# El recorrido completo de un dato en un CRM comercial

**Documento 05 — Investigación CRM**
De la primera tecla en un formulario web hasta la factura cobrada y el reporte de cierre.

---

## 0. Cómo leer este documento

### 0.1 Alcance

Se reconstruyen tres recorridos reales, con nombres de tabla y de campo plausibles, sobre el diseño que comparten HubSpot, Salesforce, Pipedrive, Attio, EspoCRM, Twenty y Odoo. No se describe un producto en particular: se describe **el patrón** y se marca en cada punto qué producto lo implementa de una forma concreta y verificable.

### 0.2 Convenciones de evidencia

| Marca | Significado |
|---|---|
| `[verificado + URL]` | Dicho en documentación pública o en el código fuente citado. |
| `[inferido]` | Deducido del comportamiento observable del producto o de la práctica estándar. No hay cita literal. |
| *Nombre de tabla* | Esquema canónico propuesto. Los productos reales usan otros nombres (p. ej. Salesforce `Opportunity`, Espo `Opportunity`, Odoo `crm_lead`). |

Todas las latencias son `[inferido]`: órdenes de magnitud de un stack típico (API + Postgres + cola), no mediciones.

### 0.3 Fuentes primarias usadas

| Fuente | Qué se tomó | URL |
|---|---|---|
| HubSpot KB — lifecycle stages | Lista de etapas, monotonía ("only move forward"), Lead Status como sub-etapas | https://knowledge.hubspot.com/records/use-lifecycle-stages |
| HubSpot KB — formularios | Email obligatorio, identificación por cookie, "always create contact for new email", lifecycle desde formulario no retrocede, aviso al owner solo si hay owner | https://knowledge.hubspot.com/forms/create-forms |
| HubSpot KB — workflows | Enrolamiento único por defecto, re-enrolamiento opt-in, delays, "Rotate record to owner", "Create quote", "Convert to invoice", "Send a webhook" | https://knowledge.hubspot.com/workflows/create-workflows · https://knowledge.hubspot.com/workflows/choose-your-workflow-actions |
| Salesforce — `convertLead()` | `LeadConvert`, fusión solo sobre campos vacíos, `overwriteLeadSource`, oportunidad por defecto, `ownerId`, `sendNotificationEmail`, máximo 100 | https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_calls_convertlead.htm |
| Salesforce — objeto `Lead` | `ConvertedAccountId/ContactId/OpportunityId`, `ConvertedDate`, `IsConverted`, lead convertido read-only | https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_lead.htm |
| Salesforce — `OpportunityHistory` | Se crea un registro al cambiar Amount/Probability/Stage/CloseDate; guarda `ForecastCategory` | https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_opportunityhistory.htm |
| Salesforce — `Task` | "Task and Event records are collectively referred to as activities" | https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_task.htm |
| Pipedrive — Activities | Tipos, vínculo automático deal→persona→organización, `marked as done time` vs `due date` | https://support.pipedrive.com/en/article/activities |
| Pipedrive — Rotting | Podredumbre por etapa según "last updated"; actividades futuras no la evitan | https://support.pipedrive.com/en/article/the-rotting-feature |
| Pipedrive — Leads Inbox | Lead separado del pipeline; conversión a deal conserva persona/organización/notas/archivos y borra el lead | https://support.pipedrive.com/en/article/leads-inbox |
| Attio — Status attribute | Único status predefinido: `stage` en el objeto deal; los statuses son columnas de kanban; list entries tienen atributos propios | https://docs.attio.com/rest-api/attribute-types/attribute-types-status |
| EspoCRM — Lead (código) | Estados `New/Assigned/In Process/Converted/Recycled/Dead`; campos `createdAccount/Contact/Opportunity` | https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Entities/Lead.php |
| EspoCRM — ConvertService | Copia campos homónimos y `convertFields`, marca `Converted`, re-parenta activities, enlaza registros creados | https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Tools/Lead/ConvertService.php |
| EspoCRM — Meeting | `parent` es `linkParent` con `entityList: [Account, Lead, Contact, Opportunity, Case]` | https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Resources/metadata/entityDefs/Meeting.json |
| EspoCRM — Web-to-Lead | POST sin autorización a `/api/v1/LeadCapture/{apiKey}`, payload configurable, target list/team/campaign, doble opt-in, reCAPTCHA v3, round-robin/least-busy vía workflow | https://docs.espocrm.com/administration/web-to-lead/ |
| EspoCRM — Quotes | Cotización desde oportunidad (copia ítems), sales order y factura desde cotización, bloqueo, expiración | https://docs.espocrm.com/user-guide/quotes/ |
| Odoo — `crm.lead` | `type` lead/opportunity en la misma tabla, `stage_id`, `probability`/`automated_probability`, `lost_reason_id`, `date_conversion`, `day_open`/`day_close`, `email_normalized`, `duplicate_lead_ids`, round-robin | https://github.com/odoo/odoo/blob/18.0/addons/crm/models/crm_lead.py |
| Odoo — `crm.stage` | `sequence`, `is_won`, `fold`, `requirements` | https://github.com/odoo/odoo/blob/18.0/addons/crm/models/crm_stage.py |
| Twenty — `timelineActivity` | `happensAt`, `properties`, `linkedRecord*`, `targetPerson/Company/Opportunity/Note/Task` | https://github.com/twentyhq/twenty/blob/main/packages/twenty-server/src/modules/timeline/standard-objects/timeline-activity.workspace-entity.ts |

---

## 1. El modelo mínimo: seis tablas y una cola

Antes de los recorridos, el esqueleto. Casi todo CRM comercial cabe en esto:

```
┌──────────────┐      ┌──────────────┐      ┌────────────────┐
│   accounts   │1────N│   contacts   │      │     users      │
│ (empresa)    │      │  (persona)   │      │ (vendedor)     │
└──────┬───────┘      └──────┬───────┘      └───────┬────────┘
       │                     │                      │ owner_id
       │1                    │1                     │
       │                     │                      │
       N                     N                      N
┌──────┴───────┐      ┌──────┴───────┐      ┌───────┴────────┐
│ opportunities│      │    leads     │      │   activities   │
│ (transacción)│      │ (pre-entidad)│      │ (verbo + tiempo)│
└──────┬───────┘      └──────────────┘      └───────┬────────┘
       │1                                            │N
       │                                             │
       N                                    parent_type + parent_id
┌──────┴───────┐                          (polimórfico: lead, contact,
│quote → order │                           account, opportunity, ticket)
│ → invoice    │
└──────────────┘
```

Y una tabla que casi nunca se dibuja en los diagramas pero sin la cual nada de lo anterior es confiable:

```
┌──────────────────────────────────────────────────────────────┐
│ outbox_events                                                │
│ id · tenant_id · aggregate_type · aggregate_id · event_type  │
│ payload(jsonb) · created_at · published_at · attempts · error│
└──────────────────────────────────────────────────────────────┘
```

**Regla de oro del diseño:** toda escritura de negocio y su evento salen en la **misma transacción** (patrón outbox). Si el evento se publica antes del commit, el consumidor lee un dato que todavía no existe. Si se publica después y el proceso muere, el evento se pierde para siempre. `[inferido]`

---

# RECORRIDO 1 — Entra un lead por formulario web

## 1.1 Diagrama de secuencia

```
Visitante    Navegador         Edge/WAF        API /forms        PostgreSQL        Worker         Vendedor
   |             |                 |                |                 |               |              |
   |--escribe--->|                 |                |                 |               |              |
   |             |--POST JSON----->|                |                 |               |              |
   |             |                 |--TLS,rate----->|                 |               |              |
   |             |                 |  limit,CORS    |--validar------->|               |              |
   |             |                 |                |--normalizar---->|               |              |
   |             |                 |                |--SELECT dedupe->|               |              |
   |             |                 |                |<--hit/miss------|               |              |
   |             |                 |                |--BEGIN--------->|               |              |
   |             |                 |                |--INSERT lead--->|               |              |
   |             |                 |                |--INSERT emails->|               |              |
   |             |                 |                |--INSERT form--->|               |              |
   |             |                 |                |--INSERT activity|               |              |
   |             |                 |                |--INSERT outbox->|               |              |
   |             |                 |                |--COMMIT-------->|               |              |
   |             |<--201 + thank you|                |                 |               |              |
   |<--pantalla--|                 |                |                 |--poll outbox->|              |
   |             |                 |                |                 |<--lead.created|              |
   |             |                 |                |<--UPDATE owner--|               |              |
   |             |                 |                |<--INSERT task---|               |              |
   |             |                 |                |                 |--email/Slack->|--notifica--->|
   |             |                 |                |                 |               |              |--abre ficha
```

## 1.2 Paso a paso, con tiempos

| t (ms) | Actor | Acción | Escribe en | Nota |
|---:|---|---|---|---|
| 0 | Visitante | Completa 5 campos y pulsa "Enviar" | — | Validación HTML5 en cliente (barata, no confiable) |
| 5 | Navegador | Serializa JSON + `form_id`, `page_url`, `utm_*`, `hubspotutk`/cookie propia, `submitted_at` del cliente | — | El `submitted_at` del cliente **no** se usa como verdad |
| 30 | Navegador | `POST /api/forms/f_7c1a/submissions` | — | `Content-Type: application/json` |
| 60 | Edge | TLS, rate limit por IP (p. ej. 10/min), CORS/allowlist de hosts, tamaño máximo | — | EspoCRM expone CORS con `leadCaptureAllowOrigin` `[verificado + https://docs.espocrm.com/administration/web-to-lead/]` |
| 75 | API | Valida esquema: tipos, longitudes, email RFC, teléfono, campos requeridos, coherencia país/teléfono | — | Rechazo → `422` con `field_errors[]` |
| 85 | API | Normaliza: email a minúsculas + `email_normalized`; teléfono a E.164 (`+56912345678`); nombre con `trim` y colapso de espacios | — | Odoo guarda `email_normalized` (índice trigram) y `phone_sanitized` `[verificado + crm_lead.py]` |
| 95 | API | Busca duplicado por `(tenant_id, email_normalized)` | `leads`, índice único parcial | |
| 105 | API | Si no hay match exacto: busca por teléfono, luego por dominio de email contra `accounts.domain` | `leads`, `accounts` | |
| 110 | API | `BEGIN` | — | |
| 115 | API | `INSERT INTO leads (...)` | `leads` | `status='new'`, `source='web_form'` |
| 125 | API | `INSERT INTO lead_emails` / `lead_phones` | tablas hijas | Soporta N emails/teléfonos por lead |
| 130 | API | `INSERT INTO form_submissions (...)` con el payload crudo | `form_submissions` | **Nunca** descartar el crudo: es la única prueba de consentimiento |
| 140 | API | `INSERT INTO activities (type='form_submitted', ...)` | `activities` | Nace la línea de tiempo |
| 145 | API | `INSERT INTO outbox_events ('lead.created.v1', payload)` | `outbox_events` | Misma transacción |
| 150 | API | `COMMIT` | — | |
| 155 | API | `201 Created` + `{ "ok": true, "redirect": "/gracias" }` | — | |
| 160 | Navegador | Pinta thank-you / redirige | — | |
| 400 | Worker | Consume `lead.created.v1` → aplica regla de asignación | `leads.owner_id`, `assignment_log` | |
| 450 | Worker | Crea tarea de primer contacto (vencimiento +5 min laborables) | `tasks` (o `activities` tipo task) | |
| 600 | Worker | Envía notificación (email interno, Slack, push) | `notifications`, `email_queue` | |
| 900 | Worker | Enriquece (dominio → empresa, país, tamaño) | `accounts`, `leads` | Opcional y **post-commit** |
| 1200 | Worker | Matricula en secuencia de nurturing si no hay respuesta | `sequence_enrollments` | Depende de workflow |

**p50 percibido por el visitante ≈ 300–400 ms. p95 ≈ 900 ms.** Todo lo que el visitante no necesita ver ocurre después del `201`. `[inferido]`

> **Decisión de arquitectura clave:** el formulario responde `201` cuando el lead está **persistido**, no cuando está asignado ni notificado. Si se espera la asignación para responder, un timeout del motor de reglas deja al visitante mirando un spinner y, peor, lo invita a reenviar.

## 1.3 Filas exactas que se escriben

### `leads`

| Campo | Valor de ejemplo | Por qué |
|---|---|---|
| `id` | `led_01JQ8Z3K9P` | ULID/UUID, ordenable por tiempo |
| `tenant_id` | `t_0042` | Multi-tenant desde el día uno |
| `first_name` / `last_name` | `Camila` / `Rojas` | |
| `company_name_raw` | `Viña Los Aromos SpA` | Texto tal como lo escribió el visitante |
| `account_id` | `NULL` → luego `acc_...` | Se resuelve al enriquecer o al convertir |
| `email_normalized` | `camila.rojas@vinaros.cl` | Clave de deduplicación |
| `phone_e164` | `+56912345678` | |
| `source` | `web_form` | |
| `source_detail` | `f_7c1a` | ID del formulario |
| `utm_source` / `utm_medium` / `utm_campaign` | `google` / `cpc` / `mar_2026` | Primer toque |
| `status` | `new` | Estado del **lead**, no de la oportunidad |
| `owner_id` | `NULL` | Se llena en t+400 ms |
| `score` | `0` | Se calcula después |
| `consent_marketing` | `true` | Con `consent_text_hash` y `consent_at` |
| `created_at` | `2026-11-16T14:02:11.412Z` | Ahora del **servidor** |
| `dedupe_of_lead_id` | `NULL` | Si se decide no crear duplicado |

### `form_submissions`

| Campo | Valor |
|---|---|
| `id` | `fsub_01JQ8Z3K9Q` |
| `form_id` | `f_7c1a` |
| `lead_id` | `led_01JQ8Z3K9P` |
| `payload` | `{"nombre":"Camila","email":"...","mensaje":"..."}` (jsonb, inmutable) |
| `ip_hash` | `sha256(ip + salt_diario)` |
| `user_agent` | `Mozilla/5.0 ...` |
| `referrer` | `https://google.com/...` |
| `idempotency_key` | `sha256(form_id + email + minuto)` |
| `created_at` | `2026-11-16T14:02:11.408Z` |

### `activities` (primer registro de la línea de tiempo)

| Campo | Valor |
|---|---|
| `id` | `act_01JQ8Z3K9R` |
| `tenant_id` | `t_0042` |
| `type` | `form_submitted` |
| `parent_type` | `lead` |
| `parent_id` | `led_01JQ8Z3K9P` |
| `actor_type` / `actor_id` | `external` / `NULL` |
| `occurred_at` | `2026-11-16T14:02:11.408Z` |
| `subject` | `Formulario "Cotizar plan"` |
| `body` | `null` |
| `metadata` | `{"form_id":"f_7c1a","fields":5}` (jsonb) |
| `visibility` | `internal` |

### `outbox_events`

```json
{
  "event_type": "lead.created.v1",
  "aggregate_type": "lead",
  "aggregate_id": "led_01JQ8Z3K9P",
  "payload": {
    "lead_id": "led_01JQ8Z3K9P",
    "tenant_id": "t_0042",
    "source": "web_form",
    "form_id": "f_7c1a",
    "email": "camila.rojas@vinaros.cl",
    "utm": {"source":"google","medium":"cpc","campaign":"mar_2026"},
    "created_at": "2026-11-16T14:02:11.412Z"
  }
}
```

## 1.4 Validaciones, en el orden exacto

El orden importa: cada capa asume que la anterior ya pasó, y así se evita gastar una consulta a la base en un email malformado.

1. **Cliente (HTML5):** `required`, `type=email`. Se puede saltar. No cuenta.
2. **Borde:** tamaño máximo del body, rate limit por IP y por `form_id`, `Content-Type` correcto, host permitido.
3. **Esquema:** campos desconocidos se guardan igual en `payload` pero se ignoran para el lead (protege contra inyección de propiedades).
4. **Semántica:** email con formato válido y dominio con MX (validación asíncrona, no bloquea); teléfono parseable a E.164 con el país del formulario; longitud de nombre 1–120; `company_name` ≤ 200.
5. **Anti-abuso:** honeypot invisible, reCAPTCHA v3 con umbral, lista de dominios desechables, detección de texto repetido.
6. **Consentimiento:** si el checkbox es obligatorio y viene vacío → `422`. Se guarda `consent_text_hash` para poder probar **qué texto** aceptó.
7. **Deduplicación:** exacta por email normalizado; luego por teléfono; luego fuzzy por dominio + nombre de empresa.

> HubSpot exige email por defecto para crear contacto y valida el formato antes de permitir el envío `[verificado + https://knowledge.hubspot.com/forms/create-forms]`. EspoCRM permite configurar exactamente qué campos viajan en el payload (`Payload Fields`) `[verificado + https://docs.espocrm.com/administration/web-to-lead/]`.

## 1.5 Cómo se detecta que ya existe

Cuatro capas, de la más barata a la más cara:

| Capa | Consulta | Coste | Resultado |
|---|---|---|---|
| 1. Cookie / `localStorage` | `visitor_id` ya visto | 0 ms | Prellena el formulario; **no** decide identidad |
| 2. Match exacto | `WHERE tenant_id=? AND email_normalized=?` | 1–5 ms (índice) | Actualiza el lead existente |
| 3. Match por teléfono | `WHERE phone_e164=?` | 2–8 ms | Candidato a merge manual |
| 4. Fuzzy empresa | trigram sobre `company_name` + dominio de email | 10–40 ms | Cola de revisión, nunca merge automático |

**Comportamiento ante duplicado:**

- **Mismo email, mismo formulario, < 60 s:** se trata como reintento de red. No se escribe nada nuevo; se devuelve `200` con el `lead_id` original (idempotencia).
- **Mismo email, formulario distinto:** se **actualiza** el lead, se agrega una fila nueva en `form_submissions` y una nueva `activity`. No se sobrescriben campos ya llenos salvo los declarados "autoritativos" por el formulario (p. ej. `mensaje`).
- **Mismo email pero el lead está `converted`:** el formulario **no** crea un lead nuevo; actualiza el contacto y, si corresponde, crea una `activity` sobre el contacto. Esto es exactamente lo que evita el clásico "el cliente ya es cliente y le volvemos a vender como prospecto".
- **Email distinto, misma persona en el mismo navegador:** es la trampa. HubSpot avisa que, con la opción por defecto, la identificación por cookie puede **sobrescribir** contactos si el mismo dispositivo envía el formulario varias veces, y que por eso existe "Always create contact for new email address" `[verificado + https://knowledge.hubspot.com/forms/create-forms]`. En el esquema canónico se resuelve guardando `visitor_id` en `form_submissions` y **no** usando la cookie como identidad.

Odoo expone el mismo problema desde el otro lado: calcula `duplicate_lead_ids` comparando email normalizado, teléfono sanitizado y nombre de empresa, y lo muestra como advertencia en la ficha, sin fusionar `[verificado + https://github.com/odoo/odoo/blob/18.0/addons/crm/models/crm_lead.py]`.

## 1.6 Asignación a un vendedor

Cinco estrategias, en orden de complejidad:

| Estrategia | Cómo se implementa | Cuándo se usa |
|---|---|---|
| **Cola compartida** | `owner_id = NULL`, `queue_id = q_inbound` | Equipos pequeños; el que llega primero lo toma |
| **Round-robin** | `assignment_log` guarda el último asignado; siguiente en la lista | Reparto justo y predecible |
| **Least busy** | `COUNT(open_leads) GROUP BY owner` + capacidad máxima | Equipos con carga desigual |
| **Por territorio / segmento** | Reglas sobre país, industria, tamaño, idioma | Equipos enterprise |
| **Por score** | Score > 80 → senior; resto → junior o secuencia automática | Modelos de madurez altos |

EspoCRM ofrece round-robin y least-busy vía Workflow/BPM, con condición por campaña para distinguir el origen `[verificado + https://docs.espocrm.com/administration/web-to-lead/]`. Odoo hace round-robin en `_handle_salesmen_assignment`, documentado en el propio docstring: 4 vendedores para 6 leads → L1-S1, L2-S2, L3-S3, L4-S4, L5-S1, L6-S2 `[verificado + https://github.com/odoo/odoo/blob/18.0/addons/crm/models/crm_lead.py]`. HubSpot lo resuelve con la acción **Rotate record to owner**, que reparte "equally within a selected team or between specified users" `[verificado + https://knowledge.hubspot.com/workflows/choose-your-workflow-actions]`.

### Escrituras de la asignación

`leads`: `owner_id`, `assigned_at`, `sla_due_at` (= `assigned_at` + SLA de primera respuesta).

`assignment_log`:

| Campo | Valor |
|---|---|
| `lead_id` | `led_01JQ8Z3K9P` |
| `from_owner_id` | `NULL` |
| `to_owner_id` | `usr_018` |
| `rule` | `round_robin:team_smb` |
| `reason` | `new_lead` |
| `created_at` | `2026-11-16T14:02:11.812Z` |

`activities`: una fila `type='assignment'` con `metadata {"rule":"round_robin","from":null,"to":"usr_018"}`. Esta fila es oro para auditar equidad de reparto.

## 1.7 Eventos publicados y quién escucha

| Evento | Publicado por | Consumidores | Efecto |
|---|---|---|---|
| `lead.created.v1` | API de formularios | Asignador, Enricher, Scoring, Notificador, Analytics | Crea tarea, calcula score, avisa |
| `lead.assigned.v1` | Asignador | Notificador, SLA timer, Forecaster | Arranca reloj de SLA, avisa al vendedor |
| `lead.duplicate_suspected.v1` | Dedupe | Cola de revisión de datos | Tarea de merge manual |
| `lead.enriched.v1` | Enricher | Scoring, Segmentación | Recalcula score con tamaño de empresa |
| `activity.created.v1` | Cualquier escritura de actividad | Timeline projector, Webhooks, Búsqueda | Indexa en timeline y en search |

**Quién escucha, en la práctica:** el mismo proceso API no hace nada de esto. Lo hacen workers idempotentes que leen de `outbox_events` con `SELECT ... FOR UPDATE SKIP LOCKED`, y cada consumidor guarda su `consumer_offset` o una tabla `processed_events(consumer, event_id)` para no aplicar dos veces.

## 1.8 Qué ve el usuario en pantalla

**Visitante**

| Momento | Pantalla |
|---|---|
| t+0 | Formulario embebido, 5 campos, botón "Enviar" |
| t+60 (submit) | Botón deshabilitado + spinner; se evita doble submit por JS (`formSubmitted = true`) |
| t+160 | Thank-you: "Te contactamos en menos de 1 hora hábil" |
| Error 422 | Campos en rojo con el mensaje exacto devuelto por el servidor |

**Vendedor (t+600 ms a t+2 s)**

| Superficie | Qué aparece |
|---|---|
| Notificación in-app | "Nuevo lead: Camila Rojas — Viña Los Aromos — formulario Cotizar plan" |
| Email interno | Mismo texto + link profundo a la ficha + botón "Llamar" |
| Slack | Tarjeta con nombre, empresa, UTM y botón "Tomar" |
| Bandeja "Mis leads" | Fila nueva arriba, con badge `Nuevo` y contador de SLA en verde |
| Ficha del lead | Cabecera con nombre/empresa/owner, panel "Línea de tiempo" con el evento *Formulario enviado*, panel derecho con tarea "Primer contacto" venciendo en 5 min |

**Jefe de ventas**

| Superficie | Qué aparece |
|---|---|
| Dashboard | +1 en "Leads hoy", embudo por `status`, gráfico de origen |
| Cola sin asignar | 0 (o el lead con badge rojo si la asignación falló) |
| Auditoría | `assignment_log` muestra a quién y por qué regla |

## 1.9 Los tres puntos donde se rompe en la vida real

### Ruptura 1 — El doble submit / reintento de red crea duplicados

**Cómo se rompe:** el visitante ve el spinner 4 segundos, cree que falló, vuelve a pulsar. O el móvil pierde la conexión justo después de que el servidor hizo `COMMIT` y el navegador reintenta. Resultado: dos leads idénticos, dos vendedores llamando a la misma persona, y el reporte de "leads por campaña" inflado 8%.

**Cómo lo mitiga el producto:**
- Índice único parcial `(tenant_id, email_normalized) WHERE status <> 'merged'`.
- `Idempotency-Key` en el POST + tabla `form_submissions(idempotency_key UNIQUE)`.
- Si el lead existe y fue creado hace < 60 s por el mismo formulario → `200` con el mismo `lead_id`.
- Botón deshabilitado en cliente + `beforeunload` guard.
- Panel de duplicados con merge auditable (Pipedrive tiene "Merge duplicates" explícito en leads `[verificado + https://support.pipedrive.com/en/article/leads-inbox]`).

### Ruptura 2 — El formulario público se convierte en buzón de spam

**Cómo se rompe:** un bot encuentra el endpoint y envía 40.000 submissions con emails aleatorios. La cola de asignación se satura, los vendedores dejan de mirar la bandeja, y cuando llega el lead real ya nadie confía en el canal.

**Cómo lo mitiga el producto:**
- reCAPTCHA v3 con umbral (EspoCRM lo soporta de fábrica) `[verificado + https://docs.espocrm.com/administration/web-to-lead/]`.
- Honeypot + `time_to_submit` mínimo (un bot llena en 200 ms; un humano no).
- Rate limit por IP y por `form_id`; bloqueo temporal tras N rechazos.
- Lista de dominios desechables y detección de patrones (`asdf@asdf.com`).
- Cuarentena: los sospechosos **no** entran a la cola del vendedor; entran a `leads(status='quarantined')` con revisión.
- CORS restringido a los hosts del sitio (`leadCaptureAllowOrigin`).

### Ruptura 3 — El lead cae en la cola de alguien que está de vacaciones

**Cómo se rompe:** el round-robin asigna al vendedor 7, que está de licencia hasta el lunes. El lead queda con `owner_id` no nulo, por lo tanto **no** aparece como "sin asignar", nadie lo ve, y el SLA vence en silencio. El visitante ya compró en otro lado.

**Cómo lo mitiga el producto:**
- **Owner ≠ disponible.** Tabla `user_availability(user_id, status, starts_at, ends_at)` alimentada por ausencias/calendario.
- El asignador **excluye** owners no disponibles y cae al **fallback**: cola del equipo (`queue_id`) o `fallback_owner_id` configurado.
- El SLA se cuenta desde `assigned_at` **efectivo**, no desde `created_at`; mientras está en cola, el reloj es de *equipo*, no de persona.
- Reasignación automática: job diario que detecta `owner` con ausencia activa y `open_leads > 0` → reasigna y emite `lead.reassigned.v1`.
- Alerta al jefe de ventas si un lead supera 2× el SLA sin primera actividad.
- HubSpot tiene un detalle que ilustra el problema: la notificación de envío se manda **al owner del contacto**, y si el contacto **no tiene owner, no se envía ninguna notificación** `[verificado + https://knowledge.hubspot.com/forms/create-forms]`. Un lead sin dueño es un lead invisible.

---

# RECORRIDO 2 — El vendedor trabaja el lead hasta cerrarlo

## 2.1 Diagrama de secuencia: registrar una llamada

```
Vendedor     UI (ficha)        API /activities      PostgreSQL        Bus/Outbox      Timeline      Otros
   |             |                   |                   |                 |              |            |
   |--"Llamar"--->|                  |                   |                 |              |            |
   |             |--POST activity--->|                   |                 |              |            |
   |             |  {type:call,...}  |--validar--------->|                 |              |            |
   |             |                   |--BEGIN----------->|                 |              |            |
   |             |                   |--INSERT activity->|                 |              |            |
   |             |                   |--INSERT participants                 |              |            |
   |             |                   |--UPDATE lead------|                 |              |            |
   |             |                   |  last_activity_at |                 |              |            |
   |             |                   |--INSERT outbox--->|                 |              |            |
   |             |                   |--COMMIT---------->|                 |              |            |
   |             |<--201 activity----|                   |                 |              |            |
   |<--aparece en timeline------------|                   |--publish------->|              |            |
   |             |                   |                   |                 |--project---->|            |
   |             |                   |                   |                 |              |--webhook-->|
   |             |                   |                   |                 |              |--score---->|
```

## 2.2 Las tres actividades básicas y sus filas

Una sola tabla `activities` con `type` como discriminador. Los campos específicos van en `metadata` (jsonb) para no llenar la tabla de columnas nulas.

### Llamada

| Campo | Valor |
|---|---|
| `type` | `call` |
| `parent_type` / `parent_id` | `lead` / `led_01JQ8Z3K9P` |
| `actor_id` | `usr_018` |
| `occurred_at` | `2026-11-16T15:10:00Z` |
| `duration_seconds` | `412` |
| `outcome` | `connected` |
| `direction` | `outbound` |
| `subject` | `Llamada de calificación` |
| `body` | `Pidió cotización formal para 3 sucursales.` |
| `metadata` | `{"recording_url":"...","disposition":"qualified","next_step":"enviar cotización"}` |

### Correo

| Campo | Valor |
|---|---|
| `type` | `email` |
| `direction` | `outbound` |
| `subject` | `Cotización plan multisede` |
| `body_html` / `body_text` | … |
| `metadata.message_id` | `<CAF=9x...@mail.empresa.cl>` |
| `metadata.thread_id` | `th_88a1` |
| `metadata.opens` | `3` |
| `metadata.clicks` | `1` |
| `sent_at` / `delivered_at` / `first_opened_at` | timestamps separados |

El `Message-ID` es la clave de idempotencia del correo. Sin él, cada sincronización IMAP duplica el hilo.

### Reunión

| Campo | Valor |
|---|---|
| `type` | `meeting` |
| `starts_at` / `ends_at` | `2026-11-18T10:00:00Z` / `11:00:00Z` |
| `status` | `planned` → `held` / `not_held` |
| `location` | `Google Meet` |
| `metadata.attendees` | `[{"email":"...","status":"accepted"}]` |
| `metadata.calendar_event_id` | `gcal_9f2...` |

EspoCRM modela la reunión con `status` `Planned / Held / Not Held`, `dateStart`, `dateEnd`, `duration` calculada, `reminders` y `parent` polimórfico hacia `Account, Lead, Contact, Opportunity, Case` `[verificado + https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Resources/metadata/entityDefs/Meeting.json]`. Pipedrive usa el mismo patrón: tipos `call, meeting, task, deadline, email, lunch` + tipos personalizados, y distingue **`due date`** (cuándo estaba agendada, editable retroactivamente) de **`marked as done time`** (cuándo se completó de verdad) `[verificado + https://support.pipedrive.com/en/article/activities]`.

Esa distinción de dos fechas es la que permite responder "¿cumplimos lo prometido?" sin mentir: si solo guardas una fecha, el vendedor la mueve y el atraso desaparece del reporte.

## 2.3 Cómo se arma la línea de tiempo

**Regla:** la línea de tiempo es una **proyección**, no la fuente de verdad. Se materializa para leer rápido, se puede reconstruir.

```
activities (fuente de verdad, append-only)
        │
        ├── activity_participants (quién participó: user, contact, external)
        │
        └──▶ timeline_entries (proyección desnormalizada, una fila por cosa que el usuario ve)
                 id · tenant_id · parent_type · parent_id · happened_at
                 icon · title · subtitle · actor_name · deep_link · rank
```

**Proyección de una fila:**

```
[16 nov 14:02]  📝 Formulario enviado        Sistema        → ver envío
[16 nov 14:02]  👤 Lead asignado             Regla RR       → usr_018
[16 nov 14:07]  ✅ Tarea completada          Camila (vend.) → "Primer contacto"
[16 nov 15:10]  📞 Llamada · 6m52s           Camila (vend.) → conectada · calificado
[16 nov 15:12]  ✉️  Correo enviado            Camila (vend.) → "Cotización plan multisede"
[16 nov 16:40]  👁  Correo abierto ×3         Sistema
[18 nov 10:00]  📅 Reunión realizada · 58m   Camila (vend.) → 2 asistentes
[18 nov 11:05]  🔁 Etapa: Calificado → Propuesta   Camila → motivo: "pidió cotización"
```

**Cómo se construye sin N+1:** el proyector escucha `activity.created.v1`, `lead.updated.v1`, `opportunity.stage_changed.v1` y hace `INSERT ... ON CONFLICT DO NOTHING` en `timeline_entries`. La UI pide `GET /timeline?parent_type=lead&parent_id=...&cursor=...` con índice `(tenant_id, parent_type, parent_id, happened_at DESC)`.

**Detalle fino:** las actividades se muestran en `occurred_at`, no en `created_at`. Una llamada registrada el lunes sobre algo del viernes debe aparecer el viernes. Si el producto ordena por fecha de creación, la narrativa del cliente se vuelve incoherente y el vendedor deja de confiar en la ficha.

## 2.4 Cambio de etapa: qué se dispara

```
Vendedor        API /opportunities      PostgreSQL            Outbox            Workers
   |                  |                     |                    |                 |
   |--mueve tarjeta-->|                     |                    |                 |
   |                  |--validar transición |                    |                 |
   |                  |  (guardrails)       |                    |                 |
   |                  |--BEGIN------------->|                    |                 |
   |                  |--UPDATE opp-------->|                    |                 |
   |                  |  stage, probability |                    |                 |
   |                  |  stage_entered_at   |                    |                 |
   |                  |--INSERT stage_hist->|                    |                 |
   |                  |--INSERT outbox----->|                    |                 |
   |                  |--COMMIT------------>|                    |                 |
   |<--200 + toast----|                     |--publish---------->|                 |
   |                  |                     |                    |--quote draft--->|
   |                  |                     |                    |--task follow-up>|
   |                  |                     |                    |--notify manager>|
   |                  |                     |                    |--forecast recalc|
```

### Escrituras

`opportunities`:

| Campo | Antes | Después |
|---|---|---|
| `stage` | `qualified` | `proposal` |
| `probability` | `40` | `65` |
| `forecast_category` | `pipeline` | `best_case` |
| `stage_entered_at` | `2026-11-14T09:12Z` | `2026-11-18T11:05Z` |
| `amount` | `4800000` | `4800000` |
| `close_date` | `2026-12-15` | `2026-12-05` |
| `updated_at` | … | `2026-11-18T11:05:03Z` |

`stage_history` (append-only, **nunca** se actualiza):

| `id` | `opportunity_id` | `from_stage` | `to_stage` | `changed_by` | `reason_code` | `amount_at_change` | `probability_at_change` | `changed_at` |
|---|---|---|---|---|---|---|---|---|
| `sh_881` | `opp_55` | `qualified` | `proposal` | `usr_018` | `quote_requested` | `4800000` | `65` | `2026-11-18T11:05:03Z` |

`activities`: fila `type='stage_changed'` con `metadata {"from":"qualified","to":"proposal","reason":"quote_requested"}`.

### Automatizaciones típicas al entrar a `proposal`

| Acción | Tabla afectada | Por qué |
|---|---|---|
| Crear borrador de cotización desde el catálogo | `quotes`, `quote_lines` | Ahorra 15 min y evita precios inventados |
| Crear tarea "Confirmar recepción de cotización" +3 días | `tasks` | Cadencia comercial |
| Notificar al gerente si `amount > 5.000.000` | `notifications` | Control de descuentos |
| Recálculo de forecast | `forecast_snapshots` | El pipeline cambió de categoría |
| Enviar plantilla de correo con la cotización adjunta | `email_queue` | Consistencia de marca |
| Actualizar `lifecycle_stage` del contacto si corresponde | `contacts` | **Solo si hay deal asociado** |

Salesforce guarda esto de forma nativa: `OpportunityHistory` crea **un registro nuevo** cada vez que cambia `Amount`, `Probability`, `Stage` o `CloseDate`, y guarda los valores vigentes de todos esos campos junto con `ForecastCategory` `[verificado + https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_opportunityhistory.htm]`. Es exactamente el patrón `stage_history` con snapshot: sin el snapshot no puedes reconstruir el pipeline de hace tres meses, porque el `amount` actual no es el `amount` de entonces.

## 2.5 Cuando el lead dice "no por ahora"

Este es el punto donde se separa un CRM de una libreta de contactos.

### Taxonomía de "no" (nunca un campo de texto libre como única opción)

| Categoría | `reason_code` | ¿Reciclable? | Vuelve en |
|---|---|---|---|
| Sin presupuesto ahora | `no_budget_timing` | Sí | 90–180 días |
| Sin urgencia | `no_urgency` | Sí | 180 días |
| Ya tiene proveedor | `has_vendor` | Sí | 270 días (renovación) |
| Decisor equivocado | `wrong_person` | No (se redirige) | — |
| Fuera de ICP | `out_of_icp` | No | Nunca |
| Competencia ganó | `lost_to_competitor` | Sí, con inteligencia | 365 días |
| Proyecto cancelado | `project_cancelled` | Sí | 180 días |
| Precio | `price` | Sí, si cambia el packaging | 120 días |
| No contesta / fantasma | `no_response` | Sí, a nurturing | 30/60/90 |
| Duplicado / basura | `invalid` | No | — |

### A dónde va y cuándo vuelve

```
                    ┌──────────────┐
                    │  nurturing   │◀── no_budget_timing, no_urgency,
                    │  (secuencia) │    has_vendor, price
                    └──────┬───────┘
                           │ 90–180 días o señal de intención
                           ▼
   lead ──▶ ┌───────────────────────┐
            │ recycled (fecha futura)│──▶ vuelve a la cola como "nuevo"
            └───────────────────────┘     con owner original o round-robin
                           │
                           │ tras N intentos (p. ej. 3 ciclos)
                           ▼
                    ┌──────────────┐
                    │    dead      │  (espacio de nombres separado,
                    └──────────────┘   sigue siendo consultable)
```

### Escrituras del "no"

`opportunities` (si ya había oportunidad):

| Campo | Valor |
|---|---|
| `stage` | `closed_lost` |
| `probability` | `0` |
| `lost_reason_code` | `no_budget_timing` |
| `lost_reason_note` | `"Congelaron capex hasta Q2 2027"` |
| `lost_to_competitor_id` | `NULL` |
| `closed_at` | `2026-11-18T17:20Z` |
| `recycle_at` | `2027-03-15` |
| `recycle_owner_id` | `usr_018` |

`activities`: `type='stage_changed'` con `to='closed_lost'` + `type='note'` con la razón en texto del vendedor.

`tasks`: una tarea futura con `due_at = recycle_at` y `type='recycle_followup'`. **Esta fila es la que hace que el lead vuelva.** Sin tarea, "reciclado" es una etiqueta decorativa.

Odoo implementa el "no" con semántica explícita: `action_set_lost()` **archiva** el registro (`active = False`), y `lost_reason_id` es un campo de primera clase; `action_set_won()` desarchiva y fuerza `probability = 100` en la etapa marcada `is_won` `[verificado + https://github.com/odoo/odoo/blob/18.0/addons/crm/models/crm_lead.py]`. Es decir: perdido = fuera del pipeline activo, pero recuperable.

Pipedrive mantiene el lead en un **Leads Inbox separado del pipeline** hasta que está calificado, y al convertir a deal el lead se borra conservando persona, organización, notas y archivos `[verificado + https://support.pipedrive.com/en/article/leads-inbox]`. Es el mismo concepto con otra topología: el embudo no se ensucia con lo no calificado.

EspoCRM tiene los estados `Recycled` y `Dead` como estados nativos del lead, separados de `Converted` `[verificado + https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Entities/Lead.php]`.

## 2.6 Conversión: lead → contacto + empresa + oportunidad

### Diagrama

```
                 ┌─────────────────────────────────────────────┐
                 │              leads (led_01JQ...)             │
                 │  status: in_process → converted              │
                 │  converted_at, converted_account_id          │
                 │  converted_contact_id, converted_opp_id      │
                 └───────┬──────────────┬──────────────┬────────┘
                         │              │              │
              copia campos      copia campos     crea desde cero
                         │              │              │
                         ▼              ▼              ▼
                  ┌───────────┐  ┌───────────┐  ┌──────────────┐
                  │ accounts  │  │ contacts  │  │opportunities │
                  │ acc_77    │  │ con_91    │  │ opp_55       │
                  └───────────┘  └───────────┘  └──────────────┘
                         ▲              ▲              ▲
                         └──────────────┴──────────────┘
                              activities se re-parentan
                         (parent_type: lead → opportunity/account)
```

### Qué se copia

| Origen (lead) | Destino | Regla |
|---|---|---|
| `first_name`, `last_name` | `contacts.first_name/last_name` | Directo |
| `email_normalized` | `contacts.email` (principal) | Primer email; los demás a `contact_emails` |
| `phone_e164` | `contacts.phone` | Directo |
| `title` | `contacts.job_title` | Directo |
| `company_name_raw` | `accounts.name` | Directo; si existe empresa por dominio, se asocia en vez de crear |
| `industry`, `employees`, `country` | `accounts.*` | Directo |
| `address_*` | `accounts.billing_*` | Directo |
| `source`, `utm_*` | `opportunities.lead_source`, `contacts.first_touch_source` | **Primer toque** al contacto, **último toque** a la oportunidad |
| `description` | `opportunities.description` + nota | Se duplica a propósito: la nota es narrativa, el campo es dato |
| `owner_id` | `accounts/contacts/opportunities.owner_id` | Salvo override explícito |
| `opportunity_amount` | `opportunities.amount` | Solo si el formulario de conversión lo pide |
| Campos custom | Destino custom | **Solo si están mapeados** |

EspoCRM copia automáticamente los campos que existen en ambos entity types **con el mismo nombre y el mismo tipo**, y permite mapeos extra vía `convertFields`; los archivos/imágenes se copian como adjuntos nuevos `[verificado + https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Tools/Lead/ConvertService.php]`.

Salesforce mapea los campos estándar automáticamente, y para los custom el administrador define el mapeo. Al fusionar con registros existentes **solo se rellenan campos vacíos**; los datos e IDs existentes no se sobrescriben. La única excepción es `overwriteLeadSource = true`, que pisa `LeadSource` en el contacto destino `[verificado + https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_calls_convertlead.htm]`.

### Qué se pierde

| Se pierde | Por qué | Mitigación |
|---|---|---|
| Campos del lead **sin mapeo** | Se quedan en el lead, que pasa a read-only | Mapear antes de convertir; auditar `convertFields` |
| `status` del lead (`in_process`, `contacted`…) | No existe en contacto/oportunidad | Guardar en `metadata.lead_status_at_conversion` |
| Historial de scoring del lead | Vive solo en `leads.score` | Copiar `score_at_conversion` a la oportunidad |
| Asignaciones y reasignaciones previas | Solo viven en `assignment_log` | Mantener el log apuntando al lead y enlazarlo |
| Notas que eran del lead y no se re-parentan | Depende del producto | Re-parentar **todas** las actividades, no solo llamadas/reuniones |
| Consentimiento granular | Vive en `form_submissions` | Enlazar `form_submissions` al contacto tras convertir |

Salesforce es explícito: **después de convertir, el lead es read-only**; solo usuarios con el permiso "View and Edit Converted Leads" pueden modificarlo, y no se puede convertir un lead cambiando `Status` por API `[verificado + https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_lead.htm]`. Esto es una decisión de integridad, no una limitación: el lead se convierte en un **registro de auditoría inmutable** de cómo entró ese negocio.

### Qué se mantiene enlazado

| Vínculo | Cómo se guarda |
|---|---|
| Lead → cuenta creada | `leads.converted_account_id` (Espo: `createdAccount`; Salesforce: `ConvertedAccountId`) |
| Lead → contacto creado | `leads.converted_contact_id` |
| Lead → oportunidad creada | `leads.converted_opportunity_id` |
| Actividades | `activities.parent_type` cambia de `lead` a `opportunity` (o `account` si no hay oportunidad) |
| Participantes | Se agrega el contacto como participante de llamadas y reuniones |
| Documentos | Se relacionan con cuenta y oportunidad (muchos-a-muchos) |
| Formularios originales | Siguen apuntando al lead; el lead apunta al contacto |
| Campaña / UTM | Copiados, no movidos: el lead conserva el original |

EspoCRM lo hace literal en `processLinks()`: recorre reuniones, llamadas, correos y documentos del lead, agrega el contacto como participante, y re-parenta cada actividad a la oportunidad si existe, o a la cuenta si no `[verificado + https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Tools/Lead/ConvertService.php]`.

Pipedrive hace lo análogo al convertir lead → deal: conserva persona, organización, notas y archivos, y elimina el lead del inbox `[verificado + https://support.pipedrive.com/en/article/leads-inbox]`.

Odoo es el caso más elegante: **no crea una entidad nueva**, cambia `type` de `lead` a `opportunity` en la misma tabla `crm_lead` y sella `date_conversion` `[verificado + https://github.com/odoo/odoo/blob/18.0/addons/crm/models/crm_lead.py]`. Consecuencia: no hay nada que copiar ni que perder; la "conversión" es un cambio de estado. El costo es que la tabla mezcla dos ciclos de vida y hay que filtrar por `type` en cada consulta.

### Escrituras de la conversión (transaccional)

```
BEGIN;
  INSERT INTO accounts (...) VALUES (...) RETURNING id;
  INSERT INTO contacts (...) VALUES (...) RETURNING id;
  INSERT INTO opportunities (...) VALUES (...) RETURNING id;
  UPDATE leads SET status='converted', converted_at=now(),
         converted_account_id=..., converted_contact_id=..., converted_opportunity_id=...
   WHERE id=? AND status <> 'converted';        -- guard idempotente
  UPDATE activities SET parent_type='opportunity', parent_id=?
   WHERE parent_type='lead' AND parent_id=?;
  INSERT INTO contact_opportunity_roles (contact_id, opportunity_id, role)
    VALUES (?, ?, 'primary');
  INSERT INTO outbox_events (event_type, aggregate_id, payload)
    VALUES ('lead.converted.v1', ?, ?);
COMMIT;
```

El `WHERE status <> 'converted'` es la defensa contra el doble clic en "Convertir". Si `UPDATE` afecta 0 filas, se hace `ROLLBACK` y se devuelve el resultado existente.

## 2.7 Eventos del recorrido 2

| Evento | Disparado por | Consumidores | Efecto |
|---|---|---|---|
| `activity.created.v1` | Cualquier actividad | Timeline, Search index, Webhooks, Scoring, Rotting timer | Aparece en ficha, resetea inactividad |
| `activity.completed.v1` | Cierre de tarea/reunión | Cadencia, SLA, Manager digest | Dispara siguiente paso |
| `email.sent.v1` / `email.opened.v1` / `email.clicked.v1` | Pasarela de correo | Scoring, Timeline, Secuencias | Sube intención |
| `opportunity.stage_changed.v1` | Cambio de etapa | Cotizaciones, Forecast, Tareas, Notificaciones, Auditoría | Ver 2.4 |
| `opportunity.amount_changed.v1` | Edición de monto | Forecast, Aprobaciones | Requiere aprobación si baja > 20% |
| `lead.recycled.v1` | Job de reciclaje | Asignador, Nurturing | Vuelve a la cola |
| `lead.converted.v1` | Conversión | Reporting, Marketing, Onboarding | Cierra el ciclo de lead |

## 2.8 Qué se ve en pantalla

| Momento | Superficie | Qué ve el usuario |
|---|---|---|
| Registra llamada | Modal sobre la ficha | Tipo, resultado (conectado/no contesta/buzón), duración, nota, "próximo paso" obligatorio |
| Guarda | Ficha | La llamada aparece arriba en la línea de tiempo; el badge "Sin actividad hace 3 días" desaparece |
| Mueve etapa | Kanban | La tarjeta se desliza, toast "Movido a Propuesta"; se abre modal pidiendo `reason_code` y, si aplica, monto y fecha |
| Etapa con guardrail | Modal bloqueante | "Para pasar a Propuesta necesitas: monto, fecha de cierre y contacto principal" |
| Dice "no por ahora" | Modal de pérdida | Selector de motivo (obligatorio) + nota + fecha de reciclaje sugerida por el sistema |
| Cierra el modal | Kanban | La tarjeta desaparece del embudo activo; contador "Reciclados: 34" sube |
| Convierte | Wizard de conversión | 3 columnas: Empresa / Contacto / Oportunidad, con checkboxes de "crear nuevo" vs "usar existente", campos prellenados y editables |
| Confirma | Ficha del contacto | Banner "Convertido desde lead el 18 nov 2026 · ver lead original"; línea de tiempo completa ya visible |

---

# RECORRIDO 3 — Se cierra la venta y pasa a operación

## 3.1 Diagrama de secuencia: ganar

```
Vendedor      API /close        PostgreSQL        Outbox        Finance       Inventory      CS
   |              |                 |                |              |             |           |
   |--"Ganado"--->|                 |                |              |             |           |
   |              |--validar------->|                |              |             |           |
   |              |  (quote, firma) |                |              |             |           |
   |              |--BEGIN--------->|                |              |             |           |
   |              |--UPDATE opp---->|                |              |             |           |
   |              |  stage=won      |                |              |             |           |
   |              |  won_at, amount |                |              |             |           |
   |              |--INSERT hist--->|                |              |             |           |
   |              |--INSERT snapshot|                |              |             |           |
   |              |--INSERT outbox->|                |              |             |           |
   |              |--COMMIT-------->|                |              |             |           |
   |<--200--------|                 |--publish------>|              |             |           |
   |              |                 |                |--order------>|             |           |
   |              |                 |                |              |--reserve--->|           |
   |              |                 |                |              |<--ok---------|           |
   |              |                 |                |              |--invoice--->|           |
   |              |                 |                |--onboarding-------------->|           |
```

## 3.2 Qué se escribe al ganar

`opportunities`:

| Campo | Valor | Nota |
|---|---|---|
| `stage` | `closed_won` | Etapa terminal |
| `probability` | `100` | |
| `forecast_category` | `closed` | Deja de contar en pipeline |
| `won_at` | `2026-12-02T16:41:09Z` | **Fecha de cierre comercial** |
| `amount` | `4800000` | |
| `amount_snapshot` | `4800000` | Congelado: no se toca nunca más |
| `currency` | `CLP` | |
| `fx_rate_at_close` | `1.0` | Si la moneda difiere de la funcional |
| `quote_id` | `quo_331` | Cotización aceptada |
| `contract_id` | `ctr_88` | Firma |
| `payment_terms` | `net_30` | |
| `closed_by` | `usr_018` | Comisión |
| `lost_reason_code` | `NULL` | Se limpia si venía de un reciclaje |

`stage_history`: fila `from='negotiation'`, `to='closed_won'`, con `amount_at_change` y `probability_at_change`.

`deal_snapshots` (una fila por cierre, inmutable):

| Campo | Valor |
|---|---|
| `opportunity_id` | `opp_55` |
| `account_id` | `acc_77` |
| `amount` | `4800000` |
| `line_items` | jsonb con productos, cantidades, precios y descuentos |
| `owner_id` | `usr_018` |
| `team_id` | `team_smb` |
| `source` | `web_form` |
| `first_touch_at` | `2026-11-16T14:02Z` |
| `won_at` | `2026-12-02T16:41Z` |
| `cycle_days` | `16` |
| `snapshot_at` | `2026-12-02T16:41:09Z` |

Este snapshot es lo que permite reportar dentro de dos años sin que un cambio de catálogo o de tipo de cambio reescriba la historia.

## 3.3 De la oportunidad a la factura

La cadena canónica es **cotización → pedido → factura → pago**, cuatro documentos con cuatro propósitos:

| Documento | Tabla | Propósito | Estado |
|---|---|---|---|
| Cotización | `quotes` + `quote_lines` | Propuesta de precio, con vigencia | `draft → sent → accepted / rejected / expired` |
| Pedido | `sales_orders` + `order_lines` | Compromiso operativo | `confirmed → fulfilled / cancelled` |
| Factura | `invoices` + `invoice_lines` | Documento tributario | `draft → posted → paid / void` |
| Pago | `payments` | Conciliación bancaria | `pending → matched` |

EspoCRM (Sales Pack) lo implementa exactamente así: la cotización se crea desde la oportunidad y **copia los ítems** de la oportunidad; desde la cotización se generan *sales order* y *invoice* copiando datos; la cotización se puede **bloquear** al completarse o cancelarse, tiene **fecha de expiración** y numeración automática configurable `[verificado + https://docs.espocrm.com/user-guide/quotes/]`.

HubSpot expone las mismas acciones como pasos de workflow: **Create quote** (a partir de un deal existente) y **Convert to invoice** (crea una factura borrador única desde un deal o cotización) `[verificado + https://knowledge.hubspot.com/workflows/choose-your-workflow-actions]`.

### Filas de la cotización

`quotes`:

| Campo | Valor |
|---|---|
| `number` | `COT-2026-0331` |
| `opportunity_id` | `opp_55` |
| `account_id` | `acc_77` |
| `contact_id` | `con_91` |
| `status` | `sent` |
| `issued_at` | `2026-11-18T12:00Z` |
| `valid_until` | `2026-12-18` |
| `currency` | `CLP` |
| `subtotal` | `4033613` |
| `tax_total` | `766387` |
| `total` | `4800000` |
| `locked` | `false` → `true` al aceptar |

`quote_lines`:

| `line_no` | `product_id` | `description` | `qty` | `unit_price` | `discount_pct` | `tax_rate` | `amount` |
|---|---|---|---|---|---|---|---|
| 1 | `prd_plan_multisede` | Plan multisede · 12 meses | 3 | `1200000` | `0` | `19` | `4284000` |
| 2 | `prd_onboarding` | Implementación | 1 | `400000` | `10` | `19` | `428400` |
| 3 | `prd_soporte` | Soporte prioritario | 1 | `87500` | `0` | `19` | `104125` |

**Punto crítico:** el precio de la línea se **congela al emitir**. Si mañana sube la lista de precios, la cotización enviada no cambia. Se guarda `price_book_id` y `price_book_version` para poder explicar el número.

### Filas de la factura

`invoices`:

| Campo | Valor |
|---|---|
| `number` | `F-2026-10842` |
| `sales_order_id` | `so_220` |
| `account_id` | `acc_77` |
| `status` | `posted` |
| `issued_at` | `2026-12-03` |
| `due_at` | `2027-01-02` |
| `net` | `4033613` |
| `tax` | `766387` |
| `total` | `4800000` |
| `currency` | `CLP` |
| `external_ref` | `SII/DTE-...` |
| `journal_entry_id` | `je_9912` |
| `paid_at` | `NULL` |

## 3.4 Conexión con inventario y caja

### Inventario

Tres movimientos distintos que la gente confunde:

| Momento | Documento | Efecto en stock | Tabla |
|---|---|---|---|
| Se gana | Pedido confirmado | **Reserva** (no descuenta) | `stock_reservations` |
| Se despacha | Guía de despacho / delivery | **Descuento real** | `stock_moves` (−) |
| Se anula | Nota de crédito / devolución | **Reingreso** | `stock_moves` (+) |

`stock_reservations`:

| Campo | Valor |
|---|---|
| `order_line_id` | `ol_551` |
| `product_id` | `prd_plan_multisede` |
| `warehouse_id` | `wh_01` |
| `qty` | `3` |
| `reserved_at` | `2026-12-02T16:41:15Z` |
| `expires_at` | `2026-12-09` |
| `status` | `active` |

Regla dura: **nunca** descontar stock al ganar. Si se descuenta y luego el cliente no paga o anula, hay que inventar un movimiento de reingreso que ensucia el kardex. Se reserva, y se descuenta cuando sale físicamente.

### Caja

`journal_entries` + `journal_lines` (partida doble, siempre cuadrada):

| Cuenta | Debe | Haber |
|---|---|---|
| `1.1.3 Clientes` | `4800000` | |
| `4.1.1 Ingresos` | | `4033613` |
| `2.1.3 IVA débito fiscal` | | `766387` |

Al cobrar: `1.1.1 Banco` al debe, `1.1.3 Clientes` al haber. El asiento se genera **al postear la factura**, no al ganar la oportunidad.

`payments`:

| Campo | Valor |
|---|---|
| `invoice_id` | `inv_...` |
| `amount` | `4800000` |
| `paid_at` | `2026-12-28` |
| `method` | `transfer` |
| `bank_ref` | `...` |
| `status` | `matched` |

## 3.5 Qué se dispara después

| Evento | Consumidor | Acción | Tabla |
|---|---|---|---|
| `opportunity.won.v1` | Finance | Crear pedido en borrador | `sales_orders` |
| `order.confirmed.v1` | Inventory | Reservar stock | `stock_reservations` |
| `invoice.posted.v1` | Accounting | Asiento contable | `journal_entries` |
| `invoice.paid.v1` | CS | Iniciar onboarding | `onboarding_plans` |
| `opportunity.won.v1` | CS | Crear proyecto + tareas + hitos | `projects`, `tasks` |
| `opportunity.won.v1` | Comms | Correo de bienvenida + kit | `email_queue` |
| `opportunity.won.v1` | Reporting | Escribir en `funnel_facts` | `funnel_facts` |
| `opportunity.won.v1` | Comisiones | Calcular comisión del trimestre | `commissions` |
| `opportunity.won.v1` | Marketing | Suprimir de campañas de adquisición | `suppression_list` |
| `onboarding.completed.v1` | CRM | `lifecycle_stage = customer` | `contacts`, `accounts` |

### Onboarding: tareas concretas

```
onboarding_plans
  id · account_id · opportunity_id · template_id · status · started_at · target_go_live

onboarding_tasks
  1. Kickoff interno                    due +1 día   owner: CSM
  2. Enviar contrato firmado al cliente due +1 día   owner: CSM
  3. Configurar accesos                 due +3 días  owner: Support   (blocker)
  4. Capacitación sesión 1              due +7 días  owner: Trainer
  5. Migración de datos                 due +10 días owner: Support   (blocker)
  6. Go-live                            due +14 días owner: CSM
  7. Revisión 30 días                   due +44 días owner: CSM
```

Cada tarea es una fila en `activities` (`type='task'`) con `parent_type='account'`. Así el onboarding **aparece en la misma línea de tiempo** que la venta: el cliente ve una historia continua, no dos sistemas pegados con cinta adhesiva.

## 3.6 Cómo queda el registro para los reportes

**Tres fechas distintas, tres preguntas distintas.** Este es el error de reporte más caro y más común:

| Fecha | Campo | Responde |
|---|---|---|
| Cierre comercial | `opportunities.won_at` | "¿Cuánto vendimos en diciembre?" (comisión, meta) |
| Emisión | `invoices.issued_at` | "¿Cuánto facturamos en diciembre?" (tributario, ingresos devengados) |
| Cobro | `payments.paid_at` | "¿Cuánta caja entró en diciembre?" (flujo de caja) |

Si el CFO y el CRO discuten porque sus números no cuadran, en el 90% de los casos están mirando fechas distintas y ninguno lo sabe.

### Tablas de hechos (modelo estrella mínimo)

```
funnel_facts
  date_key · tenant_id · owner_id · team_id · source_id
  leads_created · leads_qualified · opps_created · opps_won · opps_lost
  amount_won · amount_lost · cycle_days_sum · cycle_days_count

revenue_facts
  date_key · account_id · product_id · owner_id
  invoiced_amount · tax_amount · collected_amount · refunded_amount · fx_rate

activity_facts
  date_key · owner_id · activity_type · count · duration_seconds
```

Con eso: pipeline por etapa y owner, win rate por origen, tiempo medio por etapa, MRR, CAC payback, y cohortes de reciclaje.

## 3.7 Los tres puntos donde se rompe en la vida real

### Ruptura 1 — Se marca "ganado" y no existe cotización ni pedido

**Cómo se rompe:** el vendedor marca ganado para cerrar el mes y cobrar comisión. Finanzas no tiene documento, inventario no reservó, y el cliente recibe una factura por un monto que nadie aprobó. Al mes siguiente aparece el ajuste y el reporte de ventas del mes anterior cambia retroactivamente.

**Cómo lo mitiga el producto:**
- `closed_won` exige `quote_id` en estado `accepted` **o** `contract_id` firmado (guardrail de transición, igual que los campos requeridos de Odoo `crm.stage.requirements` `[verificado + https://github.com/odoo/odoo/blob/18.0/addons/crm/models/crm_stage.py]`).
- Cierre en **dos fases**: `won_pending_docs` (visible en el embudo, no comisionable) → `closed_won` (documento completo). El forecast usa la primera; la comisión, la segunda.
- Saga con compensación: si a los 10 minutos no se creó el pedido, se emite `order.creation_failed` y se abre tarea bloqueante al CFO. Nunca un `won` silencioso sin pedido.
- Bloqueo de período: una vez cerrado el mes contable, `won_at` no se puede mover.

### Ruptura 2 — El monto de la factura no coincide con el de la oportunidad

**Cómo se rompe:** la oportunidad dice 4.800.000, la cotización se envió con un descuento del 10% que se aplicó a mano, la factura salió con el precio de lista viejo porque el catálogo cambió entre medio, y el impuesto se calculó sobre una base distinta. Cuatro números, cuatro dueños, ninguna reconciliación.

**Cómo lo mitiga el producto:**
- **Snapshot al cerrar:** `deal_snapshots.line_items` congela productos, precios, descuentos e impuestos en el instante del cierre.
- La factura se genera **desde el snapshot o desde el pedido**, nunca recalculando desde el catálogo vigente.
- `sum(invoice_lines.amount) = opportunities.amount_snapshot` como test automatizado que corre cada noche y alerta diferencias.
- Bloqueo de emisión (EspoCRM "issuance locking" y el `locked` de la cotización) `[verificado + https://docs.espocrm.com/user-guide/quotes/]`.
- Aprobación obligatoria si `discount_pct > 15%` o si el monto baja más del 20% respecto de la etapa anterior.
- Redondeo y moneda definidos en un solo lugar (motor de impuestos), no en la plantilla del PDF.

### Ruptura 3 — Se gana, se factura, y el cliente queda botado

**Cómo se rompe:** el equipo comercial celebra, el deal sale del pipeline, y el onboarding arranca tres semanas después porque "eso lo ve el área de implementación". El cliente paga la primera factura sin haber recibido nada, entra en riesgo de churn antes del día 30, y en el CRM nadie lo nota porque el lifecycle sigue en `customer` (que suena bien) y no hay ninguna tarea vencida a la vista.

**Cómo lo mitiga el producto:**
- `opportunity.won.v1` **crea** el plan de onboarding en la misma transacción lógica (outbox, reintentos, dead-letter queue con alerta).
- El lifecycle del contacto **no** pasa a `customer` al ganar: pasa cuando `onboarding.completed.v1`. Mientras tanto es `opportunity` o un estado `onboarding` explícito.
- Tareas con dueño y SLA; si el kickoff no ocurre en 48 h, escalamiento automático al jefe de CS.
- Un health score mínimo: días desde el go-live, tareas vencidas, uso del producto, tickets abiertos.
- El reporte de "ventas ganadas" y el de "clientes activados" son **dos reportes distintos**, y el segundo es el que importa para retención.

---

# 4. El "no": comparativa y por qué es lo que más plata mueve

## 4.1 Cómo lo maneja cada producto

| Producto | Dónde vive el "no" | Motivo de pérdida | Reciclaje | Nurturing |
|---|---|---|---|---|
| **HubSpot** | `dealstage = closedlost` + `closed_lost_reason`; el contacto sigue con su `lifecycle_stage` y su `lead_status` | Propiedad de negocio, personalizable, obligatoria por workflow | Workflow con `delay` + re-enrolamiento explícito (por defecto **no** se re-enrola) `[verificado + https://knowledge.hubspot.com/workflows/create-workflows]` | Secuencias + lead status `Bad Timing` `[verificado + https://knowledge.hubspot.com/records/use-lifecycle-stages]` |
| **Salesforce** | `Opportunity.StageName = Closed Lost` + `LostReason` custom; el lead convertido queda read-only | Picklist + campo custom; validación por record type | Campañas/flow con fecha de reciclaje; el lead original no se reutiliza | Campaigns + `Lead.Status = Open - Not Contacted` |
| **Pipedrive** | Deal perdido con `lost_reason`; el lead no calificado vive en **Leads Inbox** aparte | Catálogo de motivos configurable | Archivar lead y desarchivar; "rotting" avisa si el deal se enfría en etapa `[verificado + https://support.pipedrive.com/en/article/the-rotting-feature]` | Etiquetas de lead + actividades programadas |
| **Attio** | Status attribute del deal (`stage`) con status "Lost"; atributos de motivo | Status attribute + atributo select | Listas y workflows con fecha; statuses configurables | Listas + secuencias |
| **EspoCRM** | `Opportunity.stage = Closed Lost`; `Lead.status = Recycled` o `Dead` | Campo enum + `lostReason` | `Recycled` es un estado nativo del lead `[verificado + https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Entities/Lead.php]` | Workflows/BPM + target lists |
| **Odoo** | `stage_id` en etapa `is_won=False` + `lost_reason_id`; `active=False` (archivado) | `lost_reason_id` de primera clase | `action_set_lost` archiva; desarchivar o reabrir lo devuelve `[verificado + https://github.com/odoo/odoo/blob/18.0/addons/crm/models/crm_lead.py]` | `crm.lead` + campañas de marketing |
| **Twenty** | `opportunity.stage` (string) + campo de motivo | Etapa terminal + atributo | Workflows sobre `stage` | Timeline + tareas |

## 4.2 Por qué esto es lo que más plata mueve

Un lead nuevo cuesta adquisición. Un lead reciclado cuesta **cero adquisición** y ya conoce la marca. La matemática es brutal:

| Métrica | Lead nuevo | Lead reciclado |
|---|---|---|
| Coste de adquisición | $18.000 | $0 |
| Tasa de contacto efectivo | 35% | 62% (ya hubo conversación) |
| Tasa de conversión a oportunidad | 12% | 22% |
| Ciclo de venta | 45 días | 28 días |
| Coste por venta | $150.000 | ~$8.000 (solo costo de contacto) |

Con 1.000 leads al mes, 22% dicen "no por ahora" (220), 40% de esos son reciclables (88), y si vuelven 25 de ellos con una conversión del 22% son **5,5 ventas extra al mes** que no costaron un peso de marketing. `[inferido: cifras ilustrativas, el orden de magnitud es consistente con la práctica de la industria]`

Y hay un segundo efecto, menos obvio y más grande: **el "no" bien capturado es el mejor dataset de pricing y de ICP que existe.** Cien motivos `lost_to_competitor` con el nombre del competidor te dicen dónde estás perdiendo. Cien `price` te dicen si el problema es el precio o el packaging. Cien `no_urgency` te dicen que tu propuesta de valor no tiene costo de inacción. Ninguna encuesta te da eso; la taxonomía de pérdida sí.

**El antídoto contra el "no" mal manejado** es hacer del motivo un campo obligatorio, con catálogo cerrado y una nota libre **adicional** (no sustitutiva). Texto libre solo = 400 formas de escribir "muy caro", y cero reportes.

---

# 5. La "activity" como entidad central

## 5.1 Por qué todo CRM termina con una tabla polimórfica

El CRM guarda dos clases de cosas:

1. **Sustantivos** con ciclo de vida propio: lead, contacto, empresa, oportunidad, cotización, factura. Tienen identidad, estado y relaciones.
2. **Verbos** con tiempo: llamé, escribí, me reuní, moví la etapa, envié la cotización, el cliente abrió el correo.

Los sustantivos son pocos y estables. Los verbos son **infinitos, heterogéneos y todos apuntan a cualquiera de los sustantivos**. Una llamada puede ser sobre un lead, un contacto, una empresa, una oportunidad o un ticket. Si modelas una tabla por verbo (`calls`, `emails`, `meetings`, `notes`, `tasks`, `stage_changes`, `form_submissions`…) terminas con:

- 8 tablas casi idénticas (`id, parent, actor, occurred_at, body, created_at`).
- 8 endpoints casi idénticos.
- Una línea de tiempo que requiere un `UNION ALL` de 8 tablas con `ORDER BY` y paginación por cursor en cada una. El `UNION` mata el índice.
- Cada verbo nuevo (WhatsApp, SMS, LinkedIn, transcripción) implica migración + endpoint + proyección.

Una tabla `activities` con `(type, parent_type, parent_id)` resuelve todo con un índice:

```sql
CREATE INDEX ix_activities_parent_time
  ON activities (tenant_id, parent_type, parent_id, occurred_at DESC);
```

Una consulta, un cursor, una paginación, un permiso, un webhook. Ese es el motivo por el que casi todos terminan ahí.

## 5.2 Los tres diseños que existen en la práctica

| Diseño | Ejemplo | Ventaja | Coste |
|---|---|---|---|
| **Tabla única polimórfica** | EspoCRM: `Meeting.parent` es `linkParent` a `Account, Lead, Contact, Opportunity, Case` `[verificado + https://github.com/espocrm/espocrm/blob/master/application/Espo/Modules/Crm/Resources/metadata/entityDefs/Meeting.json]` | Una consulta para la timeline; un solo ACL | Columnas específicas en JSON; sin FK real (integridad en aplicación) |
| **Una tabla por tipo + tabla de asociación** | HubSpot: calls, emails, meetings, notes, tasks como objetos separados, unidos por asociaciones | Cada tipo evoluciona solo; esquemas fuertes | La timeline necesita unir N fuentes; el "engagement" se reconstruye |
| **Tabla de timeline materializada** | Twenty: `timelineActivity` con `happensAt`, `properties`, `linkedRecord*` y `targetPerson/Company/Opportunity/Note/Task` `[verificado + https://github.com/twentyhq/twenty/blob/main/packages/twenty-server/src/modules/timeline/standard-objects/timeline-activity.workspace-entity.ts]` | Lectura instantánea; desacoplada | Es una proyección: hay que reconstruirla si se corrompe; duplica datos |

Salesforce combina los dos últimos: `Task` y `Event` son los registros, y "en la interfaz, los registros Task y Event se denominan colectivamente **activities**" `[verificado + https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_task.htm]`. El polimorfismo está en dos campos: `WhoId` (persona: lead o contacto) y `WhatId` (objeto de negocio: cuenta, oportunidad, campaña, caso). Es la misma idea con nombres distintos y una ventaja: el mismo registro puede colgar de una persona **y** de un negocio a la vez.

## 5.3 Las decisiones de diseño que importan

| Decisión | Recomendación | Por qué |
|---|---|---|
| ¿Una tabla o varias? | Una `activities` + `type` | Menos código, timeline trivial |
| ¿JSON o columnas? | Columnas para lo que se filtra; JSON para lo específico | `WHERE metadata->>'disposition'` no usa índice btree |
| ¿FK real al padre? | No (polimórfica); validar en aplicación + job de integridad | Un FK polimórfico no existe en SQL estándar |
| ¿Borrado? | Soft delete (`deleted_at`) | La historia no se borra; se oculta |
| ¿Inmutabilidad? | Actividades append-only; corrección = nueva actividad | Auditable |
| ¿Orden? | Por `occurred_at`, no `created_at` | Coherencia narrativa |
| ¿Idempotencia? | `UNIQUE (tenant_id, source, external_id)` | El sync de correo y calendario duplica sin esto |
| ¿Permisos? | ACL por `parent_type` + visibilidad por equipo | Un vendedor no debe ver la llamada de otro |
| ¿Búsqueda? | Índice full-text sobre `subject` + `body` | "¿De qué hablamos con este cliente?" |

**El test que revela si el diseño está bien:** pide "todo lo que pasó con esta cuenta en los últimos 12 meses, ordenado, en una sola pantalla". Si la respuesta requiere 6 queries y un merge en memoria, el modelo está mal.

---

# 6. Lifecycle vs stage: por qué mezclarlos rompe los reportes

## 6.1 Las dos preguntas

| | **Lifecycle stage** | **Opportunity stage** |
|---|---|---|
| Pregunta que responde | "¿Qué relación tiene esta **persona o empresa** con nosotros?" | "¿En qué punto está **esta transacción**?" |
| Entidad dueña | Contacto / empresa | Oportunidad (deal) |
| Cardinalidad | 1 por contacto, 1 por empresa | N por cuenta, a lo largo del tiempo |
| Dirección | Monotónica (avanza; retroceder es excepcional) | Cíclica (avanza, retrocede, se reabre) |
| Velocidad | Lenta (meses, años) | Rápida (días, semanas) |
| Terminal | `customer`, `evangelist`, `other` | `closed_won`, `closed_lost` |
| Quién la mueve | Marketing + sistema + ventas | Vendedor, con guardrails |
| Se usa para | Segmentación, ciclo de vida, atribución | Forecast, comisiones, pipeline, cadencia |

## 6.2 Cómo se ven en cada producto

**HubSpot** es el más explícito: el *Lifecycle stage* del contacto/empresa va de `Subscriber → Lead → MQL → SQL → Opportunity → Customer → Evangelist`, y **las herramientas de HubSpot solo pueden moverlo hacia adelante** (para retroceder hay que limpiarlo manualmente) `[verificado + https://knowledge.hubspot.com/records/use-lifecycle-stages]`. Y hay un detalle que confirma la separación: dentro de `Sales Qualified Lead` viven los *Lead Status* (`New, Open, In Progress, Open Deal, Unqualified, Attempted to Contact, Connected, Bad Timing`), que son el estado **operativo** del trabajo del vendedor, no del ciclo de vida `[verificado + mismo URL]`. Tres capas: lifecycle (relación), lead status (trabajo del lead), deal stage (transacción).

**Salesforce** separa limpio: `Lead.Status` (estado del lead), `Opportunity.StageName` (etapa del deal) y `Lifecycle` no existe como campo estándar — se modela con custom fields o con el estado del lead. La conversión deja `ConvertedAccountId/ContactId/OpportunityId` como puente `[verificado + https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_lead.htm]`.

**Attio** tiene un único status predefinido: `stage` en el objeto deal; los demás status attributes son configurables y definen columnas de kanban `[verificado + https://docs.attio.com/rest-api/attribute-types/attribute-types-status]`.

**Odoo** mezcla deliberadamente: `crm.lead.type` es `lead` u `opportunity`, y `stage_id` aplica a ambos. Es el contraejemplo útil: funciona porque la misma tabla guarda el embudo completo y `type` hace de discriminador `[verificado + https://github.com/odoo/odoo/blob/18.0/addons/crm/models/crm_lead.py]`.

## 6.3 Las cuatro formas en que se rompen los reportes

### Ruptura A — Contar oportunidades por lifecycle stage

`SELECT lifecycle_stage, COUNT(*) FROM contacts GROUP BY 1` con la intención de "ver el pipeline". Resultado: 4.800 contactos en `Lead`, 900 en `SQL`, y la conclusión "tenemos un pipeline enorme". Falso: son personas, no transacciones. Una cuenta puede tener 3 deals abiertos y 12 contactos; el contacto está en `opportunity` una sola vez.

**Correcto:** pipeline = `COUNT(*) FROM opportunities WHERE stage NOT IN ('closed_won','closed_lost')`, agrupado por `stage` y `owner`.

### Ruptura B — Poner la etapa del deal en el contacto

Alguien agrega `contact.deal_stage = 'negotiation'` "para que el vendedor lo vea rápido". Primer problema: ¿cuál de los 3 deals? Segundo: cuando uno se gana y otro se pierde, el contacto tiene un solo campo. Tercero: el forecast empieza a leer contactos y cuenta doble.

**Correcto:** el stage vive en la oportunidad; el contacto muestra un **rollup** (`open_deals_count`, `max_open_stage_rank`) que es derivado y de solo lectura.

### Ruptura C — `Closed Won` como lifecycle stage sin deal

Se marca el contacto como `customer` al cerrar. Meses después vuelve a comprar: el lifecycle ya está en `customer` y **no se mueve** (HubSpot no retrocede), así que el nuevo negocio no genera ningún evento de ciclo de vida. Peor: el reporte de "nuevos clientes por mes" cuenta contactos que pasaron a `customer`, no deals ganados; si un cliente compra dos veces, cuenta una.

**Correcto:** `customer` se deriva de "tiene al menos un deal cerrado ganado" y el reporte de nuevos clientes cuenta **la primera** `won_at` por cuenta, no el cambio de etapa.

### Ruptura D — Cohortes mezcladas

Se quiere "tiempo de conversión de lead a cliente" y se toma `customer_at - lead_created_at` sobre el contacto. Pero si el contacto se creó como lead en 2024, se convirtió, se perdió, volvió en 2026 y compró, el número sale 730 días y arruina el promedio. La etapa de la oportunidad sí permite ver el ciclo real de **ese** negocio.

**Correcto:** ciclo = `opportunity.won_at - opportunity.created_at`, y por separado "edad del lead al convertir" = `lead.converted_at - lead.created_at`. Dos métricas, dos preguntas.

## 6.4 La regla de diseño

> **Lifecycle describe a la persona/empresa. Stage describe la transacción. Nunca se escriben el uno al otro directamente; se derivan por reglas.**

```
lead.created              → contact.lifecycle_stage = 'lead'
MQL (score + fit)         → contact.lifecycle_stage = 'mql'
SQL (aceptado por ventas) → contact.lifecycle_stage = 'sql'
deal creado               → contact/company.lifecycle_stage = 'opportunity'
primer deal won           → contact/company.lifecycle_stage = 'customer'
segundo deal won          → (sin cambio; se registra en el deal)
deal lost                 → (sin cambio en lifecycle; se registra el motivo)
```

Y las consecuencias prácticas:

| Si necesitas… | Consulta… |
|---|---|
| Pipeline y forecast | `opportunities.stage` |
| Comisiones | `opportunities.won_at` + `owner_id` |
| Segmentación de marketing | `contacts.lifecycle_stage` + `lead_status` |
| Salud de la relación | `accounts.lifecycle_stage` + `open_deals` + tickets |
| Atribución | `opportunities.lead_source` + `form_submissions` + `first_touch_at` |
| Eficiencia comercial | `stage_history` (tiempo por etapa, retrocesos) |
| Reciclaje | `opportunities.lost_reason_code` + `recycle_at` |

---

# 7. Anexo: esquema de referencia

```sql
-- ============ NÚCLEO ============
CREATE TABLE accounts (
  id            text PRIMARY KEY,
  tenant_id     text NOT NULL,
  name          text NOT NULL,
  domain        text,
  industry      text,
  employees     int,
  country       char(2),
  lifecycle_stage text NOT NULL DEFAULT 'lead',
  owner_id      text,
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_accounts_domain ON accounts (tenant_id, lower(domain))
  WHERE domain IS NOT NULL;

CREATE TABLE contacts (
  id            text PRIMARY KEY,
  tenant_id     text NOT NULL,
  account_id    text REFERENCES accounts(id),
  first_name    text,
  last_name     text,
  email         text,
  email_normalized text,
  phone_e164    text,
  job_title     text,
  lifecycle_stage text NOT NULL DEFAULT 'lead',
  lead_status   text,
  first_touch_source text,
  owner_id      text,
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_contacts_email ON contacts (tenant_id, email_normalized)
  WHERE email_normalized IS NOT NULL AND merged_into_id IS NULL;

CREATE TABLE leads (
  id            text PRIMARY KEY,
  tenant_id     text NOT NULL,
  first_name    text,
  last_name     text,
  company_name_raw text,
  email_normalized text,
  phone_e164    text,
  source        text,
  source_detail text,
  utm           jsonb,
  status        text NOT NULL DEFAULT 'new',   -- new|assigned|in_process|converted|recycled|dead
  owner_id      text,
  score         int NOT NULL DEFAULT 0,
  consent_marketing boolean NOT NULL DEFAULT false,
  consent_text_hash text,
  consent_at    timestamptz,
  assigned_at   timestamptz,
  sla_due_at    timestamptz,
  converted_at  timestamptz,
  converted_account_id   text REFERENCES accounts(id),
  converted_contact_id   text REFERENCES contacts(id),
  converted_opportunity_id text,
  dedupe_of_lead_id text REFERENCES leads(id),
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_leads_email_live ON leads (tenant_id, email_normalized)
  WHERE status NOT IN ('merged','dead') AND dedupe_of_lead_id IS NULL;

CREATE TABLE opportunities (
  id            text PRIMARY KEY,
  tenant_id     text NOT NULL,
  account_id    text NOT NULL REFERENCES accounts(id),
  primary_contact_id text REFERENCES contacts(id),
  name          text NOT NULL,
  stage         text NOT NULL,
  probability   int  NOT NULL DEFAULT 0 CHECK (probability BETWEEN 0 AND 100),
  forecast_category text,
  amount        numeric(14,2) NOT NULL DEFAULT 0,
  amount_snapshot numeric(14,2),
  currency      char(3) NOT NULL DEFAULT 'CLP',
  fx_rate_at_close numeric(12,6),
  close_date    date,
  stage_entered_at timestamptz,
  won_at        timestamptz,
  closed_at     timestamptz,
  lost_reason_code text,
  lost_reason_note text,
  recycle_at    date,
  lead_source   text,
  quote_id      text,
  contract_id   text,
  owner_id      text NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_opps_pipeline ON opportunities (tenant_id, stage, owner_id, close_date);

-- ============ ACTIVIDADES (polimórfica) ============
CREATE TABLE activities (
  id            text PRIMARY KEY,
  tenant_id     text NOT NULL,
  type          text NOT NULL,   -- call|email|meeting|task|note|form_submitted|
                                 -- stage_changed|assignment|file|whatsapp
  parent_type   text NOT NULL,   -- lead|contact|account|opportunity|ticket
  parent_id     text NOT NULL,
  actor_type    text NOT NULL,   -- user|system|external
  actor_id      text,
  occurred_at   timestamptz NOT NULL,
  subject       text,
  body          text,
  direction     text,
  duration_seconds int,
  outcome       text,
  metadata      jsonb NOT NULL DEFAULT '{}'::jsonb,
  visibility    text NOT NULL DEFAULT 'internal',
  source        text NOT NULL DEFAULT 'ui',   -- ui|api|email_sync|calendar_sync|form
  external_id   text,
  created_at    timestamptz NOT NULL DEFAULT now(),
  deleted_at    timestamptz
);
CREATE INDEX ix_act_parent_time ON activities (tenant_id, parent_type, parent_id, occurred_at DESC);
CREATE UNIQUE INDEX ux_act_external ON activities (tenant_id, source, external_id)
  WHERE external_id IS NOT NULL;

CREATE TABLE activity_participants (
  activity_id   text NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
  participant_type text NOT NULL,  -- user|contact|lead|external
  participant_id   text,
  email         text,
  role          text,              -- from|to|cc|attendee|organizer
  response      text,              -- accepted|declined|tentative|none
  PRIMARY KEY (activity_id, participant_type, participant_id, role)
);

CREATE TABLE timeline_entries (      -- proyección
  id            text PRIMARY KEY,
  tenant_id     text NOT NULL,
  parent_type   text NOT NULL,
  parent_id     text NOT NULL,
  happened_at   timestamptz NOT NULL,
  kind          text NOT NULL,
  icon          text,
  title         text NOT NULL,
  subtitle      text,
  actor_name    text,
  deep_link     text,
  source_activity_id text,
  rank          int NOT NULL DEFAULT 0
);
CREATE INDEX ix_timeline ON timeline_entries (tenant_id, parent_type, parent_id, happened_at DESC);

-- ============ HISTORIA Y AUDITORÍA ============
CREATE TABLE stage_history (
  id            bigserial PRIMARY KEY,
  opportunity_id text NOT NULL REFERENCES opportunities(id),
  from_stage    text,
  to_stage      text NOT NULL,
  changed_by    text,
  reason_code   text,
  amount_at_change numeric(14,2),
  probability_at_change int,
  close_date_at_change date,
  forecast_category_at_change text,
  changed_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ix_stage_hist ON stage_history (opportunity_id, changed_at);

CREATE TABLE assignment_log (
  id            bigserial PRIMARY KEY,
  lead_id       text REFERENCES leads(id),
  opportunity_id text REFERENCES opportunities(id),
  from_owner_id text,
  to_owner_id   text NOT NULL,
  rule          text NOT NULL,
  reason        text,
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE form_submissions (
  id            text PRIMARY KEY,
  tenant_id     text NOT NULL,
  form_id       text NOT NULL,
  lead_id       text REFERENCES leads(id),
  contact_id    text REFERENCES contacts(id),
  payload       jsonb NOT NULL,
  ip_hash       text,
  user_agent    text,
  referrer      text,
  visitor_id    text,
  idempotency_key text NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_form_idem ON form_submissions (tenant_id, form_id, idempotency_key);

-- ============ COMERCIAL → OPERACIÓN ============
CREATE TABLE quotes (
  id text PRIMARY KEY, tenant_id text NOT NULL,
  number text NOT NULL, opportunity_id text REFERENCES opportunities(id),
  account_id text NOT NULL REFERENCES accounts(id), contact_id text REFERENCES contacts(id),
  status text NOT NULL DEFAULT 'draft',
  currency char(3) NOT NULL, subtotal numeric(14,2), tax_total numeric(14,2),
  total numeric(14,2), issued_at timestamptz, valid_until date,
  locked boolean NOT NULL DEFAULT false, price_book_version text,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE quote_lines (
  id text PRIMARY KEY, quote_id text NOT NULL REFERENCES quotes(id) ON DELETE CASCADE,
  line_no int NOT NULL, product_id text, description text,
  qty numeric(12,3) NOT NULL, unit_price numeric(14,2) NOT NULL,
  discount_pct numeric(5,2) NOT NULL DEFAULT 0,
  tax_rate numeric(5,2) NOT NULL DEFAULT 19,
  amount numeric(14,2) NOT NULL
);

CREATE TABLE sales_orders (
  id text PRIMARY KEY, tenant_id text NOT NULL, number text NOT NULL,
  quote_id text REFERENCES quotes(id), opportunity_id text REFERENCES opportunities(id),
  account_id text NOT NULL REFERENCES accounts(id),
  status text NOT NULL DEFAULT 'draft',
  confirmed_at timestamptz, total numeric(14,2), currency char(3)
);

CREATE TABLE invoices (
  id text PRIMARY KEY, tenant_id text NOT NULL, number text NOT NULL,
  sales_order_id text REFERENCES sales_orders(id),
  account_id text NOT NULL REFERENCES accounts(id),
  status text NOT NULL DEFAULT 'draft',   -- draft|posted|paid|void
  issued_at date, due_at date,
  net numeric(14,2), tax numeric(14,2), total numeric(14,2), currency char(3),
  external_ref text, journal_entry_id text, paid_at date
);
CREATE UNIQUE INDEX ux_invoice_order ON invoices (sales_order_id) WHERE status <> 'void';

CREATE TABLE stock_reservations (
  id bigserial PRIMARY KEY, order_line_id text NOT NULL,
  product_id text NOT NULL, warehouse_id text NOT NULL,
  qty numeric(12,3) NOT NULL, status text NOT NULL DEFAULT 'active',
  reserved_at timestamptz NOT NULL DEFAULT now(), expires_at date
);

-- ============ OUTBOX ============
CREATE TABLE outbox_events (
  id            bigserial PRIMARY KEY,
  tenant_id     text NOT NULL,
  event_type    text NOT NULL,
  aggregate_type text NOT NULL,
  aggregate_id  text NOT NULL,
  payload       jsonb NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  published_at  timestamptz,
  attempts      int NOT NULL DEFAULT 0,
  last_error    text
);
CREATE INDEX ix_outbox_pending ON outbox_events (created_at) WHERE published_at IS NULL;

CREATE TABLE processed_events (
  consumer      text NOT NULL,
  event_id      bigint NOT NULL,
  processed_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer, event_id)
);
```

---

# 8. Resumen ejecutivo en una página

1. **Todo empieza con una transacción y un evento en la misma transacción.** El `201` al visitante significa "persistido", no "asignado".
2. **Deduplicar es un problema de cuatro capas**, y la cookie nunca es identidad. El índice único por email normalizado es la última línea de defensa.
3. **Asignar es distinto de notificar**, y "tiene owner" no significa "está siendo trabajado". Sin disponibilidad de usuarios, hay leads invisibles.
4. **La actividad es la entidad central** porque los verbos son infinitos y los sustantivos son pocos. Una tabla con `(type, parent_type, parent_id)` compra timeline, auditoría y permisos con un solo índice.
5. **El "no" es el activo más subvalorado.** Motivo obligatorio con catálogo cerrado + fecha de reciclaje + tarea real = ventas que no cuestan adquisición.
6. **Lifecycle responde "qué es esta relación"; stage responde "dónde va este negocio".** Se derivan por reglas, nunca se escriben el uno al otro, y se reportan por separado.
7. **Ganar no es facturar, y facturar no es cobrar.** Tres fechas, tres reportes, tres dueños. El snapshot al cerrar es lo que evita que la historia se reescriba.
