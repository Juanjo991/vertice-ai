# 02 — Captura e ingesta: cómo entra el dato a un CRM y qué le pasa antes de guardarse

**Objetivo:** poder reconstruir desde cero cada pipeline de entrada, paso a paso, incluyendo los puntos donde se pierde o se corrompe información.

**Alcance:** CRM genérico. Fuentes primarias: Salesforce, HubSpot, Pipedrive, EspoCRM, Odoo, Gmail/Google Calendar API, Microsoft Graph, Twilio, Nylas, OpenRefine.

**Convenciones de cita**

- `[verificado + URL]` = el dato está literalmente en la documentación citada (a veces vía snapshot de Wayback Machine cuando el sitio actual es una SPA que no sirve el contenido en HTML).
- `[inferido]` = deducción de ingeniería a partir de varias fuentes o de comportamiento observado, sin una frase que lo afirme textualmente.
- Los números de límite cambian con el tiempo; se anotan con la fecha del documento consultado.

---

## 0. Resumen ejecutivo (lo que hay que saber antes de leer el resto)

1. **Todo CRM real separa tres cosas que se suelen mezclar: el payload crudo, el registro normalizado y la identidad.** Sin esas tres capas, no se puede reprocesar nada. Salesforce conserva resultados de ingesta solo 7 días; Twilio conserva el request inspector 30 días; Gmail conserva historial de cambios ~1 semana. Si no guardas el crudo, el reproceso es imposible.
2. **Los CRM grandes no deduplican por "parecido" en el camino caliente.** Salesforce compara contra *match keys* precalculados (un índice); HubSpot deduplica por email exacto (contactos) y dominio (empresas); EspoCRM arma un `WHERE` de igualdades. El fuzzy existe, pero es una capa de *sugerencia* (Salesforce: alerta de duplicado con `matchConfidence`; OpenRefine: clustering con revisión humana).
3. **La importación CSV no es un bucle "por fila → insert".** Es un job asíncrono por lotes con resultados separados en *successful / failed / unprocessed*, y el estado del job se puede reanudar.
4. **La sincronización de correo no es "polling cada minuto".** Es push (webhook) que solo dice "cambió algo" + *pull incremental* con un cursor opaco (`historyId`, `@odata.deltaLink`, `UID`). El push nunca trae el dato; el payload real siempre se vuelve a pedir.
5. **La idempotencia no se resuelve con "si existe, no insertes".** Se resuelve con una restricción única en la base + `INSERT ... ON CONFLICT`, o con una tabla de claves de idempotencia con `PRIMARY KEY`. Dos eventos en 200 ms es una carrera, no un caso borde.
6. **El dato más valioso que casi nadie guarda es el error por fila.** La fila 7.000 de 50.000 no debe detener el job: debe quedar en cuarentena con su número de fila, el valor original y el motivo.

---

## 1. Modelo de referencia: seis etapas comunes a todos los canales

Todo canal de entrada se puede descomponer en el mismo esqueleto. Cambia el transporte, no la lógica.

```
[1] CAPTURA        → se recibe bytes (HTTP POST, archivo, evento de cola)
[2] AUTENTICIDAD   → ¿viene de quien dice? (firma, OAuth, API key, allowlist)
[3] CRUDO          → se persiste el payload tal cual + metadatos de recepción
[4] NORMALIZACIÓN  → tipos, unidades, formatos canónicos, dedupe de campos
[5] IDENTIDAD      → ¿es una persona/empresa nueva o existente? (matching)
[6] PERSISTENCIA   → upsert transaccional + efectos (asignación, workflows, outbox)
                     ↓ si algo falla: cuarentena + reintento + reporte
```

**Contrato mínimo de un endpoint de ingesta** (independiente del canal):

| Elemento | Por qué |
|---|---|
| `raw_id` (ULID/UUIDv7) | Trazabilidad: une el evento con el registro creado. |
| `received_at` con zona UTC | Orden de llegada ≠ orden de ocurrencia. |
| `source` + `source_account` | Distingue "formulario web" de "import de HubSpot" del mismo cliente. |
| `external_id` | Idempotencia cuando el emisor lo provee (`CallSid`, `message.id`, `notification.id`). |
| `idempotency_key` | Derivada si el emisor no provee una. |
| `raw_payload` (JSON/bytes) | Reproceso y auditoría. |
| `schema_version` | Permite migrar el parser sin reprocesar todo. |
| `processing_status` | `received / normalized / matched / persisted / quarantined`. |

> **Regla de oro de ingeniería:** el ACK al emisor se envía **después** de persistir el crudo y **antes** de procesar. Un `200 OK` significa "lo tengo guardado", no "lo procesé". Esto es exactamente lo que recomienda Nylas para sus webhooks: responder en <10 s y procesar en una cola. `[verificado + https://developer.nylas.com/docs/cookbook/use-cases/build/retry-failed-webhooks/]`

---

## 2. Canales de entrada, uno por uno

Resumen comparativo antes del detalle:

| Canal | Volumen típico | Latencia esperada | Formato de llegada | Riesgo principal |
|---|---|---|---|---|
| Formulario web | medio | segundos | `application/json` o `x-www-form-urlencoded` | spam/bots, campos fuera de definición, doble submit |
| Correo entrante | muy alto | segundos-minutos | MIME (multipart) | hilos, firmas, HTML gigante, adjuntos |
| Calendario | medio | segundos | JSON (evento) | recurrencia, invitados, zonas horarias |
| CSV/Excel | bajo pero masivo | minutos-horas | bytes + encoding desconocido | encoding, delimitador, fechas ambiguas |
| API/webhook | alto | ms-segundos | JSON | duplicados por reintento, orden |
| Extensión de navegador | bajo | segundos | JSON desde el cliente | datos inventados por el vendedor |
| Teléfono/llamada | medio | segundos | form-encoded (Twilio) | número no E.164, llamadas perdidas |
| Redes sociales (lead ads) | variable | segundos | JSON anidado del proveedor | permisos caducados, campos custom |
| Import desde otro CRM | bajo | horas | CSV o API paginada | IDs externos, relaciones huérfanas |

---

### 2.1 Formulario web

**Llega:** un POST del navegador (o del backend del sitio) con los campos del formulario, más metadatos de contexto (IP, página, cookie de tracking, consentimiento).

**Tres implementaciones reales y sus diferencias:**

**(a) HubSpot Forms API.** Endpoint autenticado `POST https://api.hsforms.com/submissions/v3/integration/secure/submit/{portalId}/{formGuid}`. `[verificado + https://developers.hubspot.com/docs/api-reference/legacy/marketing/forms/v3-legacy/submit-data-authenticated.md]`

Contrato exacto del cuerpo:

| Campo | Tipo | Nota |
|---|---|---|
| `fields[]` | array (máx. 1000) | cada uno `{objectTypeId, name, value}`; `0-1` = contacto, `0-2` = empresa |
| `submittedAt` | string (ms epoch) | permite *backdate*; >1 mes de antigüedad da error |
| `context.hutk` | string | cookie de tracking: pega la sesión web al registro |
| `context.ipAddress` | string | validado (`INVALID_IP_ADDRESS`) |
| `context.pageUri` / `pageName` / `pageId` | string | atribución |
| `context.sfdcCampaignId` | string | campaña de Salesforce si hay integración |
| `legalConsentOptions.consent` | objeto | `consentToProcess` **obligatorio** si el form lo exige |
| `legalConsentOptions.legitimateInterest` | objeto | `value`, `subscriptionTypeId`, `legalBasis` (`CUSTOMER`/`LEAD`) |

Errores tipados que devuelve (lista parcial, literal del spec): `BLOCKED_EMAIL`, `INVALID_EMAIL`, `REQUIRED_FIELD`, `FIELD_NOT_IN_FORM_DEFINITION`, `VALUE_NOT_IN_FIELD_DEFINITION`, `MISSING_PROCESSING_CONSENT`, `FORM_HAS_RECAPTCHA_ENABLED`, `INPUT_TOO_LARGE`, `INVALID_TIMESTAMP`. `[verificado + misma URL]`

> **Hallazgo de diseño:** `FIELD_NOT_IN_FORM_DEFINITION` significa que HubSpot **rechaza** campos que no existen en la definición del formulario. Si tu integración manda un campo nuevo sin actualizar el form, pierdes la submission completa, no el campo.

**Pipeline:** llega POST JSON → se valida contra la definición del form (tipos, required, consentimiento) → se resuelve identidad por email → si el contacto existe, **se actualiza** (no se crea) → se guarda submission + se actualizan propiedades + se dispara workflow → si falla: 400 con `errors[]` por campo, el navegador debe mostrar el error; **no hay reintento automático del lado de HubSpot** porque es una llamada síncrona del sitio.

**(b) Salesforce Web-to-Lead.** El formulario es HTML plano que hace POST a `https://webto.salesforce.com/servlet/servlet.WebToLead?encoding=UTF-8`. Campos de control documentados: `oid` (org id), `retURL` (a dónde redirigir), `debug=1` (página de depuración, quitar en producción), `lead_source`, `campaign_id`, `member_status`. `[verificado + http://web.archive.org/web/20140113102309/http://help.salesforce.com/apex/HTViewHelpDoc?id=setting_up_web-to-lead.htm]`

Comportamiento verificado en esa misma página:

- Se ejecutan las **reglas de validación** del objeto antes de crear; solo se crean registros válidos.
- Los campos universalmente requeridos deben venir con valor.
- Los campos Rich Text Area se guardan como texto plano (se pierde el formato).
- Los leads nuevos quedan en estado *Unread* hasta que el dueño los abre.
- **Límite diario: 500 requests.** Al excederse, el *Default Lead Creator* recibe un correo con la información de los leads que no entraron.
- Si la configuración está mal, se notifica a Soporte de Salesforce, no al sitio.

> **Hallazgo de diseño:** Web-to-Lead es *fire-and-forget* con un techo duro de 500/día y sin reintento. Es la razón por la que casi todo el mundo pone un endpoint propio delante y usa la API REST en vez del servlet.

**(c) EspoCRM Lead Capture.** `POST /api/v1/LeadCapture/{apiKey}`, sin autorización adicional (la API key va en la URL). `[verificado + https://docs.espocrm.com/administration/web-to-lead/]` Parámetros del registro de captura: *Subscribe to Target List*, *Subscribe Contact if exists* (si ya existe contacto con ese email, lo agrega a la lista en vez de crear lead), *Target Team*, *Campaign*, *Payload Fields* (allowlist de campos aceptados). Doble opt-in opcional con `{optInUrl}`, `{optInLink}`, `{optInDateTime}`.

**Qué se descarta en formularios web:**

| Se descarta | Por qué | Consecuencia |
|---|---|---|
| Campos no declarados en el form/allowlist | HubSpot: error; EspoCRM: ignore | integración silenciosamente incompleta |
| Formato (RTA, markdown) | Salesforce lo aplana | pérdida de información comercial |
| UTMs si el form no los mapea | no hay campo destino | atribución imposible de reconstruir |
| IP / user-agent (según política) | PII | no se puede hacer fraude/abuso a posteriori |
| Consentimiento si no se persiste como evento | se guarda solo el booleano | no se puede probar *cuándo* ni *con qué texto* |

---

### 2.2 Correo entrante

**Llega:** un mensaje MIME. Del mensaje completo, lo que un CRM usa es una fracción pequeña.

Tabla de lo que se guarda vs. se descarta (práctica observada en CRMs con IMAP/Gmail/Graph):

| Parte del mensaje | ¿Se guarda? | Formato |
|---|---|---|
| `Message-ID` | sí | string; clave de idempotencia entre proveedores |
| `From`, `To`, `Cc`, `Bcc`, `Reply-To` | sí | direcciones normalizadas (lowercase, sin display name) |
| `Subject` | sí | texto, a veces truncado |
| `Date` / `receivedDateTime` | sí | UTC |
| `threadId` / `In-Reply-To` / `References` | sí | para agrupar hilos |
| Cuerpo `text/plain` | sí | texto, colapsando whitespace |
| Cuerpo `text/html` | normalmente **no** (o sí, sanitizado) | se guarda el texto extraído; el HTML se descarta por tamaño y XSS |
| Imágenes remotas / píxeles de tracking | **no** | se descartan (privacidad + tamaño) |
| Firmas y avisos de confidencialidad | **no** (se detectan y se cortan) | heurística |
| Adjuntos | metadatos sí, binario en object storage | ver §5.6 |
| Cabeceras de spam/autenticación (`Authentication-Results`, `DKIM`) | a veces | solo para auditoría |

**Pipeline:** llega webhook "hay cambios" → se pide el delta al proveedor → por cada mensaje nuevo: se calcula `Message-ID` → ¿existe ya? (idempotencia) → se resuelve la persona por dirección → se guarda `message` + `thread` + asociaciones → se actualiza `last_activity_at` del contacto → si falla el parseo MIME: se guarda el crudo y se marca `parse_failed`, no se pierde.

Detalle técnico completo en §5.

---

### 2.3 Calendario

**Llega:** un evento con `id`, `summary`, `start/end` (con `dateTime` + `timeZone` o `date` para todo-el-día), `attendees[]` (con `email`, `responseStatus`), `organizer`, `recurrence[]`, `iCalUID`, `status`.

Pipeline: llega notificación → se relee el evento → se hace upsert por `(provider, calendar_id, event_id)` → se asocian los `attendees` a personas (creando "unknown attendee" si no existen) → se calcula duración y se registra como actividad → si el evento es recurrente, se expande con `iCalUID` + `originalStartTime` como clave de instancia.

**Qué se descarta:** descripción completa del evento (a veces), adjuntos, conferencias (Meet/Zoom link se guarda como URL), y **el cuerpo de la notificación push**, que en Google Calendar directamente no existe.

> Dato clave verificado: las notificaciones de Calendar **no tienen body**; solo headers `X-Goog-*` y el estado `sync` / `exists` / `not_exists`. Además, "las notificaciones no son 100% confiables; espera un pequeño porcentaje de mensajes perdidos". `[verificado + https://developers.google.com/workspace/calendar/api/guides/push]`

---

### 2.4 CSV / Excel

Tratado en profundidad en §3. Resumen: el archivo **nunca** se procesa en streaming puro; se sube completo, se guarda, se detecta dialecto, se mapea, se normaliza, se valida, se deduplica y se hace upsert por lotes.

---

### 2.5 API / webhook

**Llega:** JSON con el objeto completo o solo el `id` + tipo de cambio.

Dos familias:

| Estilo | Ejemplo | Payload | Implicación |
|---|---|---|---|
| **Notificación rica** | Salesforce Platform Events / CDC, Graph *rich notifications* | objeto completo | se puede procesar sin llamada extra |
| **Notificación pobre** | Gmail Pub/Sub, Google Calendar, Graph *basic notifications*, HubSpot webhooks | solo IDs + tipo | hay que releer; se duplica trabajo pero se es tolerante a pérdidas |

**Pipeline:** llega POST → se verifica firma (Twilio `X-Twilio-Signature`; Nylas `X-Nylas-Signature` HMAC-SHA256 sobre el body crudo) → se persiste crudo → ACK 2xx → cola → worker hace *fetch* del objeto si la notificación es pobre → upsert idempotente → si falla: reintento con backoff y luego DLQ.

**Qué se descarta:** casi nada del payload, pero sí se descartan **campos desconocidos** si el parser es estricto (mala práctica) y se descartan los reintentos duplicados (por idempotencia).

---

### 2.6 Extensión de navegador

**Llega:** JSON construido por el cliente, normalmente a partir de:
- la página abierta (scraping del DOM: nombre, cargo, empresa de un perfil),
- una cookie de tracking (`hutk` en HubSpot) que une la sesión anónima con el formulario,
- o un correo reenviado a una dirección especial (BCC logging).

Datos típicos: nombre, email, empresa, cargo, URL de perfil, notas libres, `owner` (el vendedor).

**Pipeline:** llega JSON autenticado con token del usuario → validación de esquema **estricta** (es el canal con más basura) → se marca `source=browser_extension` y `created_by_user_id` → dedupe por email/dominio → si hay match: se crea una *nota* en el registro existente, no un registro nuevo (evita que dos vendedores dupliquen la misma cuenta) → si falla: error visible en la extensión (no silencioso).

**Qué se descarta:** todo lo que el vendedor no escribió (el DOM no se guarda completo), y en muchos casos el HTML de origen; se pierde la posibilidad de auditar de dónde salió el dato.

> Pipedrive expone explícitamente `smart_bcc_email` en el objeto persona, lo que confirma el patrón "dirección BCC por usuario para capturar correo sin OAuth". `[verificado + https://developers.pipedrive.com/docs/api/v1/Persons]`

---

### 2.7 Teléfono / llamada

**Llega:** webhook `application/x-www-form-urlencoded` de Twilio con parámetros planos. `[verificado + https://www.twilio.com/docs/voice/twiml]`

| Parámetro | Contenido |
|---|---|
| `CallSid` | identificador único de la llamada (clave de idempotencia) |
| `AccountSid` | cuenta (34 chars, empieza con `AC`) |
| `From` / `To` | E.164 (`+16175551212`) **si el carrier lo permite**; si no, string crudo |
| `CallStatus` | `queued, initiated, ringing, in-progress, completed, busy, failed, no-answer, canceled` |
| `Direction` | `inbound`, `outbound-api`, `outbound-dial` |
| `CallerName` | solo si `VoiceCallerIdLookup=true` |
| `ParentCallSid` | para llamadas con varias piernas |
| `FromCity/State/Zip/Country`, `ToCity/...` | geolocalización aproximada del número |
| `CallDuration` (en `StatusCallback`) | segundos |
| `RecordingUrl`, `RecordingSid`, `RecordingDuration` | si se grabó |

Pipeline: llega webhook de llamada entrante → se responde TwiML en <15 s → al colgar, llega `StatusCallback` con duración y grabación → se crea `call` con `external_id=CallSid` → se resuelve persona por `From` (E.164) → si no existe: se crea lead "llamada entrante" → se descarga la grabación a object storage y se guarda solo la URL interna.

**Qué se descarta:** el audio crudo en el CRM (se guarda fuera), la transcripción si no hay consentimiento, y las llamadas `canceled` en estado `queued` **no generan status callback** (verificado: "If the call is canceled while still in `queued` status, no status callbacks are fired").

> Nota operativa: Twilio documenta que su *Request Inspector* conserva el request y la respuesta HTTP **30 días**. Es el único "payload crudo" que te da el proveedor, y caduca. `[verificado + https://www.twilio.com/docs/usage/webhooks/webhooks-faq]`

---

### 2.8 Redes sociales (lead ads)

**Llega:** JSON anidado del proveedor (típicamente `entry[].changes[].value` con `leadgen_id`, `form_id`, `page_id`, `created_time`, `field_data[]` de pares `name`/`values`).

**Pipeline:** llega webhook → verificación de `hub.challenge` / firma → se encola → se pide el lead completo por API (el webhook trae IDs, no datos) → se mapean campos custom del formulario a propiedades → dedupe por email/teléfono → se crea lead con `source=social` y `campaign` → si falla el fetch (token caducado): reintento y alerta, porque el lead se pierde en el proveedor tras un tiempo.

**Qué se descarta:** `field_data` no mapeados, respuestas de formulario largo, y cualquier campo que el proveedor marque como no exportable. `[inferido: patrón consistente entre Meta/LinkedIn/TikTok lead ads; no se citó documentación específica en esta investigación]`

---

### 2.9 Importación desde otro CRM

Dos modalidades:

**(a) CSV exportado del sistema origen.** Ver §3. El problema específico es que los IDs del origen no significan nada en el destino: hay que crear un `external_id` estable.

Odoo lo formaliza: el campo **External ID** es el identificador único de un registro en otra aplicación, debe ser único **entre todos los objetos**, y se recomienda prefijarlo con el nombre de la app/tabla (`company_1`, `person_1`). Si importas el mismo archivo dos veces con External ID, **actualiza** en vez de duplicar; si lo alteras o lo quitas, **creas duplicados**. `[verificado + https://www.odoo.com/documentation/18.0/applications/essentials/export_import_data.html]`

**(b) Conector API paginado.** Se recorre el origen con cursor (`updated_since`, `since_id`), se mapea a un esquema canónico y se hace upsert por `external_id`. Pipedrive expone `updated_since` / `updated_until` en RFC3339 justo para esto. `[verificado + https://developers.pipedrive.com/docs/api/v1/Persons]`

**Orden obligatorio de importación** (por dependencias): empresas → personas → negocios → actividades. Si importas personas antes que empresas, quedan huérfanas y el "relate on import" por nombre falla. EspoCRM resuelve relaciones por nombre foráneo solo en relaciones many-to-one, y en v8.2+ por un campo *Foreign* con "Relate on Import". `[verificado + https://docs.espocrm.com/administration/import/]`

---

## 3. El pipeline de importación CSV, etapa por etapa

### 3.0 Vista de 8 pasos (la versión que cabe en una pizarra)

```
1. SUBIDA           archivo → object storage + hash (idempotencia del archivo)
2. DETECCIÓN        encoding (BOM/UTF-8/CP1252), delimitador (, ; \t |), quote, header
3. MAPEO            columna → propiedad (+ mapeo guardado reutilizable + defaults)
4. NORMALIZACIÓN    E.164, email lowercase, nombres, países ISO, fechas → ISO 8601
5. VALIDACIÓN       tipos, required, enums, regex, longitudes, reglas de negocio
6. DEDUPLICACIÓN    match contra existentes + duplicados intra-archivo
7. UPSERT           create / update / skip, por lotes, transaccional por lote
8. REPORTE          created / updated / skipped / failed(unprocessed) + CSV de errores
```

### 3.1 Subida y detección de encoding/delimitador

**Qué llega:** bytes. Nada más. El archivo puede ser UTF-8, UTF-8 con BOM, UTF-16, Windows-1252, ISO-8859-1, o una mezcla (export de Excel en español).

**Qué se hace:**

| Chequeo | Cómo | Qué falla si no se hace |
|---|---|---|
| BOM | primeros 3-4 bytes (`EF BB BF`, `FF FE`, `FE FF`) | acentos y `ñ` rotos; primera columna con nombre `\ufeffNombre` |
| Encoding | intento UTF-8 estricto → si falla, `charset-normalizer`/`chardet` → fallback CP1252 | mojibake irreversible (`MarÃ­a`) |
| Delimitador | contar ocurrencias fuera de comillas en las primeras N líneas | todo el archivo en una sola columna |
| Quote char | `"` por defecto; `'` en algunos exports | filas partidas por comas internas |
| Fin de línea | CRLF vs LF vs CR (clásico Mac) | última columna con `\r` invisible → emails inválidos |
| Header | ¿primera fila es header? ¿duplicados? | columnas sobrescritas silenciosamente |
| Encoding del *contenido* | no solo del archivo: hay celdas con `?` de un export previo | pérdida ya ocurrida, no recuperable |

Norma de referencia para el dialecto: RFC 4180 (coma, comillas dobles, escape duplicando la comilla, CRLF). `[verificado + https://www.rfc-editor.org/rfc/rfc4180]`

**Ejemplo real de lo que Odoo documenta:** el preview asume coma como separador y comillas como delimitador de texto; **si el CSV usa tabulación, Odoo no detecta la separación** y hay que cambiarlo en la hoja de cálculo. También detecta números con `32.000,00`, `32000,00`, `32,000.00`, `-32000.00`, `(32000.00)` y `$ 32.000,00`, pero **falla** con `ABC 32.000,00`. `[verificado + https://www.odoo.com/documentation/18.0/applications/essentials/export_import_data.html]`

**Qué se guarda:** el archivo original en object storage, su SHA-256, el dialecto detectado, el encoding usado y la confianza de la detección. **Nunca** se sobreescribe el original con la versión "limpia": el limpio es un artefacto derivado.

**Si falla:** `400` con `{stage: "detect", reason: "encoding_ambiguous", sample_bytes: "<hex>"}`. No se intenta adivinar en silencio: se pide al usuario que elija encoding.

---

### 3.2 Mapeo de columnas (y mapeo guardado)

**Qué se hace:** se construye una lista ordenada `columna_origen → propiedad_destino`, con tipo destino, transformación y política de sobrescritura.

HubSpot lo expone como JSON explícito y **exige una entrada por columna, en el mismo orden que el archivo**: `columnName` debe coincidir exactamente con el header; si falta una, el import arranca y luego falla con *Incorrect number of columns*. `[verificado + https://developers.hubspot.com/docs/guides/api/crm/imports]`

Estructura mínima del mapeo (esquema propio, inspirado en el de HubSpot):

```json
{
  "mapping_id": "hs_contacts_v3",
  "version": 3,
  "columns": [
    {"src": "E-mail",        "dst": "email",      "type": "email",   "unique": true,  "overwrite": "always"},
    {"src": "Nombre",        "dst": "firstname",  "type": "string",  "overwrite": "if_empty"},
    {"src": "Tel.",          "dst": "phone",      "type": "phone",   "region": "CL"},
    {"src": "Fec. alta",     "dst": "created_at", "type": "date",    "format": "DD/MM/YYYY"},
    {"src": "País",          "dst": "country",    "type": "country", "output": "ISO3166-1-alpha2"},
    {"src": "Comentarios",   "dst": null,         "type": "ignore"}
  ],
  "defaults": {"owner_id": 42, "source": "import_2026_11"},
  "on_conflict": "update_only_mapped"
}
```

**Mapeo guardado:** se persiste con `mapping_id` + `version` + hash del set de headers. En la siguiente importación se intenta reutilizar por (a) hash exacto de headers, (b) similitud de nombres (`E-mail` ≈ `email` ≈ `correo`), (c) elección manual previa del mismo usuario. **Nunca** aplicar un mapeo guardado si el set de headers cambió sin avisar: es la causa #1 de columnas corridas un lugar.

**Qué falla aquí:**

| Falla | Síntoma | Prevención |
|---|---|---|
| Header con BOM o espacio final | columna "no encontrada" | trim + strip BOM |
| Dos columnas mapeadas a la misma propiedad | gana la última, silencioso | rechazar duplicados de destino |
| Columna mapeada a propiedad de otro objeto | se ignora o rompe la asociación | validar `objectTypeId` |
| Mapeo guardado con headers distintos | datos en el campo equivocado | checksum de headers |
| Fechas sin formato declarado | `03/04/2026` interpretada al revés | `dateFormat` obligatorio (ver §3.3) |

---

### 3.3 Normalización

Esta es la etapa donde se gana o se pierde la calidad del CRM. Reglas concretas, con su justificación:

#### Teléfonos → E.164

Formato: `+` + código de país + número; máximo 15 dígitos; sin `0` inicial. `[verificado + https://www.twilio.com/docs/glossary/what-e164]`

```
normalize_phone(raw, default_region):
  s = strip_all_except_digits_and_plus(raw)
  if starts_with("00"): s = "+" + s[2:]          # prefijo internacional europeo
  if starts_with("+"):  candidate = s
  else:                 candidate = to_e164(s, default_region)   # requiere región
  if not is_valid(candidate):  → error "invalid_phone" (NO inventar)
  return candidate
```

Reglas duras:
- **Sin región por defecto no se puede normalizar** un número local (`912345678` puede ser Chile, España o México). EspoCRM lo resuelve pidiendo *Telephone country code* en el paso 1 del import y también en Lead Capture. `[verificado + https://docs.espocrm.com/administration/import/]`
- Un número inválido **no se "arregla"**: se marca error por fila. EspoCRM, con números internacionales activos, **valida al guardar y no permite guardar un número inválido**. `[verificado + https://docs.espocrm.com/administration/phone-numbers/]`
- Se guarda también el original (`phone_raw`) porque el usuario lo reconoce por cómo lo escribió.

#### Emails → minúsculas

- `trim()`, lowercase **solo del dominio** en RFC estricto, pero en la práctica **todo el email** a minúsculas (los proveedores reales son case-insensitive en la práctica y la unicidad debe ser por minúsculas).
- Validación: no regex perfecto (no existe); aceptar lo que pase un validador tipo `email-validator` + comprobar dominio con MX si se quiere calidad alta.
- Descartar `noreply@`, `no-responder@`, `postmaster@` como identidad de persona.
- Guardar `email_normalized` (índice único) + `email_display` (como lo escribió el usuario).

#### Nombres

| Regla | Motivo |
|---|---|
| `trim` + colapsar espacios + quitar caracteres de control | "Juan  Pérez" ≠ "Juan Pérez" en un índice |
| No hacer Title Case a la fuerza | rompe `McDonald`, `van der Berg`, `O'NEILL` |
| No separar `Nombre Completo` por el primer espacio | "María de los Ángeles Ruiz" → apellido incorrecto |
| Guardar `full_name` calculado, no como fuente de verdad | permite reordenar sin perder el original |
| Preservar diacríticos en el valor; normalizar (NFD→sin diacríticos) **solo** para la clave de búsqueda | "José" y "Jose" deben matchear sin destruir el dato |

#### Países

- Entrada: `Chile`, `CL`, `CHL`, `República de Chile`, `cl `.
- Salida: ISO 3166-1 alpha-2 + nombre canónico. Si el país no resuelve, se deja `null` y se marca advertencia (no error): un país mal escrito no debe bloquear un lead.

#### Fechas ambiguas (`03/04/2026`)

El problema: `03/04/2026` es 3 de abril (DD/MM) o 4 de marzo (MM/DD). **No hay forma de saberlo por el valor.**

Estrategias, en orden:

1. **Declarar el formato en el import.** HubSpot lo hace explícito en la Imports API: `dateFormat` = `MONTH_DAY_YEAR` (default), `DAY_MONTH_YEAR` o `YEAR_MONTH_DAY`. `[verificado + https://developers.hubspot.com/docs/guides/api/crm/imports]`
2. **Inferir del conjunto, no de la fila.** Si alguna fila tiene día > 12 en la primera posición → es DD/MM. Si alguna tiene > 12 en la segunda → es MM/DD. Si **ninguna** lo hace, el archivo es ambiguo y hay que preguntar.
3. **Odoo** documenta exactamente este riesgo: adivina el formato de un set de formatos comunes, y advierte que "es difícil adivinar qué parte es el día y cuál el mes en una fecha como `01-03-2016`"; la solución es fijar ISO 8601. `[verificado + https://www.odoo.com/documentation/18.0/applications/essentials/export_import_data.html]`

```
resolve_date_format(samples):
  d1 = count(day(first_component) > 12)
  d2 = count(day(second_component) > 12)
  if d1 > 0 and d2 == 0: return "DD/MM/YYYY"
  if d2 > 0 and d1 == 0: return "MM/DD/YYYY"
  if d1 > 0 and d2 > 0: return AMBIGUOUS_ERROR   # archivo inconsistente
  return ASK_USER                                 # todo ≤12
```

4. **Fechas de negocio vs. técnicas:** `created_at` del sistema debe ser del sistema (no del CSV). La fecha del CSV es `source_created_at`, un campo distinto. Mezclarlas hace que un import reescriba la antigüedad de todos los registros.

**Números y monedas:** separador de miles/decimales, paréntesis = negativo, símbolo de moneda. Tabla de casos aceptados/rechazados en Odoo (arriba). Se guarda `amount` como decimal + `currency` ISO 4217, nunca como string formateado.

---

### 3.4 Validación y errores por fila

**Qué se valida, en orden de costo (barato primero):**

| Nivel | Ejemplo | ¿Bloquea la fila? |
|---|---|---|
| Estructura | nº de columnas ≠ header | sí (fila completa) |
| Tipo | `amount = "N/A"` | sí |
| Requerido | falta `email` y `lastname` | sí |
| Formato | email sin `@`, teléfono inválido | sí |
| Catálogo | `country = "Narnia"`, enum fuera de lista | advertencia o error según campo |
| Longitud | texto > límite del campo | truncar + advertencia |
| Unicidad | email repetido dentro del archivo | sí (segunda ocurrencia) |
| Negocio | `close_date < created_date` | sí |

**Modelo de error por fila** (esto es lo que hay que guardar, siempre):

```json
{
  "row_number": 7000,
  "raw_values": ["María  Pérez", "MARIA@Example.COM ", "+56 9 1234 5678", "03/04/2026", "Santiago"],
  "normalized": {"email": "maria@example.com", "phone": "+56912345678"},
  "errors": [
    {"field": "phone", "code": "INVALID_PHONE", "detail": "número no válido para región CL"},
    {"field": "created_at", "code": "AMBIGUOUS_DATE", "detail": "03/04/2026"}
  ],
  "stage": "validation",
  "action": "quarantined"
}
```

**Códigos de error útiles** (inspirados en los `errorType` reales de HubSpot, §2.1): `REQUIRED_FIELD`, `INVALID_EMAIL`, `INVALID_PHONE`, `VALUE_NOT_IN_FIELD_DEFINITION`, `DUPLICATE_IN_FILE`, `DUPLICATE_MULTIPLE_MATCHES`, `INPUT_TOO_LARGE`, `INVALID_TIMESTAMP`.

**Qué hacen los CRM reales con los errores:**

- **EspoCRM:** todas las filas que fallan quedan en un panel *Errors* con el motivo (normalmente validación), **el índice de la fila** y los valores; se pueden **exportar a CSV** para corregir y reimportar. Distingue `Integrity-Constraint-Violation` (ID ya existe) de `Validation`. `[verificado + https://docs.espocrm.com/administration/import/]`
- **Salesforce Bulk API 2.0:** hay tres archivos de resultados distintos — `successfulResults` (con `sf__Id`, `sf__Created`), `failedResults` (con `sf__Error`, `sf__Id` y **las columnas originales**) y `unprocessedrecords` (filas que ni se procesaron porque el job se abortó). Salesforce es explícito: "Unprocessed rows are not the same as failed rows". Además, **el orden de los registros en los resultados no está garantizado**. `[verificado + https://developer.salesforce.com/docs/platform/api-asynch/guide/get-job-failed-results.md]`
- **HubSpot:** `GET /crm/v3/imports/{importId}/errors`, con estados de import `STARTED / PROCESSING / DONE / FAILED / CANCELED / DEFERRED` y un máximo de **3 imports procesando a la vez** (el cuarto queda `DEFERRED`). `[verificado + https://developers.hubspot.com/docs/guides/api/crm/imports]`

---

### 3.5 Deduplicación durante el import

Dos capas distintas que se confunden todo el tiempo:

| Capa | Qué detecta | Qué hacer |
|---|---|---|
| **Intra-archivo** | la misma persona dos veces en el CSV | fusionar filas antes de tocar la BD (o marcar duplicado) |
| **Contra la BD** | ya existe un registro con esa clave | update / skip / error según política |

Detalle de algoritmos y reglas en §4. Lo importante para el pipeline: **la clave de identidad se decide antes de la etapa 7**, no durante el upsert.

HubSpot documenta el caso patológico: si al importar un contacto **existen múltiples registros con el mismo email**, el import **falla para esa fila** y el contacto **no se importa**. `[verificado + https://knowledge.hubspot.com/records/deduplication-of-records]` Es decir: la ambigüedad no se resuelve al azar, se rechaza.

---

### 3.6 Creación / actualización (upsert)

Tres políticas, y hay que elegirlas por importación, no por sistema:

| Política | Semántica | Riesgo |
|---|---|---|
| `CREATE` | solo inserta; si existe, error/ignora | duplica si el archivo viene dos veces |
| `UPDATE` | solo actualiza; si no existe, ignora | pierde leads nuevos silenciosamente |
| `UPSERT` | crea si no existe, actualiza si existe | sobrescribe datos buenos con datos viejos |

**Sobrescritura parcial.** Dos mecanismos verificados:

- HubSpot: *Prevent property overwrite* / **Don't overwrite**. Si se marca, "el import actualizará la propiedad para registros nuevos y para existentes que nunca tuvieron valor; **no** actualizará registros que tienen valor **o que lo tuvieron en el pasado, aunque ahora estén vacíos**". `[verificado + https://knowledge.hubspot.com/import-and-export/import-objects]`
- Odoo: si un campo **no está** en el CSV, se aplica el valor por defecto; si está **vacío**, se escribe vacío (no el default). `[verificado + https://www.odoo.com/documentation/18.0/applications/essentials/export_import_data.html]`

**Upsert por external ID (Salesforce Bulk API 2.0):**

- External ID no encontrado → crea.
- Encontrado **una vez** → actualiza.
- Encontrado **múltiples veces** → **error 300**, no crea ni actualiza. `[verificado + https://developer.salesforce.com/docs/platform/api-asynch/guide/bulk-api-2-0-upsert.md]`

Ese "300" es el mejor argumento para que el external ID sea único *también* en el origen.

**Transaccionalidad:** por lote, no por archivo. Un lote de 200-2.000 filas en una transacción; si el lote falla, se reintenta el lote completo y la idempotencia por clave evita duplicados.

---

### 3.7 Reporte de resultado

El reporte debe permitir responder, sin abrir la base: ¿cuántas filas entraron, cuántas se actualizaron, cuántas se saltaron, cuántas fallaron y por qué? Y debe ser **descargable y reimportable**.

```json
{
  "import_id": "imp_01J8Z...",
  "file": {"name": "leads_nov.csv", "sha256": "9f2c...", "rows": 50000, "bytes": 7340032},
  "detected": {"encoding": "windows-1252", "delimiter": ";", "date_format": "DD/MM/YYYY"},
  "result": {
    "created": 48120,
    "updated": 1503,
    "skipped": 47,
    "failed": 330,
    "unprocessed": 0
  },
  "errors_by_code": {"INVALID_PHONE": 210, "AMBIGUOUS_DATE": 88, "DUPLICATE_MULTIPLE_MATCHES": 32},
  "artifacts": {
    "failed_csv": "s3://.../imp_01J8Z.../failed.csv",
    "unprocessed_csv": "s3://.../imp_01J8Z.../unprocessed.csv"
  },
  "duration_ms": 412000,
  "revertible": true
}
```

**Reversibilidad:** EspoCRM permite *Revert Import* (borra los registros creados) pero **no revierte actualizaciones**; además distingue "Remove duplicates" (borra lo que se marcó duplicado) y "Remove import log". `[verificado + https://docs.espocrm.com/administration/import/]` Regla de diseño: guarda `import_id` en cada registro creado; es la única forma de deshacer.

---

### 3.8 La fila 7.000 de 50.000 (qué pasa exactamente)

**Lo que NO debe pasar:** que el job muera, que se reprocesen las 6.999 anteriores creando duplicados, o que la fila 7.000 desaparezca sin rastro.

**Diseño correcto:**

```
job = create_import(file, mapping)          # estado: queued
for chunk in chunks(rows, size=500):        # 500-2000
    with transaction():
        for row in chunk:
            try:
                rec = normalize(row)
                validate(rec)                    # error → raise RowError
                key = identity_key(rec)          # ver §4
                existing = find_by_key(key)
                upsert(rec, existing)
                log_success(row.n, rec, action)
            except RowError as e:
                quarantine(row, e)               # NO aborta el chunk
            except TransientError as e:
                raise                            # aborta el chunk → reintento del chunk
    checkpoint(job, last_committed_row=chunk.end)   # commit del progreso
```

Propiedades que esto garantiza:

1. **Aislamiento:** un error de validación de una fila no afecta a las otras 499 del chunk.
2. **Reanudación:** si el proceso muere en la fila 7.000, el job retoma desde el último checkpoint. EspoCRM expone exactamente esto: `php command.php import --id={import-id} --resume` ("Resume failed import from the last processed row"). `[verificado + https://docs.espocrm.com/administration/import/]`
3. **Idempotencia del reintento:** si el chunk se reintenta completo, las filas ya insertadas se reconocen por clave y no se duplican. Sin esto, el resume crea duplicados.
4. **Cuarentena separada:** la fila 7.000 queda en `import_errors` con `row_number`, `raw_values`, `code`, `stage`. Se exporta a CSV, el usuario la corrige y la reimporta como un archivo de 1 fila.
5. **Contadores consistentes:** `created + updated + skipped + failed + unprocessed = total`. Si no cuadra, hay un bug de logging, no de datos.

**Errores transitorios vs. permanentes** (la distinción que hace que el resume funcione):

| Tipo | Ejemplo | Acción |
|---|---|---|
| Permanente (fila) | email inválido, enum desconocido | cuarentena, seguir |
| Transitorio (lote) | timeout de BD, rate limit 429, deadlock | rollback del chunk, backoff, reintentar |
| Fatal (job) | archivo corrupto, mapping inválido | abortar job, `unprocessed` = resto |

**El caso real de Salesforce Bulk API 2.0** confirma el modelo: si un job se aborta, las filas no procesadas aparecen en `unprocessedrecords` **con los datos originales** y se pueden re-subir. Y el job se puede abortar/borrar solo en estados concretos (`UploadComplete`, `JobComplete`, `Aborted`, `Failed`). `[verificado + https://developer.salesforce.com/docs/platform/api-asynch/guide/get-job-unprocessed-results.md]`

---

## 4. Deduplicación real

### 4.1 Qué compara cada sistema

| Sistema | Contactos/personas | Empresas | ¿Fuzzy? | Acción al detectar |
|---|---|---|---|---|
| **Salesforce** | Reglas de matching configurables (Email, Phone, Name, Address) | Idem (Name, Website, BillingStreet…) | **Sí** (métodos "fuzzy" + `matchConfidence`) | Block / Allow (con alerta) / reportar |
| **HubSpot** | `email` (único, automático) | `domain` (automático, **solo en import/form**, no en API) | No (exacto normalizado) | actualiza el existente / error si hay múltiples |
| **Pipedrive** | `name` + `emails[]` + `phones[]`; búsqueda y merge manual | `name` | Búsqueda por término (no fuzzy declarado) | merge explícito (`PUT /persons/{id}/merge`) |
| **EspoCRM** | `WHERE` de igualdades configurable por código | Idem | **No** (igualdad SQL) | bloquea el guardado (dup check) |
| **Odoo** | External ID / `email` | `name`, `vat` | No | update por External ID |

### 4.2 Salesforce: matching rules + duplicate rules (el modelo más completo)

Son **dos objetos separados** y conviene no confundirlos:

- **Matching rule**: define *qué es un candidato*. Se compone de campos con criterios; al activarse, **se crean match keys y se indexan los registros existentes**, de modo que la comparación solo ocurre entre registros con la misma match key. `[verificado + http://web.archive.org/web/2016/https://help.salesforce.com/apex/HTViewHelpDoc?id=matching_rules_overview.htm]`
- **Duplicate rule**: define *qué hacer* cuando el matching rule encuentra algo. Requiere al menos un matching rule asociado, y **todas las matching rules asociadas deben estar activas** para poder activar la duplicate rule. `[verificado + http://web.archive.org/web/20151023064151/https://help.salesforce.com/apex/HTViewHelpDoc?id=duplicate_rules_create.htm]`

Puntos finos, todos verificados en esa documentación:

1. **Condiciones de ejecución:** una duplicate rule puede tener condiciones (por perfil, rol, país, estado) y entonces solo corre para esos casos.
2. **En updates, la regla solo corre si se editaron campos incluidos en la matching rule.** Esto es crítico: si tu integración actualiza solo `phone` y la matching rule usa `email`, **el duplicado no se detecta**.
3. **Orden de reglas:** si la primera duplicate rule encuentra match, el registro **no se evalúa con las siguientes**. Recomendación textual: poner las reglas con `Block` antes que las de `Allow`.
4. **Truncamiento:** si comparas dos campos de texto con longitudes máximas distintas, **los datos pueden truncarse antes del matching**.
5. **Field mapping entre objetos:** al comparar dos objetos distintos, o campos custom, hay que mapear campo ↔ campo (ej. `Work Email` custom → `Email` estándar).

**Acciones posibles:** `Allow` (permite guardar, con alerta y opción de reportar) y `Block` (no permite guardar, con alerta explicando por qué y qué hacer). La alerta es parte de ambas. `[verificado + .../duplicate_rules_overview.htm]`

**Cómo se comporta por API** (esto es lo que importa a una integración):

- Cabecera `Sforce-Duplicate-Rule-Header` (API 52.0+), con tres flags por defecto en `false`: `allowSave` (permitir guardar a pesar de la alerta), `includeRecordDetails` (devolver todos los campos del duplicado, no solo el ID), `runAsCurrentUser` (aplicar reglas de compartición del usuario actual vs. del sistema). `[verificado + https://developer.salesforce.com/docs/platform/api-rest/guide/headers-duplicaterules.md]`
- Con `allowSave=false` y una regla `Block`, el guardado falla y el error es un `DuplicateError` con `duplicateResult`, `fields`, `message`, `statusCode`. `[verificado + https://developer.salesforce.com/docs/platform/api/guide/sforce-api-objects-duplicateerror.md]`
- El `duplicateResult` contiene `matchResults[]`, y cada `matchRecord` trae **`matchConfidence`** además del `record.Id` y `record.Type`. Es decir: Salesforce expone una **confianza numérica**, no un booleano.
- **`findDuplicates()`** permite preguntar "¿esto es duplicado?" **antes** de escribir, y **antes** de que exista el registro. Límite: **50 sObjects por llamada**, todos del mismo tipo. `[verificado + https://developer.salesforce.com/docs/platform/api/guide/sforce-api-calls-findduplicates.md]`

```
// Pseudocódigo de integración robusta
candidates = sf.findDuplicates([{type:"Lead", fields:{Email:e, Phone:p, Company:c}}])
if candidates.matchResults[0].matchRecords.length == 0:
    create()
elif max(confidence) >= 0.9:
    update(existing)          # match fuerte → actualizar
else:
    create(allowSave=true, flag="possible_duplicate")   # match débil → crear y marcar
```

> **Decisión de diseño:** la banda intermedia (0.6-0.9) es la más peligrosa: si actualizas, contaminas un registro; si creas, duplicas. La salida correcta es **crear y marcar para revisión**, no decidir automáticamente.

### 4.3 HubSpot: deduplicación por clave exacta, con reglas por canal

Lo verificado, que sorprende a mucha gente:

| Canal | Comportamiento |
|---|---|
| Manual | avisa que ya existe y **no permite** crear |
| Formulario | si el email existe, **actualiza** el contacto existente (no crea). Si el visitante usa su email **secundario**, ese email **sobrescribe** el primario. |
| Import | si el email existe, **actualiza**; solo se actualizan las propiedades mapeadas; **no borra** registros |
| Import sin email ni otro identificador | **cada fila se crea como contacto nuevo** (duplicados garantizados) |
| Import con email que ya existe en **múltiples** registros | **error de import**, el contacto **no se importa** |
| API (empresas) | **las empresas creadas por API no se deduplican por dominio**, incluidas las de apps de sync de terceros |

`[verificado + https://knowledge.hubspot.com/records/deduplication-of-records]`

Esa última fila es la trampa clásica: el import y el formulario deduplican empresas por dominio, la API **no**. Si construyes un conector, la deduplicación es tu responsabilidad.

**Identificadores admitidos en import** (en orden de precedencia): `Record ID` (gana sobre todo), `Email` (contactos), `Company domain name` (empresas, solo hasta el TLD), y hasta **10 propiedades custom con "requerir valores únicos"** por objeto. `[verificado + https://knowledge.hubspot.com/import-and-export/import-objects]`

**Emails adicionales:** `hs_additional_emails` permite varios emails por contacto y **también son identificadores únicos**: dos contactos no pueden compartir un email adicional. `[verificado + https://developers.hubspot.com/docs/guides/api/crm/objects/contacts]`

**Upsert por API:** `POST /crm/v3/objects/contacts/batch/upsert` con `idProperty` = `email` o un ID custom. Aviso textual: **no hay upsert parcial cuando `idProperty=email`**; para actualización parcial hay que usar una propiedad custom única. `[verificado + misma URL de contacts]`

### 4.4 EspoCRM: el matching es un `WHERE`

No hay fuzzy. El "duplicate check" se configura por entidad en `recordDefs`, con una clase que implementa `Espo\Core\Duplicate\WhereBuilder` y devuelve una condición ORM. Ejemplo de la doc: si hay `firstName` o `lastName`, compara `firstName = X AND lastName = Y`. `[verificado + https://docs.espocrm.com/development/duplicate-check/]`

Consecuencias:

- El chequeo corre **al crear**, y opcionalmente al actualizar (`updateDuplicateCheck: true`, desactivado por defecto).
- Es barato (un índice) pero **ciego a variantes**: "Jose Perez" ≠ "José Pérez" ≠ "J. Pérez".
- La calidad depende de que normalices **antes** de llamar al duplicate check. Si normalizas después, el chequeo no sirve.

### 4.5 Los algoritmos, de más estricto a más laxo

Taxonomía operativa, tomada de OpenRefine (que es la referencia open source más clara sobre esto): `[verificado + https://openrefine.org/docs/manual/cellediting#cluster-and-edit]`

**A. Key collision** (genera una clave; los que colisionan son candidatos; O(n), millones de celdas por segundo):

| Método | Qué hace | Ejemplo |
|---|---|---|
| **Fingerprint** | minúsculas, sin puntuación, sin diacríticos, colapsa espacios, **ordena las palabras alfabéticamente** | "Zhenyi, Wang" → `wang zhenyi` |
| **N-gram fingerprint** | limpia, parte en n-gramas, ordena y une | "banana" (2-gram) → `anbana`; matchea "lookout"/"look out" |
| **Metaphone3** | fonético **inglés** | "Reuben Gevorkiantz" ≈ "Ruben Gevorkyants" |
| **Cologne** | fonético **alemán** | "Greenvale" ≈ "Granville" |
| **Daitch-Mokotoff** | fonético eslavo/yiddish | apellidos |
| **Beider-Morse** | variante más estricta de D-M | apellidos |

**B. Nearest neighbor** (más lento; requiere umbral/radio):

| Método | Qué hace | Nota |
|---|---|---|
| **Levenshtein** | nº de ediciones para igualar | "New York"/"newyork" = 3; "M. Makeba"/"Miriam Makeba" = 5 (y puede confundir con "B. Makeba" = 1) |
| **PPM** | compresión (Prediction by Partial Matching) | "muy laxo incluso con radio pequeño", último recurso |

**Blocking (el truco que hace viable el fuzzy):** antes de comparar, se ordenan los valores por similitud y se agrupan; OpenRefine usa por defecto **una ventana de 6 caracteres idénticos** y recomienda no bajar de 3. Blocking más grande = más lento; más chico = más falsos negativos. `[verificado + misma URL]`

**Normalización de la distancia:** Levenshtein no considera la longitud. `Fong`/`Fang` = 1 (muy distinto en proporción) vs `Sinéad O'Connor`/`Sinéad M. O'Connor` = 3 (casi iguales). Fórmula de la doc: `levenshtein(a,b) / (max(len(a),len(b)) + 1)` → 0.2 y 0.16 respectivamente. `[verificado + misma URL]`

**Umbrales recomendados para un CRM** `[inferido: no hay documentación pública de los umbrales exactos de Salesforce/HubSpot]`:

| Comparación | Método | Umbral sugerido |
|---|---|---|
| Email | exacto sobre normalizado (lowercase, sin `+tag` si se decide) | 1.0 → match |
| Teléfono | E.164 exacto; si no, últimos 7-9 dígitos | 1.0 → match |
| Dominio de empresa | exacto sobre eTLD+1, excluyendo free-mail | 1.0 → match |
| Nombre de persona | Jaro-Winkler ≥ 0.92 **y** mismo dominio/empresa | auto-merge |
| Nombre de empresa | token-set ratio ≥ 0.9, quitando sufijos (`S.A.`, `Ltda.`, `Inc.`) | auto-merge |
| Caso intermedio | 0.75-0.9 | **sugerir**, no decidir |

**Falsos positivos que hay que bloquear por regla, no por umbral:** `info@`, `ventas@`, `admin@` (misma empresa, muchas personas); dominios genéricos (`gmail.com`) para matching de empresa; nombres de 2 letras.

### 4.6 Qué hace el sistema al encontrar un duplicado

| Acción | Cuándo | Efecto |
|---|---|---|
| **Block** | duplicado casi seguro (email exacto) | rechaza el guardado, devuelve el/los IDs existentes |
| **Alert / Allow** | duplicado probable | guarda, pero avisa y deja reportar |
| **Auto-merge** | match exacto y sin conflicto | fusiona, conserva el registro maestro |
| **Skip** | import con `UPDATE_ONLY` y no existe | no hace nada, lo cuenta |
| **Quarantine** | múltiples candidatos | no decide, lo pone en cola de revisión |

### 4.7 Fusión sin perder historial

El modelo correcto (el de Salesforce) es **no borrar el historial**: mover las relaciones.

`merge()` combina **hasta tres registros del mismo tipo** en uno (el *main record*) y **borra los otros** (*victim records*). Si un victim tiene registros relacionados, **el main record se convierte en el nuevo padre**. `[verificado + https://developer.salesforce.com/docs/platform/api/guide/sforce-api-calls-merge.md]`

Detalles verificados que definen el diseño:

- Los valores de los no-main **no ganan** salvo que los pongas en el `masterRecord` antes de fusionar.
- Los victim **no desaparecen de la base**: quedan con `isDeleted=true` y `masterRecordId != null`, consultables con `queryAll()`. La consulta textual de la doc:

```soql
SELECT Id FROM Contact
WHERE isDeleted = true AND masterRecordId != null
  AND SystemModstamp > 2026-01-01T00:00:00Z
```

- Se pueden hacer **merges sucesivos** al mismo main record (cada `MergeResult` es su propia transacción).

**Reglas de merge que hay que implementar sí o sí** (si se hace a mano):

1. **El ID sobreviviente se elige por antigüedad y por historial**, no por "el último que llegó". Si el registro viejo tiene 400 actividades, sobrevive él.
2. **Reparentar todas las tablas hijas** (actividades, notas, negocios, tickets, adjuntos, relaciones many-to-many) en una transacción.
3. **Fusionar campos por regla**, no por "el que no sea nulo": para cada campo, una política (`prefer_master`, `prefer_most_recent`, `longest_non_empty`, `concat`).
4. **Emails y teléfonos: unión, no reemplazo.** Un contacto puede quedar con 3 emails; perder uno rompe la sincronización de correo futura. HubSpot lo hace explícito con `hs_additional_emails`.
5. **Registrar el merge** (`merged_into_id`, `merged_at`, `merged_by`, `rule_version`) y **redirigir** las URLs viejas al registro nuevo.
6. **Nunca reutilizar el ID del victim.** Si el sistema origen sigue mandando eventos con ese ID, deben resolverse al main (tabla de alias de identidad).

```sql
-- tabla de alias: sobrevive a cualquier merge
CREATE TABLE identity_alias (
  source        text NOT NULL,
  external_id   text NOT NULL,
  entity_id     uuid NOT NULL,
  merged_from   uuid,
  created_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (source, external_id)
);
```

> Sin `identity_alias`, el primer merge rompe la sincronización incremental para siempre: el origen sigue hablando de un ID que ya no existe.

---

## 5. Sincronización de correo y calendario

### 5.1 El patrón general: push para despertar, pull para traer

```
proveedor ──(push: "cambió algo", sin datos)──▶ endpoint → cola
                                                    │
                              (pull: dame los cambios desde el cursor) 
                                                    ▼
                                            API del proveedor
```

Ninguno de los tres grandes (Gmail, Graph, Calendar) manda el mensaje en el push. Gmail manda `{emailAddress, historyId}` en base64; Calendar manda solo headers; Graph basic manda solo el `id`.

### 5.2 Gmail: `historyId`

**Setup:** topic de Pub/Sub → suscripción (push o pull) → dar permiso de `publish` a `gmail-api-push@system.gserviceaccount.com` → `POST /gmail/v1/users/me/watch` con `{topicName, labelIds:["INBOX"], labelFilterBehavior:"INCLUDE"}`. `[verificado + https://developers.google.com/workspace/gmail/api/guides/push]`

**Respuesta del watch:** `{historyId, expiration}`. El `historyId` es el estado actual de la casilla.

**Notificación:** `POST` con `message.data` en **base64url** que decodifica a `{"emailAddress": "...", "historyId": "9876543210"}`. Hay que hacer ACK con 200 o Pub/Sub reintenta.

**Sincronización parcial:** `GET /gmail/v1/users/me/history?startHistoryId=<último conocido>`. Devuelve `history[]` con `messagesAdded`, `messagesDeleted`, `labelsAdded`, `labelsRemoved`, más `nextPageToken` y el `historyId` actual. `[verificado + https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.history/list]`

**Las tres trampas verificadas:**

1. **Los `historyId` no son contiguos.** Textual: "History IDs increase chronologically but are **not contiguous with random gaps** in between valid IDs." Nunca hagas `last + 1`; usa el `historyId` que devuelve la respuesta.
2. **Caducan.** "Typically valid for at least a week, but in some rare circumstances may be valid for only a few hours." Si `startHistoryId` está fuera de rango → **HTTP 404** → hay que hacer **full sync**. `[verificado + guides/sync]`
3. **El push se cae.** El `watch` expira a los **7 días**; Google recomienda renovarlo **a diario**. Y el push puede perder eventos: hay que hacer fallback a `history.list` periódico aunque no llegue notificación.

**Límite de tasa:** **1 notificación por segundo por usuario**; el servicio **descarta** las que excedan. Y cuidado con los loops (procesar una notificación que genere otra). `[verificado + guides/push]`

**Full sync (el fallback):** `messages.list` → batch de `messages.get` con `format=full` o `raw` → guardar el `historyId` del mensaje más reciente. `[verificado + guides/sync]`

```
def sync_gmail(user):
    try:
        page = history_list(user, start=user.history_id)
        for h in page.history:
            for added in h.messagesAdded:
                ingest_message(user, added.message.id)   # idempotente por id
        user.history_id = page.historyId                 # NO +1
    except HTTP404:
        full_sync(user)                                  # resetea el cursor
```

### 5.3 Microsoft Graph: `deltaLink`

**Delta query (pull):** `GET /me/mailFolders/{id}/messages/delta`. Devuelve `value[]` + `@odata.nextLink` (hay más páginas) o `@odata.deltaLink` (ronda terminada). Se guarda el `deltaLink` completo y se usa tal cual en la siguiente ronda. `[verificado + https://learn.microsoft.com/en-us/graph/delta-query-messages]`

Reglas verificadas:

- **El delta es por carpeta**, no por buzón. Hay que trackear cada carpeta.
- Los tokens son **opacos**: no se inspeccionan ni se reconstruyen; se copia la URL entera.
- `changeType=created|updated|deleted` filtra el tipo de cambio **solo si se pone en la request inicial** (queda codificado en el token).
- `$filter` solo admite `receivedDateTime ge/gt` y **limita a 5.000 mensajes**; `$orderby` solo `receivedDateTime desc`; **no hay `$search`**.
- Borrados: vienen como `{"id": "...", "@removed": {"reason": "deleted"}}` (o `"changed"` si es recuperable).
- **El mismo ítem puede aparecer varias veces** en la respuesta: hay que tolerar repeticiones.
- Un `410 Gone` con `Location` sin `$deltatoken` significa **resync completo**. También `syncStateNotFound` en errores 40x.
- Los tokens de Outlook **no tienen duración fija**: dependen de una caché interna; al desbordarse, los viejos se borran y toca full sync. `[verificado + https://learn.microsoft.com/en-us/graph/delta-query-overview]`

**Change notifications (push):** `POST /subscriptions` con `resource`, `changeType`, `notificationUrl`, `expirationDateTime`, `clientState`.

| Límite verificado | Valor |
|---|---|
| Vida máxima suscripción Outlook `message` | **< 7 días** (10.080 min); con *rich notifications* **< 1 día** (1.440 min) |
| Suscripciones activas por buzón | **1.000** para todas las apps |
| Latencia media / máxima de `message` | < 1 min / 3 min |
| Validación | el endpoint debe responder al `validationToken` con `text/plain` |
| Tipos | *basic* (solo id), *rich* (con datos), *lifecycle* (avisa que la suscripción va a caducar) |

`[verificado + https://learn.microsoft.com/en-us/graph/change-notifications-overview]`

**Estrategia canónica:** suscripción corta (o *lifecycle*) para despertar + `deltaLink` para traer + full resync ante `410`.

### 5.4 IMAP: `UID` + `UIDVALIDITY`

Para buzones IMAP genéricos (EspoCRM lo soporta: host, usuario, password, SSL, *Monitored Folders*, *Fetch since*). `[verificado + https://docs.espocrm.com/user-guide/imap-smtp-configuration/]`

| Concepto | Regla |
|---|---|
| `UID` | identificador **permanente dentro de un mailbox** mientras `UIDVALIDITY` no cambie; **nunca se reutiliza** |
| `UIDVALIDITY` | si cambia, **todos los UIDs cacheados son inválidos** → re-sync completo |
| `UIDNEXT` | próximo UID a asignar; no sirve como cursor por sí solo |
| `MODSEQ` (CONDSTORE) | permite "cambios desde X" sin re-leer flags |
| `IDLE` | push real: el servidor avisa `* n EXISTS` / `* n EXPUNGE` |

`[verificado: semántica de UID/UIDVALIDITY según RFC 3501, https://www.rfc-editor.org/rfc/rfc3501]`

```
cursor = (mailbox, uidvalidity, last_uid)
if select(mailbox).uidvalidity != cursor.uidvalidity:
    full_resync(mailbox)               # los UIDs viejos ya no significan nada
else:
    for msg in fetch("UID %d:*" % (last_uid + 1), "(UID FLAGS RFC822.SIZE BODY.PEEK[])"):
        ingest(msg); cursor.last_uid = msg.uid
```

Sin `BODY.PEEK[]` se marca el mensaje como leído al sincronizarlo: un bug clásico que enfurece a los usuarios.

### 5.5 Push vs polling

| | Push | Polling |
|---|---|---|
| Latencia | segundos | intervalo |
| Costo | casi cero en reposo | proporcional al intervalo × cuentas |
| Fiabilidad | **pierde eventos** (Gmail lo dice; Calendar lo dice) | no pierde, pero puede duplicar |
| Complejidad | webhook + validación + renovación | trivial |
| Uso correcto | despertar el worker | red de seguridad (cada 15-60 min) |

**Nunca uno solo.** Push para latencia, polling para completitud, delta para el dato.

### 5.6 Cómo se asocia un correo a la persona correcta

Orden de resolución (de más fuerte a más débil):

| Paso | Regla | Resultado |
|---|---|---|
| 1 | `Message-ID` ya visto | skip (idempotencia) |
| 2 | `In-Reply-To` / `References` apuntan a un hilo ya asociado | hereda la entidad del hilo (**lo más fiable**) |
| 3 | dirección `From` ∈ emails del contacto (incluyendo `hs_additional_emails`) | asocia a ese contacto |
| 4 | dirección ∈ `Cc` y hay **un solo** contacto conocido entre los participantes | asocia a ese |
| 5 | dominio del `From` = dominio de una empresa | asocia a la empresa + "contacto desconocido" |
| 6 | ninguna | crea `unknown_sender` o un lead, según política |

**Por qué el paso 2 va antes que el 3:** en un hilo, el `From` de la respuesta puede ser un compañero del cliente que nunca fue contactado. Si matcheas por dirección primero, creas un contacto nuevo y **partes el hilo en dos**. El hilo manda.

**Contrapartida:** los hilos con `Bcc` a una dirección de logging (Pipedrive `smart_bcc_email`) no tienen `In-Reply-To` útil porque el usuario reenvía; ahí manda la dirección.

**Direcciones que NO son personas:** `no-reply@`, `notifications@`, `calendar-notification@google.com`, `mailer-daemon@`. Se descartan como identidad pero **se guardan como mensaje** si el hilo ya existe (ej. un rebote es información valiosa).

### 5.7 Qué se guarda del cuerpo y qué se descarta

| Se guarda | Se descarta |
|---|---|
| `text/plain` normalizado (whitespace colapsado) | HTML completo (se guarda solo el texto extraído) |
| `snippet` (primeros ~200 chars) para listados | imágenes remotas y píxeles de tracking |
| firmas separadas del cuerpo (si se detectan) | CSS, fuentes, adjuntos inline decorativos |
| `headers` seleccionados (`Message-ID`, `In-Reply-To`, `References`, `Date`, `From/To/Cc`) | cadena de `Received:` completa (salvo análisis anti-spam) |
| `raw_size` y `raw_sha256` | el MIME completo, salvo en `email_raw` (ver §7) |

**Por qué descartar HTML:** tamaño (10-100× el texto), XSS si se renderiza, y porque el 90% es layout. Guardar `text/plain` + `snippet` cubre búsqueda y preview. Si el cliente necesita el HTML original, se sirve desde el crudo bajo demanda.

### 5.8 Adjuntos

Modelo correcto, en tres pasos:

1. **Metadatos en el CRM:** `filename`, `content_type`, `size`, `sha256`, `message_id`. Nunca el binario en la tabla principal.
2. **Binario en object storage** (S3/GCS), con clave = `sha256` (deduplicación natural: el mismo PDF adjunto en 50 correos se guarda una vez).
3. **Escaneo** (antivirus) y **límites**: tamaño máximo, tipos bloqueados (`.exe`, `.js`, macros), y **cuarentena** si el escaneo falla.

**Límites que obligan a diseñar así:** Graph incluye adjuntos "simples" en base64 en el mensaje hasta ~3 MB y exige *upload session* para mayores; Gmail entrega el MIME completo pero con topes de tamaño; Nylas **trunca** los payloads > 1 MB y manda un evento terminado en `.truncated` sin body, obligando a re-consultar el objeto. `[verificado el caso Nylas + https://developer.nylas.com/docs/cookbook/use-cases/build/retry-failed-webhooks/]` `[inferido el umbral exacto de Graph]`

> **Regla:** si el evento de sincronización puede venir truncado, el handler **debe** tener un camino de "re-query por id". Un `.truncated` sin re-query es un correo perdido con un 200 OK como coartada.

---

## 6. Idempotencia y reintentos

### 6.1 La verdad incómoda: todo es *at-least-once*

| Sistema | Garantía | Reintentos |
|---|---|---|
| Nylas | at-least-once explícito | hasta **5**, solo en `408/429/502/503/504/507` o timeout; **otros 4xx no se reintentan** |
| Twilio | sin reintento documentado para voice; firma `X-Twilio-Signature` para autenticidad | — |
| Gmail Pub/Sub | reintenta si no hay ACK 200 | según Pub/Sub |
| Google Calendar | reintenta con backoff exponencial solo si devuelves `500/502/503/504` | — |
| Graph | entrega "best effort" con latencia variable | — |
| HubSpot Imports | job asíncrono con estados; **máx. 3 concurrentes** | — |

`[verificado: Nylas, Calendar, HubSpot]` `[inferido: Twilio]`

**Conclusión:** el consumidor tiene que ser idempotente. No hay configuración que lo evite.

### 6.2 Claves de idempotencia

| Canal | Clave natural | Clave derivada (si no hay natural) |
|---|---|---|
| Gmail | `message.id` (y `threadId` para el hilo) | `sha256(raw_sha256 + mailbox)` |
| Graph | `message.id` | `sha256(internetMessageId)` |
| IMAP | `(uidvalidity, uid)` | `Message-ID` |
| Twilio | `CallSid` | — |
| HubSpot form | no hay | `sha256(email + formGuid + submittedAt)` |
| Salesforce Web-to-Lead | no hay | `sha256(email + oid + timestamp_minuto)` |
| CSV | `sha256(file) + row_number` | `sha256(clave_negocio)` |
| Webhook genérico | `notification.id` (Nylas) / header del proveedor | `sha256(body_crudo)` |

**Patrón de tabla de deduplicación:**

```sql
CREATE TABLE webhook_delivery (
  id              text PRIMARY KEY,          -- notification.id o hash
  source          text NOT NULL,
  received_at     timestamptz NOT NULL DEFAULT now(),
  expires_at      timestamptz NOT NULL,      -- TTL: 7-30 días
  payload_sha256  text NOT NULL,
  processed_at    timestamptz
);

-- claim atómico: gana exactamente uno
INSERT INTO webhook_delivery (id, source, expires_at, payload_sha256)
VALUES ($1,$2, now() + interval '14 days', $3)
ON CONFLICT (id) DO NOTHING
RETURNING id;          -- si no devuelve fila → duplicado → 200 y salir
```

### 6.3 El mismo lead dos veces en 200 ms

Este es *el* caso que rompe implementaciones ingenuas:

```
t=0ms     request A: SELECT ... WHERE email='x' → no existe
t=120ms   request B: SELECT ... WHERE email='x' → no existe   (¡todavía!)
t=140ms   A: INSERT → ok
t=200ms   B: INSERT → duplicado
```

Un `SELECT` seguido de `INSERT` **no es atómico**. Tres soluciones, en orden de preferencia:

**(1) Restricción única + `ON CONFLICT` (la correcta):**

```sql
CREATE UNIQUE INDEX ux_person_email
  ON person (lower(email_normalized))
  WHERE deleted_at IS NULL AND email_normalized IS NOT NULL;

INSERT INTO person (id, email_normalized, ...) VALUES (...)
ON CONFLICT (lower(email_normalized)) DO UPDATE
  SET updated_at = now(),
      phone = COALESCE(person.phone, EXCLUDED.phone)
RETURNING id, (xmax = 0) AS inserted;
```

`xmax = 0` distingue created vs updated en el mismo round-trip. Sin índice único, ninguna otra técnica es fiable.

**(2) Lock por clave de negocio** (cuando no puedes poner un índice único porque la clave es compuesta o blanda):

```sql
SELECT pg_advisory_xact_lock(hashtext('person:' || $email_normalized));
-- ahora sí: SELECT, decide, INSERT/UPDATE
```

Serializa solo a los que compiten por la misma clave. `[inferido: patrón PostgreSQL]`

**(3) Cola con clave de partición** (Kafka/SQS FIFO/Redis stream): el mismo `email_normalized` como partition key garantiza que A y B caen en el mismo consumidor, en orden. No evita duplicados por sí solo, pero elimina la carrera.

**Qué NO funciona:** cache en memoria (no es compartida), `INSERT ... WHERE NOT EXISTS` (sigue sin ser atómico frente a otro transacción concurrente sin lock), y "consultar la API antes de insertar" (latencia + TOCTOU).

### 6.4 Orden de eventos

Nylas lo dice explícitamente: **el orden no está garantizado**; puede llegar un `.updated` o `.deleted` antes del `.created`. `[verificado + retry-failed-webhooks]` Graph dice que no hay que asumir secuencia en las respuestas delta. `[verificado + delta-query-overview]`

Consecuencias de diseño:

- **Nunca aplicar un evento sin `occurred_at`.** Si el evento no trae timestamp, se usa `received_at` **solo** para ordenar, nunca como dato de negocio.
- **Last-write-wins por `occurred_at`, no por orden de llegada.**
- **Los deletes son "blandos" y reversibles** en la práctica: un `.deleted` que llega antes del `.created` no debe hacer fallar el `.created` posterior. Upsert tolerante: si llega `created` después de `deleted`, se resucita con `deleted_at=null`.
- **Un evento de "updated" con datos parciales no debe borrar campos.** Solo se aplican las claves presentes (`PATCH`, no `PUT`).

### 6.5 Reintentos, backoff y DLQ

```
attempt(n):  delay = min(cap, base * 2^n) + jitter(0, base)
             base=1s, cap=15min, n=1..8
```

| Situación | Acción |
|---|---|
| Error transitorio (timeout, 429, 5xx) | reintento con backoff |
| 4xx de validación | **no reintentar** (Nylas: "never retry", se descarta) |
| Firma inválida | no reintentar, 401, alerta de seguridad |
| Poison message (falla 8 veces) | **DLQ** con el crudo + último error |
| Job de import abortado | reanudar desde checkpoint, no desde cero |

**Rate limits reales que obligan a encolar:** HubSpot `429` al pasar 80.000.000 filas/día o 1.048.576 filas / 512 MB por archivo; Salesforce Bulk 150.000.000 registros/24h y 150 MB por job; Pipedrive tiene un sistema de "cost" por endpoint (GET persons = 10, merge = 40). `[verificado + HubSpot imports; Salesforce limits; Pipedrive Persons]`

---

## 7. Dato crudo vs. dato procesado

### 7.1 Qué guardar, dónde y por cuánto tiempo

| Capa | Contenido | Dónde | Retención recomendada |
|---|---|---|---|
| **L0 — crudo** | body exacto + headers HTTP + IP + timestamp | tabla `ingest_raw` (JSONB) o object storage si >100 KB | **30-90 días** operativo; 1-7 años si hay obligación legal |
| **L1 — normalizado** | el registro parseado, sin identidad resuelta | tabla `ingest_staging` | 30 días (se puede regenerar desde L0) |
| **L2 — canónico** | persona/empresa/actividad | tablas del CRM | permanente (con soft delete) |
| **L3 — historial de cambios** | campo, valor viejo, valor nuevo, quién, cuándo | `field_history` o CDC | permanente |
| **L4 — trazabilidad** | `raw_id` → `entity_id` | `ingest_lineage` | permanente |

**Por qué L0 no es opcional:** los propios proveedores caducan rápido. Salesforce borra jobs en estado terminal **a los 7 días**; Twilio borra el request inspector a los **30 días**; Gmail conserva historial ~1 semana. Si dependes de poder re-consultar el proveedor, tu ventana de reproceso es de días. `[verificado: Salesforce limits, Twilio FAQ, Gmail sync]`

### 7.2 Esquema de referencia

```sql
CREATE TABLE ingest_raw (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  source          text NOT NULL,              -- 'hubspot_form' | 'gmail' | 'csv' | 'twilio'
  source_account  text,                       -- portal, mailbox, org
  channel         text NOT NULL,
  external_id     text,                       -- CallSid, message.id, import row
  idempotency_key text NOT NULL,
  received_at     timestamptz NOT NULL DEFAULT now(),
  occurred_at     timestamptz,                -- si el emisor lo dice
  http_method     text,
  http_headers    jsonb,
  remote_ip       inet,
  content_type    text,
  body            jsonb,                      -- o bytea
  body_sha256     text NOT NULL,
  schema_version  int NOT NULL DEFAULT 1,
  status          text NOT NULL DEFAULT 'received',
      -- received|normalized|matched|persisted|quarantined|discarded
  error           jsonb,
  processed_at    timestamptz
);

CREATE UNIQUE INDEX ux_ingest_idem ON ingest_raw (source, idempotency_key);
CREATE INDEX ix_ingest_pending  ON ingest_raw (status, received_at) WHERE status IN ('received','normalized','matched');
CREATE INDEX ix_ingest_ext      ON ingest_raw (source, external_id);
```

**Tres decisiones importantes:**

1. `body` como `jsonb` **solo si el payload es JSON**. Para MIME/CSV va `bytea` o, mejor, object storage con la referencia en la fila.
2. `schema_version` desde el día uno. Cuando el proveedor cambie el payload, podrás reprocesar los viejos con el parser viejo.
3. `status` como máquina de estados explícita, no un booleano `processed`. Necesitas saber *dónde* se quedó.

### 7.3 Reproceso

```
replay(source, from, to, filter, schema_version=None):
    rows = select * from ingest_raw
           where source=$1 and received_at between $2 and $3 and status in ('persisted','quarantined')
    for r in rows:
        if schema_version and r.schema_version != schema_version: continue
        reprocess(r)   # misma función que el camino caliente, con flag is_replay=true
```

**Reglas del replay:**

- **Nunca** re-disparar efectos externos (emails, webhooks de salida, asignaciones) sin idempotencia. Un replay no debe mandar 4.000 correos.
- **Sí** debe recalcular normalización e identidad, porque el bug suele estar ahí.
- El replay debe escribir en `ingest_raw` un nuevo `status` o en una tabla `replay_log` (`raw_id`, `run_id`, `result`), no borrar el intento anterior.
- Si el reproceso cambia la identidad asignada (el email ahora matchea con otra persona), se necesita un **merge**, no un update silencioso.

### 7.4 PII y minimización

- El crudo contiene todo: cuerpos de correo, teléfonos, direcciones. Cifrado en reposo y **acceso restringido por rol** (no lo puede leer el mismo usuario que ve el CRM).
- **Redacción selectiva al momento de guardar** si hay datos que no se necesitan (ej. números de tarjeta detectados por regex).
- **Borrado por sujeto:** si el crudo es la fuente de reproceso, borrarlo rompe la reproducibilidad; pero la ley manda. Solución: borrar el crudo del sujeto y dejar un tombstone (`raw_id`, `deleted_reason`, `deleted_at`) para que el replay sepa que faltan datos a propósito.

---

## 8. Payloads reales

### 8.1 HubSpot — form submission (request)

```json
{
  "submittedAt": "1763308800000",
  "fields": [
    {"objectTypeId": "0-1", "name": "email",     "value": "maria.perez@acme.cl"},
    {"objectTypeId": "0-1", "name": "firstname", "value": "María"},
    {"objectTypeId": "0-1", "name": "lastname",  "value": "Pérez"},
    {"objectTypeId": "0-1", "name": "phone",     "value": "+56912345678"},
    {"objectTypeId": "0-1", "name": "company",   "value": "ACME SpA"},
    {"objectTypeId": "0-2", "name": "domain",    "value": "acme.cl"}
  ],
  "context": {
    "hutk": "9f2c1e0b7a...",
    "ipAddress": "190.160.12.34",
    "pageUri": "https://acme.cl/demo?utm_source=linkedin",
    "pageName": "Solicitar demo"
  },
  "legalConsentOptions": {
    "consent": {
      "consentToProcess": true,
      "text": "Acepto que ACME trate mis datos para contactarme.",
      "communications": [
        {"value": true, "subscriptionTypeId": 999, "text": "Quiero recibir novedades"}
      ]
    }
  }
}
```

Respuesta OK: `{"redirectUri": "...", "inlineMessage": "<p>Gracias</p>"}`. Error: `{"errors":[{"message":"...","errorType":"INVALID_EMAIL"}]}`. `[verificado + https://developers.hubspot.com/docs/api-reference/legacy/marketing/forms/v3-legacy/submit-data-authenticated.md]`

**Campos que trae y para qué sirven:** `hutk` es el pegamento con la sesión anónima (sin él no hay atribución); `objectTypeId` permite crear contacto y empresa en el mismo submit; `submittedAt` permite backfill; `legalConsentOptions` es el único registro de consentimiento.

### 8.2 HubSpot — Import request

```json
{
  "name": "November Marketing Event Leads",
  "importOperations": {"0-1": "UPSERT"},
  "dateFormat": "DAY_MONTH_YEAR",
  "files": [{
    "fileName": "Nov-event-leads.csv",
    "fileFormat": "CSV",
    "fileImportPage": {
      "hasHeader": true,
      "columnMappings": [
        {"columnObjectTypeId": "0-1", "columnName": "First Name", "propertyName": "firstname"},
        {"columnObjectTypeId": "0-1", "columnName": "Last Name",  "propertyName": "lastname"},
        {"columnObjectTypeId": "0-1", "columnName": "Email",      "propertyName": "email",
         "columnType": "HUBSPOT_ALTERNATE_ID"}
      ]
    }
  }]
}
```

`columnType: HUBSPOT_ALTERNATE_ID` marca la columna como identificador único. Alternativas: `HUBSPOT_OBJECT_ID`, `FLEXIBLE_ASSOCIATION_LABEL`, `ASSOCIATION_KEYS`. `[verificado + https://developers.hubspot.com/docs/guides/api/crm/imports]`

### 8.3 Salesforce — Web-to-Lead (POST form-encoded)

```http
POST /servlet/servlet.WebToLead?encoding=UTF-8 HTTP/1.1
Host: webto.salesforce.com
Content-Type: application/x-www-form-urlencoded

oid=00D5g000004XXXX&retURL=https%3A%2F%2Facme.cl%2Fgracias
&lead_source=Web
&first_name=Mar%C3%ADa&last_name=P%C3%A9rez
&email=maria.perez%40acme.cl&phone=%2B56912345678
&company=ACME+SpA&country=Chile
&00N5g00000ABCDE=Demo+Q4
```

Notas: `oid`, `retURL`, `lead_source`, `debug` están documentados; los nombres de campo (`first_name`, `email`, `00N...` para custom) se corresponden con la API de Lead. `[verificado: oid/retURL/lead_source/debug + https://web.archive.org/web/20140113102309/http://help.salesforce.com/apex/HTViewHelpDoc?id=setting_up_web-to-lead.htm]` `[inferido: composición exacta del ejemplo]`

**Respuesta:** redirección a `retURL`; con `debug=1`, a una página de depuración. No hay body JSON: el éxito se mide por redirección, lo que hace imposible distinguir "creado" de "redirigido" sin mirar la base.

### 8.4 Salesforce — REST create (respuesta) y error de duplicado

```json
// 201 Created
{ "id": "00Q5g00000AbCdE", "errors": [], "success": true }
```

`[verificado + https://developer.salesforce.com/docs/platform/api-rest/guide/dome-sobject-create.md]`

```json
// 400 — duplicate rule en modo Block
[{
  "message": "This record looks like an existing lead. Duplicate found.",
  "errorCode": "DUPLICATES_DETECTED",
  "fields": []
}]
```

Y en SOAP / DML, el error es un `DuplicateError` con `duplicateResult.matchResults[].matchRecords[]`, cada uno con `record.Id`, `record.Type` y `matchConfidence`. `[verificado + https://developer.salesforce.com/docs/platform/api/guide/sforce-api-objects-duplicateerror.md]`

### 8.5 Gmail — notificación push (Pub/Sub)

```json
{
  "message": {
    "data": "eyJlbWFpbEFkZHJlc3MiOiAidXNlckBleGFtcGxlLmNvbSIsICJoaXN0b3J5SWQiOiAiMTIzNDU2Nzg5MCJ9",
    "messageId": "2070443601311540",
    "publishTime": "2021-02-26T19:13:55.749Z"
  },
  "subscription": "projects/myproject/subscriptions/mysubscription"
}
```

`message.data` decodificado (base64url): `{"emailAddress": "user@example.com", "historyId": "9876543210"}`. `[verificado + https://developers.google.com/workspace/gmail/api/guides/push]`

**Lo que NO trae:** asunto, remitente, cuerpo, adjuntos, labels. Nada. Es una campanilla.

### 8.6 Gmail — `history.list` (respuesta)

```json
{
  "history": [{
    "id": "1234567891",
    "messages": [{"id": "18c1f...", "threadId": "18c1e..."}],
    "messagesAdded": [{"message": {"id": "18c1f...", "threadId": "18c1e...", "labelIds": ["INBOX","UNREAD"]}}],
    "labelsAdded": [{"message": {"id": "18c1f...", "threadId": "18c1e..."}, "labelIds": ["Label_123"]}]
  }],
  "historyId": "1234567900"
}
```

`[verificado: estructura y campos + https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.history/list]` — los objetos `message` dentro de history **típicamente solo traen `id` y `threadId`** (verificado textualmente), es decir: hay que hacer `messages.get` para cada uno.

### 8.7 Microsoft Graph — delta de mensajes

```json
{
  "@odata.deltaLink": "https://graph.microsoft.com/v1.0/me/mailFolders('AQMk...')/messages/delta?$deltatoken=GwcBoTmPuoGNlgXgF1nyUNMXY",
  "value": [
    {
      "@odata.type": "#microsoft.graph.message",
      "@odata.etag": "W/\"CQAAABYAAAAR...\"",
      "id": "AAMkADNkNAAASq35xAAA=",
      "subject": "Holiday hours update",
      "isRead": false,
      "sender": {"emailAddress": {"name": "Dana Swope", "address": "danas@contoso.com"}}
    },
    {
      "@odata.type": "#microsoft.graph.message",
      "id": "AAMkADk0MGFkODE3LWE4...",
      "@removed": {"reason": "deleted"}
    }
  ]
}
```

`[verificado + https://learn.microsoft.com/en-us/graph/delta-query-messages]`

### 8.8 Twilio — webhook de llamada entrante (form-encoded)

```
POST /voice HTTP/1.1
Content-Type: application/x-www-form-urlencoded
X-Twilio-Signature: 0/KCTR6DLpKmkAf8h8d5Z...

CallSid=CA1234567890abcdef12345&AccountSid=AC1234...&From=%2B5612345678&To=%2B5698765432
&CallStatus=ringing&Direction=inbound&ApiVersion=2010-04-01
&FromCity=Santiago&FromCountry=CL&ToCountry=CL
```

Y en el `StatusCallback` al colgar: `CallStatus=completed&CallDuration=143&RecordingUrl=https://api.twilio.com/...`. `[verificado + https://www.twilio.com/docs/voice/twiml]`

### 8.9 Nylas — notificación de webhook

```json
{
  "id": "5f6a7b8c-...",
  "type": "message.created",
  "created_at": 1763308800,
  "object": {
    "grant_id": "abc...",
    "object": "message",
    "id": "msg_123",
    "thread_id": "thr_456"
  }
}
```

`id` es estable entre reintentos → clave de deduplicación. Si `type` termina en `.truncated`, hay que re-consultar el objeto. `[verificado + https://developer.nylas.com/docs/cookbook/use-cases/build/retry-failed-webhooks/]` `[inferido: forma exacta del sobre]`

### 8.10 EspoCRM — Lead Capture

```json
{
  "firstName": "María",
  "lastName": "Pérez",
  "emailAddress": "maria.perez@acme.cl"
}
```

`POST /api/v1/LeadCapture/{apiKey}`. Los campos permitidos se limitan con *Payload Fields*. `[verificado + https://docs.espocrm.com/administration/web-to-lead/]`

---

## 9. Decisiones que cambian un diseño (checklist)

1. **Guardar el crudo antes de responder 200.** Sin esto, cada caída es pérdida definitiva.
2. **Índice único en la clave de identidad + `ON CONFLICT`.** No `SELECT` y después `INSERT`.
3. **Un `external_id` por sistema origen, y tabla de alias para merges.** Sin alias, el primer merge rompe la sync.
4. **Push + delta + polling de seguridad.** Los tres. El push pierde eventos por diseño (documentado por Google).
5. **Nunca confiar en el orden de llegada.** Usar `occurred_at`, no `received_at`, para resolver conflictos.
6. **Upsert por lotes con checkpoint.** La fila que falla va a cuarentena; el lote continúa.
7. **Reporte de import con `created+updated+skipped+failed+unprocessed = total`.** Si no cuadra, hay un bug.
8. **Mapeo de columnas versionado y validado contra el hash de headers.** Es la causa #1 de corrupción silenciosa.
9. **Formato de fecha declarado, no adivinado.** Y separar `created_at` (sistema) de `source_created_at` (archivo).
10. **Teléfono sin región no se normaliza: se rechaza.** E.164 no admite ambigüedad.
11. **El hilo manda sobre la dirección** al asociar correo a persona.
12. **Los adjuntos van a object storage con clave = hash.** Nunca en la tabla.
13. **La deduplicación de empresas por dominio no ocurre en la API de HubSpot.** Si construyes conector, la implementas tú.
14. **Los webhooks con payload > 1 MB pueden venir truncados.** Camino de re-query obligatorio.
15. **Retención explícita del crudo.** 30-90 días operativo, más si hay obligación legal; los proveedores caducan en 7-30 días.

---

## 10. Fuentes consultadas

**Salesforce**
- Web-to-Lead (Setting Up Web-to-Lead, snapshot 2014): http://web.archive.org/web/20140113102309/http://help.salesforce.com/apex/HTViewHelpDoc?id=setting_up_web-to-lead.htm
- Duplicate Rules — Create or Edit (snapshot 2015): http://web.archive.org/web/20151023064151/https://help.salesforce.com/apex/HTViewHelpDoc?id=duplicate_rules_create.htm
- Duplicate Rules — Overview (snapshot 2015): http://web.archive.org/web/20150912164232/https://help.salesforce.com/apex/HTViewHelpDoc?id=duplicate_rules_overview.htm
- Matching Rules — Overview (snapshot 2016): http://web.archive.org/web/2016/https://help.salesforce.com/apex/HTViewHelpDoc?id=matching_rules_overview.htm
- Duplicate Rule Header: https://developer.salesforce.com/docs/platform/api-rest/guide/headers-duplicaterules.md
- DuplicateError: https://developer.salesforce.com/docs/platform/api/guide/sforce-api-objects-duplicateerror.md
- findDuplicates(): https://developer.salesforce.com/docs/platform/api/guide/sforce-api-calls-findduplicates.md
- merge(): https://developer.salesforce.com/docs/platform/api/guide/sforce-api-calls-merge.md
- Create a Record (REST): https://developer.salesforce.com/docs/platform/api-rest/guide/dome-sobject-create.md
- Bulk API 2.0 Ingest: https://developer.salesforce.com/docs/platform/api-asynch/guide/bulk-api-2-0-ingest.md
- Bulk API limits: https://developer.salesforce.com/docs/platform/salesforce-app-limits-cheatsheet/guide/salesforce-app-limits-platform-bulkapi.md
- Failed / Successful / Unprocessed results: `.../guide/get-job-failed-results.md`, `.../get-job-successful-results.md`, `.../get-job-unprocessed-results.md`
- Upsert (Bulk API 2.0): https://developer.salesforce.com/docs/platform/api-asynch/guide/bulk-api-2-0-upsert.md

**HubSpot**
- Forms — Submit data (schema completo): https://developers.hubspot.com/docs/api-reference/legacy/marketing/forms/v3-legacy/submit-data-authenticated.md
- Forms guide: https://developers.hubspot.com/docs/guides/api/marketing/forms
- CRM Imports: https://developers.hubspot.com/docs/guides/api/crm/imports
- Contacts API: https://developers.hubspot.com/docs/guides/api/crm/objects/contacts
- Deduplicación (KB): https://knowledge.hubspot.com/records/deduplication-of-records
- Import de múltiples objetos (KB): https://knowledge.hubspot.com/import-and-export/import-objects

**Google Workspace**
- Gmail push: https://developers.google.com/workspace/gmail/api/guides/push
- Gmail sync (full/partial): https://developers.google.com/workspace/gmail/api/guides/sync
- Gmail history.list: https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.history/list
- Calendar push: https://developers.google.com/workspace/calendar/api/guides/push

**Microsoft Graph**
- Delta query overview: https://learn.microsoft.com/en-us/graph/delta-query-overview
- Messages delta: https://learn.microsoft.com/en-us/graph/delta-query-messages
- Change notifications: https://learn.microsoft.com/en-us/graph/change-notifications-overview

**Twilio**
- TwiML / parámetros de request y StatusCallback: https://www.twilio.com/docs/voice/twiml
- Webhooks FAQ (firma, inspector 30 días): https://www.twilio.com/docs/usage/webhooks/webhooks-faq
- E.164: https://www.twilio.com/docs/glossary/what-e164

**Nylas**
- Retry e idempotencia: https://developer.nylas.com/docs/cookbook/use-cases/build/retry-failed-webhooks/
- Notifications overview: https://developer.nylas.com/docs/v3/notifications/

**Pipedrive**
- Persons API v2: https://developers.pipedrive.com/docs/api/v1/Persons

**EspoCRM**
- Import: https://docs.espocrm.com/administration/import/
- Web-to-Lead / Lead Capture: https://docs.espocrm.com/administration/web-to-lead/
- Duplicate checking (código): https://docs.espocrm.com/development/duplicate-check/
- Phone numbers (E.164, validación): https://docs.espocrm.com/administration/phone-numbers/
- IMAP/SMTP: https://docs.espocrm.com/user-guide/imap-smtp-configuration/

**Odoo**
- Export and import data: https://www.odoo.com/documentation/18.0/applications/essentials/export_import_data.html

**Algoritmos de similitud y normalización**
- OpenRefine — Cluster and edit: https://openrefine.org/docs/manual/cellediting#cluster-and-edit
- RFC 4180 (CSV): https://www.rfc-editor.org/rfc/rfc4180
- RFC 3501 (IMAP, UID/UIDVALIDITY): https://www.rfc-editor.org/rfc/rfc3501

---

*Fin del documento. Los pipelines descritos son reconstruibles de forma independiente: cada uno declara su entrada, su transformación, su persistencia y su ruta de fallo.*
