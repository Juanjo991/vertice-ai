# CRM — Investigación para construir las bases

**Fecha:** 16 de noviembre de 2026
**Qué es:** síntesis de seis investigaciones sobre cómo están construidos los CRM reales (Salesforce, HubSpot, Pipedrive, Attio, Twenty, EspoCRM, SuiteCRM, Odoo, ERPNext, Krayin). No hay opinión de marketing: hay esquemas, pipelines, decisiones y números medidos.
**Cómo leer:** esto es el índice y las conclusiones. El detalle, con DDL, payloads y URLs, está en los seis informes:

| # | Informe | Qué resuelve |
|:-:|---|---|
| 01 | [Modelo de datos](01-modelo-de-datos.md) | entidades, campos personalizados, tenancy, dinero, fechas, auditoría |
| 02 | [Captura e ingesta](02-captura-e-ingesta.md) | canales, importación CSV, deduplicación, sync de correo |
| 03 | [Frontend y UX](03-frontend-y-ux.md) | las 5 pantallas, list view, stack, patrones |
| 04 | [Backend, permisos, automatización](04-backend-permisos-automatizacion.md) | API, autorización, colas, webhooks, operación |
| 05 | [Recorrido completo](05-recorrido-completo.md) | el dato de punta a punta, con las filas exactas |
| 06 | [Reportes, búsqueda y escala](06-reportes-busqueda-escala.md) | dashboards, historia, FTS, límites medidos |

---

## 1. El esqueleto: seis tablas y una cola

Todo CRM comercial cabe en esto. Lo que cambia entre productos es el nombre, no la estructura.

```
accounts (empresa) ──1:N── contacts (persona)
      │                        │
      │1                       │1
      │N                       │N
opportunities (transacción)  leads (pre-entidad)
      │1
      │N
quote → order → invoice

activities (verbo + tiempo)   → parent_type + parent_id (polimórfico)
outbox_events                 → la tabla que nadie dibuja y sin la cual nada es confiable
```

**La regla que ordena todo:** cada escritura de negocio y su evento salen en **la misma transacción** (patrón *outbox*). Publicar antes del commit hace que el consumidor lea algo que no existe; publicar después y morir en el medio pierde el evento para siempre.

Y la regla que explica por qué el CRM es como es: **el list view manda el diseño**. No se modela para "representar la realidad", se modela para que la pantalla que se usa todo el día responda en 100 ms con los filtros puestos.

---

## 2. Las doce decisiones que la investigación cierra

### 2.1 Lead: separado, adelgazado o inexistente

Tres escuelas, todas vigentes:

| Producto | Qué hace con el lead | Por qué |
|---|---|---|
| **HubSpot** | Existe, pero es un **envoltorio de asociación**: exige un contacto y se autoborra si quitas la asociación | adoptaron el modelo de Salesforce sin querer el concepto |
| **Pipedrive** | Existe, **adelgazado**: título, vínculo a persona/empresa, valor, fecha esperada. Hereda los campos de negocios | sirve para capturar rápido antes de decidir si es cliente |
| **Attio** | **No existe**: solo personas, empresas, negocios | el lead es una persona sin calificar; crear una tabla para eso duplica todo |
| **Salesforce** | Existe y **no se borra al convertir**: queda como puntero (`ConvertedAccountId`, `ConvertedContactId`, `ConvertedDate`) | auditoría: poder responder "de dónde salió este cliente" |

**Decisión:** si el sistema captura datos de gente que todavía no es cliente (formulario, WhatsApp, feria), **el lead es una etapa de la persona, no una tabla aparte**. Si además hay que reportar "cuántos leads entraron y cuántos se convirtieron", se guarda el evento de conversión, no una copia del registro. La tabla `leads` separada solo se justifica con volumen de captura y varios vendedores.

### 2.2 Contacto ↔ Empresa es N:M, no 1:N

En B2B una persona trabaja en dos empresas y una empresa tiene varios contactos. Salesforce lo resuelve con `AccountContactRelation`; EspoCRM siempre; **Twenty lo hace mal** (`person.companyId` único).

Y el detalle que casi todos pierden: **el cargo pertenece a la relación, no a la persona**. El mismo Juan es "gerente de compras" en una empresa y "socio" en otra.

### 2.3 La etapa es una fila, no un enum

`stage` como texto libre (`'negociacion'`) rompe todo: no se puede ordenar, no se puede ponderar, no se puede cambiar el nombre sin tocar datos.

```
pipelines(id, name)
pipeline_stages(id, pipeline_id, name, position, probability, is_won, is_lost)
```

Krayin lo hace bien. Twenty lo hace mal. La probabilidad vive en la etapa, no en el registro.

### 2.4 Campos personalizados: tres capas, no una

Esta es la decisión técnica más importante del modelo, y las tres opciones puras están mal:

| Estrategia | Problema real |
|---|---|
| **EAV** (tabla atributo-valor) | cada filtro del list view es un join; Salesforce tuvo que inventar tablas de índices para sobrevivir |
| **JSONB puro** | el planner no tiene estadísticas por clave: estima mal y elige planes malos |
| **DDL en runtime** (`ALTER TABLE`) | funciona bien, pero solo si hay 1 instalación por cliente |

**La respuesta:** 60-80 campos "calientes" como columnas reales (los que aparecen en el list view, los filtros y el kanban), el resto en una columna `custom jsonb` de la misma fila, e índices de expresión **solo** para los campos que el cliente filtra de verdad.

Y el límite no es técnico: **Salesforce limita a 800 campos por objeto; HubSpot a 1.000 por objeto y 10.000 por cuenta.** El motivo es usabilidad y costo de mantener índices, no capacidad.

### 2.5 Multi-tenancy depende del número de clientes

| Clientes | Estrategia | Quién lo usa |
|---|---|---|
| 1 | DDL real, columnas normales | Sugar, Espo, Dolibarr, Frappe |
| 2–200 | **schema por tenant** | Twenty (`workspace_<base36>`, `DROP SCHEMA` para dar de baja, `pg_dump -n` para respaldo) |
| 200–10k | fila compartida + `tenant_id` | la mayoría de los SaaS |
| >10k | + particionado | Salesforce, HubSpot |

**Nunca schema-por-tenant con campos en JSONB**: pagas el fan-out de migraciones sin aprovechar las columnas.

### 2.6 Auditoría son tres cosas distintas

No existe "la auditoría". Existen tres niveles con costos y retenciones diferentes:

| Nivel | Qué guarda | Costo | Retención típica |
|---|---|---|---|
| Columnas de auditoría | `created_by`, `updated_by`, `updated_at` | 1 fila | para siempre |
| Historia de campos | valor anterior/nuevo por campo | **caro** (ver 2.12) | 18-24 meses; 20 campos por objeto |
| Timeline de actividades | qué pasó, cuándo, con quién | 1 fila por evento | para siempre |

Y el historial de etapa va **en tabla aparte**, o el reporte "tiempo en cada etapa" se vuelve un self-join con window functions.

### 2.7 El dinero se congela al escribir

Cuatro columnas, siempre: `amount`, `currency`, `amount_base`, `fx_rate` + `fx_rate_date`. Si el tipo de cambio se lee "al vuelo", el reporte de marzo cambia en abril.

Errores reales encontrados: Dolibarr usa `double(24,8)` (error de punto flotante en dinero); Krayin usa `decimal(12,4)` **sin moneda**, así que no distingue USD de CLP. EspoCRM acierta con `amount` + `amountConverted`.

**Regla:** entero en centavos o `numeric(18,4)`. Nunca `float`, nunca `double`.

### 2.8 La venta de las 22:30

Una venta a las 22:30 en Santiago es **01:30Z del día siguiente**. El dueño en Madrid la ve al día siguiente y el reporte del mes cierra distinto según quién lo mire.

Por eso:
- `close_date` se guarda como **fecha**, no como instante.
- Los reportes usan **la zona del tenant**, nunca la del usuario.
- Se guarda `occurred_local` + `occurred_tz` además del `timestamptz`.

EspoCRM tiene **una sola zona horaria de sistema**: limitación real si el equipo está distribuido.

### 2.9 Permisos: cuatro capas, y la de registro se resuelve en SQL

```
1. scope del token/app
2. permiso por objeto (CRUD + view_all/modify_all)
3. permiso por campo (si no hay fila, no hay acceso)
4. permiso por registro (owner, equipo, jerarquía, compartido)
```

La intersección manda, salvo "compartir", que solo abre.

Y la regla que evita el desastre: **una sola función `visibility_clause(usuario, objeto)` que devuelve SQL**, reutilizada por listado, detalle, búsqueda, export, PDF y webhook. Si cada endpoint la reimplementa, el que se olvide filtra datos.

Jerarquía como *materialized path* indexado, no `WITH RECURSIVE` por request. Compartir con grupos, nunca con personas: una fila por grupo, no una por miembro.

### 2.10 Síncrono y asíncrono: la venta nunca se encola

| Asíncrono | Síncrono |
|---|---|
| correo, webhooks salientes, import/export, reindexado, recálculos masivos | commit de la venta, cambio de etapa, totales, validación de permisos |

La venta no se encola: **se encola su efecto secundario**. Transporte *at-least-once* + consumidor idempotente = efecto único.

### 2.11 Los dashboards no se calculan en vivo

Ningún CRM serio lo hace. Los que lo niegan lo llaman caché. HubSpot refresca reportes cada 2 h (manual cada 15 min) y limita los gráficos a 1.000 filas; Salesforce congela con *Reporting Snapshots* de máximo 2.000 filas por corrida; **Pipedrive ni guarda historia de etapas**.

Tres capas:

```
operación (tablas de trabajo) → agregados (reconstruibles) → cierre inmutable (el pasado)
```

La consulta en vivo queda para: detalle por ID, listados con índice selectivo y ventanas cortas. Nunca "sumame todo el histórico".

### 2.12 La historia de campos cuesta 8× en escritura

Medido: agregar 2 filas de historia por insert hizo la escritura **8,2× más lenta** (200.000 inserts: 0,71 s → 5,81 s). Ese es el motivo real del límite de 20 campos de Salesforce, no un capricho.

**Decisión:** historia de etapa desde el día 1 (es la única que no se puede reconstruir después). Historia genérica de campos solo para 5-8 campos elegidos, en tabla particionada y sin FK.

### 2.13 El crudo caduca en el proveedor

Salesforce borra los jobs terminados a los 7 días; Twilio el inspector a los 30; Gmail el historial a ~1 semana. **Sin capa cruda propia, la ventana de reproceso es de días.**

**Decisión:** guardar el payload original con su hash, aparte de la tabla de trabajo, con retención declarada (90 días es razonable). Es lo que permite reprocesar cuando se descubre un error de mapeo.

### 2.14 El push nunca trae el dato

Gmail manda `{emailAddress, historyId}`; Calendar solo cabeceras y avisa que "no es 100% confiable"; Graph manda solo IDs. Y los `historyId` de Gmail **no son contiguos**: nunca `+1`, y si caduca devuelve 404 → sincronización completa.

**Decisión:** push = campanilla → cola → pull incremental por cursor opaco → `get` por cada ID → idempotencia por `Message-ID`.

---

## 3. Los seis pipelines, de un vistazo

### 3.1 Importación CSV (el que más se rompe)

```
subida (crudo + SHA-256)
  → detección (BOM, encoding, delimitador, quote, fin de línea)
  → mapeo de columnas versionado (validado contra hash de cabeceras)
  → normalización (E.164 con región obligatoria, email minúsculas, fechas por formato declarado)
  → validación con error por fila
  → deduplicación (intra-archivo + contra la base)
  → upsert por lotes con checkpoint
  → reporte: creados + actualizados + omitidos + fallidos + no procesados = total
```

**La fila 7.000 de 50.000 nunca aborta el job.** Chunks de 500-2.000 en transacción; error de validación → cuarentena con número de fila y valores crudos; error transitorio → rollback del chunk y reintento idempotente; checkpoint tras cada chunk (EspoCRM expone `--resume`, "desde la última fila procesada").

Salesforce lo formaliza en tres archivos: `successfulResults`, `failedResults` y `unprocessedrecords`, y advierte que **no procesado no es lo mismo que fallido**.

### 3.2 Deduplicación

Exacta sobre clave normalizada, **no fuzzy en el camino caliente**.

- **Salesforce** separa *matching rule* (candidatos, con match keys indexados) de *duplicate rule* (acción). Y en updates la regla **solo corre si se editaron campos de la matching rule**: si tu integración actualiza `phone` y la regla usa `email`, el duplicado no se detecta nunca.
- **HubSpot**: email único / dominio. Si hay varias coincidencias, **falla la fila**.
- **Fuzzy** (Levenshtein, fonético) existe como capa de *sugerencia* con blocking de 6 caracteres. La banda 0,75-0,9 se marca para revisión, no se decide sola.
- **Merge** = reparentar hijos, dejar el registro perdedor apuntando al maestro (`masterRecordId`), y mantener tabla de alias para no romper la sincronización.

### 3.3 Sincronización de correo

```
push (campanilla) → cola → pull incremental (historyId / deltaLink / UID+UIDVALIDITY)
→ get por ID → idempotencia por Message-ID → asociar por hilo (In-Reply-To) ANTES que por dirección
→ guardar texto plano + headers selectos; descartar HTML, píxeles y adjuntos a storage
```

Los webhooks con payload mayor a 1 MB **vienen truncados** (Nylas lo marca con `.truncated` y sin cuerpo): el handler necesita camino de re-consulta obligatorio.

### 3.4 Lead entrante (350 ms)

```
POST → edge (TLS, rate limit, CORS) → validación → normalización → dedupe en 4 capas
→ INSERT lead + emails + form_submission + activity + outbox_event  (una transacción)
→ 201 en ~350 ms
→ worker: asigna, crea tarea, notifica (0,5-2 s)
```

### 3.5 Trabajo del lead

Cada llamada, correo o reunión es **una fila en `activities`**, polimórfica, con `occurred_at` distinto de `created_at` (cuándo pasó ≠ cuándo se registró). De ahí sale la timeline.

Mover etapa escribe `opportunities` + `stage_history` (append-only, con snapshot de monto y probabilidad) y dispara cotización, tarea, forecast y aviso.

El "no por ahora" escribe `lost_reason_code` + `recycle_at` + **una tarea futura**. Eso es lo que más plata mueve y lo que casi nadie hace bien.

### 3.6 Cierre → operación

`closed_won` + `won_at` + `amount_snapshot` inmutable → pedido → reserva de stock → factura → asiento → pago → onboarding.

Y el detalle fino: el ciclo de vida pasa a "cliente" **al terminar el onboarding, no al ganar**. Si no, el reporte de clientes nuevos miente.

---

## 4. Ciclo de vida vs. etapa (el error que rompe los reportes)

| | Ciclo de vida | Etapa |
|---|---|---|
| Responde | *¿qué es esta relación?* | *¿dónde va esta transacción?* |
| Nivel | persona o empresa | oportunidad |
| Cuántos | 1 por registro | N por cuenta |
| Comportamiento | monotónico, lento | cíclico, rápido |

**Se derivan por reglas; nunca se escriben el uno al otro.** Mezclarlos hace que el pipeline cuente personas en vez de negocios, y que "clientes nuevos" cuente contactos en vez de primeras ventas.

---

## 5. La interfaz: cinco pantallas, en este orden

| # | Pantalla | Por qué en ese orden |
|:-:|---|---|
| 1 | **List view** | es el producto: define el modelo de campos; todo lo demás cuelga de ella |
| 2 | **Ficha de detalle** | motor de formularios y capacidades por rol |
| 3 | **Timeline de actividad** | obliga a un modelo de eventos append-only; es la memoria del cliente |
| 4 | **Kanban por etapa** | reutiliza la vista 1 con un campo de agrupación; necesita totales por columna o es decoración |
| 5 | **Panel de métricas** | agregaciones sobre consultas que ya existen; va último porque empezar acá produce una demo, no un CRM |

**Stack mínimo recomendado (licencias cero):** Vite + React + TypeScript; TanStack Router (filtros en la URL), TanStack Query (estado de servidor), **TanStack Table + Virtual** (headless: control total de densidad y accesibilidad), react-hook-form + Zod compartiendo esquema con el servidor, Radix para primitivas, `@dnd-kit` para el kanban, `cmdk` para ⌘K, SSE en vez de WebSocket.

Descartados con motivo: AG Grid y MUI X, porque agrupación, filtros de conjunto y panel de herramientas —justo lo que un CRM necesita a 50k filas— están en la versión Enterprise/Pro de pago. `react-data-grid` (el que usa Twenty) rinde mejor según benchmark público, pero sigue en beta.

**Cinco patrones a copiar:** vistas guardadas como pestañas con visibilidad privada/equipo/todos; panel lateral en vez de página completa; borradores con guardado explícito por lote; chips de filtro con anidado máximo 3 niveles; resolución de conflicto campo por campo (Actual / Mío / Original).

**Tres a evitar:** scroll infinito sin retorno de posición; guardar al `onBlur` sin indicador ni deshacer; filtrar y buscar en el cliente sobre el dataset completo.

---

## 6. Reportes y escala: los números medidos

| Escenario | Tiempo |
|---|---|
| Agregado "embudo por etapa", 200 k filas | 42 ms |
| Ídem, 1 M | 171 ms |
| Ídem, 10 M | 3,6 s |
| Ídem con índice **no** cubriente | **7,3-15,4 s** (el índice lo hace más lento que escanear) |
| Ídem con índice cubriente | 472 ms |
| Página profunda: `OFFSET` vs keyset | 36 ms vs 0,1 ms |
| Export 500 k filas: `OFFSET` vs keyset | ~40 min vs 1 s |

**SQLite, medido:** 10 M de negocios + 10 M de actividades + 4 índices + FTS5 = archivo de 2,82 GB; agregado mensual construido en 14,8 s y consultado en ~0 ms; FTS5 exacto 0,1 ms, prefijo 20 ms, con `bm25` 890 ms; ~5.150 escrituras/s en WAL con **un solo escritor**; export CSV de 500 k filas en 0,69 s con memoria constante.

**Traducción:** aguanta bien hasta **1-2 M de registros activos** con agregados. Se rompe con múltiples servidores, volumen de red o muchos escritores concurrentes.

**Los cinco reportes del día uno:** embudo por etapa · conversión etapa a etapa · tiempo de cierre en mediana y p90 · ventas por vendedor/región/canal · forecast del trimestre contra cuota, con el campo "vencido sin cerrar".

---

## 7. Los errores que cuestan caro

1. **Row-level como post-filtro en la aplicación** → lento y con fuga en el endpoint que alguien olvidó.
2. **Permisos de campo solo en la UI** → la API expone datos personales o sueldos: brecha notificable.
3. **Auditar todo en la misma base y para siempre** → disco lleno, y lo primero que se borra es la auditoría obligatoria.
4. **Sincronizar correo o webhook dentro del request** y encolar la venta → duplicados y pérdida silenciosa de ingresos.
5. **Automatizaciones sin tope de profundidad ni dueño por campo** → registros oscilando y pipeline inexplicable.
6. **`ALTER TABLE` en caliente** al crear un campo.
7. **Cursor sin desempate por `id`** → filas duplicadas o perdidas en la paginación.
8. **Índice no cubriente en un agregado** → 20× más lento que escanear.
9. **No guardar historia desde el día uno** → es el único error irreversible: el dato no se puede reconstruir.

---

## 8. Orden de implementación

| Fase | Qué | Por qué primero |
|:-:|---|---|
| 1 | Columnas de dinero y estado reales: `is_open`, `closed_by_id`, `amount_initial_cents`, `closed_on`, `deleted_at` | son la diferencia entre reportes correctos e incorrectos; agregarlas después exige backfill |
| 2 | Índices: `(owner_id, closed_at DESC, id DESC)`, `(parent_id, created_at DESC)`, parcial de abiertos | resuelven el 80 % de las pantallas |
| 3 | `stage_transition` + trigger | sin esto no hay historia, y no se puede recuperar después |
| 4 | Los 5 reportes, en vivo | con ≤200 k filas andan bien y validan definiciones con el negocio |
| 5 | `record_history` para 5-8 campos elegidos | el costo de escritura es real (8,2×) |
| 6 | Tabla de agregados + job nocturno | cuando el primer dashboard se ponga lento |
| 7 | Cierre mensual inmutable | antes del primer cierre con comisiones de por medio |
| 8 | Búsqueda FTS | cuando `LIKE '%texto%'` pase de 100 ms |
| 9 | Export asíncrono con streaming | cuando alguien pida "todo a Excel" |
| 10 | Particionado de historia y snapshots | desde ~10 M de filas en esas tablas |

---

## 9. Lo que queda abierto

1. **Lead como tabla o como estado de la persona** — depende de si hay captura masiva y varios vendedores (2.1).
2. **Historia de campos: qué 5-8 campos** se auditan desde el día uno.
3. **Multi-tenancy**: con una instalación por cliente, DDL real es la mejor opción; si el plan es revender un SaaS, schema por tenant. Es una decisión de negocio disfrazada de decisión técnica.
4. **Qué es "un dato crudo" en este negocio**: se decidió guardar payloads de formularios y correos, pero falta definir si se guardan los documentos (boletas, facturas) y por cuánto tiempo.
5. **El formato de la timeline**: si las actividades son polimórficas (una tabla) o una tabla por tipo. Los seis productos grandes usan polimórfica; los chicos, tablas separadas.
