# 01 — Modelo de datos de un CRM real: entidades, campos custom, multi-tenancy, auditoría, dinero y fechas

**Objetivo:** material accionable para diseñar el esquema de un CRM desde cero. Cada sección termina en una decisión concreta, no en una recomendación genérica.

**Método:** fuentes primarias únicamente — Object Reference de Salesforce, CRM API de HubSpot, referencia de la API de Pipedrive, y **código/DDL** de Twenty, EspoCRM, SuiteCRM/Sugar, Odoo, Frappe/ERPNext, Krayin, Dolibarr, CiviCRM, Monica, Attio.

**Leyenda de etiquetas**

| Etiqueta | Significado |
|---|---|
| `[V]` | Verificado: leí el documento/código. Va la URL. |
| `[V·código]` | Verificado leyendo el archivo fuente del repositorio (URL al raw). |
| `[V·sec]` | Verificado con fuente secundaria (blog de ingeniería, doc de terceros), no con doc oficial. |
| `[I]` | Inferido por mí a partir de lo anterior. No lo tomes como cita. |

---

## 1. Entidades núcleo: Account/Company vs Contact vs Lead vs Opportunity/Deal vs Activity

### 1.1 Inventario real por producto

| Producto | Empresa | Persona | Lead | Deal | Actividad | Nota |
|---|---|---|---|---|---|---|
| **Salesforce** | `Account` | `Contact` | `Lead` (objeto propio) | `Opportunity` | `Task`, `Event` (Activity) | `Note` (antes `Note`, hoy `ContentNote`) |
| **HubSpot** | `Company` (0-2) | `Contact` (0-1) | `Lead` (0-136) | `Deal` (0-3) | `Task`, `Call`, `Meeting`, `Email`, `Note`, `Communication` | "Activities" no es un objeto: son varios objetos |
| **Pipedrive** | `Organization` | `Person` | `Lead` (delgado, "Leads Inbox") | `Deal` | `Activity` | `Note` |
| **Attio** | `companies` | `people` | **no existe** | `deals` | `activities` + `tasks`/`notes` | `lists` como contenedor polimórfico |
| **Twenty** | `company` | `person` | **no existe** | `opportunity` | `task`, `note`, `calendarEvent`, `messageThread` | timeline genérico |
| **EspoCRM** | `Account` | `Contact` | `Lead` | `Opportunity` | `Task`, `Call`, `Meeting` | `Email`, `Document` |
| **Krayin** | `organizations` | `persons` | `leads` (**el deal se llama lead**) | — | `activities` | pivot `lead_activities` |
| **Dolibarr** | `llx_societe` | `llx_socpeople` | — (usa `llx_prospect`? no: usa `societe.client=2` para prospecto) | `llx_propal` (propuesta) | `llx_actioncomm` | — |
| **CiviCRM** | `civicrm_contact` (organización **y** persona en la misma tabla) | idem | — | contribuciones | `civicrm_activity` | — |
| **Monica** | — (personal CRM) | `contacts` | — | — | `activities` | — |

Fuentes: `[V]` https://developer.salesforce.com/docs/platform/object-reference/references/standard-objects/object-opportunity.html · `[V]` https://developers.hubspot.com/docs/guides/api/crm/understanding-the-crm · `[V]` https://developers.pipedrive.com/docs/api/v1/Leads · `[V·código]` https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-server/src/modules/person/standard-objects/person.workspace-entity.ts · `[V·código]` https://raw.githubusercontent.com/espocrm/espocrm/master/application/Espo/Modules/Crm/Resources/metadata/entityDefs/Contact.json · `[V·código]` https://raw.githubusercontent.com/krayin/laravel-crm/2.2/packages/Webkul/Lead/src/Database/Migrations/2021_04_22_164215_create_leads_table.php

### 1.2 Por qué existe `Lead` separado de `Contact`

No es una decisión de marketing, es de **modelo de dominio y de seguridad**. Un lead:

1. **Tiene dueño semántico distinto.** En Salesforce el objeto `Lead` tiene su propio modelo de compartición (Lead Sharing) separado del de Contact/Account. Un SDR puede ver 5.000 leads y 0 cuentas.
2. **Tiene volumen y vida corta.** Los leads entran por formularios/webhooks en lotes; el 90% nunca se convierte. Mezclarlos con `contact` contamina la tabla que se usa en list views de todo el equipo.
3. **Tiene campos que no tienen sentido después de convertir** (`Status`, `Rating`, `LeadSource`, campaña de origen, formulario de origen). En `Contact` esos campos son ruido.
4. **La conversión es una transacción de 3 escrituras.** Salesforce: `Lead` → `Account` + `Contact` + `Opportunity` **atómicamente**, y el lead queda marcado, no borrado.

Lo verificable en Salesforce:

```
Lead.ConvertedAccountId        -> Id de la Account creada
Lead.ConvertedContactId        -> Id del Contact creado
Lead.ConvertedOpportunityId    -> Id de la Opportunity creada
Lead.ConvertedDate             -> date
Lead.IsConverted               -> boolean
```

`[V]` https://developer.salesforce.com/docs/platform/object-reference/references/standard-objects/object-lead.html — la doc dice literalmente "Object reference ID that points to the account into which the lead converted" y "Indicates whether the lead has been converted (true) or not (false)".

Es decir: **Salesforce no borra el lead, lo convierte en un puntero**. Eso es una decisión de auditoría, no de UX.

EspoCRM replica el patrón:

```json
// Lead.json — links
"createdAccount":     {"type":"hasOne","entity":"Account","foreign":"originalLead"},
"createdContact":     {"type":"hasOne","entity":"Contact","foreign":"originalLead"},
"createdOpportunity": {"type":"hasOne","entity":"Opportunity","foreign":"originalLead"}
// campos: "convertedAt", "accountName" (texto libre, pre-conversión)
```

`[V·código]` https://raw.githubusercontent.com/espocrm/espocrm/master/application/Espo/Modules/Crm/Resources/metadata/entityDefs/Lead.json

Detalle importante: Espo tiene `accountName` en el Lead (texto) **y** `createdAccount` (FK). Es el patrón "nombre libre hasta que exista la entidad". Todo CRM lo necesita: el comercial escribe "Falabella" antes de que exista la cuenta.

### 1.3 Quién eliminó el Lead (y quién no)

**Corrección de la premisa:** Pipedrive **no eliminó** el lead; lo **adelgazó**. Attio sí lo eliminó. HubSpot lo **reintrodujo** en 2025 con una semántica nueva.

| Producto | Estado | Semántica real | Fuente |
|---|---|---|---|
| **Pipedrive** | Existe, pero es delgado | `title` + `person_id`/`organization_id` (obligatorio uno de los dos) + `value{amount,currency}` + `expected_close_date` + `label_ids`. **Hereda los campos custom de Deals**: "leads do not have a separate set of custom fields, instead they inherit the custom fields' structure from deals". Al convertir: "the lead is marked as deleted". | `[V]` https://developers.pipedrive.com/docs/api/v1/Leads |
| **Attio** | **No existe** | Su API expone como *standard objects* solo People, Companies, Users, Deals, Workspaces. No hay grupo "Leads" en el índice de endpoints. Modelo: objects + attributes + records + **lists** (el pipeline es una list con entries). | `[V]` https://docs.attio.com/rest-api/endpoint-reference (índice: sin grupo Leads) · `[V]` https://docs.attio.com/sql/available-data (schemas `objects`, `lists`, `activities`) |
| **HubSpot** | Existe como **envoltorio de asociación** | "Must be associated with an existing contact". Y: "If you remove all primary associations to the lead, the lead will automatically be deleted." Tiene `hs_lead_name`, `hs_lead_type`, `hs_lead_label` y su propio pipeline (limit 15). | `[V]` https://developers.hubspot.com/docs/api-reference/latest/crm/objects/leads/guide |
| **Twenty** | No existe | Solo `person` + `opportunity`. | `[V·código]` estructura de `packages/twenty-server/src/modules/*/standard-objects/` |
| **Krayin** | Inversión de nombres | La tabla `leads` **es** el deal: `lead_value`, `lead_pipeline_id`, `lead_stage_id`, `closed_at`, `lost_reason`. No hay `deals`. | `[V·código]` migración `create_leads_table.php` |

**Lectura de diseño:** hay tres soluciones al problema "no quiero contaminar `contact` con prospectos", y son **mutuamente excluyentes**:

- **A. Objeto Lead completo (Salesforce, EspoCRM).** Tabla propia con nombre, email, teléfono, empresa en texto. Coste: duplicación de datos de persona + un proceso de merge/dedupe entre `lead` y `contact` que hay que construir y mantener para siempre.
- **B. Lead delgado con FK a persona/empresa (Pipedrive, HubSpot).** No duplica datos de persona. Coste: no puedes capturar un lead sin crear antes la persona (o crearla al vuelo), y el lead no puede tener campos que la persona no tenga.
- **C. Sin Lead (Attio, Twenty).** Todo deal nace ya con empresa/persona. Coste: el equipo comercial usa una etapa "Prospecto" del pipeline como si fuera lead, y las métricas de marketing (leads por campaña) se pierden o se ponen en un campo del deal.

**Decisión recomendada: B.** Razones concretas:

1. El Lead de tipo A obliga a un dedupe continuo (`lead.email` vs `contact.email`) — el problema más caro de un CRM y el que más bugs produce.
2. B permite que la conversión sea **una sola escritura** (`INSERT INTO deals ... SELECT ... FROM leads`), sin transacción de 3 tablas.
3. B no impide tener métricas de marketing: `leads.source`, `leads.campaign_id`, `leads.created_at`.
4. B permite borrar el lead al convertir **sin perder historia**, porque la persona y la empresa ya existían.

DDL recomendado (ver §8 para el esquema completo):

```sql
CREATE TABLE leads (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id     uuid NOT NULL,
  title         text NOT NULL,                    -- lo que el comercial escribe
  contact_id    uuid REFERENCES contacts(id),      -- al menos uno de los dos
  company_id    uuid REFERENCES companies(id),
  pipeline_id   uuid NOT NULL REFERENCES pipelines(id),
  stage_id      uuid NOT NULL REFERENCES pipeline_stages(id),
  amount        numeric(18,4),
  currency      char(3),
  expected_close_date date,
  source        text,
  campaign_id   uuid,
  owner_id      uuid NOT NULL REFERENCES users(id),
  converted_deal_id uuid REFERENCES deals(id),     -- puntero, no borrado
  converted_at  timestamptz,
  archived_at   timestamptz,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now(),
  deleted_at    timestamptz,
  CONSTRAINT lead_needs_a_party CHECK (contact_id IS NOT NULL OR company_id IS NOT NULL)
);
```

Regla: `lead` **no** guarda nombre/email/teléfono de la persona. Si el lead llega sin persona, se crea la persona y se enlaza. (Esto es exactamente lo que hace HubSpot: "if the person does not exist yet, it needs to be created first" — `[V]` doc de Pipedrive, mismo texto.)

### 1.4 Cardinalidades reales: Account ↔ Contact

Este es el punto donde más productos se equivocan, y donde la realidad del negocio gana.

| Producto | Modelo | Implementación |
|---|---|---|
| **Salesforce** | 1:N por defecto (`Contact.AccountId`) **+ N:M opcional** | Objeto `AccountContactRelation` ("Represents a relationship between a contact and one or more accounts", API 37.0+), con `Roles`, `IsActive`, `StartDate`, `EndDate`, `IsDirect` |
| **EspoCRM** | **N:M siempre**, con `account` (primario) además | Tabla pivote `AccountContact` con `role varchar(100)` e `isInactive bool` |
| **HubSpot** | N:M con etiquetas | Asociación contact↔company con `associationTypeId` = label ("Decision maker", "Billing contact"), límite 50.000 asociaciones por registro |
| **Twenty** | **1:N puro** | `person.companyId` único FK. No hay N:M. |
| **Dolibarr** | 1:N puro | `llx_socpeople.fk_soc` |
| **CiviCRM** | N:M vía `civicrm_relationship` | tabla de relaciones tipadas con `is_active`, fechas |

Fuentes: `[V]` https://developer.salesforce.com/docs/platform/object-reference/references/standard-objects/object-accountcontactrelation.html · `[V·código]` Contact.json (`"accounts": {"type":"hasMany","additionalColumns":{"role":...,"isInactive":...}}`) · `[V]` https://developers.hubspot.com/docs/guides/api/crm/understanding-the-crm · `[V·código]` Person/Company entity de Twenty · `[V·código]` `llx_socpeople.sql`

**Por qué el N:M es obligatorio en B2B real:**

- Un contacto es "Gerente de Compras de Falabella" **y** "Director de la fundación familiar". Dos empresas, una persona.
- Cambio de trabajo: si tu modelo es 1:N, mover el contacto de empresa **borra la historia comercial** (el deal de 2024 apuntaba a la empresa vieja).
- Filiales: un contacto puede comprar a 3 razones sociales del mismo grupo con precios distintos.
- Consultores y brokers: N empresas por definición.

**Decisiones concretas:**

1. Tabla puente `company_contacts` con atributos **de la relación**, no de la persona: `role`, `is_primary`, `is_active`, `job_title` (¡el cargo es de la relación!), `started_at`, `ended_at`.
2. Denormalizar `contacts.primary_company_id` para que el list view de contactos no necesite el join. Es lo que hace Salesforce con `Contact.AccountId`.
3. El deal guarda **empresa y contacto primario**, no la relación: `deals.company_id` + `deals.primary_contact_id`. Si el contacto se va, el deal sigue en la empresa.
4. Nunca borrar filas de `company_contacts`: `is_active=false` + `ended_at`.

```sql
CREATE TABLE company_contacts (
  company_id  uuid NOT NULL REFERENCES companies(id),
  contact_id  uuid NOT NULL REFERENCES contacts(id),
  role        text,
  job_title   text,
  is_primary  boolean NOT NULL DEFAULT false,
  is_active   boolean NOT NULL DEFAULT true,
  started_at  date,
  ended_at    date,
  PRIMARY KEY (company_id, contact_id)
);

-- solo puede haber un primario activo por contacto
CREATE UNIQUE INDEX uq_contact_primary_company
  ON company_contacts (contact_id)
  WHERE is_primary AND is_active;
```

Ese índice parcial es la pieza que casi nadie implementa y que evita el bug clásico "el contacto tiene 3 empresas primarias".

### 1.5 Opportunity/Deal: cómo se modela la etapa (y por qué casi todos lo hacen mal)

| Producto | Etapa | Probabilidad | Ganado/Perdido |
|---|---|---|---|
| **Salesforce** | `StageName` picklist → objeto `OpportunityStage` | por etapa (`Probability`), y `Opportunity.Probability` (percent) editable | `IsClosed`/`IsWon` booleanos **derivados** de la etapa (`Defaulted on create`, sin `Update`) |
| **EspoCRM** | `stage` enum + `probabilityMap` en metadata | `probability` int por registro; el mapa de etapa es solo default | `Closed Won`/`Closed Lost` como opciones del enum |
| **Krayin** | `lead_stage_id` + **`lead_pipeline_stages`** con `probability` y `sort_order` por pipeline | por (pipeline, etapa) | no hay flag: se infiere de la etapa |
| **HubSpot** | `dealstage` de un `pipeline` | por etapa | flags de etapa cerrada |
| **Twenty** | `stage: string` — **texto libre** | no hay | no hay |

Fuentes: `[V]` Object Reference de Opportunity (bloques de `probability`=percent y de booleanos "Defaulted on create", sin Update) · `[V·código]` Opportunity.json de Espo (`"probabilityMap": {"Prospecting":10,...,"Closed Won":100,"Closed Lost":0}`) · `[V·código]` `create_lead_pipeline_stages_table.php` · `[V·código]` https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-server/src/modules/opportunity/standard-objects/opportunity.workspace-entity.ts (`stage: string`).

**Por qué la etapa debe ser una fila y no un enum:**

1. Cada tenant define sus etapas. Un enum en columna te obliga a DDL o a un CHECK que hay que reescribir.
2. Cada tenant tiene **varios pipelines** (nuevo negocio, renovación, licitaciones), y la misma "Negociación" tiene probabilidad distinta en cada uno. Krayin lo resolvió bien: `lead_pipeline_stages(lead_pipeline_id, lead_stage_id, probability, sort_order)`.
3. El forecast necesita `forecast_category` por etapa (Pipeline / Best Case / Commit / Closed) — es un atributo de la etapa, no del deal.
4. Necesitas "tiempo en etapa", y eso exige historial de etapa (§4), no solo el valor actual.

**Decisiones concretas:**

```sql
CREATE TABLE pipelines (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  name text NOT NULL, entity text NOT NULL DEFAULT 'deal',  -- 'deal' | 'lead' | 'ticket'
  is_default boolean NOT NULL DEFAULT false
);

CREATE TABLE pipeline_stages (
  id uuid PRIMARY KEY, pipeline_id uuid NOT NULL REFERENCES pipelines(id),
  name text NOT NULL,
  position int NOT NULL,                 -- orden en el kanban
  probability int NOT NULL DEFAULT 0 CHECK (probability BETWEEN 0 AND 100),
  is_won boolean NOT NULL DEFAULT false,
  is_lost boolean NOT NULL DEFAULT false,
  forecast_category text,                -- pipeline|best_case|commit|closed
  rot_days int,                          -- SLA: días para considerar estancado
  UNIQUE (pipeline_id, position)
);

CREATE TABLE deals (
  ...
  pipeline_id uuid NOT NULL REFERENCES pipelines(id),
  stage_id    uuid NOT NULL REFERENCES pipeline_stages(id),
  stage_entered_at timestamptz NOT NULL DEFAULT now(),  -- para "time in stage"
  probability int,                                      -- override manual, nullable
  won_at timestamptz, lost_at timestamptz, lost_reason text,
  amount numeric(18,4), currency char(3), amount_base numeric(18,4),
  fx_rate numeric(18,8), fx_rate_date date,
  close_date date NOT NULL,            -- fecha, NO timestamp
  ...
);
```

Invariante que debes hacer cumplir en la capa de aplicación **y** con un trigger: `is_won`/`is_lost` de la etapa son la única fuente de verdad; `deals.won_at` se escribe cuando la etapa destino tiene `is_won`.

### 1.6 Activity / Task / Note / Email: los tres patrones reales

| Patrón | Producto | Cómo | Problema que resuelve / crea |
|---|---|---|---|
| **A. Padre polimórfico** | EspoCRM, Dolibarr | `Task.parent_type` + `parent_id`; `Email.parent_type/parent_id`; Dolibarr `llx_actioncomm.elementtype` + `fk_element` | Simple, 1 actividad → 1 padre. **No permite** una reunión con 4 contactos y 1 deal sin duplicar |
| **B. Junction de targets** | Twenty | `taskTarget(taskId, targetEntityType, targetEntityId)`, igual `noteTarget`, `calendarEventTarget` | 1 actividad → N entidades. **Requiere** un join por timeline |
| **C. Tabla ancha única + extensiones** | Dolibarr, Krayin | `llx_actioncomm` con TODOS los campos de todos los tipos (email_subject, email_from, recurrule, location, calling_duration...) | Una sola query de timeline. Tabla con 60+ columnas mayormente NULL |

Fuentes: `[V·código]` `note-target.workspace-entity.ts` / `task-target.workspace-entity.ts` de Twenty · `[V·código]` `llx_actioncomm.sql` (ver §7.1) · `[V·código]` `create_lead_activities_table.php` (Krayin: tabla puente `activity_id`+`lead_id`).

Krayin usa el patrón B pero **con una tabla puente por entidad** (`lead_activities`, `person_activities`), que es lo peor de ambos mundos: N tablas y N queries en la timeline. Twenty usa B con **una sola tabla** y un `targetEntityType` discriminador — eso sí escala.

**Decisión recomendada: B con una única tabla de enlaces.**

```sql
CREATE TABLE activities (
  id uuid PRIMARY KEY, tenant_id uuid NOT NULL,
  kind text NOT NULL CHECK (kind IN ('note','task','call','meeting','email','system')),
  title text,
  body text,
  occurred_at timestamptz NOT NULL,      -- cuándo pasó (o cuándo se creó la nota)
  due_at timestamptz,                    -- solo task
  completed_at timestamptz,              -- solo task
  duration_minutes int,                  -- solo call/meeting
  owner_id uuid REFERENCES users(id),
  is_pinned boolean NOT NULL DEFAULT false,
  -- extras específicos del tipo, sin 60 columnas
  meta jsonb,
  created_at timestamptz NOT NULL DEFAULT now(),
  deleted_at timestamptz
);

CREATE TABLE activity_links (
  activity_id uuid NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
  entity_type text NOT NULL,       -- 'company'|'contact'|'deal'|'lead'|'ticket'
  entity_id   uuid NOT NULL,
  role text,                       -- 'attendee'|'organizer'|'about'
  PRIMARY KEY (activity_id, entity_type, entity_id)
);

-- LA consulta que define la tabla: timeline de un registro
CREATE INDEX idx_activity_links_timeline
  ON activity_links (entity_type, entity_id, activity_id);
```

Y una vista materializada o un índice sobre `activities(tenant_id, occurred_at DESC)` para el feed global.

**Nota sobre Notes vs Tasks:** no las unas en una tabla si vas a hacer filtros de productividad. "Mis tareas de esta semana" con `kind='task' AND completed_at IS NULL AND due_at BETWEEN ...` es un índice parcial perfecto:

```sql
CREATE INDEX idx_tasks_open ON activities (tenant_id, owner_id, due_at)
  WHERE kind = 'task' AND completed_at IS NULL AND deleted_at IS NULL;
```

Eso es la diferencia entre 4 ms y 400 ms en el home del vendedor.

---

## 2. Campos personalizados: EAV vs JSONB vs DDL dinámico

### 2.1 Qué usa cada producto (verificado)

| Producto | Estrategia | Evidencia |
|---|---|---|
| **Salesforce** | **EAV con slots ("flex columns") + tabla pivote de índices** | `MT_Data` con `Value0…ValueN` de tipo string universal; `MT_Fields` mapea campo→slot; `_MT_Indexes_` con `StringValue/NumValue/DateValue`; `_MT_Clob_`, `_MT_Unique_Indexes`, `_MT_Relationships`, `MT_Name_Denorm` |
| **Krayin** | **EAV clásico, con columnas tipadas** | `attributes(id, code, type, entity_type, ...)` + `attribute_values(text_value, boolean_value, integer_value, float_value, datetime_value, date_value, json_value, entity_id, attribute_id)` con `UNIQUE(entity_type, entity_id, attribute_id)` |
| **HubSpot** | EAV lógico invisible tras la API de "properties" | `properties` con `objectTypeId`; 1.000 props/objeto, 10.000 por cuenta |
| **SugarCRM / SuiteCRM** | **DDL dinámico, tabla `<módulo>_cstm`** | `CREATE TABLE accounts_cstm (id_c char(36) NOT NULL, PRIMARY KEY (id_c)); ALTER TABLE accounts_cstm ADD COLUMN ejemplo_c varchar(255) NULL;` + fila en `fields_meta_data` |
| **Dolibarr** | **DDL dinámico, tabla de extrafields por objeto** | `llx_societe_extrafields(rowid, tms, fk_object, import_key)` y columnas `x_<campo>` añadidas con ALTER TABLE |
| **EspoCRM** | **DDL dinámico desde `entityDefs` (JSON de metadatos)** | `SchemaManager` + `ColumnPreparator` con `precision`/`scale`/`dbType`; el campo custom es una columna real |
| **Frappe / ERPNext** | **DDL dinámico desde DocType** | `DBTable.sync()` → `add_column()` → `alter table tab<DocType> add column if not exists ...` |
| **Twenty** | **DDL dinámico por workspace (schema propio)** | `generateWorkspaceSchemaDdl()` emite `CREATE TYPE` + `CREATE TABLE`; `fieldMetadataTypeToColumnType()` mapea tipo→`numeric|text|timestamptz|jsonb|tsvector`. Los campos nuevos se convierten en `ALTER TABLE ... ADD COLUMN` |
| **Odoo** | **DDL dinámico + JSONB opcional** | `ir.model.fields` refleja columnas reales; `fields.Serialized` + `sparse='data'` guarda varios campos dentro de **un** JSONB; `Properties` guarda pseudo-campos en JSONB |
| **CiviCRM** | **DDL dinámico, una tabla por grupo de campos custom** | `civicrm_value_<grupo>_<id>(id, entity_id, <columna_por_campo>...)` con FK a `civicrm_contact` y `UNIQUE(entity_id)` |
| **Attio** | JSON por atributo (cada atributo es una entidad con su propio tipo) | API `/objects/{object}/records/{id}/attributes/{attribute}/values` — los valores son direccionables por atributo, no un blob |

Fuentes:
`[V]` https://architect.salesforce.com/docs/architect/fundamentals/guide/platform-multitenant-architecture.html
`[V·código]` https://raw.githubusercontent.com/krayin/laravel-crm/2.2/packages/Webkul/Attribute/src/Database/Migrations/2021_04_06_122751_create_attribute_values_table.php
`[V]` https://support.sugarcrm.com/documentation/sugar_developer/sugar_developer_guide_14.0/data_framework/vardefs/manually_creating_custom_fields/
`[V·código]` https://raw.githubusercontent.com/Dolibarr/dolibarr/develop/htdocs/install/mysql/tables/llx_societe_extrafields.sql
`[V·código]` https://raw.githubusercontent.com/espocrm/espocrm/master/application/Espo/Core/Utils/Database/Schema/ColumnPreparators/MysqlColumnPreparator.php
`[V·código]` https://raw.githubusercontent.com/frappe/frappe/develop/frappe/database/schema.py (líneas 487‑510)
`[V·código]` https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-server/src/database/commands/workspace-export/utils/generate-workspace-schema-ddl.util.ts
`[V·código]` https://raw.githubusercontent.com/odoo/odoo/20.0/addons/base_sparse_field/models/models.py
`[V·código]` https://raw.githubusercontent.com/civicrm/civicrm-core/master/sql/civicrm_generated.mysql (líneas 12928, 12952, 12975)

### 2.2 El caso Salesforce en detalle (el más instructivo)

Cita literal de la guía de arquitectura:

> *"Each row in the MT_Data table also has a Name field ... Value0 … ValueN flex columns, also known as slots, store application data that maps to the tables and fields declared in MT_Objects and MT_Fields, respectively. All flex columns use a variable-length string data type so that they can store any structured type of application data (strings, numbers, dates, and so on)."*

> *"it is not practical to create native database indexes for the flex columns of MT_Data because the platform uses a single flex column to store the data of many fields with varying structured data types. Instead, the platform manages an index of MT_Data by synchronously copying field data marked for indexing to an appropriate column in an _MT_Indexes_ pivot table."*

Traducción a decisiones de ingeniería:

1. **No hay una columna por campo custom. Hay slots.** Un `Value0` guarda el campo "RUT" de la cuenta A y el campo "Industria" del lead B. Eso es EAV con un layout físico preasignado (no filas, columnas).
2. **Los tipos se pierden.** Todo se guarda como string canónico y se castea al leer (`TO_NUMBER`, `TO_DATE`). Consecuencia: `WHERE amount_custom > 1000` no usa índice salvo que el campo esté marcado como indexado.
3. **El índice es una tabla aparte.** `_MT_Indexes_` con `StringValue`, `NumValue`, `DateValue` — es decir, **ellos también acabaron construyendo un EAV de índices**. La lección: con EAV puro, el índice deja de ser gratis y hay que mantenerlo a mano, de forma síncrona, en cada escritura.
4. **Unicidad también es tabla aparte** (`MT_Unique_Indexes`), porque un índice único sobre un slot compartido no tiene sentido.
5. **Los CLOB van fuera de línea** (`_MT_Clob_`), igual que haría Postgres con TOAST.

Y aun así, Salesforce **limita a 800 campos custom por objeto** (500 en Enterprise, +100 de paquetes gestionados, tope duro ~900 por org):

`[V·sec]` https://help.salesforce.com/s/articleView?id=platform.custom_field_allocations.htm&type=5 · tabla por edición en https://help.salesforce.com/s/articleView?id=005223994

**Esto es el dato más importante de la sección:** el producto con la arquitectura más elástica del mercado (slots, no columnas) **igual pone un límite de 800**. El límite no es del motor de base de datos: es de **usabilidad y de coste de mantener índices**. Un CRM con 5.000 campos custom por objeto es un CRM que nadie entiende.

### 2.3 Comparativa de costes reales

| Dimensión | EAV (tabla valores) | JSONB (columna) | DDL por tenant (columna real) |
|---|---|---|---|
| **Lectura de 1 registro** | N joins o 1 query extra + pivot en app | 0 coste extra (misma fila) | 0 coste extra |
| **List view filtrado por 1 campo custom** | 1 join + índice `(entity_id, attribute_id)`; selectividad mala | `col ? 'k' AND col->>'k' = 'v'` → **seq scan** salvo índice de expresión | índice btree normal, selectividad real |
| **List view ordenado por campo custom** | requiere pivot en SQL (`crosstab`/agregación) o sort en app | ordena por `col->>'k'` (texto, colación rara) | `ORDER BY col` con índice |
| **Filtro por rango numérico/fecha** | necesita columna tipada (`numeric_value`) y cast | necesita índice de expresión con cast — y si el dato es mixto, error | nativo |
| **Nº de filtros simultáneos** | coste ~lineal en joins (3 filtros = 3 joins) | constante | constante |
| **Índices** | 1 índice por `(entity_id, attribute_id)` + 1 por columna tipada; no cubre rangos | GIN (contención) o 1 índice de expresión **por campo filtrado** | 1 índice por campo filtrado, creado por la misma migración |
| **Añadir campo** | `INSERT` en tabla de metadatos. Sin lock. Milisegundos | `INSERT` en metadatos. Sin lock | `ALTER TABLE ADD COLUMN`. **ACCESS EXCLUSIVE** (rápido en PG11+ si es NULL sin default, pero bloquea) |
| **Migrar N tenants** | 1 migración de esquema (compartido) | 1 migración de esquema | **N migraciones**, con reintentos, idempotencia y ventana de mantenimiento |
| **Límite práctico** | ilimitado en teoría; el nº de joins te mata a ~20 campos filtrados | ~cientos por fila, pero un JSONB >2 KB va a TOAST (lectura extra) | **1.600 columnas por tabla** en PostgreSQL `[V]` https://www.postgresql.org/docs/current/limits.html · 1.017 en InnoDB |
| **Integridad** | sin FK, sin NOT NULL, sin FK real (salvo tabla por tipo) | sin constraints (salvo CHECK con expresión) | FK, NOT NULL, CHECK, UNIQUE, todo gratis |
| **Renombrar un campo** | `UPDATE` de metadatos. Trivial | `UPDATE` de metadatos. Trivial (pero hay que migrar claves si usas el nombre como key) | `ALTER TABLE RENAME COLUMN` (Odoo lo **prohíbe** explícitamente en sparse: *"Renaming sparse field is not allowed"*) |
| **Borrar un campo** | `DELETE` de filas | dejas la clave huérfana | `DROP COLUMN` (destructivo, bloqueante) |
| **Aislamiento de tenant** | 1 `tenant_id` en la tabla de valores | 1 fila | **físico** (schema distinto) |

### 2.4 Límites documentados (para tener números en la cabeza)

| Producto | Límite | Fuente |
|---|---|---|
| Salesforce | 500 custom fields/objeto (Enterprise), **800** (Unlimited/Performance), +100 de paquetes, tope duro ~900/org. 200 campos auditables con Field Audit Trail | `[V·sec]` help.salesforce.com |
| HubSpot | **1.000 propiedades por objeto**, **10.000 custom properties por cuenta**, 20 objetos custom (Enterprise), 15.000.000 registros por objeto, 50.000 asociaciones por registro | `[V]` https://developers.hubspot.com/docs/api-reference/latest/crm/limits-tracking/guide |
| PostgreSQL | **1.600 columnas por tabla**, limitadas además por el tamaño de tupla en una página | `[V]` https://www.postgresql.org/docs/current/limits.html |
| Twenty | sin límite documentado; práctico = 1.600 − columnas estándar | `[I]` |
| EspoCRM / SuiteCRM / Frappe / Odoo | sin límite documentado; práctico = límite de columnas del motor | `[I]` |
| Krayin (EAV) | sin límite (1 fila por valor) | `[V·código]` |

### 2.5 Veredicto: ¿EAV, JSONB o DDL?

**Ninguno de los tres, en estado puro. Usa tres capas.** Concretamente:

```
Capa 1 — Campos "calientes" (≈60-80 por objeto): COLUMNAS REALES.
          Los que aparecen en el list view, en los filtros guardados,
          en los kanban y en los reportes. Creados por migración versionada,
          no en runtime.

Capa 2 — Campos "fríos" (el resto): UNA columna JSONB en la misma tabla.
          custom jsonb NOT NULL DEFAULT '{}'.
          GIN para búsqueda de contenido (custom @> '{"rut":"123"}').

Capa 3 — Campos custom que el cliente quiere filtrar/ordenar:
          ÍNDICE DE EXPRESIÓN creado al guardar la definición del campo:
          CREATE INDEX ON deals ((custom->>'rut')) WHERE deleted_at IS NULL;
```

Por qué esta combinación y no otra:

- **EAV puro está descartado** para el list view. El list view es la consulta que manda (§6) y con EAV cada filtro añade un join. Salesforce, que tiene dinero infinito, tuvo que montar `_MT_Indexes_` para sobrevivir; tú no vas a montar eso.
- **JSONB puro está descartado** para los campos calientes, por dos motivos medibles: (a) `col->>'x'` devuelve `text`, así que un filtro por fecha o por monto exige cast y pierdes el índice salvo expresión; (b) PostgreSQL **no tiene estadísticas por clave de JSONB**, así que el planner estima mal y elige nested loops catastróficos en cuanto la tabla crece.
- **DDL en runtime está descartado** si tu producto es multi-tenant SaaS con más de ~50 tenants: N × `ALTER TABLE`, cada uno con lock exclusivo, sin transacción global, y con el riesgo de dejar tenants en esquemas distintos (drift). Twenty lo hace y por eso necesita un motor de migración de workspace completo (`workspace-migration-builder` + `runner` + acciones idempotentes).
- **DDL en runtime es la mejor opción** si el producto se despliega on-premise por cliente (1 tenant por instalación): SugarCRM, EspoCRM, Dolibarr, Frappe y Odoo llevan 15 años demostrando que funciona, y te da constraints, FKs e índices gratis.

**Regla de decisión por número de tenants:**

| Tenants | Estrategia |
|---|---|
| 1 (on-premise / self-host por cliente) | DDL real por campo. Esquema versionado + `ALTER TABLE` en la migración del release. |
| 2 – 200 | Schema por tenant + DDL. Ventana de mantenimiento aceptable; `DROP SCHEMA` para borrar un cliente es oro. |
| 200 – 10.000 | Tablas compartidas con `tenant_id`, campos calientes como columnas, resto JSONB + índices de expresión. |
| > 10.000 | Igual que arriba + particionado por `tenant_id` (hash) y, si el volumen lo pide, un índice pivote tipo `MT_Indexes`. |

---

## 3. Multi-tenancy

### 3.1 Qué hace cada producto (verificado)

| Producto | Modelo | Evidencia |
|---|---|---|
| **Salesforce** | **1 base de datos, 1 esquema, filas de todos los tenants**, particionado físicamente por `OrgID`, con "partition pruning" | `[V]` architect.salesforce.com: *"A single shared multitenant database with a single schema"* + *"All the platform data, metadata, and pivot table structures ... are physically partitioned by OrgID"* |
| **Twenty** | **Schema por workspace** dentro de una misma base: `workspace_<base36(uuid)>` | `[V·código]` `getWorkspaceSchemaName = (workspaceId) => \`workspace_${uuidToBase36(workspaceId)}\`` — https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-server/src/engine/workspace-datasource/utils/get-workspace-schema-name.util.ts |
| **Frappe / ERPNext** | **Base de datos por tenant** ("site") | `[V]` https://docs.frappe.io/framework/user/en/basics/sites — *"Frappe is a multitenant platform and each tenant is called a site. A site has its own database."* |
| **Odoo** | **Base de datos por instalación/tenant**; multi-*company* dentro de la misma base con `company_id` + record rules | `[V]` https://www.odoo.com/documentation/18.0/applications/general/companies/multi_company.html — *"multiple companies can be configured under one database"* |
| **EspoCRM / SuiteCRM / Vtiger / Krayin / Monica** | **Single-tenant por instalación** (no hay noción de tenant en el esquema) | `[I]` por ausencia de `tenant_id` en todos los esquemas revisados |
| **Dolibarr** | Single-tenant por instalación, pero con **`entity`** (multi-company) en cada tabla | `[V·código]` `llx_societe.entity integer DEFAULT 1 NOT NULL -- multi company id` |
| **HubSpot** | Multi-tenant compartido (`portal_id`) | `[I]` |

### 3.2 Implicaciones operativas

| Tarea | DB por tenant | Schema por tenant | Fila compartida (`tenant_id`) |
|---|---|---|---|
| **Respaldo de 1 cliente** | `pg_dump -d db` → limpio y completo | `pg_dump -n workspace_x` → limpio | `pg_dump --where` (no existe) → **hay que escribir exportador propio** |
| **Restaurar 1 cliente** | `createdb` + restore | `CREATE SCHEMA` + restore. Cuidado con los `CREATE TYPE` (van por schema, bien) | Filtrar e insertar respetando FKs → **doloroso y propenso a error** |
| **Borrar 1 cliente (GDPR)** | `DROP DATABASE` → 1 comando | `DROP SCHEMA ... CASCADE` → 1 comando | `DELETE FROM` 80 tablas en orden de FK, o `ON DELETE CASCADE` desde `tenants` |
| **Migración de esquema** | N veces, una por base; paralelizable; una base rota no afecta al resto | N veces, un `ALTER` por schema; mismo DB, locks que afectan a otros tenants | **1 vez**. Esta es la gran ventaja |
| **Aislamiento** | Total (proceso/DB separado) | Alto (permisos por schema) | Lógico: **un bug de `WHERE tenant_id` filtra datos de otro** |
| **Noisy neighbour** | Ninguno | Medio (mismo WAL, mismo buffer pool, mismo `shared_buffers`) | Alto |
| **Conexiones** | 1 pool por tenant (o pool compartido con `dbname` dinámico) | 1 pool, `SET search_path` por request | 1 pool, sin trucos |
| **Reportes cross-tenant** | Imposible sin ETL | Posible con `UNION ALL` sobre schemas | Trivial |
| **Coste por cliente pequeño** | Alto (overhead de base ~10 MB + WAL) | Bajo | Mínimo |

### 3.3 Veredicto

- Si el producto se **vende on-premise o self-hosted por cliente**: no inventes multi-tenancy. Añade igual `tenant_id`/`entity` a las tablas (como Dolibarr con `entity DEFAULT 1`), porque es gratis hoy y te salva el día que quieras lanzar un SaaS.
- Si es **SaaS con clientes que personalizan mucho** (campos custom, pipelines, permisos): **schema por tenant** es el mejor punto medio. Es la decisión de Twenty y es defendible: `DROP SCHEMA` para el churn, `pg_dump -n` para el backup selectivo, y aislamiento real sin pagar una base por cliente.
- Si es **SaaS de cientos de miles de cuentas** (freemium, autoservicio): **fila compartida + `tenant_id` + particionado**. Es la decisión de Salesforce y no es por elegancia: es la única que escala económicamente.
- **Nunca** "schema por tenant" con campos custom en JSONB (lo peor de ambos: pagas el fan-out de migraciones y no aprovechas las columnas reales).

El detalle que casi nadie prevé en schema-por-tenant: **las secuencias, tipos ENUM y extensiones también son por schema**. Twenty genera `CREATE TYPE workspace_x."opportunity_stage_enum" AS ENUM (...)` por workspace — `[V·código]` `generateWorkspaceSchemaDdl`. Si usas `ENUM` para las etapas, cada tenant tiene su copia y no puedes comparar pipelines entre tenants sin castear a texto.

---

## 4. Historial y auditoría

### 4.1 Tres niveles distintos (no los confundas)

| Nivel | Pregunta que responde | Volumen | Retención típica |
|---|---|---|---|
| **1. Audit columns** (`created_at`, `created_by`, `updated_at`, `updated_by`) | "¿quién tocó esto por última vez?" | 1 fila | para siempre |
| **2. Field history** (por campo auditado) | "¿quién cambió la etapa y cuándo?" | 1 fila por cambio de campo | 18–24 meses (12 meses con FAT hasta 10 años) |
| **3. Activity/timeline** | "¿qué pasó con este cliente?" (llamadas, emails, notas, cambios) | 1 fila por evento | para siempre |

### 4.2 Cómo lo hace cada producto

**Salesforce (field history):**
- Cada objeto tiene un objeto hijo `<Object>History` (p.ej. `OpportunityHistory`). La doc de Opportunity expone relaciones `LastAmountChangedHistory` / `LastCloseDateChangedHistory` que apuntan a `OpportunityHistory` — `[V]` Object Reference de Opportunity.
- **20 campos auditables por objeto** por defecto; **200** con Field Audit Trail. Retención por defecto **18 meses** (mínimo 1, máximo 18 en el objeto de política), y con FAT se archiva hasta 10 años.
- `[V]` https://developer.salesforce.com/docs/platform/field-history-retention/guide/field-audit-trail.html · `[V·sec]` https://developer.salesforce.com/blogs/isv/2015/04/field-audit-trail-briefing-isvs (*"up to 20 fields per object ... stored in a related object, `<object>__history`, for up to 18 months"*)
- La escritura es **asíncrona**: *"the system asynchronously records information about the changes made to the field (old and new values, change date, and so on) using an internal pivot table as an audit trail"* — `[V]` architect.salesforce.com.

Ese "asíncrono" es una decisión de coste enorme: no pagas la escritura de auditoría en la latencia del usuario, pero **puedes perder eventos** si el proceso muere y **no puedes garantizar lectura-de-lo-escrito**.

**EspoCRM:** el flag es por campo, en los metadatos: `"amount": {"type":"currency", "audited": true}`, `"stage": {..., "audited": true}`, `"closeDate": {"type":"date","required":true,"audited":true}` — `[V·código]` Opportunity.json. El resultado se ve en el *stream* del registro.

**Twenty:** doble mecanismo. (a) Flag `isAuditLogged` en `fieldMetadata` (existe un upgrade command dedicado: `ADD_FIELD_METADATA_IS_AUDIT_LOGGED_UPGRADE_COMMAND_NAME`) — `[V·código]` `field-metadata.entity.ts`. (b) Tabla `timelineActivity` con `happensAt`, `properties jsonb`, `timelineActivityTypeId`, `linkedRecordId`, `workspaceMemberId` — `[V·código]` `timeline-activity.workspace-entity.ts`. Es decir: **un solo log de eventos** que sirve de timeline y de auditoría, con el payload en JSONB.

**HubSpot:** historial por propiedad vía `propertiesWithHistory` en la lectura, con timestamps y fuente — `[V]` guía de Leads API.

**Dolibarr:** no tiene field history; tiene **tombstones** en `llx_deletion_log` (append-only, purgado a los `MAIN_DELETION_LOG_RETENTION_DAYS` días) con `entity`, `fk_actioncomm`, `uid`, `fk_user_action`, `assigned_users`, `date_deletion`, `fk_user`. Sirve para que un cliente de calendario sepa **qué desapareció** — `[V·código]` `llx_deletion_log.sql`.

### 4.3 DDL recomendado para auditoría

```sql
-- 1. Historial de campos: append-only, una fila por campo cambiado
CREATE TABLE audit_log (
  id           bigserial PRIMARY KEY,
  tenant_id    uuid NOT NULL,
  entity_type  text NOT NULL,
  entity_id    uuid NOT NULL,
  field        text NOT NULL,
  old_value    jsonb,          -- jsonb y no text: guarda números como números
  new_value    jsonb,
  changed_by   uuid,           -- NULL = proceso/sistema
  changed_at   timestamptz NOT NULL DEFAULT now(),
  source       text NOT NULL DEFAULT 'ui',  -- ui|api|import|workflow|system
  txid         bigint DEFAULT txid_current(),
  request_id   text
) PARTITION BY RANGE (changed_at);

CREATE INDEX ON audit_log (entity_type, entity_id, changed_at DESC);
CREATE INDEX ON audit_log (tenant_id, changed_at DESC);

-- 2. Snapshots completos (para "ver la ficha como estaba el 3 de marzo")
CREATE TABLE record_versions (
  id bigserial PRIMARY KEY,
  entity_type text NOT NULL, entity_id uuid NOT NULL,
  version int NOT NULL,
  snapshot jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  created_by uuid,
  UNIQUE (entity_type, entity_id, version)
);

-- 3. Historial de etapa, desnormalizado y barato (el que usan los reportes)
CREATE TABLE deal_stage_history (
  id bigserial PRIMARY KEY,
  deal_id uuid NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
  from_stage_id uuid REFERENCES pipeline_stages(id),
  to_stage_id   uuid NOT NULL REFERENCES pipeline_stages(id),
  entered_at timestamptz NOT NULL DEFAULT now(),
  left_at    timestamptz,          -- NULL = etapa actual
  changed_by uuid,
  seconds_in_stage bigint GENERATED ALWAYS AS
    (EXTRACT(EPOCH FROM (left_at - entered_at))::bigint) STORED
);
CREATE INDEX ON deal_stage_history (deal_id, entered_at DESC);
```

**Por qué `deal_stage_history` aparte y no solo `audit_log`:** "tiempo promedio en Negociación" es un reporte que se pide siempre y que sobre `audit_log` obliga a un self-join con window functions. Una tabla dedicada con `left_at` y `seconds_in_stage` generada convierte ese reporte en un `AVG()`.

**Reglas duras del `audit_log`:**
1. `REVOKE UPDATE, DELETE ON audit_log FROM app_role;` — solo `INSERT`.
2. Particionar por mes y hacer `DETACH` + archivado a object storage a los 24 meses.
3. `old_value`/`new_value` en `jsonb`, no `text`: si guardas `text`, "100" y "100.00" son distintos y los diffs mienten.
4. Un trigger genérico por tabla que lea `to_jsonb(NEW) - to_jsonb(OLD)`. Ojo: es fácil hacerlo mal y duplicar filas si el UPDATE no cambió nada → comparar `IS DISTINCT FROM` campo a campo.

### 4.4 Soft delete: `deleted_at` vs tabla de papelera

| Producto | Mecanismo | Evidencia |
|---|---|---|
| **Salesforce** | Columna `IsDeleted` en `MT_Data` + **Recycle Bin 15 días**; API `getDeleted()` para sync de tombstones; `undelete()` | `[V]` architect.salesforce.com (*"the platform simply marks the row for deletion by modifying the row's IsDeleted field ... restore ... for up to 15 days"*) + Object Reference (`getDeleted`, `undelete`) |
| **EspoCRM** | Columna booleana `deleted` en cada tabla (visible en los where-clauses de los entityDefs: `"deleted": false`) | `[V·código]` Contact.json |
| **Twenty** | `deletedAt: string \| null` en cada entity | `[V·código]` company/person/opportunity workspace-entity |
| **HubSpot** | "recycling bin", restaurable | `[V]` guía de Leads API (*"will add the lead to the recycling bin"*) |
| **Dolibarr** | Borrado real + `llx_deletion_log` (tombstone) | `[V·código]` |
| **Odoo** | `active boolean` (archivar), no borrar | `[I]` convención de Odoo; no hay columna `deleted` en `res.partner` |

**Por qué casi nunca se borra de verdad:**

1. **Integridad referencial.** Un contacto con 3 deals, 40 actividades y 200 filas de auditoría. `ON DELETE CASCADE` te borra la historia comercial de la empresa.
2. **Sincronización.** Si un integrador hace pull incremental, un borrado físico es invisible. Necesitas un tombstone (`getDeleted()` de Salesforce, `llx_deletion_log` de Dolibarr) o el integrador resucita el registro en el siguiente sync.
3. **Undo.** El usuario borra "Falabella S.A." y a los 3 segundos dice "no, era la otra". Deshacer es más barato que restaurar un backup.
4. **Legal.** Muchos contratos exigen conservar registros comerciales N años.
5. **Coste.** `DELETE` de una fila con 12 índices es más caro que `UPDATE deleted_at`.

**El patrón correcto:**

```sql
deleted_at timestamptz,          -- NULL = vivo
deleted_by uuid,

-- índice parcial: el list view solo ve vivos y el índice es 60% más pequeño
CREATE INDEX idx_deals_live ON deals (tenant_id, updated_at DESC)
  WHERE deleted_at IS NULL;

-- ¡CRÍTICO! las unicidades también deben ser parciales
CREATE UNIQUE INDEX uq_contacts_email
  ON contacts (tenant_id, lower(email))
  WHERE deleted_at IS NULL;
```

Sin el `WHERE deleted_at IS NULL` en la unique constraint, **no puedes volver a crear un contacto con el email de uno borrado**. Es el bug de soft-delete más común y aparece siempre en producción, nunca en tests.

Y el contrapunto: **GDPR no se resuelve con soft delete.** Necesitas un job de *pseudonimización*: `UPDATE contacts SET email = NULL, phone = NULL, name = '[redactado]', anonymized_at = now() WHERE id = ...`, manteniendo la fila y el `audit_log`. Borrar la fila rompe la auditoría; no borrar nada viola la ley.

---

## 5. Dinero y fechas

### 5.1 Cómo guarda cada uno los montos

| Producto | Tipo de columna | Moneda | Evidencia |
|---|---|---|---|
| **Odoo** | `numeric` (`_column_type = ('numeric','numeric')`) | `Many2one(res.currency)` por registro, vía `currency_field` | `[V·código]` `odoo/orm/fields_numeric.py`, clase `Monetary`: *"Encapsulates a float expressed in a given res_currency ... The decimal precision and currency symbol are taken from the currency_field attribute"* |
| **EspoCRM** | `currency` (decimal) **+ `amountConverted`** (a moneda base) **+ `amountWeightedConverted`** (calculado en SQL, no almacenado) | `amountCurrency` (FK a `Currency`), tabla `CurrencyRecordRate(record, baseCode, date, rate numeric(20,8))` | `[V·código]` Opportunity.json + CurrencyRecordRate.json |
| **Salesforce** | `currency` (Amount), con multi-currency: `CurrencyIsoCode` por registro + `DatedConversionRate` | por registro | `[V]` Object Reference (campo de tipo `currency` en Opportunity) · `[I]` nombres exactos de los campos de multidivisa |
| **HubSpot** | `amount` numérico + propiedad de moneda del deal | por registro | `[I]` |
| **Krayin** | `decimal(12,4)` — **sin columna de moneda** | ✗ | `[V·código]` `create_leads_table.php`: `$table->decimal('lead_value', 12, 4)->nullable();` |
| **Dolibarr** | `double(24,8)` para `total_ht`, `total_tva`, `total_ttc` + `fk_currency varchar(3)` + `multicurrency_total_ht/ttc` | por registro | `[V·código]` `llx_propal.sql` |

**Dos errores reales, documentados en el código:**

1. **Krayin:** `lead_value decimal(12,4)` sin moneda. No puedes distinguir 1.000 USD de 1.000 CLP. En un CRM multi-país eso es un bug de negocio, no de esquema.
2. **Dolibarr:** `double(24,8)` para totales monetarios. `double` es binario: `0.1 + 0.2 != 0.3`. Al sumar 10.000 líneas de propuesta, los totales divergen centavos y el cliente de contabilidad lo detecta. `[V·código]` `llx_propal.sql`.

### 5.2 `numeric` vs entero en centavos

| | `numeric(18,4)` | `bigint` en unidades mínimas |
|---|---|---|
| Exactitud | Exacta (decimal) | Exacta (entero) |
| Multi-moneda con exponentes distintos | Trivial: el exponente lo da la moneda | **Complejo**: necesitas tabla `currency(iso, exponent)` porque JPY tiene 0 decimales, CLP 0, KWD 3 |
| Precio unitario con 4-6 decimales | Natural | Necesitas escala fija (micro-unidades) |
| `AVG`, porcentajes, prorrateo | Natural | Requiere convertir a numeric de todos modos |
| Rendimiento | ~2× más lento que bigint en agregaciones masivas | Más rápido |
| Riesgo de bug | Bajo | Alto (olvidar multiplicar/desplazar) |
| Formato / JSON | Serializa como string en muchos drivers (¡ojo!) | Entero limpio |

**Decisión: `numeric(18,4)` para montos y `numeric(18,8)` para tipos de cambio.** Un CRM no es un libro mayor con millones de asientos por segundo; la legibilidad y la multi-moneda valen más que el 2× en agregaciones. (Si algún día haces facturación electrónica con libros de IVA, ahí sí bigint o un ledger aparte.)

Nunca `float`/`double`/`real`. Nunca `money` (tiene precisión fija por locale y rompe en multi-moneda).

### 5.3 Multi-moneda: el esquema correcto

```sql
CREATE TABLE currencies (
  code char(3) PRIMARY KEY,      -- ISO 4217
  exponent smallint NOT NULL,    -- 2 EUR/USD, 0 CLP/JPY, 3 KWD
  symbol text
);

-- tipo de cambio CON FECHA (no un valor global que se sobreescribe)
CREATE TABLE fx_rates (
  base char(3) NOT NULL REFERENCES currencies(code),
  quote char(3) NOT NULL REFERENCES currencies(code),
  rate_date date NOT NULL,
  rate numeric(18,8) NOT NULL CHECK (rate > 0),
  source text,                   -- 'BCCh' | 'manual' | 'ECB'
  PRIMARY KEY (base, quote, rate_date)
);
```

Y en `deals` / `quotes` / `invoices` **siempre las cinco columnas**:

```sql
amount        numeric(18,4) NOT NULL,   -- en la moneda del registro
currency      char(3) NOT NULL REFERENCES currencies(code),
amount_base   numeric(18,4) NOT NULL,   -- congelado al momento de la escritura
fx_rate       numeric(18,8) NOT NULL,   -- el rate usado para amount_base
fx_rate_date  date NOT NULL             -- de qué día salió ese rate
```

**Por qué congelar `amount_base` y no convertir al leer:**

- El tipo de cambio cambia. Si conviertes al leer con el rate de hoy, el forecast del mes pasado **cambia cada día** y los reportes dejan de ser auditables.
- Con `amount_base` congelado, el total del pipeline de octubre es estable y reproducible.
- Necesitas `fx_rate` y `fx_rate_date` guardados junto al monto: sin ellos no puedes reconstruir la conversión ni defenderla ante un auditor.

EspoCRM implementa exactamente esta idea con `amount` (original) + `amountConverted` (a moneda base), y calcula `amountWeightedConverted` (monto × probabilidad × rate) **en SQL, no lo almacena**:

```json
"amountWeightedConverted": {
  "type":"float","readOnly":true,"notStorable":true,
  "select": {"select":"DIV:(MUL:(amount, probability, amountCurrencyRate.rate), 100)", ...}
}
```

`[V·código]` Opportunity.json. Es una decisión correcta: el ponderado depende de `probability`, que cambia; si lo almacenas, tienes que recalcularlo en cada edición.

**Regla de reporte:** todo reporte muestra **dos cifras**: en moneda del registro y en moneda base, y declara la fecha de conversión. Si no, el CFO y el vendedor discuten sobre números que son ambos correctos.

### 5.4 Zonas horarias: el caso "la venta de las 22:30"

**El escenario, con números:**

Un vendedor en Santiago cierra un deal el **16 de noviembre de 2026 a las 22:30** (America/Santiago, UTC−3 en noviembre → horario de verano chileno).

| Representación | Valor | Quién lo ve mal |
|---|---|---|
| UTC | `2026-11-17 01:30:00+00` | Correcto como instante |
| Render en Santiago | 16 nov 22:30 | ✓ |
| Render en Madrid (UTC+1) | **17 nov 02:30** | El CEO en Madrid ve la venta **al día siguiente** |
| Render en Ciudad de México (UTC−6) | 16 nov 19:30 | ✓ día, ✗ hora |
| Guardado como `timestamp` sin tz con hora local | `2026-11-16 22:30` | Al leerlo un usuario en Madrid: 22:30 (7 horas de error) o 23:30 si el driver aplica UTC a ciegas |

**Las tres consecuencias no obvias:**

1. **El "día" del reporte no es universal.** "Ventas del 16 de noviembre" da 12 resultados en Santiago y 11 en Madrid. Solución: el tenant define `reporting_timezone`, y los reportes hacen `(closed_at AT TIME ZONE tenants.reporting_timezone)::date`. **Nunca** `closed_at::date` (eso usa el timezone del servidor de base de datos) y **nunca** el timezone del usuario que consulta.
2. **`close_date` no es un instante, es una fecha.** "Cierra el 30 de noviembre" es una fecha de calendario. Si la guardas como `timestamptz` a medianoche UTC, en Chile se muestra el 29 de noviembre y en Tokio el 30 a las 09:00. Por eso **`close_date date NOT NULL`** y no `timestamptz`.
3. **Guarda la zona original del evento.** Si el usuario dice "reunión el 16 a las 22:30", necesitas saber que **era** 22:30 en Santiago. `timestamptz` guarda el instante pero pierde la intención: si Chile cambia su regla de DST (lo ha hecho), el render futuro cambia y la reunión aparece a las 21:30.

**DDL:**

```sql
CREATE TABLE users (
  id uuid PRIMARY KEY,
  timezone text NOT NULL DEFAULT 'UTC'   -- IANA: 'America/Santiago'
    CHECK (timezone IN (SELECT name FROM pg_timezone_names))
);

CREATE TABLE tenants (
  id uuid PRIMARY KEY,
  reporting_timezone text NOT NULL DEFAULT 'UTC'
);

CREATE TABLE activities (
  ...
  occurred_at    timestamptz NOT NULL,   -- el instante (verdad única)
  occurred_local timestamp   NOT NULL,   -- "22:30" tal como lo dijo el humano
  occurred_tz    text        NOT NULL,   -- "America/Santiago"
  ...
);
```

`occurred_local` + `occurred_tz` es redundante **a propósito**: `occurred_at` es para ordenar y filtrar; `occurred_local`+`occurred_tz` es para mostrar y para reconstruir la intención si la base de zonas horarias cambia.

**Comparativa de lo que hace cada producto:**

| Producto | Manejo de TZ | Veredicto |
|---|---|---|
| **PostgreSQL `timestamptz`** | Guarda UTC, convierte en la sesión | Correcto si fijas `TimeZone` de sesión explícitamente |
| **Odoo** | Columna `timestamp` **sin** tz, convención: se guarda en UTC y se convierte con el tz del usuario en la capa ORM (`context_timestamp`) | Funciona, pero el dato crudo es ambiguo: quien lea la DB sin Odoo se equivoca |
| **EspoCRM** | `Settings.timeZone` **global del sistema**; no hay timezone por usuario (el entityDefs de `User` no tiene campo de zona) | **Limitación real** para equipos distribuidos: todos ven la hora del servidor |
| **Salesforce** | TZ por usuario, `Date` vs `DateTime` como tipos distintos | Referencia a copiar |
| **Twenty** | `createdAt: string` + `timestamptz` en columnas de fecha-hora; `date` para fechas | `[I]` por `fieldMetadataTypeToColumnType`: `DATE_TIME → timestamptz`, `DATE → date` — el mapeo correcto |

`[V·código]` https://raw.githubusercontent.com/espocrm/espocrm/master/application/Espo/Resources/metadata/entityDefs/Settings.json (`timeZone`, default `"UTC"`) · `[V·código]` `User.json` sin campo de zona · `[V·código]` https://raw.githubusercontent.com/odoo/odoo/20.0/odoo/orm/fields_temporal.py (`class Datetime: _column_type = ('timestamp','timestamp')`) · `[V·código]` `field-metadata-type-to-column-type.util.ts`

**Regla de oro:** en PostgreSQL **nunca** uses `timestamp` (sin tz). Usa `timestamptz` para instantes y `date` para fechas de calendario. No existe un tercer caso.

---

## 6. Índices y consultas: el list view manda el diseño

### 6.1 La consulta que define tu esquema

El 80% de las queries de un CRM es esto:

```sql
SELECT d.id, d.name, d.amount, d.currency, d.close_date,
       s.name AS stage, u.name AS owner, c.name AS company
FROM deals d
JOIN pipeline_stages s ON s.id = d.stage_id
JOIN users u           ON u.id = d.owner_id
JOIN companies c       ON c.id = d.company_id
WHERE d.tenant_id = $1
  AND d.deleted_at IS NULL
  AND d.owner_id   = ANY($2)      -- "mis deals" o "los del equipo"
  AND d.stage_id   = ANY($3)      -- columnas visibles del kanban
  AND d.close_date BETWEEN $4 AND $5
ORDER BY d.updated_at DESC
LIMIT 50 OFFSET 0;
```

Por qué esta query manda:

1. **Es la de mayor QPS.** Se ejecuta cada vez que alguien abre la app, cambia de pestaña, arrastra una tarjeta.
2. **Tiene malísima selectividad.** "Mis deals abiertos" puede ser 1 de 3 o 1 de 3.000. El planner no puede acertar siempre.
3. **Ordena.** `ORDER BY updated_at DESC` obliga a materializar y ordenar, o a un índice que ya esté ordenado así. Con `LIMIT 50` el índice puede servir; sin él, `Sort` + `top-N heapsort` sobre 40.000 filas.
4. **Hace 3 joins para mostrar nombres.** Por eso Salesforce mantiene `MT_Name_Denorm` ("a lean data table that stores the ObjID and Name of each record ... to execute a relatively simple query") — `[V]` architect.salesforce.com. Tú necesitas lo mismo.
5. **Pagina con OFFSET.** `OFFSET 4000` lee y descarta 4.000 filas. Es el mayor asesino de latencia en un CRM.

### 6.2 Qué indexar (y qué no)

```sql
-- 1. El default del list view: "lo más reciente que puedo ver"
CREATE INDEX idx_deals_tenant_updated
  ON deals (tenant_id, updated_at DESC)
  WHERE deleted_at IS NULL;

-- 2. Los tres filtros que la gente usa de verdad, cada uno con el orden ya incluido
CREATE INDEX idx_deals_owner ON deals (tenant_id, owner_id, updated_at DESC)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_deals_stage ON deals (tenant_id, stage_id, updated_at DESC)
  WHERE deleted_at IS NULL;
CREATE INDEX idx_deals_close ON deals (tenant_id, close_date)
  WHERE deleted_at IS NULL AND won_at IS NULL AND lost_at IS NULL;

-- 3. Pipeline / kanban: etapa + orden manual de la tarjeta
CREATE INDEX idx_deals_kanban ON deals (tenant_id, pipeline_id, stage_id, position)
  WHERE deleted_at IS NULL;

-- 4. Búsqueda: columna generada + GIN (exactamente lo que hace Twenty)
ALTER TABLE deals ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('spanish', coalesce(name,'')), 'A') ||
    setweight(to_tsvector('spanish', coalesce(description,'')), 'B')
  ) STORED;
CREATE INDEX idx_deals_search ON deals USING gin (search_vector);

-- 5. Timeline (la segunda query más frecuente)
CREATE INDEX idx_activities_entity
  ON activities (tenant_id, occurred_at DESC);
```

Twenty genera precisamente eso: `tsvector` como columna `GENERATED ALWAYS AS (...) STORED` — `[V·código]` `build-sql-column-definition.util.ts` (*"parts.push(\`GENERATED ALWAYS AS (${column.asExpression})\`)"*) y `fieldMetadataTypeToColumnType` devolviendo `'tsvector'`.

**Lo que NO debes indexar:**

| No indexar | Por qué |
|---|---|
| `is_won`, `is_active`, `deleted_at` solos | Cardinalidad 2. Usa índice **parcial** (`WHERE is_won`), que es 100× más pequeño |
| `created_at` si ya tienes `updated_at` | Redundante para el 95% de los list views; añade coste de escritura |
| Cada campo custom | Solo los que aparecen en filtros guardados. Cada índice cuesta ~10% de throughput de escritura |
| `lower(email)` para búsqueda | Ya lo tienes en la unique parcial; para búsqueda usa trigram (`pg_trgm`) |
| Texto libre con `LIKE '%x%'` | Btree no sirve. Usa `pg_trgm` GIN o el `tsvector` |

### 6.3 Paginación: keyset, no offset

```sql
-- MAL: OFFSET 4000 lee 4050 filas y tira 4000
... ORDER BY updated_at DESC LIMIT 50 OFFSET 4000;

-- BIEN: keyset (cursor). Índice (tenant_id, updated_at DESC, id)
... AND (updated_at, id) < ($cursor_ts, $cursor_id)
ORDER BY updated_at DESC, id DESC LIMIT 50;
```

Requiere que el índice incluya `id` como desempate (`(tenant_id, updated_at DESC, id DESC)`). Con `updated_at` de precisión de microsegundos las colisiones son raras, pero existen en imports masivos (mismo `now()` en toda la transacción) → sin `id` en el ORDER BY, la paginación **pierde o duplica filas**.

### 6.4 Los extras que hacen que un CRM se sienta rápido

1. **Contadores desnormalizados.** El kanban muestra "12 deals / $340.000" por columna. Ese `COUNT(*) + SUM()` por columna y por usuario es inviable en vivo. Solución: tabla `pipeline_stats(tenant_id, pipeline_id, stage_id, owner_id, deal_count, amount_sum, updated_at)` recalculada por trigger o por job. Es lo que hace Salesforce con los roll-up summary fields, calculados nativamente.
2. **`name_denorm`.** Para mostrar "Falabella" en la lista de deals sin join: `deals.company_name_cache text` mantenido por trigger. Salesforce tiene una tabla entera para esto.
3. **Vistas guardadas = queries guardadas.** El "list view" no es una query ad-hoc: es una fila (`view_filters` con `field, operator, value`). Twenty tiene `view`, `viewField`, `viewFilter`, `viewSort` como entidades de primera clase — `[V·código]` imports de `field-metadata.entity.ts`. Eso te permite cachear el plan, precargar y hasta crear índices a demanda para los filtros más usados.
4. **Búsqueda global separada.** El buscador (arriba a la derecha) no debe hacer `LIKE` sobre 8 tablas. Un `tsvector` por tabla + `UNION ALL` con `LIMIT`, o un índice externo. Salesforce lo tiene fuera de la base transaccional: *"the platform uses a search engine that is separate from its transaction engine"* — `[V]` architect.salesforce.com.

### 6.5 Prueba de que el list view manda el diseño

| Decisión de esquema | ¿Por qué? |
|---|---|
| `stage_id` como FK y no `stage text` | Para filtrar por `stage_id IN (...)` con índice en vez de `stage LIKE` |
| `owner_id` en todas las tablas principales | Todo list view filtra por dueño; sin él no hay índice útil |
| `updated_at` en todas las tablas | Es el `ORDER BY` por defecto de todas las listas |
| `deleted_at` + índices parciales | La lista nunca debe ver borrados, y el índice debe ignorarlos |
| `amount_base` congelado | Para sumar sin join a `fx_rates` en cada list view |
| `search_vector` en la misma tabla | Para que el buscador no haga 6 joins |
| `position` en `deals` (int) | El kanban permite reordenar arrastrando; necesita un orden estable y barato |
| `company_name_cache` | Para no hacer join en la lista de contactos |

---

## 7. DDL real de productos

### 7.1 Dolibarr — SQL literal del repositorio

Archivo: `htdocs/install/mysql/tables/llx_socpeople.sql`
`[V·código]` https://raw.githubusercontent.com/Dolibarr/dolibarr/develop/htdocs/install/mysql/tables/llx_socpeople.sql

```sql
create table llx_socpeople
(
  rowid                integer AUTO_INCREMENT PRIMARY KEY,
  datec                datetime,
  tms                  timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  fk_soc               integer,                     -- lien vers la societe
  use_thirdparty_address smallint DEFAULT NULL,
  entity               integer DEFAULT 1 NOT NULL,  -- multi company id
  ref_ext              varchar(255),
  name_alias           varchar(255),
  fk_parent            integer NULL,

  civility             varchar(6),
  lastname             varchar(50),
  firstname            varchar(50),
  address              varchar(255),
  zip                  varchar(25),
  town                 varchar(255),
  fk_departement       integer,
  fk_pays              integer DEFAULT 0,

  geolat               double(24,8) DEFAULT NULL,
  geolong              double(24,8) DEFAULT NULL,
  geopoint             point DEFAULT NULL,
  georesultcode        varchar(16),

  birthday             date,
  poste                varchar(255),      -- 'poste' = cargo
  phone                varchar(30),
  phone_perso          varchar(30),
  phone_mobile         varchar(30),
  fax                  varchar(30),
  url                  varchar(255),
  email                varchar(255),

  socialnetworks       text DEFAULT NULL, -- json with socialnetworks
  photo                varchar(255),
  no_email             smallint NOT NULL DEFAULT 0,
  priv                 smallint NOT NULL DEFAULT 0,
  fk_prospectlevel     varchar(12),
  fk_stcommcontact     integer DEFAULT 0 NOT NULL,
  fk_user_creat        integer DEFAULT 0,
  fk_user_modif        integer,
  note_private         text,
  note_public          text,
  default_lang         varchar(6),
  canvas               varchar(32),
  import_key           varchar(14),
  statut               tinyint DEFAULT 1 NOT NULL,
  ip                   varchar(250)
) ENGINE=innodb;
```

Cosas que enseñan este DDL:
- `entity integer DEFAULT 1` — multi-company por columna, sin `tenant_id`. Barato y suficiente para on-premise.
- `tms timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` — `updated_at` gratis en MySQL (en PostgreSQL necesitas trigger).
- `fk_user_creat` / `fk_user_modif` — auditoría mínima, sin nombres: dos enteros, no dos uuids.
- `import_key varchar(14)` — **marca de importación**. Sirve para hacer rollback de una importación masiva. Detalle de producto que casi nadie modela.
- `socialnetworks text` con JSON — JSONB "a la antigua", antes de que existiera JSONB.
- `ip varchar(250)` — la IP de origen del registro (para formularios públicos / cumplimiento). Otra cosa que se olvida siempre.

Y la tabla de campos custom:
`[V·código]` `llx_societe_extrafields.sql`

```sql
create table llx_societe_extrafields
(
  rowid         integer AUTO_INCREMENT PRIMARY KEY,
  tms           timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  fk_object     integer NOT NULL,
  import_key    varchar(14)
) ENGINE=innodb;
```

**No tiene columnas de datos.** Las columnas `x_<nombre>` se añaden con `ALTER TABLE` desde el módulo de administración. Es el patrón "tabla satélite de extrafields": la tabla principal se mantiene limpia, y los campos custom van a una tabla 1:1 que crece. Es una variante muy buena del DDL dinámico: **si el `ALTER` falla o bloquea, solo bloquea la tabla de extrafields, no la de contactos**.

Propuesta concreta (ventas): `[V·código]` `llx_propal.sql`

```sql
create table llx_propal
(
  rowid              integer AUTO_INCREMENT PRIMARY KEY,
  ref                varchar(30) NOT NULL,
  entity             integer DEFAULT 1 NOT NULL,
  fk_soc             integer,
  fk_projet          integer DEFAULT NULL,
  datec              datetime,
  datep              date,             -- fecha de la propuesta (DATE!)
  fin_validite       datetime,
  date_valid         datetime,
  date_signature     datetime,
  date_cloture       datetime,
  fk_user_author     integer,
  fk_user_modif      integer,
  fk_statut          smallint DEFAULT 0 NOT NULL,  -- 0=draft 1=validated 2=accepted 3=refused 4=billed
  total_ht           double(24,8) DEFAULT 0,       -- ⚠ float para dinero
  total_tva          double(24,8) DEFAULT 0,       -- ⚠
  total_ttc          double(24,8) DEFAULT 0,       -- ⚠
  fk_currency        varchar(3),
  multicurrency_total_ht  double(24,8) DEFAULT 0,
  multicurrency_total_ttc double(24,8) DEFAULT 0
) ENGINE=innodb;
```

Lecciones: (a) separa `datep` (date) de `datec` (datetime) — la fecha comercial no es un instante; (b) el estado es un `smallint` con comentario, no un enum — cero migraciones al añadir estados; (c) `double` para dinero es un error que arrastran desde 2003.

### 7.2 Twenty — DDL generado por el código

Twenty no tiene `.sql` en el repo: **el DDL se genera**. El generador es `generateWorkspaceSchemaDdl()` y emite, por cada objeto del workspace:

```ts
// packages/twenty-server/src/database/commands/workspace-export/utils/generate-workspace-schema-ddl.util.ts
statements.push(
  `CREATE TYPE ${schemaName}.${enumName} AS ENUM (${escapedValues});`,
);
...
statements.push(
  `CREATE TABLE ${schemaName}.${tableName} (\n${columnsSql}\n);`,
);
```

`[V·código]` https://raw.githubusercontent.com/twentyhq/twenty/main/packages/twenty-server/src/database/commands/workspace-export/utils/generate-workspace-schema-ddl.util.ts

El resultado, para el objeto `company` de un workspace `workspace_3k9x2ab`, es equivalente a:

```sql
CREATE SCHEMA IF NOT EXISTS "workspace_3k9x2ab";

CREATE TABLE "workspace_3k9x2ab"."company" (
  "id"                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  "createdAt"             timestamptz NOT NULL DEFAULT now(),
  "updatedAt"             timestamptz NOT NULL DEFAULT now(),
  "deletedAt"             timestamptz,
  "name"                  text,
  "domainNamePrimaryLinkUrl"   text,        -- composite LinksMetadata → columnas planas
  "domainNamePrimaryLinkLabel" text,
  "annualRevenueAmount"   numeric,          -- composite CurrencyMetadata
  "annualRevenueCurrencyCode" text,
  "addressAddressStreet1" text,             -- composite AddressMetadata
  "addressAddressCity"    text,
  "addressAddressCountry" text,
  "position"              float NOT NULL DEFAULT 0,
  "createdBySource"       text,             -- ActorMetadata
  "createdByWorkspaceMemberId" uuid,
  "createdByName"         text,
  "searchVector"          tsvector GENERATED ALWAYS AS (...) STORED,
  "accountOwnerId"        uuid REFERENCES "workspace_3k9x2ab"."workspaceMember"("id")
);
```

Los mapeos son verificables en el código:
- `UUID → uuid`, `NUMERIC → numeric`, `NUMBER|POSITION → float`, `BOOLEAN → boolean`, `DATE_TIME → timestamptz`, `DATE → date`, `SELECT|MULTI_SELECT|RATING → enum`, `FILES|RAW_JSON → jsonb`, `TS_VECTOR → tsvector`, texto → `text`.
  `[V·código]` `field-metadata-type-to-column-type.util.ts`
- Los tipos compuestos (`LinksMetadata`, `CurrencyMetadata`, `FullNameMetadata`, `EmailsMetadata`, `PhonesMetadata`, `ActorMetadata`, `AddressMetadata`) se **aplanan** en varias columnas con sufijo pascal-case: `name + CamelCase(subcampo)`.
  `[V·código]` `compute-column-name.util.ts`: `return \`${name}${pascalCase(compositeProperty.name)}\``
- El `tsvector` es `GENERATED ALWAYS AS (...) STORED` con una expresión derivada de los campos indexados.
  `[V·código]` `build-sql-column-definition.util.ts`

Modelo de entidades (el equivalente a mi `Company`/`Person`/`Opportunity`):
`[V·código]` `company.workspace-entity.ts` / `person.workspace-entity.ts` / `opportunity.workspace-entity.ts`

```ts
export class CompanyWorkspaceEntity {
  id: string; createdAt: string; updatedAt: string; deletedAt: string | null;
  name: string | null;
  domainName: LinksMetadata;
  annualRevenue: CurrencyMetadata | null;
  address: AddressMetadata;
  position: number;
  createdBy: ActorMetadata;
  updatedBy: ActorMetadata;
  searchVector: string;
  accountOwnerId: string | null;
  // relaciones
  people: EntityRelation<PersonWorkspaceEntity[]>;
  opportunities: EntityRelation<OpportunityWorkspaceEntity[]>;
  taskTargets: EntityRelation<TaskTargetWorkspaceEntity[]>;
  noteTargets: EntityRelation<NoteTargetWorkspaceEntity[]>;
  timelineActivities: EntityRelation<TimelineActivityWorkspaceEntity[]>;
}

export class OpportunityWorkspaceEntity {
  name: string;
  amount: CurrencyMetadata | null;
  closeDate: Date | null;
  stage: string;                    // ⚠ string libre, no FK
  probability: string;              // ⚠ @deprecated
  companyId: string | null;
  pointOfContactId: string | null;  // un solo contacto
  ownerId: string | null;
  searchVector: string;
}
```

**Lo bueno de Twenty:** `deletedAt` en todo, `searchVector` generado, `createdBy`/`updatedBy` como struct (`source` + `workspaceMemberId` + `name`) que sobrevive al borrado del usuario — eso es mejor que un FK a `users` que se pone NULL.
**Lo malo:** `stage: string` sin FK a un pipeline; `pointOfContactId` único (sin N:M de contactos); `probability` como string deprecado.

### 7.3 SugarCRM / SuiteCRM — el DDL que genera la herramienta

Salida literal de *Quick Repair and Rebuild* al crear el primer campo custom de un módulo:

```sql
/*Checking Custom Fields for module : <module>*/

CREATE TABLE <module>_cstm (
  id_c char(36) NOT NULL,
  PRIMARY KEY (id_c)
) CHARACTER SET utf8 COLLATE utf8_general_ci;

/*MISSING IN DATABASE - example_c - ROW*/

ALTER TABLE <module>_cstm ADD COLUMN example_c varchar(255) NULL;
```

Y el registro de metadatos:

```sql
INSERT INTO fields_meta_data
  (id, name, vname, comments, custom_module, type, len, required,
   deleted, audited, massupdate, duplicate_merge, reportable, importable)
VALUES
  ('<module>example_c','example_c','LBL_EXAMPLE_C','Example Varchar Vardef',
   '<module>','varchar',255,0,0,0,0,0,1,'true');
```

`[V]` https://support.sugarcrm.com/documentation/sugar_developer/sugar_developer_guide_14.0/data_framework/vardefs/manually_creating_custom_fields/

Detalles de diseño notables:
- La tabla `_cstm` **solo tiene la PK** al principio. Cada campo custom añade una columna. Es una tabla 1:1 con la principal: `accounts` ←→ `accounts_cstm` por `id`.
- La columna `deleted` **existe en `fields_meta_data`**: borrar un campo es un soft delete del metadato, no un `DROP COLUMN`. Eso evita perder datos por accidente — buen patrón.
- `audited` es un campo del metadato: activar auditoría por campo es un `UPDATE` de una fila.
- El nombre del campo custom **debe** terminar en `_c` y tener `source='custom_fields'`. La convención como contrato.
- SuiteCRM es un fork de Sugar 6.5 y hereda este mismo mecanismo `[I]`, aunque su repo no incluye un dump del esquema (solo `tests/_data/dump.sql`, que es un placeholder: `/* Replace this file with actual dump of your database */`).

**Por qué es mejor que hacer `ALTER TABLE accounts` directamente:** la tabla principal nunca se bloquea por completo si el motor permite `ALTER` online en la tabla satélite, y el `SELECT *` del ORM no arrastra 60 columnas custom. Es el mismo razonamiento que Dolibarr con `_extrafields`.

### 7.4 Krayin — Laravel migrations (EAV real)

```php
// packages/Webkul/Lead/src/Database/Migrations/2021_04_22_164215_create_leads_table.php
Schema::create('leads', function (Blueprint $table) {
    $table->increments('id');
    $table->string('title');
    $table->text('description')->nullable();
    $table->decimal('lead_value', 12, 4)->nullable();     // ⚠ sin moneda
    $table->boolean('status')->nullable();
    $table->text('lost_reason')->nullable();
    $table->datetime('closed_at')->nullable();

    $table->integer('user_id')->unsigned()->index();
    $table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
    $table->integer('person_id')->unsigned();
    $table->foreign('person_id')->references('id')->on('persons')->onDelete('cascade');
    $table->integer('lead_source_id')->unsigned();
    $table->foreign('lead_source_id')->references('id')->on('lead_sources')->onDelete('cascade');
    $table->integer('lead_type_id')->unsigned();
    $table->foreign('lead_type_id')->references('id')->on('lead_types')->onDelete('cascade');
    $table->integer('lead_pipeline_id')->unsigned()->nullable();
    $table->foreign('lead_pipeline_id')->references('id')->on('lead_pipelines')->onDelete('cascade');
    $table->integer('lead_stage_id')->unsigned();
    $table->foreign('lead_stage_id')->references('id')->on('lead_stages')->onDelete('cascade');
    $table->timestamps();
});
```

Equivalente SQL:

```sql
CREATE TABLE `leads` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `title` varchar(255) NOT NULL,
  `description` text NULL,
  `lead_value` decimal(12,4) NULL,
  `status` tinyint(1) NULL,
  `lost_reason` text NULL,
  `closed_at` datetime NULL,
  `user_id` int unsigned NOT NULL,
  `person_id` int unsigned NOT NULL,
  `lead_source_id` int unsigned NOT NULL,
  `lead_type_id` int unsigned NOT NULL,
  `lead_pipeline_id` int unsigned NULL,
  `lead_stage_id` int unsigned NOT NULL,
  `created_at` timestamp NULL,
  `updated_at` timestamp NULL,
  PRIMARY KEY (`id`),
  CONSTRAINT `leads_user_id_foreign` FOREIGN KEY (`user_id`) REFERENCES `users`(`id`) ON DELETE CASCADE,
  CONSTRAINT `leads_person_id_foreign` FOREIGN KEY (`person_id`) REFERENCES `persons`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB;
```

Y el EAV de campos custom:

```sql
CREATE TABLE `attributes` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `code` varchar(255) NOT NULL,
  `name` varchar(255) NOT NULL,
  `type` varchar(255) NOT NULL,
  `lookup_type` varchar(255) NULL,
  `entity_type` varchar(255) NOT NULL,
  `sort_order` int NULL,
  `validation` varchar(255) NULL,
  `is_required` tinyint(1) NOT NULL DEFAULT 0,
  `is_unique` tinyint(1) NOT NULL DEFAULT 0,
  `quick_add` tinyint(1) NOT NULL DEFAULT 0,
  `is_user_defined` tinyint(1) NOT NULL DEFAULT 1,
  PRIMARY KEY (`id`),
  UNIQUE KEY `attributes_code_entity_type_unique` (`code`,`entity_type`)
);

CREATE TABLE `attribute_values` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `entity_type` varchar(255) NOT NULL DEFAULT 'leads',
  `text_value` text NULL,
  `boolean_value` tinyint(1) NULL,
  `integer_value` int NULL,
  `float_value` double NULL,
  `datetime_value` datetime NULL,
  `date_value` date NULL,
  `json_value` json NULL,
  `entity_id` int unsigned NOT NULL,
  `attribute_id` int unsigned NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `entity_type_attribute_value_index_unique` (`entity_type`,`entity_id`,`attribute_id`),
  CONSTRAINT `attribute_values_attribute_id_foreign` FOREIGN KEY (`attribute_id`) REFERENCES `attributes`(`id`) ON DELETE CASCADE
);
```

`[V·código]` migraciones citadas arriba.

Observaciones: (a) `double` para `float_value` (aceptable, no es dinero); (b) **8 columnas tipadas** en vez de una `value text` — eso es lo que hace que EAV sea usable: filtros numéricos usan `float_value` con índice; (c) el `UNIQUE(entity_type, entity_id, attribute_id)` es obligatorio y suele olvidarse; (d) `entity_type` como string en la tabla de valores: si un día renombras `leads`→`deals`, tienes que migrar datos.

### 7.5 CiviCRM — una tabla de valores custom por grupo

```sql
CREATE TABLE `civicrm_value_constituent_information_1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `entity_id` int(10) unsigned NOT NULL,
  `most_important_issue_1` varchar(255) DEFAULT NULL,
  `marital_status_2` varchar(255) DEFAULT NULL,
  `marriage_date_3` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `unique_entity_id` (`entity_id`),
  INDEX `INDEX_most_important_issue_1` (`most_important_issue_1`),
  CONSTRAINT `FK_..._entity_id` FOREIGN KEY (`entity_id`)
    REFERENCES `civicrm_contact` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

`[V·código]` https://raw.githubusercontent.com/civicrm/civicrm-core/master/sql/civicrm_generated.mysql (línea 12928)

Es el híbrido más elegante de los tres: **DDL real (columnas tipadas, índices, FK) pero agrupado por "custom group"**, no por objeto. Ventajas: un grupo de 15 campos se administra como una unidad; puedes borrar el grupo entero con `DROP TABLE`; el `ON DELETE CASCADE` limpia solo. Desventaja: para leer un contacto con 3 grupos custom necesitas 3 joins.

---

## 8. Síntesis: esquema recomendado

Consolidando todo lo anterior. PostgreSQL 15+.

```sql
-- ============ TENANCY ============
CREATE TABLE tenants (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  reporting_timezone text NOT NULL DEFAULT 'UTC',
  base_currency char(3) NOT NULL DEFAULT 'USD',
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE users (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  email citext NOT NULL,
  name text NOT NULL,
  timezone text NOT NULL DEFAULT 'UTC',
  is_active boolean NOT NULL DEFAULT true,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON users (tenant_id, email) WHERE is_active;

-- ============ NÚCLEO ============
CREATE TABLE companies (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name text NOT NULL,
  domain citext,
  industry text,
  employee_count int,
  annual_revenue numeric(18,4),
  annual_revenue_currency char(3),
  address jsonb,                       -- capa 2: compuesto, nunca se filtra por calle
  owner_id uuid REFERENCES users(id),
  custom jsonb NOT NULL DEFAULT '{}',  -- capa 2: campos fríos
  search_vector tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('simple', coalesce(name,'')),'A') ||
    setweight(to_tsvector('simple', coalesce(domain::text,'')),'B')
  ) STORED,
  created_at timestamptz NOT NULL DEFAULT now(),
  created_by uuid,
  updated_at timestamptz NOT NULL DEFAULT now(),
  updated_by uuid,
  deleted_at timestamptz
);
CREATE INDEX ON companies (tenant_id, updated_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX ON companies (tenant_id, owner_id) WHERE deleted_at IS NULL;
CREATE INDEX ON companies USING gin (search_vector);
CREATE UNIQUE INDEX ON companies (tenant_id, domain) WHERE deleted_at IS NULL AND domain IS NOT NULL;

CREATE TABLE contacts (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  first_name text, last_name text NOT NULL,
  email citext, phone text, mobile text, linkedin text,
  primary_company_id uuid REFERENCES companies(id),  -- denormalizado para list views
  owner_id uuid REFERENCES users(id),
  custom jsonb NOT NULL DEFAULT '{}',
  search_vector tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('simple', coalesce(first_name,'')||' '||coalesce(last_name,'')),'A') ||
    setweight(to_tsvector('simple', coalesce(email::text,'')),'B')
  ) STORED,
  created_at timestamptz NOT NULL DEFAULT now(), updated_at timestamptz NOT NULL DEFAULT now(),
  deleted_at timestamptz
);
CREATE INDEX ON contacts (tenant_id, updated_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX ON contacts (tenant_id, primary_company_id) WHERE deleted_at IS NULL;
CREATE INDEX ON contacts USING gin (search_vector);
CREATE UNIQUE INDEX ON contacts (tenant_id, lower(email))
  WHERE deleted_at IS NULL AND email IS NOT NULL;

CREATE TABLE company_contacts (
  company_id uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
  contact_id uuid NOT NULL REFERENCES contacts(id) ON DELETE CASCADE,
  role text, job_title text,
  is_primary boolean NOT NULL DEFAULT false,
  is_active boolean NOT NULL DEFAULT true,
  started_at date, ended_at date,
  PRIMARY KEY (company_id, contact_id)
);
CREATE UNIQUE INDEX ON company_contacts (contact_id) WHERE is_primary AND is_active;

-- ============ PIPELINE ============
CREATE TABLE pipelines (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name text NOT NULL,
  entity text NOT NULL DEFAULT 'deal',
  is_default boolean NOT NULL DEFAULT false
);

CREATE TABLE pipeline_stages (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  pipeline_id uuid NOT NULL REFERENCES pipelines(id) ON DELETE CASCADE,
  name text NOT NULL,
  position int NOT NULL,
  probability int NOT NULL DEFAULT 0 CHECK (probability BETWEEN 0 AND 100),
  is_won boolean NOT NULL DEFAULT false,
  is_lost boolean NOT NULL DEFAULT false,
  forecast_category text,
  UNIQUE (pipeline_id, position)
);

-- ============ DEALS ============
CREATE TABLE deals (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name text NOT NULL,
  company_id uuid REFERENCES companies(id),
  primary_contact_id uuid REFERENCES contacts(id),
  pipeline_id uuid NOT NULL REFERENCES pipelines(id),
  stage_id uuid NOT NULL REFERENCES pipeline_stages(id),
  stage_entered_at timestamptz NOT NULL DEFAULT now(),
  probability int CHECK (probability BETWEEN 0 AND 100),  -- override; NULL = usar el de la etapa
  amount numeric(18,4),
  currency char(3) NOT NULL DEFAULT 'USD',
  amount_base numeric(18,4),
  fx_rate numeric(18,8),
  fx_rate_date date,
  close_date date,
  won_at timestamptz, lost_at timestamptz, lost_reason text,
  position int NOT NULL DEFAULT 0,
  owner_id uuid REFERENCES users(id),
  custom jsonb NOT NULL DEFAULT '{}',
  search_vector tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('simple', coalesce(name,'')),'A')
  ) STORED,
  created_at timestamptz NOT NULL DEFAULT now(), updated_at timestamptz NOT NULL DEFAULT now(),
  deleted_at timestamptz,
  CHECK (amount IS NULL OR (currency IS NOT NULL AND fx_rate IS NOT NULL))
);
CREATE INDEX ON deals (tenant_id, updated_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX ON deals (tenant_id, owner_id, updated_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX ON deals (tenant_id, stage_id, updated_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX ON deals (tenant_id, pipeline_id, stage_id, position) WHERE deleted_at IS NULL;
CREATE INDEX ON deals (tenant_id, close_date) WHERE deleted_at IS NULL AND won_at IS NULL AND lost_at IS NULL;

-- ============ ACTIVIDADES ============
CREATE TABLE activities (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  kind text NOT NULL CHECK (kind IN ('note','task','call','meeting','email','system')),
  title text, body text,
  occurred_at timestamptz NOT NULL DEFAULT now(),
  occurred_local timestamp, occurred_tz text,
  due_at timestamptz, completed_at timestamptz,
  duration_minutes int,
  owner_id uuid REFERENCES users(id),
  meta jsonb,
  created_at timestamptz NOT NULL DEFAULT now(),
  deleted_at timestamptz
);
CREATE INDEX ON activities (tenant_id, occurred_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX ON activities (tenant_id, owner_id, due_at)
  WHERE kind = 'task' AND completed_at IS NULL AND deleted_at IS NULL;

CREATE TABLE activity_links (
  activity_id uuid NOT NULL REFERENCES activities(id) ON DELETE CASCADE,
  entity_type text NOT NULL,
  entity_id uuid NOT NULL,
  role text,
  PRIMARY KEY (activity_id, entity_type, entity_id)
);
CREATE INDEX ON activity_links (entity_type, entity_id, activity_id);

-- ============ HISTORIAL ============
CREATE TABLE deal_stage_history (
  id bigserial PRIMARY KEY,
  deal_id uuid NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
  from_stage_id uuid REFERENCES pipeline_stages(id),
  to_stage_id uuid NOT NULL REFERENCES pipeline_stages(id),
  entered_at timestamptz NOT NULL DEFAULT now(),
  left_at timestamptz,
  changed_by uuid
);
CREATE INDEX ON deal_stage_history (deal_id, entered_at DESC);

CREATE TABLE audit_log (
  id bigserial PRIMARY KEY,
  tenant_id uuid NOT NULL,
  entity_type text NOT NULL, entity_id uuid NOT NULL,
  field text NOT NULL,
  old_value jsonb, new_value jsonb,
  changed_by uuid, changed_at timestamptz NOT NULL DEFAULT now(),
  source text NOT NULL DEFAULT 'ui'
) PARTITION BY RANGE (changed_at);
CREATE INDEX ON audit_log (entity_type, entity_id, changed_at DESC);
```

Decisiones que este esquema toma y que puedes defender:

| Decisión | Alternativa rechazada | Por qué |
|---|---|---|
| `company_contacts` N:M con `is_primary` parcial | `contacts.company_id` 1:N | Cambio de trabajo y filiales. El único índice parcial evita 3 primarios |
| `pipeline_stages` como filas con `probability`/`is_won` | enum en `deals.stage` | Multi-pipeline, multi-tenant, forecast, sin DDL |
| `deals.close_date` como `date` | `timestamptz` | "Cierra el 30" no tiene hora; con tz se corre un día |
| `amount` + `currency` + `amount_base` + `fx_rate` + `fx_rate_date` | solo `amount` | Reportes auditables y estables |
| `activities` + `activity_links` | `parent_type`/`parent_id` | Una reunión con 4 contactos y 1 deal |
| `audit_log` particionado + `deal_stage_history` | solo audit | El reporte de tiempo-en-etapa se pide siempre |
| `deleted_at` + índices parciales + unique parciales | borrado físico | Sync, undo, integridad, y poder reusar emails |
| `custom jsonb` + índices de expresión | EAV | El list view no paga joins |
| `search_vector` generado + GIN | `LIKE '%x%'` | Búsqueda global en <50 ms |

---

## Anexo A — Fuentes primarias citadas

**Salesforce**
- Arquitectura multi-tenant (UDD, MT_Data, flex columns, MT_Indexes, Recycle Bin, auditoría asíncrona): https://architect.salesforce.com/docs/architect/fundamentals/guide/platform-multitenant-architecture.html
- Opportunity: https://developer.salesforce.com/docs/platform/object-reference/references/standard-objects/object-opportunity.html
- Account: https://developer.salesforce.com/docs/platform/object-reference/references/standard-objects/object-account.html
- Contact: https://developer.salesforce.com/docs/platform/object-reference/references/standard-objects/object-contact.html
- Lead (campos `Converted*`, `IsConverted`): https://developer.salesforce.com/docs/platform/object-reference/references/standard-objects/object-lead.html
- AccountContactRelation: https://developer.salesforce.com/docs/platform/object-reference/references/standard-objects/object-accountcontactrelation.html
- Field Audit Trail: https://developer.salesforce.com/docs/platform/field-history-retention/guide/field-audit-trail.html
- Field History (20 campos / 18 meses), blog de ISV: https://developer.salesforce.com/blogs/isv/2015/04/field-audit-trail-briefing-isvs
- Límite de campos custom: https://help.salesforce.com/s/articleView?id=platform.custom_field_allocations.htm&type=5

**HubSpot**
- Objetos, objectTypeIds, associations, properties, search: https://developers.hubspot.com/docs/guides/api/crm/understanding-the-crm
- Leads (requiere asociación a contact; se borra al quitar asociaciones): https://developers.hubspot.com/docs/api-reference/latest/crm/objects/leads/guide
- Límites (1.000 props/objeto, 10.000 total, 20 objetos custom, 15M registros): https://developers.hubspot.com/docs/api-reference/latest/crm/limits-tracking/guide

**Pipedrive**
- Leads (delgados, heredan custom fields de Deals, conversión marca el lead como borrado): https://developers.pipedrive.com/docs/api/v1/Leads

**Attio**
- Índice de API (objetos estándar: People, Companies, Users, Deals, Workspaces; sin Leads): https://docs.attio.com/rest-api/endpoint-reference
- SQL (schemas `objects`, `lists`, `activities`; una tabla por objeto): https://docs.attio.com/sql/available-data

**Twenty (código)**
- Entidades: `.../src/modules/{company,person,opportunity,note,task}/standard-objects/*.workspace-entity.ts`
- Generación de DDL: `.../src/database/commands/workspace-export/utils/generate-workspace-schema-ddl.util.ts`
- Mapeo tipo→columna: `.../workspace-migration-runner/utils/field-metadata-type-to-column-type.util.ts`
- Nombre de schema por workspace: `.../workspace-datasource/utils/get-workspace-schema-name.util.ts`
- Metadatos de campos: `.../metadata-modules/field-metadata/field-metadata.entity.ts`
- Definición de columna con `GENERATED ALWAYS AS`: `.../twenty-orm/workspace-schema-manager/utils/build-sql-column-definition.util.ts`
- Timeline: `.../modules/timeline/standard-objects/timeline-activity.workspace-entity.ts`

**EspoCRM (código)**
- `Contact.json`, `Account.json`, `Lead.json`, `Opportunity.json`, `Settings.json`, `CurrencyRecordRate.json` en `application/Espo/Modules/Crm/Resources/metadata/entityDefs/` y `application/Espo/Resources/metadata/entityDefs/`
- Schema builder: `application/Espo/Core/Utils/Database/Schema/*`

**SugarCRM**
- Creación de campos custom (`_cstm`, `fields_meta_data`): https://support.sugarcrm.com/documentation/sugar_developer/sugar_developer_guide_14.0/data_framework/vardefs/manually_creating_custom_fields/

**Odoo (código)**
- `Monetary` (numeric + `currency_field`): https://raw.githubusercontent.com/odoo/odoo/20.0/odoo/orm/fields_numeric.py
- `Datetime` (`timestamp` sin tz): https://raw.githubusercontent.com/odoo/odoo/20.0/odoo/orm/fields_temporal.py
- Campos sparse (JSONB): https://raw.githubusercontent.com/odoo/odoo/20.0/addons/base_sparse_field/models/models.py
- Multi-company: https://www.odoo.com/documentation/18.0/applications/general/companies/multi_company.html

**Frappe / ERPNext (código y docs)**
- `DBTable.sync` / `add_column` / `DEFAULT_COLUMNS`: https://raw.githubusercontent.com/frappe/frappe/develop/frappe/database/schema.py
- Un site = una base de datos: https://docs.frappe.io/framework/user/en/basics/sites

**Krayin (código)**
- `leads`, `organizations`, `persons`, `attributes`, `attribute_values`, `activities`, `lead_pipelines`, `lead_pipeline_stages` — migraciones en `packages/Webkul/{Lead,Contact,Attribute,Activity}/src/Database/Migrations/`

**Dolibarr (código)**
- `llx_societe.sql`, `llx_socpeople.sql`, `llx_societe_extrafields.sql`, `llx_propal.sql`, `llx_actioncomm.sql`, `llx_deletion_log.sql` en `htdocs/install/mysql/tables/`

**CiviCRM**
- DDL generado con tablas de valores custom: https://raw.githubusercontent.com/civicrm/civicrm-core/master/sql/civicrm_generated.mysql

**PostgreSQL**
- Límites (1.600 columnas/tabla): https://www.postgresql.org/docs/current/limits.html

---

## Anexo B — Checklist para la primera migración

1. `timestamptz` en todo lo que sea instante; `date` en lo que sea calendario. Cero `timestamp` sin tz.
2. `deleted_at` + índice parcial + **unique parcial** en toda tabla con un identificador natural.
3. `amount` + `currency` + `amount_base` + `fx_rate` + `fx_rate_date`. Nunca floats.
4. `pipeline_stages` como tabla, con `probability`, `is_won`, `is_lost`, `position`.
5. `company_contacts` N:M con `job_title` en la relación, y único parcial de primario.
6. `activity_links` con `(entity_type, entity_id)`, no `parent_type`/`parent_id`.
7. `custom jsonb` en las tablas principales desde el día uno (aunque no haya UI todavía).
8. `search_vector` generado + GIN en las 4 tablas con nombre.
9. `audit_log` append-only, particionado por mes, sin permisos de UPDATE/DELETE.
10. `deal_stage_history` con `left_at` + `seconds_in_stage` generado.
11. `owner_id` en toda entidad que aparezca en un list view.
12. Keyset pagination desde el principio (índices con `id` como desempate).
