# Reportes, búsqueda y escala en un CRM

**Qué hay que construir para que un dashboard no mienta y no tumbe la base de datos.**
Documento de ingeniería. CRM genérico, sin vendor lock-in. Fecha: noviembre 2026.

Cómo leerlo:

- `[verificado + URL]` = lo dice la documentación oficial o una fuente citable que revisé.
- `[medido]` = lo corrí en este servidor (4 vCPU, 7 GB RAM, SSD, SQLite 3.45.1 vía Python 3, WAL, `synchronous=NORMAL`, cache 200 MB). Los scripts están descritos en §9.4 para que se puedan repetir.
- `[inferido]` = razonamiento de ingeniería a partir de los datos anteriores, sin fuente externa. Si el número importa para una decisión, se vuelve a medir.

Advertencia sobre los benchmarks propios: la base de 10 M filas pesa 2,9 GB y el servidor tiene 7 GB de RAM, así que **la mayoría de las mediciones son con caché tibia**. En frío (primer hit tras reiniciar) los números de escaneo secuencial empeoran 3–8×. Los órdenes de magnitud y las comparaciones *entre* consultas se mantienen; los valores absolutos no son universales.

---

## 0. Veredicto en diez líneas

1. **Ningún CRM serio calcula el dashboard en vivo contra las tablas de trabajo.** Todos precalculan, y los que lo niegan lo precalculan igual con otro nombre (caché, "report engine", snapshot).
2. Los dashboards se sirven desde **tablas de agregados** (una fila por mes × pipeline × etapa × owner × región × canal). Medido: 0,0 ms frente a 7–15 s de la misma pregunta contra 10 M de oportunidades (§6.3).
3. La **consulta en vivo** queda reservada para: detalle por ID, listados filtrados por índice selectivo y ventanas de tiempo cortas. Nunca "sumame todo el pipeline histórico".
4. La **historia de campos** no es un `log` decorativo: es la única forma de responder "cuánto tiempo estuvo en cada etapa" y "cuánto cambió el monto". Cuesta caro: duplicar el número de filas de historial hizo el insert **8,2× más lento** (§2.5).
5. Los reportes históricos **no se recalculan**. El dato de ayer ya no existe si el registro es mutable y nadie lo congeló. Se congela por noche/mes en tablas inmutables de cierre.
6. Búsqueda: Postgres (`tsvector` + GIN + `unaccent`) cubre el 90 % de los casos hasta decenas de millones de documentos; SQLite FTS5 es sorprendentemente bueno hasta ~10 M filas; Elasticsearch/OpenSearch se justifica cuando necesitás facetas ricas, ranking afinado y varios idiomas a la vez.
7. A 10 M de filas el problema no es "la base es lenta": es que **un índice mal elegido hace el agregado 20× más lento que un escaneo secuencial** (§6.2). El optimizador no sabe que tu `SUM()` necesita un índice cubriente.
8. `OFFSET` profundo es una trampa: medido 36 ms vs 0,1 ms del keyset en la misma página (§7.2). En un export de 500 k filas, `OFFSET` es la diferencia entre 1 segundo y 40 minutos.
9. SQLite aguanta mucho más de lo que la fama dice — 10 M de filas, 2,9 GB, agregados en 0,5 s con índice cubriente, 15 k escrituras/s — pero tiene **un solo escritor** y no vive en red. Sirve hasta cierto punto y ese punto se puede medir (§9).
10. El día uno se resuelve con **cinco reportes** (§10). Todo lo demás es optimización prematura o vanidad de producto.

---

## 1. Cómo se calcula un dashboard: consulta viva vs tablas de snapshot/agregados

### 1.1 Lo que hace cada producto (y por qué importa)

| Producto | Qué se calcula en vivo | Qué está precalculado / congelado | Frecuencia | Fuente |
|---|---|---|---|---|
| **Salesforce** | Ejecución del reporte al abrirlo (motor de reportes sobre SOQL, con caché de resultado por Running User) | Nada de serie para tendencias; para eso existe **Reporting Snapshot**, que copia el resultado de un reporte a un objeto custom | Snapshot: diario/semanal/mensual, **máx. 2.000 registros por corrida** | `[verificado]` https://help.salesforce.com/s/articleView?id=analytics.data_setting_up_analytic_snap.htm&type=5 |
| **HubSpot** | Nada: los reportes se sirven de un almacén derivado | Todo. Reportes del builder custom se refrescan **cada 2 h**; refresco manual **1 vez cada 15 min**; dato nuevo tarda **10–15 min** en aparecer | 2 h automático | `[verificado]` https://knowledge.hubspot.com/reports/how-often-do-analytics-in-hubspot-update · https://knowledge.hubspot.com/reports/create-reports-with-the-custom-report-builder |
| **HubSpot (límite duro)** | — | Reportes que **no son tabla** están limitados a **1.000 filas únicas**; los de tabla paginan | — | `[verificado]` misma URL |
| **Pipedrive Insights** | Reporte se calcula al pedirlo | No expone snapshots; el reporte de *progress* usa **la etapa más reciente**, no el histórico completo | — | `[verificado]` https://support.pipedrive.com/en/article/insights-reports-deal-progress |
| **Metabase / Superset / Looker (BI sobre CRM)** | Consulta SQL en vivo, salvo que configures "scheduled question" / "persist" | Persistencia opcional por tabla/periodo | Configurable | `[inferido]` (comportamiento documentado de cada tool) |

Dos lecturas que casi nadie hace:

- **HubSpot, con todo el dinero del mundo, refresca cada 2 horas y limita a 1.000 filas los gráficos.** No es una limitación vergonzosa: es la única forma de que 50 personas abran el dashboard a las 9:00 sin encolar consultas. Si tu CRM tiene 200 k registros y querés dashboard instantáneo, vas a terminar haciendo lo mismo.
- **Pipedrive no guarda historia de etapas.** Su reporte de *progress* te dice en qué etapa está hoy cada negocio, y "cuánto tiempo estuvo" lo deriva de la fecha de entrada a la etapa actual. Es la decisión de producto que permite no tener una tabla de eventos — y la razón por la que no puede responder "cuánto tiempo estuvo en la etapa 2 el trimestre pasado". `[verificado]` (misma URL)

### 1.2 Por qué la consulta en vivo se vuelve impagable (con números)

Un dashboard no es una consulta. Son 6–10 widgets, y cada widget es un agregado con `GROUP BY` sobre el mismo conjunto de filas.

**Medición propia, tabla `deals` sintética, agregado "embudo por etapa" (`GROUP BY stage_id`, `COUNT(*)`, `SUM(amount)`), mediana de 3 corridas:**

| Filas | Escaneo secuencial (sin índice de etapa) | Con índice no cubriente `(stage_id, won)` | Con índice cubriente `(stage_id, amount_cents, won)` |
|---:|---:|---:|---:|
| 200.000 | 41,6 ms | 117,9 ms | — |
| 1.000.000 | 171,1 ms | 499,2 ms | **55,2 ms** |
| 5.000.000 | 1.065,5 ms | 3.677,9 ms | — |
| 10.000.000 | 3.579,1 ms | **7.260–15.436 ms** | **472,3 ms** |

`[medido]`

Tres conclusiones incómodas:

1. **El costo es lineal en filas.** 200 k → 10 M es ×50 en filas y ×60–130 en tiempo. No hay tuning que cambie la pendiente; solo cambia la constante.
2. **Poner un índice no siempre ayuda.** A 10 M, el índice `(stage_id, won)` hace el agregado **2× a 4× más lento** que escanear la tabla, porque recorre el índice y después salta al heap por cada fila para leer `amount_cents` (accesos aleatorios). Es el error de performance más común en CRM: "le puse índice a la columna del filtro" y empeoró.
3. **El índice cubriente cambia el juego** (15× a 30× mejor), pero solo sirve para *ese* `GROUP BY`. Cada widget nuevo pide otro índice cubriente. Ahí es cuando conviene la tabla de agregados.

**La aritmética del dashboard:**

```
1 dashboard = 8 widgets × agregado de 500 ms (1 M filas)  = 4 s de CPU
30 usuarios abriendo el dashboard a las 9:00              = 120 s de CPU
4 vCPU                                                     = 30 s de saturación pura
```

Con 10 M de filas, cada widget son 0,5–15 s → el dashboard directamente no se puede servir en vivo. Y mientras esos agregados corren, la base está ocupada y **la operación de venta se pone lenta**: es el mismo pool de conexiones, el mismo buffer cache.

### 1.3 La arquitectura que sí funciona: tres capas

```
┌─ Capa 1: operación (tablas normalizadas) ────────────────────────┐
│ deal, account, contact, activity, stage_transition, record_history│
│ Índices B-tree sobre (owner_id, closed_at), (account_id), (stage_id)│
│ Consultas: detalle por ID, listados filtrados, ventanas cortas.    │
└───────────────────────────────────────────────────────────────────┘
                    │  escritura (trigger / CDC / job)
                    ▼
┌─ Capa 2: agregados (tablas derivadas, reconstruibles) ───────────┐
│ agg_deal_daily   (día × owner × etapa × región × canal)           │
│ agg_deal_monthly (mes × ...)                                      │
│ Reconstruible en cualquier momento. Se puede tirar y regenerar.   │
└───────────────────────────────────────────────────────────────────┘
                    │  cierre (job nocturno / mensual, append-only)
                    ▼
┌─ Capa 3: cierre (inmutable, es la verdad histórica) ─────────────┐
│ close_run (metadata del cierre)                                   │
│ fact_deal_close (una fila por deal cerrado, congelada)            │
│ agg_close_monthly (una fila por dimensión y mes, congelada)       │
│ Nunca se hace UPDATE. Se agrega un close_run nuevo si hay ajuste. │
└───────────────────────────────────────────────────────────────────┘
```

Regla de asignación:

| Pregunta | Capa | Latencia aceptable |
|---|---|---|
| "Mostrame el deal #4821" | 1 (vivo, por PK) | < 10 ms |
| "Deals de mi equipo cerrados este mes, ordenados por monto" | 1 (índice `(owner_id, closed_at)`) | < 100 ms |
| "Embudo del trimestre por etapa" | 2 | < 50 ms |
| "Ventas por vendedor, últimos 12 meses" | 2 | < 50 ms |
| "Cómo cerró marzo 2025 (número congelado)" | 3 | < 20 ms |
| "Cuánto cambió el monto de este deal" | 1 + `record_history` | < 50 ms |

### 1.4 Tabla de agregados diaria (la pieza que falta en casi todos los CRM caseros)

```sql
CREATE TABLE agg_deal_daily (
  day           date    NOT NULL,
  pipeline_id   int     NOT NULL,
  stage_id      int     NOT NULL,
  owner_id      bigint  NOT NULL,
  region_id     int     NOT NULL,
  channel_id    int     NOT NULL,
  deals_open    int     NOT NULL DEFAULT 0,
  amount_open   bigint  NOT NULL DEFAULT 0,   -- centavos, siempre enteros
  deals_won     int     NOT NULL DEFAULT 0,
  amount_won    bigint  NOT NULL DEFAULT 0,
  deals_lost    int     NOT NULL DEFAULT 0,
  amount_lost   bigint  NOT NULL DEFAULT 0,
  refreshed_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (day, pipeline_id, stage_id, owner_id, region_id, channel_id)
);

-- Índice para "por owner en un rango de días" y "por región/canal"
CREATE INDEX ix_agg_day_owner ON agg_deal_daily (owner_id, day);
CREATE INDEX ix_agg_day_region ON agg_deal_daily (region_id, channel_id, day);
```

Cuántas filas son: `días_retrocedidos × pipelines × etapas × owners × regiones × canales`. Con 730 días (2 años) × 2 × 6 × 40 × 5 × 5 = **8,7 M de filas**. Demasiado. Por eso lo normal es:

- **Diario solo para 90 días** (~1 M filas, entra en RAM) y
- **Mensual para todo el histórico** (~12 × 2 × 6 × 40 × 5 × 5 = 144 k filas, con 5 años).

Y para dashboards de "hoy", un agregado *intradía* de una sola fila por dimensión que se recalcula cada 5–15 minutos (que es exactamente lo que hace HubSpot con sus 2 h y 15 min: `[verificado]` arriba).

Refresco incremental de la capa 2 (Postgres, sin bloquear lecturas):

```sql
-- Recalcular solo los últimos 7 días (ventana móvil, absorbe correcciones tardías)
INSERT INTO agg_deal_daily (day, pipeline_id, stage_id, owner_id, region_id, channel_id,
                            deals_open, amount_open, deals_won, amount_won, deals_lost, amount_lost, refreshed_at)
SELECT d.closed_on, d.pipeline_id, d.stage_id, d.owner_id, d.region_id, d.channel_id,
       COUNT(*) FILTER (WHERE d.is_open),
       COALESCE(SUM(d.amount_cents) FILTER (WHERE d.is_open), 0),
       COUNT(*) FILTER (WHERE d.won),
       COALESCE(SUM(d.amount_cents) FILTER (WHERE d.won), 0),
       COUNT(*) FILTER (WHERE d.lost),
       COALESCE(SUM(d.amount_cents) FILTER (WHERE d.lost), 0),
       now()
FROM deal d
WHERE d.closed_on >= current_date - 7
GROUP BY 1,2,3,4,5,6
ON CONFLICT (day, pipeline_id, stage_id, owner_id, region_id, channel_id)
DO UPDATE SET deals_open = EXCLUDED.deals_open,
              amount_open = EXCLUDED.amount_open,
              deals_won = EXCLUDED.deals_won,
              amount_won = EXCLUDED.amount_won,
              deals_lost = EXCLUDED.deals_lost,
              amount_lost = EXCLUDED.amount_lost,
              refreshed_at = EXCLUDED.refreshed_at;
```

**Costo medido de construir un agregado mensual desde 10 M de filas: 14,8 s** `[medido]`. Eso corre a las 3:00 y no le importa a nadie. Si tuvieras que pagarlo en cada carga de pantalla, sería inviable.

---

## 2. Historia de campos: "cuánto tiempo en cada etapa" y "cuánto cambió el monto"

### 2.1 Lo que hace Salesforce (y por qué el límite existe)

| Límite | Estándar | Con Field Audit Trail (Shield) |
|---|---|---|
| Campos rastreados por objeto | **20** (Tasks/Events: 6) | ~60–200 |
| Retención | **18 meses** en UI, **24 meses** vía API | Hasta 10 años |
| Backfill al activarlo | **No** (no hay historia previa) | No |
| Acceso | Lista de historial, reportes, API | Big Object `FieldHistoryArchive`, **solo API** |
| Fuentes | `[verificado]` https://www.salesforceben.com/salesforce-field-history-tracking · https://nickfrates.com/field-audit-trail-vs-field-history · https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_fieldhistoryarchive.htm | |

El límite de 20 campos no es tacañería: es el reconocimiento de que **la historia cuesta escritura, espacio e índices**. Salesforce prefiere que el cliente elija qué 20 campos le importan antes que escribir 40 filas de auditoría por cada update en una base multi-tenant.

### 2.2 El modelo mínimo (dos tablas, no una)

**Tabla A — transiciones de etapa (append-only).** Es la que responde las preguntas de tiempo.

```sql
CREATE TABLE stage_transition (
  id            bigserial PRIMARY KEY,
  deal_id       bigint      NOT NULL,
  pipeline_id   int         NOT NULL,
  from_stage_id int,                       -- NULL en la creación
  to_stage_id   int         NOT NULL,
  entered_at    timestamptz NOT NULL DEFAULT now(),
  changed_by    bigint,
  via           text        NOT NULL DEFAULT 'ui',   -- ui|api|import|automation|merge
  CONSTRAINT uq_transition UNIQUE (deal_id, entered_at, to_stage_id)  -- idempotencia de imports
);
CREATE INDEX ix_tr_deal_time ON stage_transition (deal_id, entered_at);
CREATE INDEX ix_tr_time      ON stage_transition (entered_at);  -- para reportes por período
```

Por qué **append-only y no una fila con `left_at` que se actualiza**: si actualizás, perdés el "quién cambió qué y cuándo" (el dato que necesitás el día que un vendedor jura que él no movió el negocio), y generás contención de escritura sobre la misma fila. Con eventos, el intervalo se calcula con `lead()`. Con 200 k deals × 5 transiciones = 1 M de filas; a ~90 bytes/fila son ~90 MB. Trivial.

**Tabla B — historia de campos genérica.** Para "cuánto cambió el monto", "quién cambió el owner", "cuándo pasó a perdido".

```sql
CREATE TABLE record_history (
  id          bigserial,
  entity      text        NOT NULL,      -- 'deal' | 'account' | 'contact'
  record_id   bigint      NOT NULL,
  field       text        NOT NULL,      -- 'amount_cents' | 'owner_id' | 'close_date'
  old_value   jsonb,
  new_value   jsonb,
  changed_at  timestamptz NOT NULL DEFAULT now(),
  changed_by  bigint,
  via         text,
  txid        bigint,                    -- agrupa los N campos del mismo UPDATE
  PRIMARY KEY (entity, record_id, changed_at, id)
) PARTITION BY RANGE (changed_at);

CREATE TABLE record_history_2026 PARTITION OF record_history
  FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
CREATE INDEX ix_rh_rec ON record_history (entity, record_id, changed_at DESC);
CREATE INDEX ix_rh_field_time ON record_history (field, changed_at) WHERE field = 'amount_cents';
```

Puntos de diseño que importan:

- `jsonb` y no `text`: podés guardar de todo sin migrar el esquema cuando aparece un campo nuevo, y comparar tipos.
- `txid`: sin esto, un update que cambió 4 campos produce 4 filas indistinguibles y no podés reconstruir "el estado después de ese click".
- Particionar por `changed_at`: el 95 % de las consultas son "de este deal" o "de este mes", y borrar historia vieja es `DROP TABLE` de una partición, no un `DELETE` de 40 M de filas.
- **Sin FK a `deal`**: la historia debe sobrevivir al borrado del registro (si no, perdés la auditoría justo cuando la necesitás).

### 2.3 Las dos preguntas, resueltas

**"¿Cuánto tiempo estuvo esta oportunidad en cada etapa?"**

```sql
WITH ev AS (
  SELECT deal_id,
         to_stage_id AS stage_id,
         entered_at,
         lead(entered_at) OVER (PARTITION BY deal_id ORDER BY entered_at) AS left_at
  FROM stage_transition
  WHERE pipeline_id = $1
)
SELECT s.name AS etapa,
       COUNT(*)                                                             AS visitas,
       ROUND(AVG(EXTRACT(epoch FROM (COALESCE(left_at, now()) - entered_at)) / 86400)::numeric, 1) AS dias_promedio,
       ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (
              ORDER BY EXTRACT(epoch FROM (COALESCE(left_at, now()) - entered_at)) / 86400)::numeric, 1) AS dias_mediana,
       ROUND(PERCENTILE_CONT(0.9) WITHIN GROUP (
              ORDER BY EXTRACT(epoch FROM (COALESCE(left_at, now()) - entered_at)) / 86400)::numeric, 1) AS dias_p90
FROM ev JOIN stage s ON s.id = ev.stage_id
GROUP BY s.name
ORDER BY dias_promedio DESC;
```

Usá **mediana y p90, no promedio**. Un solo negocio que estuvo 400 días en "Negociación" arruina el promedio y hace que el gerente tome una decisión sobre un número que no representa a nadie. En datos de ventas la distribución de tiempos es fuertemente asimétrica; el promedio es una mentira estadística cómoda. `[inferido]`

**"¿Cuánto cambió el monto desde que se creó?"**

Opción rápida y la que recomiendo en producción: **denormalizar el valor inicial en el deal**.

```sql
ALTER TABLE deal ADD COLUMN amount_initial_cents bigint;
-- se setea UNA vez, en el INSERT (o en el primer backfill), y no se toca nunca más

SELECT d.id, d.title,
       d.amount_initial_cents,
       d.amount_cents AS amount_actual,
       d.amount_cents - d.amount_initial_cents AS delta,
       ROUND(100.0 * (d.amount_cents - d.amount_initial_cents)
             / NULLIF(d.amount_initial_cents, 0), 1) AS delta_pct,
       (SELECT COUNT(*) FROM record_history h
         WHERE h.entity='deal' AND h.record_id=d.id AND h.field='amount_cents') AS n_cambios
FROM deal d
WHERE d.amount_cents <> d.amount_initial_cents
ORDER BY ABS(d.amount_cents - d.amount_initial_cents) DESC
LIMIT 100;
```

Opción auditable (sin denormalizar), más lenta pero correcta incluso si el backfill se olvidó:

```sql
SELECT DISTINCT ON (h.record_id)
       h.record_id AS deal_id,
       COALESCE(h.old_value, h.new_value)::text::bigint AS monto_inicial,
       h.changed_at AS primer_cambio
FROM record_history h
WHERE h.entity = 'deal' AND h.field = 'amount_cents'
ORDER BY h.record_id, h.changed_at ASC;
```

La regla: **el valor "de creación" es un dato del negocio, no un dato derivado.** Guardalo. Derivarlo de la historia funciona hasta que aparece un import masivo sin historia y el reporte de "descuento promedio desde la creación" empieza a devolver `NULL` para el 30 % de la cartera.

### 2.4 Historia por día vs historia por evento

Si solo necesitás "cuántos días estuvo en cada etapa" agregado por mes (para un dashboard), no necesitás eventos: necesitás un **snapshot diario de etapa**.

```sql
CREATE TABLE deal_stage_daily (
  day      date   NOT NULL,
  deal_id  bigint NOT NULL,
  stage_id int    NOT NULL,
  owner_id bigint NOT NULL,
  amount_cents bigint NOT NULL,
  PRIMARY KEY (day, deal_id)
) PARTITION BY RANGE (day);
```

- Tamaño: 200 k deals abiertos × 365 días = **73 M de filas/año**. A ~40 bytes comprimido, ~3 GB/año. Viables con particionado y retención de 24 meses.
- Ventaja brutal: "días en etapa" es `COUNT(*)` por etapa, sin ventanas ni `lead()`, y es **inmune a que alguien edite la historia**.
- Desventaja: no sabés *quién* movió el negocio ni *cuántas veces* volvió. Para eso están los eventos.

**Mi recomendación:** eventos (`stage_transition`) como fuente de verdad + snapshot diario solo si el volumen de reportes de tiempo-en-etapa lo justifica. No las dos cosas "por las dudas".

### 2.5 El costo real de escribir historia

Medición propia: insertar 200.000 deals en una transacción, con y sin trigger que escribe 2 filas de historia por insert. `[medido]`

| Escenario | Tiempo | Throughput | Factor |
|---|---:|---:|---:|
| 200 k inserts, sin historia | 0,71 s | ~282.000 filas/s | 1,0× |
| 200 k inserts, con 2 filas de historia por insert (400 k filas extra) | 5,81 s | ~34.000 filas/s | **8,2×** |

Es decir: **la historia multiplica por 8 el costo de escritura**, no por 2, porque cada fila de historia toca 2 índices adicionales y ensucia páginas. Ese es el número que hay que tener en la cabeza cuando alguien pide "rastrear todos los campos de todos los objetos".

Qué hacer con eso:

1. **Rastrear solo campos que alguien va a consultar.** Etapa, monto, owner, fecha de cierre, probabilidad. No "todos".
2. **Escribir la historia en la misma transacción** (trigger `AFTER UPDATE`) para no perderla nunca. El costo se paga, pero se paga una vez.
3. **Si el pico de escritura duele**: historia a una cola (outbox) y un worker que inserta en lote. Cambia durabilidad inmediata por throughput. Solo si el volumen lo exige (imports masivos, sincronización con ERP).
4. **Borrar historia con `DROP PARTITION`**, nunca con `DELETE`.

---

## 3. Los reportes clásicos y la consulta que hay detrás

Notación: `deal` tiene `stage_id`, `pipeline_id`, `owner_id`, `amount_cents` (entero, centavos), `currency`, `region_id`, `channel_id`, `created_at`, `closed_at`, `won bool`, `lost bool`, `is_open` (generada), `probability int`.

### 3.1 Embudo por etapa (pipeline snapshot)

**Pregunta:** "¿Cuánto tengo abierto ahora, por etapa?"
**Tablas:** `deal` + `stage`. **Precalculado:** nada (es snapshot del presente, se puede cachear 60 s).

```sql
SELECT s.position,
       s.name AS etapa,
       COUNT(*)                                    AS deals,
       SUM(d.amount_cents) / 100.0                 AS monto,
       ROUND(AVG(d.amount_cents) / 100.0, 0)       AS ticket_promedio,
       ROUND(AVG(EXTRACT(day FROM now() - d.created_at)), 0) AS edad_promedio_dias
FROM deal d
JOIN stage s ON s.id = d.stage_id
WHERE d.is_open
  AND d.pipeline_id = $1
GROUP BY s.position, s.name
ORDER BY s.position;
```

Nota: **`WHERE d.is_open` es obligatorio y su ausencia es el bug #1 de los embudos.** Sin él, el "pipeline" incluye ganados y perdidos históricos y el número queda inflado 10×. `is_open` debe ser una columna generada (`GENERATED ALWAYS AS (won = false AND lost = false) STORED`) e indexada, no una condición repetida a mano en 40 consultas.

### 3.2 Tasa de conversión etapa → etapa

**Pregunta:** "De los que llegaron a la etapa 3, ¿cuántos pasaron a la 4?"
**Tablas:** `stage_transition`. **Precalculado:** nada, o `agg_stage_conversion` mensual si el volumen es alto.

```sql
WITH reach AS (
  SELECT deal_id, to_stage_id AS stage_id, MIN(entered_at) AS first_at
  FROM stage_transition
  WHERE pipeline_id = $1 AND entered_at >= $2 AND entered_at < $3
  GROUP BY deal_id, to_stage_id
)
SELECT s.position,
       s.name,
       COUNT(*) AS llegaron,
       COUNT(*) FILTER (WHERE EXISTS (
         SELECT 1 FROM reach r2
         WHERE r2.deal_id = r.deal_id AND r2.stage_id > r.stage_id)) AS avanzaron,
       ROUND(100.0 * COUNT(*) FILTER (WHERE EXISTS (
         SELECT 1 FROM reach r2
         WHERE r2.deal_id = r.deal_id AND r2.stage_id > r.stage_id)) / COUNT(*), 1) AS pct_avance
FROM reach r
JOIN stage s ON s.id = r.stage_id
GROUP BY s.position, s.name
ORDER BY s.position;
```

**Trampa de definición:** "conversión etapa 3→4" puede significar (a) llegaron a 4 en algún momento, (b) están en 4 hoy, (c) pasaron directo de 3 a 4. Son tres números distintos y los tres se llaman "conversión" en las reuniones. Elegí una, documentala en el tooltip del widget.

### 3.3 Tiempo de cierre promedio (y mediana)

**Pregunta:** "¿Cuántos días tardamos en cerrar?"
**Tablas:** `deal` (o `stage_transition` si querés por etapa). **Precalculado:** `agg_close_monthly` con la mediana ya calculada.

```sql
SELECT date_trunc('month', closed_at) AS mes,
       COUNT(*) AS cerrados,
       ROUND(AVG(EXTRACT(epoch FROM closed_at - created_at)/86400)::numeric, 1) AS dias_promedio,
       ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (
             ORDER BY EXTRACT(epoch FROM closed_at - created_at)/86400)::numeric, 1) AS dias_mediana,
       ROUND(PERCENTILE_CONT(0.9) WITHIN GROUP (
             ORDER BY EXTRACT(epoch FROM closed_at - created_at)/86400)::numeric, 1) AS dias_p90
FROM deal
WHERE won AND closed_at >= $1 AND closed_at < $2
  AND deleted_at IS NULL
GROUP BY 1
ORDER BY 1;
```

**Trampa:** usar `closed_at - created_at` mide el ciclo completo, incluido el tiempo en que el negocio estuvo en un cajón. Si querés "velocidad comercial", medí desde el primer contacto real o desde que entró al pipeline calificado. Y `created_at` de un import masivo miente: 3.000 deals "creados" el mismo día.

### 3.4 Ventas por vendedor / región / canal

**Pregunta:** "¿Quién vendió qué, dónde, por qué canal?"
**Tablas:** con 200 k filas, `deal` + índices; con millones, `agg_deal_monthly`. **Precalculado:** el agregado mensual.

```sql
SELECT u.name AS vendedor,
       r.name AS region,
       c.name AS canal,
       COUNT(*)                             AS deals_ganados,
       SUM(d.amount_cents)/100.0            AS monto,
       ROUND(AVG(d.amount_cents)/100.0, 0)  AS ticket_promedio,
       ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY u.id), 1) AS pct_del_vendedor
FROM deal d
JOIN "user" u   ON u.id = d.owner_id
JOIN region r   ON r.id = d.region_id
JOIN channel c  ON c.id = d.channel_id
WHERE d.won
  AND d.closed_at >= $1 AND d.closed_at < $2
  AND d.deleted_at IS NULL
GROUP BY u.id, u.name, r.name, c.name
ORDER BY monto DESC;
```

**Trampa de atribución (la más grave de todas):** `deal.owner_id` es el owner **de hoy**. Si María cerró el negocio y después se fue de la empresa y el registro se reasignó a Pedro, el reporte del año pasado le va a dar el crédito a Pedro. La venta histórica de María desaparece. Solución: guardar `closed_by_id` en el momento del cierre (inmutable) y usarlo para reportes históricos. Esto es un campo, no un sistema. `[inferido]`

### 3.5 Cohortes de clientes

**Pregunta:** "De los clientes que entraron en marzo, ¿cuántos siguen comprando en el mes 6?"
**Tablas:** `account` (fecha de alta), `deal` (compras), o mejor `fact_revenue` si hay facturación.
**Precalculado:** matriz de cohortes (una fila por cohorte × mes de vida).

```sql
WITH cohort AS (
  SELECT id AS account_id,
         date_trunc('month', first_won_at) AS cohort_month
  FROM account
  WHERE first_won_at >= $1 AND first_won_at < $2
),
actividad AS (
  SELECT c.cohort_month,
         c.account_id,
         (EXTRACT(year FROM age(date_trunc('month', d.closed_at), c.cohort_month)) * 12
          + EXTRACT(month FROM age(date_trunc('month', d.closed_at), c.cohort_month)))::int AS mes_vida,
         SUM(d.amount_cents)/100.0 AS monto
  FROM cohort c
  JOIN deal d ON d.account_id = c.account_id AND d.won
  GROUP BY 1,2,3
)
SELECT cohort_month,
       mes_vida,
       COUNT(DISTINCT account_id) AS clientes_activos,
       SUM(monto)                 AS monto
FROM actividad
WHERE mes_vida BETWEEN 0 AND 12
GROUP BY 1,2
ORDER BY 1,2;
```

**Trampa:** una cohorte mal definida (por fecha de creación del registro en el CRM, no por primera compra) produce una "retención" que en realidad mide cuándo se digitalizó la cartera. En un CRM migrado, la cohorte de marzo puede ser "el mes en que el becario cargó 4.000 clientes viejos".

### 3.6 Forecast / pipeline ponderado

**Pregunta:** "¿Cuánto voy a cerrar este trimestre?"
**Tablas:** `deal` (abiertos con `close_date` y `probability`), `quota`. **Precalculado:** `agg_forecast_weekly` por owner y por mes de cierre esperado.

```sql
SELECT u.name AS vendedor,
       SUM(d.amount_cents)/100.0 AS pipeline_bruto,
       SUM(d.amount_cents * COALESCE(d.probability, s.default_probability) / 100.0)/100.0 AS forecast_ponderado,
       SUM(d.amount_cents) FILTER (WHERE d.close_date < current_date)/100.0 AS vencido_sin_cerrar,
       q.amount_cents/100.0 AS cuota,
       ROUND(100.0 * SUM(d.amount_cents * COALESCE(d.probability, s.default_probability)/100.0)
             / NULLIF(q.amount_cents, 0), 1) AS pct_cuota
FROM deal d
JOIN "user" u ON u.id = d.owner_id
JOIN stage s  ON s.id = d.stage_id
LEFT JOIN quota q ON q.owner_id = u.id
                 AND q.period_start = date_trunc('quarter', current_date)::date
WHERE d.is_open
  AND d.close_date >= date_trunc('quarter', current_date)
  AND d.close_date <  date_trunc('quarter', current_date) + interval '3 months'
GROUP BY u.id, u.name, q.amount_cents
ORDER BY forecast_ponderado DESC;
```

**Trampa:** `probability` editada a mano por el vendedor el último día del trimestre convierte el forecast en una declaración de deseos. Los CRM serios guardan **historia de `probability`** (sí, otra vez `record_history`) para poder medir el sesgo: "este equipo pondera 90 % los negocios que cierran al 40 %". Ese reporte —*forecast accuracy*— vale más que el forecast.

### 3.7 Qué se precalcula, resumen

| Reporte | Toca | En vivo hasta | Precalcular cuando |
|---|---|---|---|
| Embudo actual | `deal`, `stage` | 200 k–1 M | > 1 M filas abiertas → `agg_pipeline_now` cada 5 min |
| Conversión por etapa | `stage_transition` | 5 M eventos | > 5 M → `agg_stage_conversion` mensual |
| Tiempo de cierre | `deal` | 1 M | siempre (es caro por los percentiles) |
| Ventas por dimensión | `deal` + dims | 500 k | > 500 k → `agg_deal_monthly` |
| Cohortes | `deal`, `account` | 200 k | siempre (matriz precalculada) |
| Forecast | `deal`, `quota` | 100 k abiertos | `agg_forecast_weekly` |

---

## 4. Snapshots y cierres: por qué el pasado no se puede recalcular

### 4.1 El problema, en una frase

**Un reporte histórico calculado sobre tablas mutables no es historia: es el presente disfrazado.**

Ejemplo concreto:

- El 31 de marzo, el pipeline abierto era **$420 M** y el dashboard se lo mostró al directorio.
- El 15 de abril, un vendedor movió 30 negocios a "Perdido" y corrigió montos.
- Hoy, alguien pide "el pipeline al 31 de marzo" y la consulta devuelve **$180 M**.

Nadie mintió y el número cambió. Eso es lo que destruye la confianza en el CRM: no el error, sino la inconsistencia. Y no se arregla con mejor SQL: se arregla **congelando**.

Salesforce lo resolvió con Reporting Snapshots (copiar el resultado de un reporte a un objeto custom cada día/semana/mes) y con un límite duro de 2.000 filas por corrida `[verificado]` (URL en §1.1). Es decir: **el vendor más grande del mercado te está diciendo que el snapshot es la respuesta, y que su implementación es de juguete para volúmenes reales.** Para 200 k registros hay que hacerlo uno mismo.

### 4.2 Qué se congela y cuándo

| Frecuencia | Qué | Para qué | Costo |
|---|---|---|---|
| Cada noche | `fact_deal_snapshot` (estado de cada deal abierto) | "cómo evolucionó el pipeline", tiempo en etapa | alto en filas (200 k/día → 73 M/año) |
| Cada noche | `agg_deal_daily` (agregados, no detalle) | dashboards de tendencia | bajo (~1 M filas) |
| **Cierre mensual** | `fact_deal_close` (deals cerrados en el mes, congelados) | comisiones, revenue, comparación año contra año | 1 fila por deal cerrado |
| **Cierre mensual** | `agg_close_monthly` (agregados congelados) | reportes oficiales del mes | decenas de miles de filas |
| Al cambiar un deal | `record_history` | auditoría, "cuánto cambió" | alto (ver §2.5) |

### 4.3 Tabla de cierre mensual: diseño

```sql
-- 1) Metadata de cada corrida de cierre: el "libro" de contabilidad del CRM
CREATE TABLE close_run (
  id            bigserial PRIMARY KEY,
  period        date NOT NULL,                  -- '2026-03-01'
  kind          text NOT NULL,                  -- 'monthly' | 'quarterly' | 'yearly'
  status        text NOT NULL DEFAULT 'running',-- running|done|failed|restated
  started_at    timestamptz NOT NULL DEFAULT now(),
  finished_at   timestamptz,
  rows_written  bigint,
  checksum      text,                           -- hash del resultado, para detectar cambios
  restates_run  bigint REFERENCES close_run(id),-- si este cierre reemplaza a otro
  notes         text
);

-- 2) Hecho congelado: una fila por deal cerrado. NUNCA se actualiza.
CREATE TABLE fact_deal_close (
  close_run_id     bigint NOT NULL REFERENCES close_run(id),
  deal_id          bigint NOT NULL,
  period           date   NOT NULL,             -- mes de cierre (closed_at)
  closed_at        timestamptz NOT NULL,
  created_at       timestamptz NOT NULL,
  cycle_days       int NOT NULL,                -- cerrado - creado, congelado
  pipeline_id      int NOT NULL,
  stage_id         int NOT NULL,
  owner_id         bigint NOT NULL,             -- owner AL MOMENTO DEL CIERRE
  closed_by_id     bigint,                      -- quién lo cerró (atribución histórica)
  account_id       bigint,
  region_id        int,
  channel_id       int,
  amount_cents     bigint NOT NULL,             -- monto congelado
  currency         char(3) NOT NULL,
  amount_clp_cents bigint NOT NULL,             -- convertido con la tasa del día de cierre
  fx_rate          numeric(18,8) NOT NULL,      -- tasa usada, guardada
  is_won           boolean NOT NULL,
  is_lost          boolean NOT NULL,
  source           text,                        -- 'crm' | 'erp' | 'import'
  PRIMARY KEY (close_run_id, deal_id)
) PARTITION BY RANGE (period);

CREATE INDEX ix_fdc_period_owner ON fact_deal_close (period, owner_id);
CREATE INDEX ix_fdc_period_dim   ON fact_deal_close (period, region_id, channel_id);

-- 3) Agregado congelado del mes (lo que lee el dashboard oficial)
CREATE TABLE agg_close_monthly (
  close_run_id bigint NOT NULL REFERENCES close_run(id),
  period       date   NOT NULL,
  pipeline_id  int    NOT NULL,
  owner_id     bigint NOT NULL,
  region_id    int    NOT NULL,
  channel_id   int    NOT NULL,
  deals_won    int    NOT NULL,
  amount_won_clp_cents bigint NOT NULL,
  deals_lost   int    NOT NULL,
  amount_lost_clp_cents bigint NOT NULL,
  avg_cycle_days numeric(8,2),
  median_cycle_days numeric(8,2),
  PRIMARY KEY (close_run_id, period, pipeline_id, owner_id, region_id, channel_id)
);
```

Decisiones que no son obvias:

1. **`close_run_id` en la clave primaria.** Permite tener dos versiones del mismo mes (el original y el re-expresado) sin pisar nada. El reporte "oficial" filtra por el `close_run` marcado como vigente. Sin esto, el primer ajuste contable te obliga a hacer `UPDATE` sobre historia, que es exactamente lo que querías evitar.
2. **`fx_rate` guardada.** Si convertís a moneda local con la tasa de hoy, el revenue de marzo de 2024 cambia cada vez que se mueve el dólar. Congelá la tasa y el monto convertido. Es la diferencia entre un reporte que se puede auditar y uno que no.
3. **`closed_by_id` separado de `owner_id`.** Ver §3.4.
4. **`checksum`.** Al re-ejecutar el cierre del mismo mes, comparás el hash. Si cambió, es un *restatement* y hay que decidir explícitamente si se publica. Sin checksum, los cambios entran silenciosos.
5. **Sin `UPDATE` ni `DELETE` en `fact_deal_close`.** Se puede hacer cumplir con permisos: `REVOKE UPDATE, DELETE ON fact_deal_close FROM app_user;`

### 4.4 El job de cierre (idempotente)

```sql
BEGIN;

INSERT INTO close_run (period, kind, status)
VALUES ('2026-03-01', 'monthly', 'running')
RETURNING id;  -- :run_id

INSERT INTO fact_deal_close (
  close_run_id, deal_id, period, closed_at, created_at, cycle_days,
  pipeline_id, stage_id, owner_id, closed_by_id, account_id,
  region_id, channel_id, amount_cents, currency, amount_clp_cents, fx_rate,
  is_won, is_lost, source)
SELECT :run_id, d.id, date_trunc('month', d.closed_at)::date, d.closed_at, d.created_at,
       EXTRACT(day FROM d.closed_at - d.created_at)::int,
       d.pipeline_id, d.stage_id, d.owner_id, d.closed_by_id, d.account_id,
       d.region_id, d.channel_id, d.amount_cents, d.currency,
       ROUND(d.amount_cents * fx.rate), fx.rate,
       d.won, d.lost, d.source
FROM deal d
JOIN fx_rate fx ON fx.currency = d.currency
               AND fx.day = date_trunc('month', d.closed_at)::date
WHERE d.closed_at >= '2026-03-01' AND d.closed_at < '2026-04-01'
  AND (d.won OR d.lost)
  AND d.deleted_at IS NULL
ON CONFLICT (close_run_id, deal_id) DO NOTHING;   -- idempotencia

INSERT INTO agg_close_monthly
SELECT :run_id, period, pipeline_id, owner_id, region_id, channel_id,
       COUNT(*) FILTER (WHERE is_won),
       COALESCE(SUM(amount_clp_cents) FILTER (WHERE is_won), 0),
       COUNT(*) FILTER (WHERE is_lost),
       COALESCE(SUM(amount_clp_cents) FILTER (WHERE is_lost), 0),
       AVG(cycle_days) FILTER (WHERE is_won),
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY cycle_days) FILTER (WHERE is_won)
FROM fact_deal_close
WHERE close_run_id = :run_id
GROUP BY 2,3,4,5,6;

UPDATE close_run SET status='done', finished_at=now() WHERE id = :run_id;
COMMIT;
```

Consulta del reporte oficial — nótese que **no toca `deal`**:

```sql
SELECT period, SUM(deals_won) AS won, SUM(amount_won_clp_cents)/100.0 AS monto
FROM agg_close_monthly
WHERE close_run_id = (SELECT id FROM close_run
                      WHERE period = $1 AND kind='monthly' AND status='done'
                      ORDER BY id DESC LIMIT 1)
GROUP BY period;
```

### 4.5 Cómo consultar "el estado en una fecha" sin snapshot

Si no tenés snapshot de un día pasado, todavía podés reconstruirlo **si tenés los eventos** (y solo si los tenés):

```sql
-- Pipeline abierto al cierre del 31/03/2025, reconstruido desde eventos
WITH ultimo_estado AS (
  SELECT DISTINCT ON (t.deal_id)
         t.deal_id, t.to_stage_id, t.entered_at
  FROM stage_transition t
  WHERE t.entered_at < '2025-04-01'
  ORDER BY t.deal_id, t.entered_at DESC
),
cierres AS (
  SELECT deal_id FROM stage_transition
  WHERE entered_at < '2025-04-01'
    AND to_stage_id IN (SELECT id FROM stage WHERE is_terminal)
)
SELECT s.name, COUNT(*), SUM(d.amount_cents)/100.0
FROM ultimo_estado u
JOIN deal d ON d.id = u.deal_id
JOIN stage s ON s.id = u.to_stage_id
WHERE u.deal_id NOT IN (SELECT deal_id FROM cierres)
  AND d.created_at < '2025-04-01'
GROUP BY s.name;
```

Esto funciona, es lento (escanea toda la historia), y **miente si el monto cambió** (el monto no está en `stage_transition`). Es un plan B para el primer mes, no una arquitectura.

### 4.6 Las siete formas en que un reporte miente (checklist de QA)

| # | Mentira | Cómo se detecta | Cómo se evita |
|---|---|---|---|
| 1 | Incluye ganados/perdidos en el "pipeline" | suma ≠ filtro de abiertos | `is_open` en columna generada, usada siempre |
| 2 | Atribuye a owner actual una venta vieja | el total del año pasado cambió | `closed_by_id` inmutable |
| 3 | Convierte monedas con la tasa de hoy | revenue histórico se mueve solo | `fx_rate` + monto convertido congelados |
| 4 | Cuenta deals borrados (soft delete) | contactos - deals ≠ reporte | `deleted_at IS NULL` en toda consulta o vista que lo filtre |
| 5 | Zona horaria: cierre del 31 a las 23:00 cae en el mes siguiente | el último día del mes tiene menos ventas | `timestamptz` + `closed_on` (date) calculado en la TZ del negocio |
| 6 | Duplica por JOIN con actividades o productos | el monto se multiplica por N líneas | `SUM(DISTINCT)` está mal; agregar en subquery |
| 7 | "Promedio" sobre distribución asimétrica | el promedio no lo alcanza nadie | mediana + p90, y mostrar n |

El #6 merece un ejemplo porque es el más frecuente:

```sql
-- MAL: si un deal tiene 3 productos, cuenta 3 veces el monto
SELECT SUM(d.amount_cents) FROM deal d JOIN deal_product p ON p.deal_id = d.id;

-- BIEN
SELECT SUM(x.amount_cents) FROM (
  SELECT d.id, d.amount_cents FROM deal d WHERE d.won
) x;
```

---

## 5. Búsqueda

### 5.1 Las tres opciones, honestamente

| Criterio | Postgres FTS (`tsvector` + GIN) | SQLite FTS5 | Elasticsearch / OpenSearch |
|---|---|---|---|
| Dónde vive | en la misma base, transaccional | en el mismo archivo | proceso/JVM aparte |
| Consistencia con la escritura | inmediata (misma transacción) | inmediata | eventual (refresh ~1 s `[verificado]` https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html) |
| Complejidad operativa | baja | **nula** | alta (cluster, heap, shards, snapshots) |
| Ranking | `ts_rank`, `ts_rank_cd` (TF-IDF-ish, sin BM25 real) | `bm25()` real | BM25 + `function_score`, boosting |
| Stemming español | sí (`spanish` config) | **no** (Porter es solo inglés) | sí (analyzer `spanish`) |
| Tildes | `unaccent` (diccionario) | `remove_diacritics 2` (tokenizer) | `asciifolding` |
| Substring / typo | `pg_trgm` (GIN/GiST) | tokenizer `trigram` | `fuzziness`, `ngram` |
| Facetas | `GROUP BY` sobre el resultado (2 queries) | `GROUP BY` sobre FTS (idem) | **aggregations** nativas |
| Búsqueda global multi-entidad | `UNION ALL` manual | `UNION ALL` manual | multi-index + `collapse` |
| Escala cómoda | decenas de millones de docs | ~10 M (medido, §9) | miles de millones |
| Cuándo NO usarlo | ranking muy afinado, multi-idioma, facetas ricas | concurrencia de escritura, > 20 M filas | equipo sin nadie que opere un cluster |

Fuentes: `[verificado]` https://www.postgresql.org/docs/current/textsearch.html · https://www.postgresql.org/docs/current/pgtrgm.html · https://sqlite.org/fts5.html · https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html

### 5.2 Postgres: la configuración que realmente funciona en español

El error clásico: `to_tsvector('spanish', ...)` en la consulta, sin columna persistida ni índice → escaneo secuencial + parseo de todos los textos en cada búsqueda. Medido por otros: de 3.042 ms a < 1 ms solo agregando la columna persistida + GIN. `[verificado]` https://danielabaron.me/blog/speed-up-pg-fts-with-persistent-ts-vectors

```sql
CREATE EXTENSION IF NOT EXISTS unaccent;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 1) Configuración de español que ADEMÁS ignora tildes
CREATE TEXT SEARCH CONFIGURATION es_unaccent (COPY = spanish);
ALTER TEXT SEARCH CONFIGURATION es_unaccent
  ALTER MAPPING FOR hword, hword_part, word
  WITH unaccent, spanish_stem;

-- 2) Columna generada (Postgres 12+). Se recalcula sola en cada INSERT/UPDATE.
ALTER TABLE deal
  ADD COLUMN search_es tsvector
  GENERATED ALWAYS AS (
    to_tsvector('es_unaccent',
      coalesce(title,'') || ' ' ||
      coalesce(notes,'') || ' ' ||
      coalesce(account_name,'') || ' ' ||
      coalesce(contact_name,''))
  ) STORED;

CREATE INDEX ix_deal_search ON deal USING GIN (search_es);

-- 3) Búsqueda con ranking
SELECT id, title,
       ts_rank_cd(search_es, q) AS rank,
       ts_headline('es_unaccent', notes, q,
                   'StartSel=<mark>, StopSel=</mark>, MaxWords=30') AS snippet
FROM deal, websearch_to_tsquery('es_unaccent', $1) q
WHERE search_es @@ q
  AND deleted_at IS NULL
ORDER BY rank DESC, updated_at DESC
LIMIT 20;
```

`websearch_to_tsquery` y no `to_tsquery`: acepta entrada humana (`"cierre anual" -borrador`) sin explotar con sintaxis inválida. Es la diferencia entre un buscador y un 500.

**Tildes, plurales y apodos — el caso español, resuelto pieza por pieza:**

| Problema | Solución | Ejemplo |
|---|---|---|
| "Muñoz" vs "Munoz" | `unaccent` en el diccionario | ambas indexan como `munoz` |
| "Ñ" | `unaccent` la mapea a `n`; si querés preservarla, usá un diccionario propio | `[inferido]` |
| Plurales ("clientes" vs "cliente") | `spanish_stem` ya hace stemming | `clientes` → `client` |
| Apodos ("Pepe" ↔ "José") | **no lo resuelve ningún stemmer**: tabla de sinónimos + `tsquery` expandida | ver abajo |
| Typos ("Gonzalez" vs "Gonsalez") | `pg_trgm` con `%` / `similarity()` | ver abajo |

```sql
-- Sinónimos / apodos: tabla propia, expandida en la aplicación
CREATE TABLE search_synonym (
  term  text NOT NULL,
  alias text NOT NULL,
  PRIMARY KEY (term, alias)
);
INSERT INTO search_synonym VALUES
  ('jose','pepe'), ('jose','chepe'), ('francisco','paco'), ('francisco','pancho'),
  ('maria','mari'), ('conversion','cierre'), ('cotizacion','presupuesto');

-- Expansión de la query (en la app o en SQL)
WITH base AS (SELECT websearch_to_tsquery('es_unaccent', $1) AS q),
     alts AS (
       SELECT string_agg(alias, ' | ') AS or_terms
       FROM search_synonym, base
       WHERE term = ANY (string_to_array($1, ' '))
     )
SELECT id, title
FROM deal, base
WHERE search_es @@ (base.q || to_tsquery('es_unaccent',
        COALESCE((SELECT or_terms FROM alts), 'zzzz')))
LIMIT 20;
```

```sql
-- Tolerancia a typos sobre nombres propios (índice trigram)
CREATE INDEX ix_account_name_trgm ON account USING GIN (lower(name) gin_trgm_ops);

SELECT id, name, similarity(lower(name), lower($1)) AS sim
FROM account
WHERE lower(name) % lower($1)          -- usa el índice
ORDER BY sim DESC
LIMIT 10;
```

Costo real de `pg_trgm`: índice GIN de **1,5× a 4× el tamaño de la columna** y penalización de escritura de ~2× en inserts por lote. `[verificado]` https://aws.amazon.com/blogs/database/index-types-supported-in-amazon-aurora-postgresql-and-amazon-rds-for-postgresql-using-extensions-bloom-pg_trgm-and-pg_bigm y https://www.runxbuild.com/blog/postgresql-pg-trgm-extension

### 5.3 Búsqueda en campos personalizados (el problema real de un CRM)

Un CRM tiene campos que el cliente inventa. Hay tres modelos y cada uno tiene un costo distinto:

**(a) Columnas reales (`ALTER TABLE ADD COLUMN`)** — rápido de consultar, índice normal, pero un `ALTER TABLE` por cada campo que pide un cliente y una tabla con 300 columnas. Es lo que hace EspoCRM: los campos custom se materializan como columnas reales, con su definición e índices en metadata JSON (ver `entityDefs` e `indexes` en `Lead.json`). `[verificado]` https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Resources/metadata/entityDefs/Lead.json

**(b) `jsonb`** — un `custom jsonb` en cada entidad, con GIN:

```sql
CREATE INDEX ix_deal_custom ON deal USING GIN (custom jsonb_path_ops);
SELECT id FROM deal WHERE custom @> '{"industria":"mineria"}';
```

Funciona bien para *filtros de igualdad*, mal para full-text: `to_tsvector` de un `jsonb` no se puede indexar por campo arbitrario. Para buscar texto dentro de campos custom hay que mantener un **documento de búsqueda**:

```sql
-- Columna de texto "plana" con todos los campos custom relevantes, mantenida por trigger
ALTER TABLE deal ADD COLUMN search_doc text;

CREATE OR REPLACE FUNCTION deal_search_doc() RETURNS trigger AS $$
BEGIN
  NEW.search_doc := coalesce(NEW.title,'') || ' ' ||
                    coalesce(NEW.custom->>'industria','') || ' ' ||
                    coalesce(NEW.custom->>'contacto_tecnico','') || ' ' ||
                    coalesce(NEW.custom->>'notas','');
  NEW.search_es := to_tsvector('es_unaccent', NEW.search_doc);
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_deal_search BEFORE INSERT OR UPDATE ON deal
  FOR EACH ROW EXECUTE FUNCTION deal_search_doc();
```

Costo: el trigger en cada escritura (ver §2.5, el mismo orden de magnitud). Beneficio: un solo GIN, búsqueda uniforme sobre campos estándar y custom.

**(c) Tabla EAV indexada** — `custom_value(record_id, field_id, value_text)` con GIN sobre `value_text`. Flexible total, join caro, y ordenar por un campo custom requiere un índice por campo igual. Solo si los campos son realmente dinámicos (un CRM de plataforma, no un CRM de producto).

**Recomendación:** (a) para los 20–30 campos que usan todos, (b) para los custom de largo tail. La (c) casi nunca.

### 5.4 Búsqueda global con resultados agrupados

El patrón: una consulta por entidad, cada una con su `LIMIT`, unidas. Nunca un `UNION` sin límites pensando en paginar después.

```sql
WITH q AS (SELECT websearch_to_tsquery('es_unaccent', $1) AS tsq)
SELECT * FROM (
  (SELECT 'deal' AS tipo, id, title AS label,
          ts_rank_cd(search_es, q.tsq) AS r, updated_at
   FROM deal, q WHERE search_es @@ q.tsq AND deleted_at IS NULL
   ORDER BY r DESC LIMIT 5)
  UNION ALL
  (SELECT 'account', id, name, ts_rank_cd(search_es, q.tsq), updated_at
   FROM account, q WHERE search_es @@ q.tsq AND deleted_at IS NULL
   ORDER BY 4 DESC LIMIT 5)
  UNION ALL
  (SELECT 'contact', id, full_name, ts_rank_cd(search_es, q.tsq), updated_at
   FROM contact, q WHERE search_es @@ q.tsq AND deleted_at IS NULL
   ORDER BY 4 DESC LIMIT 5)
) x
ORDER BY tipo, r DESC;
```

Detalles que hacen la diferencia:

- **Normalizá el score entre entidades.** `ts_rank_cd` de un `contact` con nombre corto y de un `deal` con notas largas no son comparables. Si querés un ranking global, normalizá por `max(r)` de cada grupo o aplicá un peso por entidad (`deal` 1,0, `account` 0,8...).
- **Cada grupo tiene su propio índice.** Un solo `search_es` gigante multi-entidad no existe en Postgres; en Elastic sí, y es la ventaja real de Elastic para búsqueda global.
- **Facetas sin N+1:** una query para los resultados y una para los conteos, con los mismos filtros:

```sql
SELECT count(*) FILTER (WHERE stage_id = 3) AS en_negociacion,
       count(*) FILTER (WHERE region_id = 1) AS metropolitana,
       count(*) FILTER (WHERE amount_cents > 1000000) AS sobre_10k
FROM deal, websearch_to_tsquery('es_unaccent', $1) q
WHERE search_es @@ q;
```

### 5.5 SQLite FTS5

```sql
-- Tokenizer con tildes resueltas y prefijos indexados (para plurales)
CREATE VIRTUAL TABLE deal_fts USING fts5(
  title, account_name, notes,
  content='deal', content_rowid='id',
  tokenize="unicode61 remove_diacritics 2",
  prefix='2 3 4'
);

-- Triggers de sincronización (obligatorios: FTS5 no se sincroniza solo)
CREATE TRIGGER deal_ai AFTER INSERT ON deal BEGIN
  INSERT INTO deal_fts(rowid, title, account_name, notes)
  VALUES (new.id, new.title, new.account_name, new.notes);
END;
CREATE TRIGGER deal_ad AFTER DELETE ON deal BEGIN
  INSERT INTO deal_fts(deal_fts, rowid, title, account_name, notes)
  VALUES ('delete', old.id, old.title, old.account_name, old.notes);
END;
CREATE TRIGGER deal_au AFTER UPDATE ON deal BEGIN
  INSERT INTO deal_fts(deal_fts, rowid, title, account_name, notes)
  VALUES ('delete', old.id, old.title, old.account_name, old.notes);
  INSERT INTO deal_fts(rowid, title, account_name, notes)
  VALUES (new.id, new.title, new.account_name, new.notes);
END;

-- Búsqueda con prefijo (cubre plurales: "client*" → cliente, clientes)
SELECT d.id, d.title, bm25(deal_fts, 5.0, 2.0, 1.0) AS score
FROM deal_fts f JOIN deal d ON d.id = f.rowid
WHERE deal_fts MATCH 'constructora* AND (ñuñoa OR nunoa)'
ORDER BY score
LIMIT 20;
```

Lo que hay que saber de FTS5:

- `remove_diacritics 2` es **obligatorio** en español: convierte `á é í ó ú ñ` en `a e i o u n` correctamente (el valor `1` tiene un bug conocido con caracteres de doble diacrítico; el `2` lo corrige — está documentado en la propia página de FTS5). `[verificado]` https://sqlite.org/fts5.html
- **No hay stemming en español.** `porter` es inglés. El reemplazo práctico es `prefix='2 3 4'` + buscar con `palabra*`, que cubre plurales y género pero no conjugaciones.
- `trigram` tokenizer = substring real (`LIKE '%algo%'` indexado), pero infla el índice y los términos de < 3 caracteres no matchean. `[verificado]` (misma URL)
- `bm25(tabla, w1, w2, w3)` con pesos por columna es la forma de decir "el título importa 5×, las notas 1×".
- `detail=none` reduce el índice ~5× (743 MiB → 134 MiB en el test de la doc) pero pierde frases y `NEAR`. `[verificado]` (misma URL)

### 5.6 Elasticsearch / OpenSearch: cuándo y cómo

Se justifica cuando se cumplen **dos o más** de estas:

- > 20 M documentos y varias entidades buscables a la vez.
- Facetas con conteos en tiempo real sobre muchos campos (aggregations).
- Ranking que necesita afinarse (boosting por campo, `function_score` por recencia/popularidad).
- Varios idiomas o sinónimos administrados por el negocio sin deploy.
- Búsqueda "typo-tolerant" seria (`fuzziness: AUTO`).

Esbozo mínimo (un índice por entidad, o uno solo con `type`):

```json
PUT /crm_v1
{ "settings": { "analysis": { "analyzer": {
    "es_fold": { "type":"custom", "tokenizer":"standard",
      "filter":["lowercase","asciifolding","spanish_stop","spanish_stemmer"] } } } },

  "mappings": { "properties": {
    "type":        { "type":"keyword" },
    "title":       { "type":"text", "analyzer":"es_fold", "boost":3 },
    "notes":       { "type":"text", "analyzer":"es_fold" },
    "amount":      { "type":"long" },
    "stage":       { "type":"keyword" },
    "region":      { "type":"keyword" },
    "owner_id":    { "type":"keyword" },
    "updated_at":  { "type":"date" } } } }

GET /crm_v1/_search
{ "size": 5,
  "query": { "bool": {
    "must":   [ { "multi_match": { "query":"constructora ñuñoa",
                                   "fields":["title^3","notes","account_name^2"],
                                   "fuzziness":"AUTO" } } ],
    "filter": [ { "term": { "type":"deal" } },
                { "term": { "region":"metropolitana" } } ] } },
  "aggs": { "por_etapa": { "terms": { "field":"stage" } } },
  "collapse": { "field": "account_id" } }
```

Tres cosas que se pagan caro si se ignoran: **`refresh_interval`** (indexás 50 k registros y esperás — o lo subís a `-1` durante el bulk y lo restaurás), **el `heap` de la JVM** (≤ 50 % de la RAM del nodo, nunca swap), y **la reindexación** (necesitás un alias `crm_actual` apuntando a `crm_v1`, para poder construir `crm_v2` y cambiar el alias sin downtime).

---

## 6. Volumen: qué pasa a los 100 k, 1 M y 10 M

### 6.1 La tabla de referencia (medición propia, SQLite)

Dataset: `deal` con 15 columnas, 40 owners, 6 etapas, 5 regiones, 5 canales, 60 % cerrados. Más `activity` (1 fila por deal), 4 índices B-tree y un FTS5. `[medido]`

| Métrica | 200 k | 1 M | 5 M | 10 M |
|---|---:|---:|---:|---:|
| Tamaño del archivo | 55 MB | 277 MB | 1,40 GB | **2,82 GB** |
| Insert masivo (1 txn) | 1,5 s | 9,6 s | 26,6 s | 59,4 s |
| Crear 4 índices | 0,4 s | 1,8 s | 10,0 s | 24,7 s |
| Construir FTS5 | 0,4 s | 1,8 s | 10,6 s | 24,7 s |
| Embudo (seq scan) | 42 ms | 171 ms | 1.066 ms | 3.579 ms |
| Embudo (índice no cubriente) | 118 ms | 499 ms | 3.678 ms | 7.260–15.436 ms |
| Embudo (índice cubriente) | — | 55 ms | — | **472 ms** |
| Ganado por owner, 90 días | 17 ms | 66 ms | 519 ms | 9.992 ms |
| Ciclo de cierre promedio | 23 ms | 89 ms | 533 ms | 1.353 ms |
| `LIKE '%Maria%'` | 15 ms | 85 ms | 534 ms | 1.220 ms |
| Filtro por campo JSON | 56 ms | 336 ms | 1.699 ms | 3.325 ms |
| FTS5 match exacto | ~0 ms | ~0 ms | ~0 ms | **0,1 ms** |
| FTS5 prefijo | 0,4 ms | 1,6 ms | 9,3 ms | 20,3 ms |
| FTS5 + `bm25` (ordenar) | 15 ms | 75 ms | 442 ms | **890 ms** |
| Página 4.000 (`OFFSET 199950`) | 5,4 ms | 4,5 ms | 4,5 ms | **36,3 ms** |
| Keyset (misma página) | ~0 ms | ~0 ms | ~0 ms | **0,1 ms** |
| Construir agregado mensual | — | — | — | 14,8 s |
| Consulta sobre el agregado | — | — | — | **~0 ms** |

Lectura de la tabla:

- **100 k–200 k:** todo funciona en vivo. Un CRM en SQLite sin agregados puede sobrevivir acá. No optimices.
- **1 M:** el agregado sin índice cubriente ya está en 0,5 s; 8 widgets = 4 s. **Necesitás capa de agregados.**
- **5 M:** el escaneo secuencial cruza el segundo. La historia y los snapshots dominan el espacio.
- **10 M:** cualquier agregado en vivo es inviable (7–15 s); el agregado mensual responde en 0 ms. La diferencia entre arquitectura correcta e incorrecta es de **cuatro órdenes de magnitud**.

### 6.2 La lección del índice cubriente (y su límite)

A 10 M de filas, `GROUP BY stage_id` con `COUNT(*)` y `SUM(amount_cents)`:

- índice `(stage_id, won)` → **7,3–15,4 s** (recorre índice + salto al heap por fila)
- escaneo secuencial → **3,6 s**
- índice `(stage_id, amount_cents, won)` → **0,47 s** (15–30× mejor)

Esto se generaliza mal: **necesitarías un índice cubriente por cada combinación de `GROUP BY` + `SUM` que pida el producto.** Con 6 dimensiones, eso son decenas de índices, cada uno encareciendo cada escritura. Ahí es donde la tabla de agregados deja de ser una optimización y pasa a ser la única opción racional.

Regla práctica: si un `GROUP BY` de dashboard no tiene un índice cubriente, **no lo corras en vivo por encima de ~1 M de filas**.

### 6.3 Diseño de índices compuestos (orden de columnas)

El orden no es negociable: **igualdad primero, rango después, ordenamiento al final.**

```sql
-- "mis deals cerrados este trimestre, ordenados por fecha"
CREATE INDEX ix_deal_owner_closed ON deal (owner_id, closed_at DESC, id DESC);
--                                       ^igualdad   ^rango+orden   ^desempate

-- "deals de esta cuenta, más recientes primero"
CREATE INDEX ix_deal_account_created ON deal (account_id, created_at DESC);

-- "pipeline abierto por etapa" (índice parcial: solo abiertos, mucho más chico)
CREATE INDEX ix_deal_open_stage ON deal (stage_id, amount_cents)
  WHERE won = false AND lost = false AND deleted_at IS NULL;

-- búsqueda por rango de fechas para reportes ad-hoc
CREATE INDEX ix_deal_closed_brin ON deal USING BRIN (closed_at);
```

El **índice parcial** es la joya poco usada: si el 80 % de las consultas son sobre deals abiertos y solo el 5 % de las filas están abiertas, el índice pesa 1/20 y vive entero en caché. `[inferido]`

**BRIN** para tablas append-only con correlación temporal (historia, logs, snapshots): índice de kilobytes en vez de gigabytes, a cambio de leer más páginas. Ideal para `record_history` y `fact_deal_close` cuando ya están particionadas.

### 6.4 Particionado por fecha

```sql
CREATE TABLE record_history (
  ...
) PARTITION BY RANGE (changed_at);

-- Una partición por mes, creada por un job
CREATE TABLE record_history_2026_11 PARTITION OF record_history
  FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');
```

Cuándo particionar (criterios, no superstición):

| Señal | Particionar |
|---|---|
| La tabla supera la RAM del servidor | sí |
| Necesitás borrar por antigüedad (retención) | **sí, siempre** — `DROP TABLE` vs `DELETE` masivo + `VACUUM` |
| Las consultas siempre filtran por un rango de fecha | sí |
| Tabla de 2 M de filas, consultas por PK | **no** |
| Muchas dimensiones de filtro sin fecha dominante | no (hash, o nada) |

Postgres advierte explícitamente: demasiadas particiones degradan el *planning* y consumen memoria por sesión; el planner maneja bien hasta unos pocos miles. `[verificado]` https://www.postgresql.org/docs/current/ddl-partitioning.html

### 6.5 El patrón "skinny table" de Salesforce

Salesforce guarda los campos estándar y los custom en **tablas físicas separadas**, lo que obliga a un join invisible en cada consulta. Una *skinny table* es una copia con solo los campos usados, de ambas familias, sin registros borrados: la consulta deja de necesitar el join y lee filas más angostas (más filas por página). `[verificado]` https://developer.salesforce.com/docs/platform/salesforce-large-data-volumes-bp/guide/ldv-deployments-infrastructure-skinny-tables.html

Restricciones documentadas: máximo **200 columnas**, no puede traer campos de otros objetos, solo sobre Account/Contact/Opportunity/Lead/Case y custom objects, **la crea el soporte** (no el admin), y hay que pedirla con la consulta concreta en la mano. Advertencia textual de Salesforce: *"no son una varita mágica… mantener tablas separadas que copian datos vivos tiene costo; usarlas mal degrada el rendimiento"*. `[verificado]` (misma URL)

Cómo se traduce esto a Postgres, sin soporte de por medio:

```sql
-- "Vista materializada angosta" para el listado principal del CRM
CREATE MATERIALIZED VIEW deal_list_mv AS
SELECT d.id, d.title, d.amount_cents, d.currency, d.stage_id, d.owner_id,
       d.close_date, d.created_at, d.updated_at,
       a.name AS account_name, u.name AS owner_name,
       d.custom->>'industria' AS industria
FROM deal d
LEFT JOIN account a ON a.id = d.account_id
LEFT JOIN "user"  u ON u.id = d.owner_id
WHERE d.deleted_at IS NULL;

CREATE UNIQUE INDEX ix_dlm_id ON deal_list_mv (id);
CREATE INDEX ix_dlm_owner ON deal_list_mv (owner_id, updated_at DESC);
CREATE INDEX ix_dlm_stage ON deal_list_mv (stage_id, amount_cents);
```

Refresco sin bloquear lecturas:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY deal_list_mv;
```

Requiere un índice `UNIQUE` sobre columnas reales (no expresión, no parcial). Bloquea escrituras pero **no lecturas**, y es 1,5–3× más lento que un refresh normal. `[verificado]` https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html · https://www.hivenet.com/post/postgresql-materialized-views

Alternativa superior cuando el refresh se pone pesado: **tabla real + job incremental** (`INSERT ... ON CONFLICT DO UPDATE` solo de las filas con `updated_at > last_run`). El refresh de una MV es todo o nada; el incremental escala.

### 6.6 Números por umbral, en resumen

| Filas en la tabla principal | Qué duele primero | Qué hacer |
|---|---|---|
| 10 k – 100 k | nada | nada. Índices por PK y por owner. |
| 100 k – 500 k | agregados de dashboard (~100–300 ms × 8 widgets) | caché de resultado 60 s; empezar `agg_*` |
| 500 k – 2 M | agregados (> 1 s), export, `LIKE` | `agg_deal_daily` + `agg_deal_monthly`, FTS, keyset |
| 2 M – 10 M | agregados (segundos), índices grandes, `VACUUM` | agregados obligatorios, particionado de historia, índice cubriente o MV |
| 10 M – 100 M | todo lo anterior + mantenimiento | Postgres con particiones + réplica de lectura; quizás motor de búsqueda aparte |
| > 100 M | la operación normal | separar OLTP de analítica (réplica/almacén), archivar cerrados |

---

## 7. Paginación y exportación

### 7.1 Por qué `OFFSET` se degrada

`OFFSET 199950 LIMIT 50` no salta 199.950 filas: **las lee, las evalúa, las descarta y devuelve 50**. Costo `O(offset + limit)`. Y hay un segundo problema, peor que el rendimiento: si entre la página 3 y la 4 alguien inserta un registro, la página 4 **repite** un elemento; si borra, **salta** uno. En un CRM con actividad constante, esto se nota. `[inferido]` a partir del modelo de ejecución; confirmado por múltiples fuentes: https://www.trinitylogic.co.uk/blog/spring-data-keyset-pagination-java

Medición propia (10 M filas, `ORDER BY created_at DESC, id DESC`):

| Página | Offset | Tiempo |
|---|---:|---:|
| 1 | 0 | 0,1 ms |
| 500 | 24.950 | 1,0 ms |
| 4.000 | 199.950 | **36,3 ms** |
| keyset equivalente | — | **0,1 ms** |

El offset es 360× más lento en la página 4.000 y crece linealmente. Benchmark publicado en una tabla de 10 M: página 1 = 8 ms, página 1.000 = 890 ms, página 10.000 = **8.200 ms**. `[verificado]` https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset-vs-keyset

### 7.2 Keyset (cursor) pagination

```sql
-- Página 1
SELECT id, title, amount_cents, created_at
FROM deal
WHERE owner_id = $1 AND deleted_at IS NULL
ORDER BY created_at DESC, id DESC
LIMIT 51;                         -- 51 para saber si hay siguiente sin COUNT

-- Página siguiente: el cursor es (created_at, id) del último elemento recibido
SELECT id, title, amount_cents, created_at
FROM deal
WHERE owner_id = $1 AND deleted_at IS NULL
  AND (created_at, id) < ($2, $3)    -- comparación de fila: usa el índice compuesto
ORDER BY created_at DESC, id DESC
LIMIT 51;
```

Requisitos y límites:

1. **Índice compuesto que cubra el `ORDER BY` completo**, incluyendo el desempate (`id`). Sin el desempate, dos filas con el mismo `created_at` hacen que la paginación pierda o repita registros.
2. **No podés saltar a la página 47.** Si el producto exige "ir a la página 47", keyset no alcanza. Solución honesta: no ofrecer números de página (scroll infinito / "cargar más"), o mantener un `COUNT(*)` cacheado para mostrar "≈ 12.400 resultados" sin permitir salto.
3. **El cursor va firmado/opaco** (base64 del par + HMAC) si es una API pública: no expongas el orden interno ni permitas que el cliente lo falsifique.
4. `WHERE (a,b) < (x,y)` es una comparación de fila y usa el índice en Postgres. En SQLite también está soportada (medido: 0,1 ms a 10 M filas). `[medido]`

### 7.3 Exportar 500.000 filas sin tumbar el servidor

Lo que **no** se hace: `SELECT *` de la tabla entera a memoria, serializar a un string, y devolverlo en la respuesta HTTP.

```python
# Mal: 500k filas en memoria, la respuesta se arma entera, timeout del proxy
rows = cur.execute("SELECT * FROM deal").fetchall()
return json.dumps(rows)

# Bien: cursor del servidor + streaming al cliente en chunks
with conn.cursor(name="export_deals") as cur:          # server-side cursor (psycopg)
    cur.itersize = 5000
    cur.execute("""SELECT id, title, amount_cents, closed_at
                   FROM deal WHERE closed_at >= %s AND closed_at < %s
                   ORDER BY id""", (start, end))
    for chunk in iter(lambda: list(islice(cur, 5000)), []):
        yield render_csv(chunk)
```

Medición propia en SQLite (10 M filas en la tabla): exportar 500.000 filas a CSV — **0,69 s y 24 MB, con memoria constante**; cargar todo con `fetchall` primero — 0,41 s pero con 500 k tuplas residentes en RAM (≈ 150–250 MB según el ancho de fila). `[medido]`

Arquitectura recomendada para export grande:

```
POST /exports               → 202 Accepted {job_id}
   ↓ worker (fuera del proceso web)
   cursor + streaming → archivo temporal / S3, en lotes de 5–10 k filas
   ↓
GET /exports/{job_id}       → {status: "done", url: "..."} (URL firmada, TTL 24 h)
```

Reglas:

- **Nunca** en el request HTTP del usuario. Un export de 500 k puede tardar minutos; el proxy corta a los 30–60 s y el usuario reintenta, duplicando la carga.
- **Lotes de 5.000–10.000 filas** por round trip. Más grande, más memoria; más chico, más overhead.
- **Sin `OFFSET` en el exportador.** Paginá por keyset (`WHERE id > last_id`) o por rangos de fecha. Un export con `OFFSET` de 500 k filas es el peor caso posible.
- **Timeout y `statement_timeout`** en la conexión del worker, más `LIMIT` de seguridad.
- **Formato**: CSV con `\N` para nulos y UTF-8 con BOM si el destino es Excel en español (sin BOM, los acentos se rompen). `[inferido]`
- **Cuota**: máximo N exports concurrentes por organización. Un usuario puede pedir 5 exports de 500 k y saturar el pool.

Referencia de cómo lo hace un grande: Salesforce muestra **2.000 filas** en pantalla, exporta **100.000** en `.xlsx` (100 columnas), **20.000** en reportes unidos, y **trunca silenciosamente** lo que sobra. El export "detalles solamente" en CSV no tiene ese tope, y los reportes tienen timeout de **10 minutos** (extendible a 20). `[verificado]` https://www.saleswingsapp.com/salesforce/reporting-tips-40-expert-faqs-to-create-reliable-and-accurate-salesforce-reports-and-dashboards y https://metrica.hashnode.dev/salesforce-reporting-at-enterprise-scale-gaps-limits-and-options

La lección del truncado silencioso es la más importante de esta sección: **si tu export tiene un límite, decilo en el archivo y en la UI.** Un CSV que dice "100.000 de 500.000 filas" es un bug de confianza; uno que lo declara es un producto.

---

## 8. Concurrencia: leer mientras se escribe

### 8.1 Los dos modelos

**Postgres: MVCC.** Cada escritura crea una versión nueva de la fila; los lectores ven el snapshot del inicio de su sentencia (READ COMMITTED, el default) o de su transacción (REPEATABLE READ). **Un `SELECT` nunca bloquea un `INSERT`, y un `INSERT` nunca bloquea un `SELECT`.** Lo que sí bloquea: `ALTER TABLE`, `VACUUM FULL`, `REFRESH MATERIALIZED VIEW` sin `CONCURRENTLY`, y los `LOCK` explícitos.

**SQLite: un escritor a la vez.** En WAL, los lectores no bloquean al escritor ni viceversa, y hay múltiples lectores concurrentes; pero **solo hay un escritor**. Un segundo escritor espera (o recibe `SQLITE_BUSY`). `[verificado]` https://sqlite.org/wal.html

### 8.2 Lo que medí

Lectura de 0,48 s (self-join de 10 M filas contra 10 M de actividades) mientras otro hilo escribía sin parar: **15.468 escrituras completadas en 3 s (~5.150/s) y la lectura terminó en 0,48 s sin esperar**. `[medido]`

Es la demostración de que **un reporte largo no tiene por qué bloquear la operación** — si usás WAL y el motor es MVCC. En SQLite la lectura larga tampoco bloquea las escrituras, pero **sí puede bloquear el checkpoint**: si hay siempre un lector activo, el WAL crece sin límite y las lecturas se degradan. La doc lo dice explícitamente ("checkpoint starvation"). `[verificado]` (misma URL)

### 8.3 Reglas para que un reporte no tumbe la operación

| Riesgo | Mitigación |
|---|---|
| Reporte pesado compite por CPU con la venta | `statement_timeout` (ej. 30 s) + `work_mem` acotado por sesión de reportes |
| Reporte pesado compite por conexiones | **pool separado** para reportes (5 conexiones) o **réplica de lectura** |
| `REFRESH MATERIALIZED VIEW` bloquea todo | `CONCURRENTLY` + índice único; nunca en horario pico |
| Vacuum/analyze compite con I/O | autovacuum afinado por tabla; `VACUUM` manual en ventana |
| Export masivo satura el pool | cola de trabajos con concurrencia limitada |
| Deadlocks en escritura | ordenar los `UPDATE` por `id`; transacciones cortas |
| Long-running transaction retiene versiones viejas (bloat) | `idle_in_transaction_session_timeout` |

```sql
-- Aislamiento para reportes: snapshot consistente, sin bloquear escritores
BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY;
SET LOCAL statement_timeout = '30s';
-- ... todas las consultas del reporte ven el mismo instante ...
COMMIT;

-- Usuario de reportes sin permisos de escritura y con límites propios
ALTER ROLE crm_reporter SET statement_timeout = '30s';
ALTER ROLE crm_reporter SET work_mem = '32MB';
ALTER ROLE crm_reporter SET default_transaction_read_only = on;
```

`REPEATABLE READ` + `READ ONLY` es lo correcto para un reporte de varias consultas: sin eso, el total del widget de arriba y el desglose del widget de abajo pueden venir de instantes distintos y no sumar. **Ese es un bug de reporte que nadie reporta como bug** — el usuario ve "los números no cuadran" y deja de confiar. `[inferido]`

`READ ONLY` es defensa en profundidad: si un bug de la app manda un `UPDATE` desde el usuario de reportes, falla en vez de corromper.

---

## 9. Qué se puede hacer bien con SQLite (con números)

### 9.1 Los límites del motor

| Límite | Valor | Condición |
|---|---|---|
| Tamaño máximo de base de datos | **281 TB** teórico / **17,5 TB** con páginas de 4 KB | `SQLITE_MAX_PAGE_COUNT` = 4.294.967.294; desde 3.45.0 ese es el default |
| Tamaño máximo con páginas de 64 KB | 281 TB | `page_size=65536` |
| Filas por tabla | 2⁶⁴ teórico | inalcanzable; el tamaño del archivo llega antes |
| Columnas por tabla | 2.000 por defecto (32.767 compilado) | `SQLITE_MAX_COLUMN` |
| Valor máximo (BLOB/TEXT) | 1 GB | `SQLITE_MAX_LENGTH` |

`[verificado]` https://sqlite.org/limits.html

Ojo con el número grande: 281 TB es la aritmética del formato, no una recomendación. El propio equipo de SQLite dice que si los datos van a acercarse a los terabytes, conviene un motor cliente/servidor. Y hay un límite práctico que no aparece en la tabla: **el archivo tiene que caber en un solo filesystem y no puede estar en NFS** (WAL necesita memoria compartida). `[verificado]` https://sqlite.org/whentouse.html

### 9.2 Lo que medí en este servidor

| Escenario | Resultado |
|---|---|
| 10 M deals + 10 M activities, 4 índices, FTS5 | **2,82 GB**, insert 59 s, índices 25 s, FTS 25 s |
| Agregado mensual desde 10 M filas | 14,8 s (job nocturno, irrelevante) |
| Consulta sobre el agregado (mes × owner) | **~0 ms** |
| Embudo con índice cubriente, 10 M filas | 472 ms |
| Embudo sin índice cubriente, 10 M filas | 7.260–15.436 ms |
| Búsqueda FTS5 exacta, 10 M filas | 0,1 ms |
| Búsqueda FTS5 con prefijo | 20 ms |
| FTS5 con `bm25` ordenado | 890 ms |
| Escrituras concurrentes (WAL, autocommit) | ~5.150/s |
| Insert masivo en una transacción | ~170.000 filas/s (10 M en 59 s) |
| Insert con 2 filas de historia por fila | 34.000/s (8,2× más lento) |
| Export CSV 500 k filas, streaming | 0,69 s, 24 MB |
| Lectura de 0,48 s concurrente con escritura | sin bloqueo |

`[medido]`

### 9.3 Qué se puede hacer bien, y qué no

**Se puede hacer bien:**

- Un CRM de **una sola organización** con hasta ~1–2 M de registros activos, con agregados precalculados y FTS5. Los dashboards responden en milisegundos porque leen 100 k filas de agregados, no 10 M.
- Búsqueda full-text de buena calidad hasta ~10 M documentos (0,1 ms exacta, 20 ms con prefijo). El ranking (`bm25`) es lo que se degrada primero (890 ms a 10 M).
- Exportaciones y respaldos triviales: `cp` del archivo. Backup incremental: `VACUUM INTO` o Litestream. `[inferido]`
- Despliegue: un binario + un archivo. Cero administración de base de datos.

**No se puede hacer bien:**

- **Muchos escritores concurrentes.** Un solo writer. Con 200 usuarios escribiendo a la vez, la cola de escritura se vuelve el cuello. A 5.000 escrituras/s de operaciones puntuales está bien; a 50 usuarios haciendo drag-and-drop simultáneo en un tablero Kanban, no. `[medido]` + `[inferido]`
- **Multi-tenant con 50 organizaciones en el mismo archivo** y expectativas de aislamiento. Se puede con `tenant_id` + índices, pero perdés el aislamiento operativo (un `VACUUM` afecta a todos).
- **Red / contenedores con volúmenes de red.** WAL no funciona sobre NFS. `[verificado]`
- **Réplica de lectura y failover.** No existe el equivalente a un standby con streaming replication.
- **Analítica ad-hoc sobre 50 M de filas.** No hay paralelismo de consulta, ni `JIT`, ni particionado declarativo.
- **`ALTER TABLE` sobre tablas grandes con la app corriendo.** Bloquea; en un CRM eso significa ventana de mantenimiento.

### 9.4 Reproducir el benchmark

```python
# Estructura del test (SQLite 3.45+, WAL, synchronous=NORMAL, cache 200 MB)
# 1) crear deal/activity, insertar N filas en una transacción
# 2) medir "embudo" ANTES de crear índices secundarios (seq scan)
# 3) crear índices, volver a medir
# 4) crear índice cubriente (stage_id, amount_cents, won), volver a medir
# 5) crear FTS5 external-content + triggers, medir match/prefix/bm25
# 6) offset profundo vs keyset
# 7) agregado mensual y consulta sobre el agregado
# 8) concurrencia: hilo lector + hilo escritor, contar escrituras
```

Los tiempos reportados son la mediana de 3 corridas, con la base ya poblada. La varianza entre corridas fue alta en el caso del índice no cubriente (7,3 s vs 15,4 s a 10 M) porque depende del estado de la caché de páginas: por eso el rango y no un número único.

### 9.5 Veredicto honesto

SQLite es una elección **correcta** para: CRM de un solo tenant, hasta ~1 M de registros activos, pocos escritores concurrentes, servidor único, y con la capa de agregados bien hecha. En ese escenario es más rápido que Postgres para lecturas por clave (no hay round-trip de red) y muchísimo más simple de operar. Benchmark publicado por el propio equipo: el protocolo de SQLite puede ser hasta **35 % más rápido que el filesystem** para archivos < 10 KB. `[verificado]` (vía https://www.hendrik-erz.de/post/why-you-shouldnt-use-sqlite, que cita la doc de SQLite)

SQLite es una elección **incorrecta** cuando: hay más de un servidor de aplicación, hay un volumen de red, hay más de un puñado de escritores concurrentes, o el negocio va a pedir "un reporte de los últimos 5 años sobre 60 M de actividades" sin precalcular.

La transición razonable no es "reescribir todo en Postgres el día que duele". Es: **mantener SQLite para la operación y mover los reportes a un archivo derivado** (un `.db` de agregados regenerado cada noche, o un Postgres de analítica). Eso compra años.

---

## 10. Los cinco reportes del día uno

Criterio de selección: cada uno responde una pregunta que alguien hace **en la primera semana** de usar un CRM, y entre los cinco cubren las tres capas (vivo, agregado, congelado).

### Reporte 1 — Embudo actual (pipeline por etapa)

**Pregunta del negocio:** "¿Cuánto tengo abierto y dónde está trabado?"
**Capa:** vivo (o agregado de 5 min). **Tablas:** `deal`, `stage`.

```sql
SELECT s.position AS orden,
       s.name     AS etapa,
       COUNT(*)                                   AS oportunidades,
       COALESCE(SUM(d.amount_cents), 0) / 100.0   AS monto,
       ROUND(AVG(d.amount_cents) / 100.0, 0)      AS ticket_promedio,
       ROUND(AVG(EXTRACT(day FROM now() - d.created_at)), 0) AS edad_dias,
       ROUND(100.0 * SUM(d.amount_cents)
             / SUM(SUM(d.amount_cents)) OVER (), 1)            AS pct_del_pipeline
FROM deal d
JOIN stage s ON s.id = d.stage_id
WHERE d.is_open
  AND d.deleted_at IS NULL
  AND d.pipeline_id = $1
GROUP BY s.position, s.name
ORDER BY s.position;
```

### Reporte 2 — Conversión etapa a etapa (y global)

**Pregunta:** "¿Dónde se cae el negocio?"
**Capa:** vivo sobre eventos hasta ~5 M; después, tabla de conversión mensual. **Tablas:** `stage_transition`, `stage`.

```sql
WITH alcanzo AS (
  SELECT deal_id, to_stage_id AS stage_id, MIN(entered_at) AS t
  FROM stage_transition
  WHERE entered_at >= $1 AND entered_at < $2
  GROUP BY deal_id, to_stage_id
),
por_etapa AS (
  SELECT s.position, s.id, s.name,
         COUNT(*) AS llegaron,
         COUNT(*) FILTER (WHERE EXISTS (
           SELECT 1 FROM alcanzo a2
           WHERE a2.deal_id = a.deal_id AND a2.stage_id <> a.stage_id
             AND (SELECT position FROM stage WHERE id = a2.stage_id)
               > (SELECT position FROM stage WHERE id = a.stage_id)
         )) AS avanzaron
  FROM alcanzo a
  JOIN stage s ON s.id = a.stage_id
  GROUP BY s.position, s.id, s.name
)
SELECT position AS orden, name AS etapa, llegaron, avanzaron,
       ROUND(100.0 * avanzaron / NULLIF(llegaron, 0), 1) AS pct_avance
FROM por_etapa
ORDER BY position;
```

### Reporte 3 — Tiempo de cierre (mediana y p90, no promedio)

**Pregunta:** "¿Cuánto tardamos en cerrar, y el mes pasado fue peor?"
**Capa:** agregado mensual congelado. **Tablas:** `agg_close_monthly` (o `deal` si aún no hay cierre).

```sql
SELECT to_char(period, 'YYYY-MM') AS mes,
       SUM(deals_won)                      AS ganados,
       ROUND(SUM(amount_won_clp_cents)/100.0, 0) AS monto,
       ROUND(AVG(median_cycle_days)::numeric, 1) AS dias_mediana_pond,
       ROUND(AVG(avg_cycle_days)::numeric, 1)    AS dias_promedio
FROM agg_close_monthly
WHERE period >= date_trunc('month', current_date) - interval '12 months'
  AND close_run_id = (SELECT max(id) FROM close_run
                      WHERE kind='monthly' AND status='done')
GROUP BY period
ORDER BY period;
```

Versión en vivo (sin cierre, para el mes en curso):

```sql
SELECT ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (
         ORDER BY EXTRACT(epoch FROM closed_at - created_at)/86400)::numeric, 1) AS dias_mediana,
       ROUND(PERCENTILE_CONT(0.9) WITHIN GROUP (
         ORDER BY EXTRACT(epoch FROM closed_at - created_at)/86400)::numeric, 1) AS dias_p90,
       COUNT(*) AS cerrados
FROM deal
WHERE won AND closed_at >= date_trunc('month', current_date) AND deleted_at IS NULL;
```

### Reporte 4 — Ventas por vendedor, región y canal

**Pregunta:** "¿Quién vende, dónde y por qué canal?"
**Capa:** agregado mensual. **Tablas:** `agg_deal_monthly`, `user`, `region`, `channel`.

```sql
SELECT u.name AS vendedor,
       r.name AS region,
       c.name AS canal,
       SUM(a.deals_won)                AS ganados,
       SUM(a.amount_won_cents)/100.0   AS monto,
       ROUND(100.0 * SUM(a.amount_won_cents)
             / SUM(SUM(a.amount_won_cents)) OVER (), 1) AS pct_total,
       ROUND(SUM(a.amount_won_cents)/100.0 / NULLIF(SUM(a.deals_won),0), 0) AS ticket_promedio
FROM agg_deal_monthly a
JOIN "user"  u ON u.id = a.owner_id
JOIN region  r ON r.id = a.region_id
JOIN channel c ON c.id = a.channel_id
WHERE a.month >= $1 AND a.month < $2
GROUP BY u.name, r.name, c.name
ORDER BY monto DESC;
```

**Este reporte es el que más miente** si `owner_id` es mutable. Para el histórico, usar `closed_by_id` (ver §3.4): una columna, un `GROUP BY` distinto, y el reporte del año pasado deja de cambiar cuando alguien se va de la empresa.

### Reporte 5 — Forecast del trimestre vs cuota

**Pregunta:** "¿Vamos a llegar? ¿Qué está en riesgo?"
**Capa:** vivo sobre abiertos (pocas filas: solo el pipeline abierto del trimestre). **Tablas:** `deal`, `stage`, `quota`.

```sql
WITH trim AS (
  SELECT date_trunc('quarter', current_date)::date AS ini,
         (date_trunc('quarter', current_date) + interval '3 months')::date AS fin
)
SELECT u.name AS vendedor,
       COUNT(d.id)                                        AS oportunidades,
       SUM(d.amount_cents)/100.0                          AS pipeline_bruto,
       ROUND(SUM(d.amount_cents * COALESCE(d.probability, s.default_probability) / 100.0)/100.0, 0)
                                                          AS forecast_ponderado,
       SUM(d.amount_cents) FILTER (WHERE d.close_date < current_date)/100.0
                                                          AS vencido_sin_cerrar,
       q.amount_cents/100.0                               AS cuota,
       ROUND(100.0 * SUM(d.amount_cents * COALESCE(d.probability, s.default_probability)/100.0)
             / NULLIF(q.amount_cents, 0), 0)              AS pct_cuota,
       SUM(d.amount_cents * COALESCE(d.probability, s.default_probability)/100.0)/100.0
         - q.amount_cents/100.0                           AS brecha
FROM deal d
JOIN "user" u ON u.id = d.owner_id
JOIN stage  s ON s.id = d.stage_id
CROSS JOIN trim t
LEFT JOIN quota q ON q.owner_id = u.id AND q.period_start = t.ini
WHERE d.is_open
  AND d.deleted_at IS NULL
  AND d.close_date >= t.ini
  AND d.close_date <  t.fin
GROUP BY u.name, q.amount_cents
ORDER BY brecha ASC;
```

El campo `vencido_sin_cerrar` es el que convierte este reporte de decorativo en accionable: son los negocios con fecha de cierre pasada que nadie tocó. Si ese número crece, el forecast es ficción.

### Y el sexto, que no es del día uno pero llega en el mes dos

**Cohortes de clientes** (§3.5). Es el único que responde "¿el negocio es sustentable?" en vez de "¿cómo vamos este mes?". Necesita `account.first_won_at` poblado correctamente — y esa columna hay que crearla antes de tener datos, no después.

---

## Apéndice A — Orden de implementación

| Fase | Qué | Por qué primero |
|---|---|---|
| 1 | `is_open`, `closed_by_id`, `amount_initial_cents`, `closed_on`, `deleted_at` como columnas reales | son la diferencia entre reportes correctos e incorrectos; agregarlas después requiere backfill |
| 2 | Índices: `(owner_id, closed_at DESC, id DESC)`, `(account_id, created_at DESC)`, parcial de abiertos | resuelven el 80 % de las pantallas |
| 3 | `stage_transition` + trigger | sin esto no hay historia y **no se puede recuperar después** |
| 4 | Los 5 reportes de §10, en vivo | con ≤ 200 k filas andan bien y validan las definiciones con el negocio |
| 5 | `record_history` solo para 5–8 campos elegidos | el costo de escritura es real (8,2×) |
| 6 | `agg_deal_monthly` + job nocturno | el primer dashboard que se pone lento |
| 7 | `close_run` + `fact_deal_close` + `agg_close_monthly` | antes del primer cierre de mes con comisiones de por medio |
| 8 | FTS5/`tsvector` + búsqueda global | cuando el `LIKE '%texto%'` pase de 100 ms |
| 9 | Export asíncrono con streaming | cuando alguien pida "todos los deals a Excel" |
| 10 | Particionado de historia y snapshots | a partir de ~10 M de filas en esas tablas |

## Apéndice B — Errores que cuestan caro

1. **Dashboard en vivo contra la tabla principal.** Funciona a 50 k filas y explota a 1 M. Se descubre tarde.
2. **No guardar historia desde el día uno.** No hay backfill posible. Es el único error de esta lista que es **irreversible**.
3. **Índice por columna de filtro sin pensar en el `SUM`.** Medido: 2–4× más lento que no tener índice.
4. **`OFFSET` en la API y en el exportador.** Degradación lineal + filas duplicadas/saltadas.
5. **`owner_id` para reportes históricos.** Las ventas cambian de dueño retroactivamente.
6. **Recalcular el pasado.** El reporte del mes pasado cambia y nadie sabe por qué.
7. **Convertir moneda al vuelo con la tasa del día.** El revenue histórico se mueve solo.
8. **Promedios sobre distribuciones asimétricas.** Usar mediana y p90.
9. **`REFRESH MATERIALIZED VIEW` sin `CONCURRENTLY`** en horario de oficina: bloquea todas las lecturas del dashboard.
10. **Un solo pool de conexiones para operación y reportes.** Un reporte lento deja al vendedor sin poder guardar.

---

### Fuentes citadas

**Vendors de CRM**
- Salesforce, Reporting Snapshot (2.000 filas por corrida): https://help.salesforce.com/s/articleView?id=analytics.data_setting_up_analytic_snap.htm&type=5
- Salesforce, límites de reportes (2.000 en pantalla / 100.000 export / 20.000 unidos): https://www.saleswingsapp.com/salesforce/reporting-tips-40-expert-faqs-to-create-reliable-and-accurate-salesforce-reports-and-dashboards
- Salesforce, reportes a escala empresarial (truncado silencioso, Bulk API): https://metrica.hashnode.dev/salesforce-reporting-at-enterprise-scale-gaps-limits-and-options
- Salesforce, field history tracking (20 campos, 18/24 meses): https://www.salesforceben.com/salesforce-field-history-tracking
- Salesforce, Field Audit Trail vs estándar: https://nickfrates.com/field-audit-trail-vs-field-history
- Salesforce, `FieldHistoryArchive` (big object, solo API): https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_fieldhistoryarchive.htm
- Salesforce, Apex governor limits (50.000 filas SOQL, 120 s): https://developer.salesforce.com/docs/platform/apex-dev/references/apex-dev/apex-gov-limits.html
- Salesforce, skinny tables y LDV: https://developer.salesforce.com/docs/platform/salesforce-large-data-volumes-bp/guide/ldv-deployments-infrastructure-skinny-tables.html
- Salesforce, selectividad de índices (10 % / 30 %, topes 333 k / 1 M): https://micropyramid.com/blog/how-to-manage-large-data-volumes-using-skinny-tables-and-indexes
- HubSpot, frecuencia de actualización (40 min / 2 h / 15 min manual): https://knowledge.hubspot.com/reports/how-often-do-analytics-in-hubspot-update
- HubSpot, custom report builder (1.000 filas únicas, 10–15 min de lag): https://knowledge.hubspot.com/reports/create-reports-with-the-custom-report-builder
- Pipedrive, Insights deal progress (usa la etapa más reciente): https://support.pipedrive.com/en/article/insights-reports-deal-progress
- Pipedrive, Insights deal duration: https://support.pipedrive.com/en/article/insights-reports-deal-duration
- EspoCRM, metadata y definición de índices/`textFilterFields`: https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Resources/metadata/entityDefs/Lead.json · https://docs.espocrm.com/development/metadata

**PostgreSQL**
- Full text search: https://www.postgresql.org/docs/current/textsearch.html
- `REFRESH MATERIALIZED VIEW` (requisito de índice único, `CONCURRENTLY`): https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html
- Comportamiento de locks en refresh concurrente: https://www.hivenet.com/post/postgresql-materialized-views
- Table partitioning (pruning, límites prácticos): https://www.postgresql.org/docs/current/ddl-partitioning.html
- `pg_trgm`: https://www.postgresql.org/docs/current/pgtrgm.html
- Tamaño de índices trigram (3–4× el dato): https://aws.amazon.com/blogs/database/index-types-supported-in-amazon-aurora-postgresql-and-amazon-rds-for-postgresql-using-extensions-bloom-pg_trgm-and-pg_bigm
- Penalización de escritura de GIN trigram (~2×): https://www.runxbuild.com/blog/postgresql-pg-trgm-extension
- tsvector persistido (3.042 ms → 1 ms): https://danielabaron.me/blog/speed-up-pg-fts-with-persistent-ts-vectors
- Benchmark nativo vs motores BM25 en 10 M filas: https://neon.com/blog/postgres-full-text-search-vs-elasticsearch
- Tamaño de índice FTS sobre Wikipedia (12 GB textsearch, 7,6 GB pg_trgm): https://pgroonga.github.io/reference/pgroonga-versus-textsearch-and-pg-trgm.html

**SQLite**
- Límites del motor (281 TB / 17,5 TB, 2⁶⁴ filas): https://sqlite.org/limits.html
- Cuándo usar SQLite (y cuándo no): https://sqlite.org/whentouse.html
- WAL: concurrencia, checkpoint starvation: https://sqlite.org/wal.html
- FTS5: tokenizers, `remove_diacritics`, `bm25`, `detail`, prefijos: https://sqlite.org/fts5.html

**Búsqueda / paginación**
- Elasticsearch near-real-time (refresh ~1 s): https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html
- Offset vs keyset (página 1 = 8 ms, página 10.000 = 8.200 ms): https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset-vs-keyset
- Offset = O(n), degradación y filas fantasma: https://www.trinitylogic.co.uk/blog/spring-data-keyset-pagination-java
