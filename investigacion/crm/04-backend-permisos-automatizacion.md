# Backend de un CRM: API, permisos, automatización, auditoría y operación

**Documento de arquitectura — especificación desde cero, sin copiar errores.**
Versión 1.0 · Alcance: capa servidor de un CRM multi-usuario (SaaS de un tenant por despliegue o self-hosted).
Fuentes primarias: Salesforce, HubSpot, EspoCRM, Odoo, Twenty, Pipedrive, Stripe, Temporal, n8n, RFC 7644/9865.

Convención de marcas: **[verificado + URL]** = el dato fue leído en la fuente citada. **[inferido]** = conclusión propia de arquitectura, sin cita directa.

---

## Índice

0. [Resumen de decisiones](#0-resumen-de-decisiones)
1. [Estilo de API](#1-estilo-de-api)
2. [Autenticación](#2-autenticación)
3. [Autorización](#3-autorización-la-parte-que-todos-subestiman)
4. [Auditoría](#4-auditoría)
5. [Automatización sin código](#5-automatización-sin-código)
6. [Colas y trabajos](#6-colas-y-trabajos)
7. [Webhooks salientes](#7-webhooks-salientes)
8. [Operación](#8-operación)
9. [Cinco errores caros](#9-cinco-errores-de-diseño-caros-y-comunes)
10. [Self-hosted en 2–4 GB de RAM](#10-qué-haría-distinto-en-un-servidor-modesto)
11. [Apéndice: DDL mínimo y checklist](#11-apéndice)

---

## 0. Resumen de decisiones

| Dimensión | Decisión recomendada | Por qué |
|---|---|---|
| Estilo de API | REST orientado a recursos + `fields=` explícito. GraphQL solo si hay un cliente único y se limita profundidad/complejidad | REST es cacheable, observable y barato de servir; GraphQL traslada costo de cómputo al servidor y complica permisos por campo |
| Campos personalizados | Metadata-driven: definición en tabla, almacenamiento híbrido (columnas core + `JSONB custom`), proyección dinámica | Evita `ALTER TABLE` en caliente y evita EAV puro |
| Paginación | Cursor (keyset) sobre `(sort_key, id)` con `id` como desempate obligatorio | Offset se degrada y duplica/pierde filas con escrituras concurrentes |
| Auth | Sesión (cookie httpOnly) para la UI, API keys por integración, OAuth2 para apps de terceros, SAML/OIDC para SSO, SCIM para provisión | Cada uno resuelve un problema distinto; mezclarlos produce agujeros |
| Permisos | 4 capas: scope de app → objeto → campo → registro. Evaluación "lo más restrictivo gana", salvo sharing que solo abre | Es el modelo mental que el cliente ya conoce de Salesforce/EspoCRM |
| Row-level | Precompilar la visibilidad a **una cláusula SQL** (`owner`, `team`, `hierarchy`, `shares`) — nunca post-filtrar en la aplicación | Elimina N+1 y elimina la clase de bug "el listado muestra lo que el detalle oculta" |
| Auditoría | Append-only, particionada por mes, solo campos marcados, diff y no payload, retención por capas | La auditoría total es el mayor consumidor de disco de un CRM |
| Automatización | Motor de reglas como datos, ejecución en un solo paso por transacción, guardas de profundidad y de campo cambiado | Los bucles son el fallo número uno de los motores no-code |
| Async | Todo lo que sale del proceso: correo, webhooks, importación, exportación, recálculo. **Nunca** el commit de la venta | El usuario espera confirmación; el resto puede esperar |
| Webhooks | HMAC-SHA256 sobre `timestamp.cuerpo_crudo`, `event_id`, reintentos con backoff, sin garantía de orden | Es el estándar de facto (Stripe) y sobrevive a fallos parciales |
| Despliegue modesto | Monolito modular + PostgreSQL único + cola en PostgreSQL (`SKIP LOCKED`) + FTS nativo | Cada componente extra son ~150–300 MB de RAM que no hay |

---

## 1. Estilo de API

### 1.1 Qué usa cada producto

| Producto | Estilo primario | Estilo secundario | Notas verificables |
|---|---|---|---|
| **Salesforce** | REST (`/services/data/vXX.X/sobjects/...`), SOAP legacy | Bulk API 2.0 (CSV asíncrono), Composite API (hasta 25 subrequests), GraphQL (beta/limitado) | La API REST respeta FLS y sharing del usuario autenticado **[verificado + https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/limits.htm]** |
| **HubSpot** | REST con versión en la ruta (`/crm/objects/2026-09/contacts`) | Search API (POST con filtros), Batch endpoints, GraphQL (limitado) | La versión va en el path y no cambia los límites de tasa **[verificado + https://developers.hubspot.com/docs/developer-tooling/platform/usage-guidelines]** |
| **Twenty** | GraphQL (`/graphql` para datos, `/metadata` para esquema) | REST (`/rest/...`) con `starting_after`/`ending_before` | Dos endpoints GraphQL separados: datos vs metadata **[verificado + https://github.com/twentyhq/twenty/pull/20092]** |
| **Odoo** | RPC (JSON-RPC/XML-RPC) sobre modelos | REST solo vía módulos de terceros | No hay recursos: se invocan métodos de modelo (`search_read`, `write`) **[verificado + https://www.odoo.com/documentation/17.0/developer/reference/backend/security.html]** |
| **EspoCRM** | REST orientado a entidades (`/api/v1/Contact`) | — | ACL se evalúa por scope y por campo en cada request **[verificado + https://docs.espocrm.com/administration/roles-management/]** |
| **Pipedrive** | REST v1 y v2 (v2 más eficiente en tokens) | — | v2 permite páginas de hasta 500 y cuesta menos tokens **[verificado + https://pipedrive.readme.io/docs/core-api-concepts-rate-limiting]** |
| **Stripe** | REST, versionado por fecha, cursor `starting_after` | Search API con `page`/`next_page` | `limit` 1–100, default 10, orden cronológico inverso **[verificado + https://docs.stripe.com/api/pagination]** |

**Lectura de arquitecto.** Nadie con volumen real usa un solo estilo. El patrón común es: REST para el 90 % del CRUD, un endpoint de búsqueda aparte (porque la semántica de filtros es distinta de la de listar), un canal masivo asíncrono para import/export, y un lote (batch/composite) para reducir round-trips.

### 1.2 REST vs GraphQL vs RPC — el criterio honesto

| Criterio | REST | GraphQL | RPC |
|---|---|---|---|
| Permisos por campo | Proyección explícita (`fields=`) — fácil de auditar | Selection set — elegante, pero hay que validar profundidad, alias, fragmentos y `__typename` | Difícil: el contrato es el método, no el dato |
| Costo de una consulta | Predecible | Impredecible (N+1 del cliente, *resolvers* anidados) | Predecible |
| Cache HTTP | Sí, nativo | Muy difícil (POST, cuerpo variable) | No |
| Versionado | Path (`/v1`) o fecha | Evolutivo con deprecación de campos | Por método |
| Bulk / import | Endpoint aparte | Mal encaje | Buen encaje |
| Depuración en producción | `curl` + logs | Requiere tooling | `curl` |

**Caso Twenty, evidencia de por qué GraphQL no es gratis.** Su paginación por cursor falló de tres formas distintas y simultáneas, todas detectables solo con tests exhaustivos:

1. El cursor se calculaba solo con la primera columna de `orderBy`; con valores duplicados la segunda página repetía la primera **[verificado + https://github.com/twentyhq/twenty/issues/20520]**.
2. El cursor serializaba `undefined` cuando el campo de orden no estaba en el *selection set* (porque el `SELECT` se recorta por permisos de campo), y la paginación seguía por `id`: perdía ~50 % de filas por salto y reportaba `hasNextPage: false` **[verificado + https://github.com/twentyhq/twenty/pull/24354]**.
3. Las condiciones keyset no eran *null-aware*: `campo > NULL` no devuelve nada, así que la paginación moría exactamente en el primer `NULL` **[verificado + mismo PR]**.

Conclusión aplicable a cualquier stack: **el cursor debe construirse desde la misma lista de claves de orden que usa el SQL, incluir desempate por `id`, y tratar `NULL` explícitamente.** Si el `SELECT` se recorta por permisos, los valores de orden deben seleccionarse igual (columnas ocultas) aunque no se devuelvan.

### 1.3 Endpoints para objetos con campos personalizados

Tres modelos de almacenamiento, con consecuencias distintas:

| Modelo | Cómo se ve | Ventaja | Defecto |
|---|---|---|---|
| **EAV** (una fila por valor) | `record_values(record_id, field_id, value_text, value_num, value_date)` | Cualquier campo sin migración | Toda lectura es un `PIVOT`; filtros y ordenamiento lentos; tipos inconsistentes |
| **Columnas físicas + metadata** | `ALTER TABLE ADD COLUMN` cuando el admin crea un campo | Índices normales, SQL simple | `ALTER TABLE` bloqueante en tablas grandes (ver §8.1) |
| **Híbrido (recomendado)** | Columnas reales para campos núcleo y de negocio; `custom JSONB` + GIN para el resto; metadata dice dónde vive cada campo | Consultas rápidas sobre lo importante, flexibilidad para lo demás | Requiere una capa de traducción campo→expresión SQL |

Regla de negocio que casi nadie implementa y debería: **un campo personalizado creado por el usuario no debe ser indexable por defecto**, y debe tener un tipo declarado y validado en el servidor (no "todo texto").

**¿El cliente pide los campos o el servidor decide?**

Los tres patrones existen en producción:

- **Por defecto completo, con exclusión** (Salesforce): devuelve todos los campos; `?fields=` en algunos endpoints.
- **Por defecto mínimo, con inclusión** (HubSpot): `?properties=email,firstname`; si no pides, recibes un set reducido.
- **Selection set** (Twenty/GraphQL): el cliente declara exactamente la forma.

Recomendación: **por defecto, un conjunto "core" estable; inclusión explícita para el resto; y una regla dura: los campos pedidos se intersectan con los permisos de campo del usuario, y los no permitidos se omiten del payload (no se devuelven como `null`)**. Devolver `null` filtra la existencia del campo y confunde a la UI; devolver el campo con valor `null` cuando existe y no es legible es una fuga de información sutil (el cliente deduce que hay dato).

```
GET /v1/deals?fields=name,amount,stage,close_date,custom.industry&limit=50&after=eyJ...
                                                                  │
                                          ┌───────────────────────┘
                                          ▼
                        ┌──────────────────────────────────┐
                        │ 1. validar que los campos existen │
                        │ 2. intersectar con FLS(read)      │
                        │ 3. proyectar solo esos en SQL     │
                        │ 4. devolver meta.allowed_fields   │
                        └──────────────────────────────────┘
```

### 1.4 Paginación: cursor vs offset

| Esquema | Ejemplo real | Ventajas | Límites concretos |
|---|---|---|---|
| **Offset/limit** | `?offset=200&limit=50` | Simple, permite "página 7" | `OFFSET` grande = escaneo; con inserciones concurrentes duplica y salta filas; profunda degradación |
| **Cursor opaco (keyset)** | Stripe `starting_after=obj_x`; HubSpot `after`; Twenty `after` | Estable con escrituras, O(log n) con índice | No permite saltar a un número de página; requiere orden total determinista |
| **Cursor con tope** | HubSpot Search: 10.000 resultados por query, luego `400` | Protege al servidor | Obliga a *chunking* por rango de fechas para exportar **[verificado + https://developers.hubspot.com/docs/api-reference/latest/crm/search-the-crm]** |

Números reales que conviene copiar como referencia de diseño:

- Stripe: `limit` 1–100, **default 10**, `has_more`, orden cronológico inverso, `starting_after` y `ending_before` mutuamente excluyentes **[verificado + https://docs.stripe.com/api/pagination]**.
- HubSpot: 200 objetos por página, máximo 5 `filterGroups` y 18 filtros, cuerpo de query ≤ 3.000 caracteres, 5 req/s, 10.000 resultados por query **[verificado + misma URL de Search API]**.
- Pipedrive v2: hasta 500 por página (y más barato en tokens que v1) **[verificado + https://pipedrive.readme.io/docs/core-api-concepts-rate-limiting]**.

**Contrato de paginación recomendado (copiable):**

```json
{
  "data": [ ... ],
  "page": {
    "next_cursor": "eyJ2IjoxLCJrIjpbIjIwMjYtMTEtMTZUMTA6MDA6MDBaIiwiMDE4Zi4uLiJdfQ",
    "has_more": true,
    "limit": 50
  }
}
```

```python
# Cursor = (sort_value, id). NUNCA solo sort_value.
def build_keyset(order_by, last):
    # order_by = [(col, 'asc'|'desc'), ..., ('id', 'asc')]  # id SIEMPRE al final
    parts, params = [], []
    for i, (col, direction) in enumerate(order_by):
        eq_prefix = " AND ".join(f"{c} = %s" for c, _ in order_by[:i])
        op = ">" if direction == "asc" else "<"
        if last[col] is None:
            null_cond = f"{col} IS NOT NULL" if direction == "asc" else "FALSE"
            parts.append(f"({eq_prefix} AND {null_cond})")
        else:
            parts.append(f"({eq_prefix} AND ({col} {op} %s OR {col} IS NULL))"
                         if direction == "asc" else
                         f"({eq_prefix} AND {col} {op} %s)")
        params.extend([last[c] for c, _ in order_by[:i]] + ([last[col]] if last[col] is not None else []))
    return "(" + " OR ".join(parts) + ")", params
```

Tres reglas que evitan el 90 % de los bugs de paginación:

1. **Desempate obligatorio por `id`** (o clave primaria monotónica). Sin él, dos filas con el mismo `updated_at` hacen que la página 2 repita la 1.
2. **`NULL` explícito**: decidir `NULLS FIRST/LAST` y reflejarlo en la comparación.
3. **El cursor se valida**: si el cliente manda un cursor generado con otro `order_by`, se responde `400 INVALID_CURSOR` en vez de continuar en silencio. Twenty terminó implementando esto **[verificado + PR 24354]**.

### 1.5 Errores, idempotencia y lotes

- **Formato de error único** (`type`, `code`, `message`, `details[]`, `request_id`). HubSpot devuelve `category`, `correlationId`, `context` — el `correlationId` es lo que hace soportable un ticket **[verificado + https://developers.hubspot.com/docs/developer-tooling/platform/usage-guidelines]**.
- **Idempotency-Key** en todo `POST` que crea dinero o registros: guardar `(key, endpoint, hash_del_cuerpo) → respuesta`, con TTL de 24 h. Stripe lo popularizó y es la única defensa real contra el reintento del cliente **[verificado + https://docs.stripe.com/api/idempotent_requests]**.
- **Batch**: aceptar hasta N operaciones por request y devolver resultado por ítem con status individual (`207`-like). Es lo que permite a un integrador sincronizar 10.000 contactos sin 10.000 round-trips.

---

## 2. Autenticación

### 2.1 Los cinco mecanismos y para qué sirve cada uno

| Mecanismo | Quién lo usa | Vida del credencial | Riesgo principal |
|---|---|---|---|
| **Cookie de sesión** (httpOnly, SameSite=Lax, Secure) | UI web | 8–12 h + refresh silencioso | CSRF; se mitiga con token anti-CSRF y `SameSite` |
| **API key** (header `Authorization: Bearer` o `X-API-Key`) | Integraciones internas, scripts, BI | Larga, revocable, **con scopes** | Se filtra en logs/repos; debe ser hash en DB, mostrada una vez |
| **OAuth2 authorization code + PKCE** | Apps de terceros, marketplace | Access 30–60 min + refresh rotativo | Redirect URI mal validada, scopes excesivos |
| **OAuth2 client credentials** | Servicio a servicio | Corta | Sin usuario → hay que decidir de quién son los permisos |
| **SAML 2.0 / OIDC (SSO)** | Empresas | Sesión del IdP | Mapeo de atributos y *just-in-time provisioning* mal hecho crea usuarios duplicados |

**El detalle que se olvida siempre:** una API key no es "un usuario". Es un *principal* con su propio conjunto de permisos, y debe aparecer distinto en la auditoría (`actor_type='api_key'`). Si la key hereda los permisos de su creador, rotar al creador rompe la integración y nadie sabe por qué.

### 2.2 Scopes: pedir, otorgar y **verificar lo otorgado**

HubSpot es la mejor referencia porque separa tres listas:

- `requiredScopes`: sin ellas la instalación falla.
- `optionalScopes`: el admin puede rechazarlas; la instalación sigue.
- `conditionallyRequiredScopes`: solo si el `scope` del install URL las pide.
**[verificado + https://developers.hubspot.com/blog/build-with-confidence-app-governance]**

Y lo crítico: **el token devuelve el array `scopes` realmente otorgado**, que puede ser distinto del pedido. El código debe leer ese array y apagar features, no asumir **[verificado + misma fuente]**.

HubSpot migró de scopes amplios (`tickets`) a granulares (`crm.objects.tickets.read` / `.write`), obligatorios para apps nuevas desde el 8 de septiembre de 2026 **[verificado + misma fuente]**. Lección de diseño: **nombrar el scope como `objeto.acción`, no como módulo**. `crm.read` es un permiso que nadie puede auditar.

Matriz mínima viable de scopes para un CRM:

| Scope | Concede |
|---|---|
| `crm.objects.<obj>.read` | Leer registros del objeto (sujeto a row-level) |
| `crm.objects.<obj>.write` | Crear/editar |
| `crm.objects.<obj>.delete` | Borrar (separado de write: casi nadie debería tenerlo) |
| `crm.schemas.<obj>.read/write` | Ver/crear campos personalizados |
| `crm.objects.owners.read` | Ver propietarios y equipos (necesario para asignación) |
| `settings.users.read` | Leer usuarios (para mapeos de SSO/SCIM) |
| `webhooks.write` | Gestionar suscripciones salientes |
| `offline_access` | Refresh token |

### 2.3 SSO y SCIM

**SSO (SAML/OIDC)** resuelve autenticación, **no** provisión. Sin SCIM, el ciclo de vida (alta, cambio de área, baja) se hace a mano y las bajas se olvidan: la cuenta del ex-empleado sigue con acceso hasta que alguien la desactive.

**SCIM 2.0 (RFC 7644)** es el estándar de facto. Lo mínimo que un IdP exige:

| Requisito | Detalle | Fuente |
|---|---|---|
| `GET /scim/v2/Users?filter=userName eq "x"` | Okta **exige** este filtro para decidir si crea o actualiza | **[verificado + https://developer.okta.com/docs/api/openapi/okta-scim/guides/scim-20]** |
| `POST /Users`, `PUT`, `PATCH`, `DELETE` | `PATCH` con `PatchOp` (`add`/`replace`/`remove`) | **[verificado + https://www.rfc-editor.org/rfc/rfc7644]** |
| Desactivar ≠ borrar | La baja es `PATCH {"op":"replace","path":"active","value":false}`; el `DELETE` se reserva para GDPR | **[verificado + Okta SCIM guide]** |
| `/Groups` requiere `PATCH` | Entra ID solo soporta grupos si el endpoint implementa `PATCH` | **[verificado + https://learn.microsoft.com/en-us/entra/identity/app-provisioning/use-scim-to-provision-users-and-groups]** |
| Paginación `startIndex`/`count` | `startIndex` es 1-based; si hay demasiados resultados, `400` con `scimType: "tooMany"` | **[verificado + RFC 7644 §3.4.2.1]** |
| No aceptar passwords por SCIM | El IdP no debe enviar credenciales | **[verificado + guías de provisioning]** |
| Cursor pagination (nuevo) | RFC 9865 añade paginación por cursor a SCIM; Entra ya la usa | **[verificado + https://learn.microsoft.com/en-us/entra/identity/app-provisioning/entra-id-scim-api-reference]** |

Diseño recomendado: **un token SCIM por tenant, hasheado en DB, con scope único `scim:provision`**, y un log de cada operación (quién creó a quién). El error clásico es reutilizar el token de administrador de la API.

### 2.4 Middleware de autenticación (pseudocódigo)

```python
def authenticate(request) -> Principal:
    # 1. Orden de precedencia explícito y documentado
    if tok := request.headers.get("Authorization", "").removeprefix("Bearer "):
        if tok.startswith("sk_live_"):
            return api_key_principal(tok)          # hash lookup, cache 60s
        return oauth_principal(tok)                # JWT verify + introspection fallback
    if sid := request.cookies.get("sid"):
        return session_principal(sid)              # server-side session, no JWT en cookie
    raise Unauthorized("no_credentials")

# Principal SIEMPRE lleva tenant_id: sin él, el aislamiento depende del WHERE de cada query
@dataclass(frozen=True)
class Principal:
    tenant_id: str
    actor_id: str            # usuario real
    actor_type: str          # user | api_key | oauth_app | system
    on_behalf_of: str | None # impersonación auditada
    scopes: frozenset[str]
    role_ids: tuple[str, ...]
    team_ids: tuple[str, ...]
    hierarchy_path: str      # materialized path: "/1/4/9/"
```

`hierarchy_path` como *materialized path* es lo que hace que "mi jerarquía ve lo de sus subordinados" sea un `LIKE '/1/4/%'` indexable en vez de un recorrido recursivo por request.

---

## 3. Autorización, la parte que todos subestiman

### 3.1 Las cuatro capas y su orden

```
  ┌─────────────────────────────────────────────────────────────┐
  │ 1. SCOPE / APP      ¿esta app puede tocar este objeto?      │  ← se rechaza temprano (401/403)
  ├─────────────────────────────────────────────────────────────┤
  │ 2. OBJETO (CRUD)    ¿este usuario puede editar deals?       │  ← perfil/rol/permiso
  ├─────────────────────────────────────────────────────────────┤
  │ 3. CAMPO (FLS)      ¿puede ver/editar `amount`?             │  ← proyección + validación de input
  ├─────────────────────────────────────────────────────────────┤
  │ 4. REGISTRO (RLS)   ¿puede ver ESTE deal?                   │  ← cláusula SQL
  └─────────────────────────────────────────────────────────────┘
```

Regla de composición: **las capas 1–3 son intersección (lo más restrictivo gana). La capa 4 también es intersección, salvo que dentro de ella las "sharing rules" solo pueden ampliar.** Salesforce lo dice explícito: los permisos de objeto y de campo tienen precedencia sobre las reglas de sharing, y una sharing rule nunca puede ser más estricta que el OWD **[verificado + https://developer.salesforce.com/docs/platform/apex-dev/references/apex-dev/apex-classes-perms-enforcing.html]** y **[verificado + https://help.salesforce.com/s/articleView?id=platform.security_about_sharing_rules.htm&language=en_US&type=5]**.

### 3.2 RBAC: roles, no "tipos de usuario"

Los tres productos resuelven lo mismo con vocabulario distinto:

| Concepto | Salesforce | EspoCRM | Odoo |
|---|---|---|---|
| Unidad de permiso | Profile / Permission Set | Role | `res.groups` |
| Asignación | 1 perfil + N permission sets | N roles (usuario **y** equipos) | N grupos (+ implícitos) |
| Composición | Unión (el permiso existe si algún set lo da) | **Unión permisiva**: "a more permissive rule will have a higher priority" **[verificado + https://docs.espocrm.com/administration/roles-management/]** | **Unión aditiva**: "a user's accesses are the union of the accesses they get through all their groups" **[verificado + https://www.odoo.com/documentation/17.0/developer/reference/backend/security.html]** |
| Nivel mínimo | — | `no` | Sin ACL = sin acceso |
| Rol base | — | "baseline role" aplicado a todos los internos (v9.2+) **[verificado + misma fuente]** | Grupo `base.group_user` |

**Decisión de diseño:** roles componibles y aditivos, con un **rol base** que garantice el mínimo (leer lo propio, comentar) y roles que sumen. Nunca un modelo donde "quitar un rol" degrada silenciosamente: eso produce el bug "el usuario no ve nada y nadie sabe qué rol falta".

Límite práctico que Salesforce documenta y conviene imitar: **máximo ~10 niveles de jerarquía** y **300 sharing rules por objeto, de las cuales 50 pueden ser por criterio** **[verificado + https://help.salesforce.com/s/articleView?id=platform.security_about_sharing_rules.htm&language=en_US&type=5]**. No porque el software no aguante más, sino porque nadie puede razonar sobre 800 reglas.

### 3.3 Permisos por objeto

CRUD + dos "válvulas de escape" que hay que tener y hay que auditar:

- `view_all` (Salesforce: *View All Records*): ignora row-level para lectura.
- `modify_all`: ignora row-level para escritura.
- Salesforce advierte que `View All` **no** da visibilidad de campos: FLS sigue aplicando **[verificado + https://salesforce.stackexchange.com/questions/237117 — cita la doc oficial de "Permissions and the Salesforce REST API"]**.

Esas válvulas son la causa del 80 % de los incidentes de "por qué este usuario ve todo". Deben requerir aprobación explícita, expirar, y generar una alerta cuando se activan.

### 3.4 Field-Level Security (FLS)

**Cómo funciona de verdad en Salesforce.** Existe una tabla `FieldPermissions` con `(ParentId, SObjectType, Field, PermissionsRead, PermissionsEdit)`. La regla no obvia: **si no hay fila, no hay acceso** — no existen filas con ambos permisos en `false`, se borran **[verificado + https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_fieldpermissions.htm vía https://wiki.sfxd.org/books/best-practices/page/field-permissions---basic-functionality]**.

Consecuencias operativas:

1. **Leer sin FLS no es "devolver null": es un error de columna inexistente.** Si quitas `Industry` al usuario, un `GET ...?fields=Name,Industry` responde "No such column 'Industry'" **[verificado + https://salesforce.stackexchange.com/questions/237117]**.
2. **Escribir un campo no permitido debe fallar, no ignorarse.** Ignorar el campo hace que el usuario crea que guardó y el dato se pierda en silencio.
3. **El `describe` es parte del contrato**: un endpoint que dice qué campos existen *para este usuario*, con `readable`/`editable`. Sin él, cada cliente hardcodea el esquema.

En EspoCRM el FLS es más simple y más limitado: solo `yes`/`no` por campo y por acción, sin niveles `own`/`team` — una limitación que su propio foro discute como carencia **[verificado + https://docs.espocrm.com/administration/roles-management/]**.

**Implementación recomendada (la que evita fugas):**

```
       Request: GET /deals?fields=name,amount,salary_band
                          │
        ┌─────────────────▼──────────────────┐
        │ allowed = fsl_read(principal, Deal)│  ← 1 query, cacheada por (rol, objeto)
        └─────────────────┬──────────────────┘
                          ▼
        SELECT name, amount          ← salary_band NO aparece en el SQL
          FROM deals
         WHERE <visibilidad> AND <filtros>
```

- La proyección se decide **antes** del SQL. Nunca `SELECT *` + borrar claves en Python.
- La caché de FLS se invalida por evento (`role_changed`, `field_created`), no por TTL corto.
- Los `JOIN` también: si `deals.owner_id` no es legible, no se puede ordenar por él (Twenty tuvo que rechazar el orden por campo restringido porque el cursor embebe el valor **[verificado + PR 24354]**).

### 3.5 Row-level: los tres modelos que hay que entender

#### 3.5.1 Salesforce — "precomputar y compartir"

El modelo tiene cuatro piezas:

1. **OWD (Organization-Wide Defaults)** por objeto: `Private`, `Public Read Only`, `Public Read/Write`, `Controlled by Parent`. Es el punto de partida **más restrictivo** posible; todo lo demás amplía **[verificado + https://help.salesforce.com/s/articleView?id=platform.security_data_access.htm&language=en_US&type=5]**.
2. **Jerarquía de roles**: el acceso fluye **hacia arriba**. Un gerente ve lo de sus subordinados; dos personas del mismo rol **no** se ven entre sí **[verificado + misma fuente]**.
3. **Sharing rules**: excepciones automáticas al OWD, por **propietario** (registros de miembros del grupo A → grupo B) o por **criterio** (campo del registro = X → grupo B). Solo abren, nunca cierran **[verificado + https://help.salesforce.com/s/articleView?id=platform.security_about_sharing_rules.htm&language=en_US&type=5]**.
4. **Manual sharing / teams / implicit sharing**: compartir a mano, equipos de cuenta, y el acceso implícito padre↔hijo (ver un contacto da lectura a su cuenta) **[inferido a partir de la documentación de sharing]**.

Por debajo hay **tablas de share** (`AccountShare`, `MyObject__Share`) donde cada fila tiene: id del registro, id del usuario **o grupo**, nivel de acceso, y **row cause** (`Owner`, `Manual`, `Rule`, `ImplicitParent`) **[verificado + https://developer.salesforce.com/docs/platform/salesforce-record-access-under-the-hood/guide/uth-entity.html]**.

El detalle que hace que esto escale: **compartir con un grupo crea UNA fila, no una por miembro**. En tiempo de lectura se cruzan la tabla de shares con las tablas de membresía de grupos **[verificado + https://developer.salesforce.com/blogs/engineering/2013/10/behind-the-scenes-of-record-ownership-in-salesforce]**.

Y el costo que hay que presupuestar: **las shares se recalculan cuando cambia la membresía o el rol**. Con *ownership skew* (un usuario dueño de cientos de miles de registros) ese recálculo es caro; Salesforce lo movió a asíncrono cuando conviene, y advierte que entonces las share rows **no están disponibles de inmediato** **[verificado + https://help.salesforce.com/s/articleView?id=005103630&language=en_US&type=1]**. La guía de Large Data Volumes recomienda que **ningún usuario sea dueño de más de 10.000 registros de un objeto** y que ningún padre tenga más de 10.000 hijos **[verificado + guía LDV de Salesforce, citada en https://www.stevetecharc.com/courses/data-architect/scale-ldv/volume-meets-sharing]**.

Esto es una lección de arquitectura, no un detalle: **el row-level precomputado traslada costo de lectura a escritura, y hay que aceptarlo conscientemente.** Si tu CRM tiene 3 usuarios, es sobre-ingeniería. Si tiene 300 y reglas por territorio, es la única forma de que un listado abra en 200 ms.

#### 3.5.2 Odoo — "compilar el dominio a SQL"

Dos mecanismos separados y con semántica **distinta**, que es donde todos se equivocan:

| | `ir.model.access` (access rights) | `ir.rule` (record rules) |
|---|---|---|
| Nivel | Modelo (tabla) | Registro (fila) |
| Operaciones | `perm_read/write/create/unlink` | Mismos nombres, **pero significan "a qué operación aplica la regla"**, no "concede" |
| Composición | Unión entre grupos | **Global: intersección. De grupo: unión.** Y global ∩ grupo |
| Default | Sin ACL = **sin acceso** | **Default-allow**: si no hay regla, se permite **[verificado + https://www.odoo.com/documentation/17.0/developer/reference/backend/security.html]** |

```xml
<!-- Access right: el grupo "Ventas" puede leer/escribir el modelo -->
<record id="access_sale_lead" model="ir.model.access">
  <field name="model_id" ref="model_crm_lead"/>
  <field name="group_id" ref="group_sales"/>
  <field name="perm_read" eval="True"/>
  <field name="perm_write" eval="True"/>
</record>

<!-- Record rule: solo los leads de mi equipo o míos -->
<record id="rule_lead_team" model="ir.rule">
  <field name="model_id" ref="model_crm_lead"/>
  <field name="domain_force">
    ['|', ('team_id', 'in', user.sale_team_ids.ids), ('user_id', '=', user.id)]
  </field>
</record>
```

**Lo bueno del diseño de Odoo**: `domain_force` es una expresión declarativa que el ORM **inyecta como `WHERE`** en la consulta — no hay post-filtrado, no hay N+1. El dominio puede usar `user`, `company_id`, `company_ids` y `time` **[verificado + misma fuente]**.

**Lo peligroso, documentado por Odoo mismo**: *"Creating multiple global rules is risky as it's possible to create non-overlapping rulesets, which will remove all access"* — dos reglas globales se intersectan, y si no se solapan, nadie ve nada **[verificado + misma fuente]**. Un CRM debe impedir por diseño que un admin configure dos reglas globales sin una vista previa del efecto ("¿cuántos registros quedan visibles para este usuario?").

Nota fina: las reglas pueden forzar *post-filtrado* si el dominio referencia campos no almacenados o `sudo()`; ahí es donde Odoo pierde el rendimiento. Regla de arquitectura: **toda condición de visibilidad debe ser sobre columnas almacenadas y con índice**.

#### 3.5.3 EspoCRM — "niveles discretos, fáciles de explicar"

Cinco niveles por acción: `yes` (todo), `all`, `team`, `own`, `no`, sobre 5 acciones (`create`, `read`, `edit`, `delete`, `stream`) **[verificado + https://docs.espocrm.com/administration/roles-management/]**.

- `own` = es el `assignedUser` (o el creador).
- `team` = el registro tiene alguno de mis equipos en su campo Teams.
- Roles del usuario **y** de sus equipos se fusionan permisivamente.
- `Assignment Permission` controla si puede reasignar a otros (y de paso, si puede escribir en el stream ajeno).
- `User Permission` convierte al usuario en "supervisor" y **puede saltarse el row-level** — el propio doc advierte que se dé solo a privilegiados **[verificado + misma fuente]**.

Ventaja: un admin entiende la matriz en 30 segundos. Desventaja: no hay sharing por criterio ni jerarquía; para eso necesitas equipos y "collaborators" **[verificado + misma fuente]**.

### 3.6 Cómo evaluar row-level sin N+1

El antipatrón, que aparece en casi todo CRM hecho a mano:

```python
# ❌ N+1 Y fuga de datos en export/report
deals = db.query("SELECT * FROM deals WHERE tenant_id=%s", tenant)
visible = [d for d in deals if can_read(user, d)]   # 1 query + N chequeos
```

Consecuencias: 10.000 registros = 10.000 chequeos (o 10.000 queries si `can_read` consulta la DB), y todo camino que no pase por esa lista (export, PDF, búsqueda, webhook) se olvida del filtro.

#### Las cinco estrategias, con su costo

| # | Estrategia | Cómo | Costo lectura | Costo escritura | Cuándo |
|---|---|---|---|---|---|
| A | **Compilar a SQL** (Odoo) | La regla es un predicado que se inyecta en `WHERE` | O(log n) con índice | 0 | Reglas basadas en columnas del propio registro |
| B | **Tabla de shares precomputada** (Salesforce) | Fan-out en escritura a `record_share` | JOIN pequeño | Alto (fan-out + recálculo) | Reglas por grupo/jerarquía con muchas combinaciones |
| C | **Conjunto de visibilidad por principal** | Materializar `visible_record_ids(user)` | Trivial | Caro de mantener | Reporting/BI, no OLTP |
| D | **PostgreSQL RLS** | `CREATE POLICY ... USING (owner_id = current_setting('app.uid')::uuid)` | Plan por rol, no por usuario | 0 | Self-hosted, un rol de DB por "clase" de usuario |
| E | **Predicados de scope** (recomendado) | `owner_id = :me OR team_id = ANY(:my_teams) OR hierarchy_path LIKE :prefix OR id IN (SELECT record_id FROM shares WHERE grantee_id = ANY(:my_principals))` | O(log n) con índices compuestos | 0 (solo shares manuales) | El 95 % de los CRMs |

**El modelo mínimo viable es E + un bolsillo de B**: predicados para owner/equipo/jerarquía (que cubren el 90 % de los casos) y una tabla `record_share` solo para lo que se comparte a mano o por criterio.

```sql
-- Una sola cláusula, reutilizada por TODO camino de lectura
CREATE OR REPLACE FUNCTION deal_visibility(p_principal uuid) RETURNS text
LANGUAGE sql STABLE AS $$
  SELECT format(
    '(d.owner_id = %1$L
      OR d.team_id = ANY (SELECT team_id FROM principal_teams(%1$L))
      OR d.owner_id IN (SELECT user_id FROM subordinates(%1$L))
      OR EXISTS (SELECT 1 FROM record_share s
                  WHERE s.object = ''deal'' AND s.record_id = d.id
                    AND s.grantee_id IN (SELECT principal_id FROM principal_closure(%1$L))))',
    p_principal);
$$;
```

Y el listado queda:

```sql
SELECT d.id, d.name, d.amount
  FROM deals d
 WHERE d.tenant_id = $1
   AND (d.owner_id = $2 OR d.team_id = ANY($3) OR d.owner_id = ANY($4)
        OR EXISTS (SELECT 1 FROM record_share s WHERE ...))
   AND d.stage = $5
 ORDER BY d.updated_at DESC, d.id DESC
 LIMIT 51;
```

**Un `EXPLAIN` con `EXISTS` sobre `record_share(record_id, grantee_id)` con índice compuesto resuelve esto en un solo plan.** No hay N+1 porque no hay N.

#### El cierre transitivo de la jerarquía

Para "mi jerarquía ve lo de abajo", hay tres implementaciones y una sola correcta a escala:

| Técnica | Consulta | Problema |
|---|---|---|
| `WITH RECURSIVE` por request | Recursión en cada listado | Costoso y no indexable |
| Lista de subordinados en la app | `owner_id IN (lista)` | Listas de 5.000 ids en el `WHERE` |
| **Materialized path** (`/1/4/9/`) | `owner_path LIKE '/1/4/%'` | Requiere mantener al mover un nodo (un `UPDATE` de subtree) |
| **Tabla de cierre** (`user_closure(ancestor, descendant)`) | `JOIN user_closure c ON c.descendant = d.owner_id WHERE c.ancestor = :me` | Una fila por par; crece O(n²) en jerarquías planas |

Recomendación: **materialized path con prefijo indexado** (`text_pattern_ops`) para jerarquías de hasta ~50.000 usuarios. Es un índice B-tree que responde `LIKE 'prefijo%'` sin escanear.

#### Los cuatro caminos que hay que cubrir con la MISMA cláusula

```
                   ┌──────────────────────────┐
                   │  visibilidad(principal)  │  ← una sola definición
                   └────────────┬─────────────┘
        ┌───────────┬───────────┼───────────┬────────────┐
        ▼           ▼           ▼           ▼            ▼
     listado     detalle     búsqueda    export/PDF   webhook
      (SQL)      (SQL+404)    (FTS+SQL)    (SQL)       (filtro)
```

El bug clásico: la búsqueda usa un índice de texto y devuelve el `name` de registros que el detalle niega con 404. **La búsqueda debe devolver el mismo conjunto que el listado, no un superconjunto.** Prueba de aceptación: para cada endpoint, un test que compare `ids(search(q)) ⊆ ids(list())`.

#### Cómo se prueba (y esto es lo que separa un CRM de una demo)

1. **Matriz de permisos**: N usuarios × M objetos × 5 acciones, como test parametrizado. Cada celda con resultado esperado.
2. **Test de conteo de queries**: el listado de 100 registros no puede ejecutar más de K queries (K constante, no proporcional a N). Si sube, alguien metió un `can_read` en un loop.
3. **Test de coherencia entre caminos**: mismo registro, mismos permisos, mismo resultado en list/detail/search/export.
4. **Test de "usuario sin rol"**: debe ver cero, no todo.
5. **Test de campo restringido en `order_by`/`filter`**: debe dar `400`, no filtrar en silencio.

### 3.7 Impersonación y delegación

Cuando soporte "entra como" el cliente, hay que registrar **dos actores**: `actor_id` (quien ejecuta) y `on_behalf_of` (el usuario efectivo). Sin eso, la auditoría es inútil justo en el caso en que más se necesita. Y la impersonación debe ser de solo lectura por defecto, con expiración y banner visible.

---

## 4. Auditoría

### 4.1 Tres auditorías distintas que se confunden en una

| Tipo | Pregunta que responde | Volumen | Retención típica |
|---|---|---|---|
| **Negocio / field history** | ¿quién cambió el monto de este deal y de cuánto a cuánto? | Alto (proporcional a ediciones) | 18–24 meses caliente, archivo después |
| **Seguridad / acceso** | ¿quién inició sesión, quién cambió permisos, quién exportó? | Medio | 6–12 meses mínimo legal |
| **Operativa / API** | ¿qué request falló, con qué `request_id`? | Muy alto | 7–30 días |

Mézclarlas en una tabla es el error que revienta el disco: la operativa (que es la más voluminosa y la menos valiosa) desplaza a la de negocio (que es la que un auditor pide).

### 4.2 Qué registrar (y qué no)

**Sí:**
- `occurred_at`, `tenant_id`, `actor_id`, `actor_type`, `on_behalf_of`, `ip`, `user_agent`, `request_id`, `source` (`ui|api|automation|import|webhook`).
- `object`, `record_id`, `action` (`create|update|delete|view|export|share|login`).
- `changes`: solo los campos que cambiaron, con `old`/`new` **y solo para campos marcados como auditables**.
- `automation_run_id` cuando el cambio lo hizo una regla (si no, nadie puede explicar por qué cambió un campo).

**No:**
- El payload completo del request (contiene PII y multiplica el volumen por 20).
- Cada `GET`/listado (a menos que sea `export` o acceso a campo sensible).
- Valores de campos cifrados o de tipo "nota larga" (Salesforce documenta que en textos >255 caracteres registra *que hubo cambio* pero no los valores **[verificado + https://help.salesforce.com/s/articleView?id=xcloud.tracking_field_history.htm&language=en_US]** — es una decisión de costo, no de capacidad).

### 4.3 Cuánto pesa, con números

Supuesto: 100 usuarios, 50 ediciones/día/usuario, 3 campos por edición.

```
100 × 50 × 3        = 15.000 filas/día
15.000 × 365        = 5.475.000 filas/año
~250 B/fila + índices (~1,5×)  ≈ 3,4 GB/año   ← solo field history
```

Con un `payload JSONB` completo por cambio, la misma cifra se va a **20–40 GB/año**. En un servidor de 4 GB de RAM con 80 GB de disco, eso es la diferencia entre "tengo CRM" y "tengo un disco lleno en marzo".

Referencias verificadas de cómo lo acotan los grandes:

| Producto | Límite / retención |
|---|---|
| Salesforce Field History Tracking | Hasta **20 campos por objeto**, retención 18–24 meses **[verificado + https://developer.salesforce.com/docs/atlas.en-us.field_history_retention.meta/field_history_retention/field_audit_trail.htm]** |
| Salesforce Field Audit Trail | Hasta **200 campos**, archivado a `FieldHistoryArchive` (big object), retención indefinida, **solo API**, y **no cuenta contra el storage** **[verificado + misma fuente]** |
| Salesforce Event Monitoring | 30 días por defecto, extensible a 365 con Shield/add-on; **los logs no consumen storage del org** **[verificado + https://help.salesforce.com/s/articleView?id=000396295&language=en_US&type=1]** |
| Salesforce Setup Audit Trail | **180 días, no extensible** — hay que exportarlo **[verificado + https://help.salesforce.com/s/articleView?id=xcloud.shr_set_up_audit_trail.htm&language=en_US&type=5]** |

La lección de diseño: **los campos auditables son una decisión de producto con un tope explícito**, y el archivo va a un storage distinto (big object / S3 / Parquet) que no compite con el disco transaccional.

### 4.4 Diseño que aguanta

```sql
-- Append-only, particionada por mes, sin FKs, sin UPDATE, sin DELETE
CREATE TABLE audit_event (
  id           bigserial,
  occurred_at  timestamptz NOT NULL DEFAULT now(),
  tenant_id    uuid NOT NULL,
  actor_id     uuid,
  actor_type   text NOT NULL,
  on_behalf_of uuid,
  source       text NOT NULL,
  request_id   text,
  object       text,
  record_id    uuid,
  action       text NOT NULL,
  changes      jsonb,          -- {"amount": {"old": 1000, "new": 1200}}
  context      jsonb           -- ip, ua, automation_run_id
) PARTITION BY RANGE (occurred_at);

CREATE INDEX ON audit_event USING BRIN (occurred_at);          -- barato, ideal para rangos
CREATE INDEX ON audit_event (tenant_id, record_id, occurred_at DESC); -- "historial de este deal"
CREATE INDEX ON audit_event (tenant_id, actor_id, occurred_at DESC);  -- "qué hizo este usuario"
```

Cinco decisiones que marcan la diferencia:

1. **Partición mensual** + `DETACH`/`DROP` de la partición vieja = retención sin `DELETE` masivo (que genera bloat y rompe el autovacuum).
2. **`BRIN` en `occurred_at`** en vez de B-tree: en datos append-only, `BRIN` ocupa ~1 % del tamaño y basta para rangos.
3. **Sin foreign keys** hacia `users`/`deals`: si borras un usuario, la auditoría debe sobrevivir. Guardar `actor_id` como valor, no como referencia.
4. **Escritura fuera de la transacción caliente** vía outbox: el `INSERT` de auditoría no debe añadir latencia al `UPDATE` del negocio, y si falla la auditoría no debe revertir la venta (o sí, según el requisito legal — decidirlo explícitamente; para field history regulado, **sí** debe ser transaccional).
5. **Compresión**: `ALTER TABLE ... SET (toast_compression = lz4)` y archivado mensual a Parquet/S3 con `COPY`. 10:1 típico en JSON repetitivo.

**Cómo se consulta** (si no se puede consultar, no es auditoría, es basura):

| Vista | Query | Índice que la sirve |
|---|---|---|
| Historial de un registro | `WHERE tenant_id=? AND record_id=? ORDER BY occurred_at DESC` | `(tenant_id, record_id, occurred_at)` |
| Actividad de un usuario | `WHERE actor_id=? AND occurred_at > ?` | `(tenant_id, actor_id, occurred_at)` |
| Cambios de un campo | `WHERE changes ? 'amount' AND occurred_at BETWEEN ?` | GIN sobre `changes` (o tabla `audit_field_change` si el volumen lo pide) |
| Export completo de un periodo | `WHERE occurred_at BETWEEN ?` | BRIN |

**Retención por capas, explícita:**

```
  0–90 días    → PostgreSQL (consulta interactiva desde la UI)
 90 d–2 años   → PostgreSQL comprimido / solo si hay requisito
 2 años+       → almacenamiento frío (S3/Parquet), consulta vía job, no vía UI
```

Y una prueba que casi nadie hace: **restaurar un backup y verificar que la auditoría del periodo está completa**. Un backup sin restore probado no es un backup.

---

## 5. Automatización sin código

### 5.1 La anatomía común

Todo motor no-code de CRM tiene la misma forma:

```
  EVENTO                CONDICIÓN                 ACCIÓN
  ─────                 ─────────                 ──────
  record.created   →   ¿state == 'won'?     →   update / email / webhook /
  record.updated        ¿campo X cambió?         assign / create task /
  record.deleted        ¿usuario en grupo?       start sub-workflow
  time.reached          ¿monto > 10k?            (con delay opcional)
  manual / scheduled
```

La diferencia entre productos está en **cuándo** se evalúa (antes o después del commit), **cómo** se evita el bucle, y **qué límites** comparte con el resto del sistema.

### 5.2 Salesforce Flow — el más restrictivo y el mejor documentado

| Aspecto | before-save ("Fast Field Updates") | after-save |
|---|---|---|
| Corre | Antes del commit, en el mismo save | Después del commit |
| Puede | Cambiar campos del **mismo** registro | Crear/actualizar otros, emails, acciones, subflows |
| Costo | 0 DML extra | Un DML extra (y re-ejecuta la save procedure) |
| Riesgo | Ninguno de recursión | **Recursión** si actualiza el registro que lo disparó |

Recomendación oficial y verificable: **usar before-save para cambios en el mismo registro**, porque evita un segundo DML y un ciclo de guardado completo **[verificado + https://architect.salesforce.com/docs/architect/decision-guides/guide/record-triggered]**.

Control de recursión — tres mecanismos, en orden de preferencia:

1. **Entry criteria de transición**: "solo cuando el registro se actualiza para cumplir las condiciones" (`$Record.Amount != $Record__Prior.Amount`), en vez de "cada vez que se actualiza y cumple" **[verificado + misma fuente]**.
2. **Guard por campo cambiado** (`ISCHANGED`), que además hace la regla idempotente.
3. **Antes que nada, no hacer el segundo save**: mover la lógica a before-save.

Dato clave que sorprende a todos: un after-save flow que actualiza el mismo registro **no re-dispara flows** (solo triggers Apex), pero **sí re-ejecuta la save procedure** y puede encadenar recursión entre objetos (Opp→Project→Account→Opp) **[verificado + https://salesforce.stackexchange.com/questions/387588 vía discusión de Trailblazer Community]**.

Y el límite que obliga a diseñar bien: los flows **comparten los governor limits** de la transacción — 100 SOQL y 150 DML en síncrono **[verificado + https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_apexgov.htm]**. Un `Get Records` dentro de un loop es el mismo error que un SOQL en un `for` de Apex.

### 5.3 Odoo — potente y peligroso

`base_automation` ofrece: `on_create`, `on_write`, `on_create_or_write`, `on_unlink`, `on_change`, `based_on_timed_condition` **[verificado + https://github.com/odoo/odoo/blob/f399f99d/addons/test_base_automation/tests/test_flow.py]**.

La pieza elegante es el par de dominios:

- `filter_pre_domain`: evalúa el estado **antes** del write (valores viejos).
- `filter_domain`: evalúa el estado **después**.
- Juntos detectan **transiciones** ("no estaba 'returned' y ahora sí") sin necesidad de un flag **[verificado + misma fuente / doc de automatizaciones]**.

Odoo **no bloquea la recursión**: el propio test suite incluye `test_021_recursive` con un ciclo `draft→pending→open→done→draft` de cuatro automatizaciones, y `test_020_recursive` verifica que una regla se ejecuta recursivamente por un cambio secundario **[verificado + misma fuente]**. Es decir: **el motor es potente porque no te protege**. En self-hosted eso significa que un admin puede colgar el servidor con dos reglas y nadie sabrá cuál.

### 5.4 EspoCRM — el que sí bloquea por defecto

Triggers: `after record created`, `after record updated`, `after record created or updated`, `manual`, `scheduled` (cron + List Report), `sequential`, `signal` **[verificado + https://docs.espocrm.com/administration/workflows/]**.

Dos decisiones de diseño notables:

1. **Safeguard anti-bucle incorporado**: "A built-in safeguard prevents looping in basic cases" — y en la práctica, **un registro creado por un workflow no dispara los triggers `after record created` de otros workflows**; hay que usar explícitamente `sequential` + `Trigger another Workflow` **[verificado + misma fuente y https://forum.espocrm.com/forum/general/113197]**.
2. **Reconoce su límite**: "It's not recommended to rely on an Update Target Record action triggering another after-update Workflow. In some cases it won't work." **[verificado + misma fuente]**. Es honestidad de diseño: el encadenamiento implícito es frágil, el explícito es verificable.

### 5.5 Cómo se diseña el motor (lo que haría distinto)

**Reglas como datos, no como código desplegado.** Una tabla `automation_rule` con `trigger`, `conditions` (JSON), `actions` (JSON), `order`, `active`, `stop_on_match`, `max_depth`.

```python
def run_automations(ctx: ChangeContext, depth: int = 0) -> None:
    if depth > MAX_DEPTH:                      # 1. tope duro
        log.warn("automation_depth_exceeded", ctx)
        emit_metric("automation.depth_exceeded")
        return

    rules = load_rules(ctx.object, ctx.trigger)  # ordenadas por `order`
    fired = []
    for rule in rules:
        if rule.id in ctx.visited:             # 2. no re-ejecutar la misma regla
            continue
        if not matches(rule, ctx.before, ctx.after):
            continue
        ctx.visited.add(rule.id)
        # 3. transición, no estado: evita re-disparo por ediciones irrelevantes
        if rule.transition_only and not changed(rule.watch_fields, ctx.before, ctx.after):
            continue

        result = execute(rule.actions, ctx)    # 4. idempotente por (rule, record, version)
        fired.append(rule.id)

        if result.changed_records:
            for sub in result.changed_records:
                run_automations(sub, depth + 1)  # 5. recursión ACOTADA y visible
        if rule.stop_on_match:
            break

    ctx.audit(fired)                            # 6. qué reglas corrieron y qué cambiaron
```

Seis guardas, y ninguna es opcional:

| Guarda | Qué evita |
|---|---|
| **Profundidad máxima** (10) | Bucle infinito que quema CPU |
| **Set de reglas visitadas por transacción** | La misma regla disparándose en cascada |
| **`transition_only` + `watch_fields`** | Re-ejecución por ediciones que no importan |
| **Idempotency key `(rule_id, record_id, version)`** | Duplicados por reintento |
| **Un solo "dueño" por campo** (validación al guardar la regla) | Dos reglas escribiendo `stage` y peleándose |
| **`stop_on_match` + orden explícito** | "No sé por qué ganó la otra regla" |

**El conflicto de escritura es el bug silencioso más caro.** Si la regla A pone `stage='won'` y la regla B pone `stage='negotiation'` cuando `amount>0`, el registro oscila en cada edición y nadie lo nota hasta que un reporte da cifras imposibles. Solución: al guardar una regla, calcular qué campos escribe y **rechazar si otro workflow activo ya es dueño de ese campo** (o exigir un override explícito y auditado).

### 5.6 Asignación automática

Cuatro estrategias, en orden de complejidad:

| Estrategia | Cómo | Cuidado |
|---|---|---|
| Round-robin | Cursor por equipo | Persistir el cursor; si no, todos al primero |
| Por carga | `COUNT(open_deals) GROUP BY owner` | Requiere índice en `(owner_id, is_open)` |
| Por territorio/región | Match de campos del registro | Reglas solapadas: definir precedencia |
| Por reglas tipo "lead scoring" | Puntaje + umbrales | El puntaje debe ser reproducible y auditable |

Regla: **la asignación es una decisión de negocio y se registra como evento** (`assigned_from`, `assigned_to`, `rule_id`, `reason`). Sin eso, "¿por qué este lead es de Juan?" no tiene respuesta.

### 5.7 Escalamiento por tiempo (y por qué el cron que escanea es un error)

```
❌  cada minuto:  SELECT * FROM deals WHERE stage='new' AND created_at < now()-'2 days'
                → escanea toda la tabla, cada minuto, para siempre
✅  al crear/editar:  next_action_at = business_add(created_at, '2 days', calendar)
    cada minuto:      SELECT ... WHERE next_action_at <= now() AND state='pending'
                      ORDER BY next_action_at LIMIT 500   -- índice parcial
```

Detalles que definen si funciona:

- **Índice parcial**: `CREATE INDEX ON deals (next_action_at) WHERE state = 'pending'`. Solo indexa lo pendiente, no el histórico.
- **Horario hábil y feriados**: "2 días hábiles" no es `+2 days`. Calendario por tenant, por equipo, con zona horaria.
- **Reprogramación al cambiar la condición**: si el deal pasa a `won`, hay que **cancelar** el `next_action_at` pendiente, no dejarlo dispararse.
- **Idempotencia**: el job de escalamiento debe poder correr dos veces sin mandar dos correos (`escalation_fired_at` no nulo → skip).

---

## 6. Colas y trabajos

### 6.1 La línea divisoria

**Síncrono** = el usuario debe ver el resultado para continuar, y tarda < ~300 ms de presupuesto.
**Asíncrono** = tiene efectos externos, es lento, o puede fallar y reintentarse.

| Operación | ¿Async? | Por qué |
|---|---|---|
| Crear/editar un registro y verlo guardado | **No** | El commit es el contrato con el usuario |
| Cambiar etapa de una oportunidad | **No** | Es el corazón del negocio; si se "encola", la venta se pierde |
| Calcular totales/impuestos mostrados | **No** | Debe ser determinista y visible |
| Validar permisos | **No** | Nunca se difiere una decisión de seguridad |
| Enviar correo / notificación | **Sí** | Lento, externo, reintentable |
| Webhook saliente | **Sí** | El receptor puede estar caído |
| Importación CSV | **Sí** | Minutos, con progreso |
| Exportación / reporte grande | **Sí** | Bloquea la DB |
| Recalcular campos derivados masivos | **Sí** | No debe bloquear la edición |
| Indexar para búsqueda | **Sí** | Consistencia eventual aceptable |
| Ejecutar automatizaciones de un cambio | **Depende**: síncrono si el usuario debe ver el efecto | Si el efecto es "el stage cambió a won", el usuario debe verlo |
| Enviar el cobro a la pasarela | **Sí, pero con idempotency key** | Y la **decisión** de cobrar es síncrona; el `POST` al proveedor puede reintentarse |

**La venta no se encola.** Lo que se encola es el *efecto secundario*: el correo de confirmación, la sincronización contable, el webhook. Si encolas la transacción, un worker caído significa ventas perdidas sin que nadie lo sepa.

### 6.2 Semántica: at-least-once + idempotencia = efecto exactly-once

No existe entrega exactly-once. Existe **at-least-once en el transporte + idempotencia en el consumidor**. Temporal lo formula así: las activities son at-least-once por defecto (retry policy), y la idempotencia del código da el efecto único **[verificado + https://temporal.io/blog/idempotency-and-durable-execution]**.

Dos conceptos que se confunden y hay que separar:

- **Activity ID / idempotency key**: evita *enviar* el trabajo dos veces.
- **Código idempotente**: evita que *un reintento* duplique el efecto.
Temporal dice explícitamente que "you need both" **[verificado + https://docs.temporal.io/standalone-activity]**.

### 6.3 Outbox: el patrón que evita el evento fantasma

El problema: `COMMIT` en la DB y `PUBLISH` a la cola no son atómicos. Si publicas antes del commit y el commit falla, el consumidor ve un evento de algo que no existe. Si publicas después y el proceso muere, el evento nunca sale.

```
   ┌─────────────── transacción ───────────────┐
   │ INSERT INTO deals (...)                   │
   │ INSERT INTO outbox (topic, payload, ...)  │  ← mismo COMMIT
   └───────────────────┬───────────────────────┘
                       ▼
             ┌───────────────────┐
             │ relay worker      │  SELECT ... FOR UPDATE SKIP LOCKED
             │ (cada 1s)         │  → publica / envía
             └─────────┬─────────┘
                       ▼
             marca published_at, o reintenta
```

Es el *transactional outbox pattern*: la tabla outbox en la misma transacción, un relay que publica y marca **[verificado + https://microservices.io/patterns/data/transactional-outbox.html]**.

En un servidor modesto, **la tabla outbox es la cola**. No hace falta Redis ni RabbitMQ.

### 6.4 Reintentos y dead letter

Clasificar antes de reintentar:

| Clase | Ejemplo | Acción |
|---|---|---|
| **Permanente** | `400` de validación, payload inválido, destinatario inexistente | **No reintentar**. DLQ directa + alerta |
| **Transitorio** | timeout, `502`, conexión rechazada | Backoff exponencial con jitter, hasta N intentos (8–12) |
| **Rate limit** | `429` con `Retry-After` | Respetar el header; backoff lineal con tope (60 s) |
| **Desconocido** | excepción no clasificada | Tratar como transitorio; el costo de perder un evento es mayor que el de un intento extra |

```
intento 1: +30 s
intento 2: +2 min
intento 3: +10 min   (×2 con jitter ±20 %)
intento 4: +1 h
intento 5: +6 h
intento 6: +24 h
→ DLQ con payload, último error, stack, y botón "replay"
```

**DLQ sin herramienta de replay es solo un basurero.** Debe incluir: payload íntegro, número de intentos, última traza, y un comando `replay --id` que reencola con el mismo idempotency key.

### 6.5 Worker en PostgreSQL (sin infraestructura extra)

```sql
-- Job queue mínima, robusta, sin servicios adicionales
CREATE TABLE job (
  id            bigserial PRIMARY KEY,
  queue         text NOT NULL,              -- email | webhook | import | automation
  payload       jsonb NOT NULL,
  run_at        timestamptz NOT NULL DEFAULT now(),
  attempts      int NOT NULL DEFAULT 0,
  max_attempts  int NOT NULL DEFAULT 10,
  locked_by     text,
  locked_until  timestamptz,
  last_error    text,
  idempotency_key text UNIQUE,
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON job (queue, run_at) WHERE locked_until IS NULL OR locked_until < now();
```

```python
def worker_loop(worker_id):
    while True:
        with db.transaction():
            job = db.one("""
                SELECT * FROM job
                 WHERE queue = ANY(%s)
                   AND run_at <= now()
                   AND (locked_until IS NULL OR locked_until < now())
                 ORDER BY run_at
                 FOR UPDATE SKIP LOCKED
                 LIMIT 1
            """, (WORKER_QUEUES,))
            if not job:
                sleep(1); continue
            db.execute("UPDATE job SET locked_until = now() + interval '5 min', locked_by=%s "
                       "WHERE id=%s", (worker_id, job.id))
        try:
            handle(job)                      # idempotente
            db.execute("DELETE FROM job WHERE id=%s", job.id)
        except PermanentError as e:
            dead_letter(job, e);             # sin reintento
        except Exception as e:
            schedule_retry(job, e)           # backoff + jitter, o DLQ si attempts >= max
```

`FOR UPDATE SKIP LOCKED` es la pieza que permite N workers sin coordinación y sin bloqueos entre ellos. Funciona con decenas de miles de jobs/día en un Postgres modesto.

**Colas separadas por tipo** es obligatorio: si la cola de webhooks (que puede tener 5.000 pendientes porque el cliente está caído) comparte worker con la de correos, los correos se retrasan horas. Al menos: `email`, `webhook`, `import`, `automation`, `maintenance`.

### 6.6 Temporal y n8n: para qué sí y para qué no

| Herramienta | Fortaleza | No usar para |
|---|---|---|
| **Temporal** | Workflows duraderos, deterministas, con historial y replay; activities con retry/heartbeat; `Activity ID` como dedupe **[verificado + https://docs.temporal.io/standalone-activity]** | CRUD simple; cuesta RAM y operación (server + DB + workers) |
| **n8n** | Integraciones rápidas, cientos de conectores, bueno para prototipos y automatizaciones internas | El camino crítico de una venta: sin garantías transaccionales ni determinismo frente a ediciones del workflow |
| **Cron + tabla** | Lo más simple que funciona | Nada que requiera sub-minuto o fan-out masivo |

Regla para un servidor de 2–4 GB: **Temporal no entra** (su stack son varios cientos de MB). El outbox + `SKIP LOCKED` cubre el 95 % de los casos. n8n entra solo como servicio externo/opcional para integraciones de cliente, nunca dentro de la ruta de escritura del CRM.

### 6.7 Importaciones

Las importaciones son el trabajo async que más rompe:

- **Chunking** de 500–1.000 filas por job, con checkpoint (`last_processed_row`).
- **Validación en dos fases**: dry-run (reporta errores sin escribir) y apply.
- **Idempotencia por fila**: `(import_id, row_number)` único; re-ejecutar no duplica.
- **Modo upsert con clave de negocio declarada** (email, external_id), no con `id`.
- **Reporte de errores descargable** con número de fila y motivo.
- **Cancelación cooperativa**: el worker consulta `cancelled_at` cada chunk.

---

## 7. Webhooks salientes

### 7.1 El contrato

```json
POST /webhooks/crm  HTTP/1.1
Content-Type: application/json
X-Event-Id: evt_01J8Z...
X-Event-Type: deal.stage_changed
X-Signature: t=1763300000,v1=5f3a...c9
X-Delivery-Attempt: 1
```
```json
{
  "id": "evt_01J8Z...",
  "type": "deal.stage_changed",
  "occurred_at": "2026-11-16T10:00:00Z",
  "api_version": "2026-11-01",
  "tenant_id": "t_123",
  "data": {
    "object": "deal",
    "id": "d_456",
    "version": 42,
    "changed": ["stage"],
    "before": {"stage": "negotiation"},
    "after":  {"stage": "won"}
  }
}
```

Decisiones: **snapshot mínimo + `changed`** en vez del registro completo (minimiza PII y tamaño), **`version` monotónica por registro** para que el consumidor pueda descartar eventos viejos, y **`api_version`** para poder evolucionar el payload.

### 7.2 Firma: cómo se hace bien

Stripe es la referencia: header `Stripe-Signature` con `t=<timestamp>` y `v1=<hmac>`, HMAC-SHA256 sobre `"{timestamp}.{cuerpo_crudo}"`, secreto `whsec_...` **[verificado + https://docs.stripe.com/webhooks/signature]**.

Cuatro reglas que se violan constantemente:

1. **Firmar el cuerpo crudo**, no el JSON re-serializado. Cualquier framework que parsee y vuelva a serializar cambia espacios y orden de claves y rompe la verificación — Stripe lo documenta como la causa #1 de fallos **[verificado + misma fuente]**.
2. **Incluir el timestamp en lo firmado** para evitar replay; rechazar si `|now - t| > 300 s`.
3. **Comparación en tiempo constante** (`hmac.compare_digest`).
4. **Rotación con doble secreto**: aceptar `v1` de secreto nuevo y `v1` de secreto viejo durante la ventana de rotación.

```python
def verify(raw_body: bytes, header: str, secrets: list[bytes], tolerance=300) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    t = int(parts["t"])
    if abs(time.time() - t) > tolerance:
        return False
    signed = f"{t}.".encode() + raw_body
    return any(hmac.compare_digest(hmac.new(s, signed, hashlib.sha256).hexdigest(), parts["v1"])
               for s in secrets)
```

**Nota de seguridad verificada**: Stripe advierte que las entradas `secret`, `destination` y `payload` del header son **solo para depuración y no están firmadas**; un atacante puede generarlas para una request falsa. **Solo una firma `v1` válida autentica** **[verificado + https://docs.stripe.com/webhooks/signature]**.

### 7.3 Reintentos, orden y duplicados

| Propiedad | Comportamiento de referencia (Stripe) |
|---|---|
| Ventana de reintento | Hasta **3 días** con backoff exponencial en live; 3 intentos en sandbox **[verificado + https://docs.stripe.com/webhooks]** |
| Timeout | No publicado; un timeout cuenta como fallo |
| Éxito | Solo `2xx`. **Los `3xx` cuentan como fallo** |
| Orden | **No garantizado** — "Stripe doesn't guarantee the delivery of events in the order that they're generated" |
| Duplicados | Posibles; deduplicar por `event.id` |
| Replay manual | Dashboard hasta 15 días, CLI hasta 30 días |
| Retención de eventos | 30 días consultables por API |
| Endpoints por cuenta | 16 |

**Diseño del orden.** Como el orden no se garantiza, hay dos opciones:

1. **No depender del orden** (recomendado): cada evento lleva `version` del recurso; el consumidor ignora `version <= last_seen`. Y si necesita el estado actual, hace `GET` del recurso. Stripe recomienda exactamente esto: "trata el webhook como una notificación de que algo cambió, y reconcilia con el estado actual" **[verificado + https://docs.stripe.com/webhooks]**.
2. **Ordenar por recurso** (más caro): una cola por `record_id` con un solo worker, o un `sequence` por stream. Solo si el negocio lo exige (p. ej. movimientos de saldo).

**Deduplicación en el receptor** (pseudocódigo):

```python
@app.post("/webhooks/crm")
def receive(request):
    raw = request.body                       # crudo, sin parsear
    if not verify(raw, request.headers["X-Signature"], SECRETS):
        return Response(status=401)
    evt = json.loads(raw)

    # INSERT ... ON CONFLICT DO NOTHING sobre event_id
    if not store_event_once(evt["id"], evt):
        return Response(status=200)          # ya procesado → 200 para cortar reintentos

    enqueue(evt)                             # trabajo real fuera del request
    return Response(status=200)              # responder rápido, siempre
```

Responder `200` antes de procesar es lo que evita que un pico de trabajo se convierta en una tormenta de reintentos.

### 7.4 Operación de webhooks

- **Estado por endpoint**: `enabled`, `disabled_after_failures`, `last_success_at`, `consecutive_failures`.
- **Deshabilitar tras N fallos consecutivos** (p. ej. 100) y avisar por correo. Stripe hace algo equivalente: si el destino está deshabilitado al momento del reintento, no reintenta **[verificado + misma fuente]**.
- **Visibilidad para el cliente**: una pantalla con los últimos 30 días de entregas, código HTTP, tiempo de respuesta, y botón "reenviar". Sin esto, cada incidente es un ticket.
- **Minimización**: no enviar campos que el scope del endpoint no cubre; no enviar PII innecesaria; el payload es una salida más y **respeta los mismos permisos** que la API.
- **SSRF**: el destino lo elige el cliente. Bloquear rangos privados/link-local, resolver DNS y validar la IP final, no seguir redirects.
- **Aislamiento**: si el cliente está caído, no debe consumir los workers de correo (§6.5).

---

## 8. Operación

### 8.1 Migraciones en caliente con clientes activos

La regla: **nunca un `ALTER TABLE` que bloquee escrituras sobre una tabla grande, y nunca un deploy que requiera downtime.**

Patrón expand/contract en cuatro fases:

```
Fase 1  EXPAND    agregar columna NULLable / tabla nueva / índice CONCURRENTLY
                  (deploy N)     la app escribe en viejo Y nuevo (dual write)
Fase 2  BACKFILL  job por lotes de 1.000, con pausa, idempotente
Fase 3  SWITCH    leer de lo nuevo (feature flag), verificar, y dejar de escribir lo viejo
Fase 4  CONTRACT  DROP de lo viejo cuando ya no hay tráfico (deploy N+2)
```

Recetas concretas para PostgreSQL:

| Cambio | Cómo | Nunca |
|---|---|---|
| Agregar columna | `ADD COLUMN ... NULL` (metadata-only desde PG 11) | `ADD COLUMN ... NOT NULL DEFAULT` en tabla grande (antes de PG 11) |
| Poner `NOT NULL` | 1) `CHECK (col IS NOT NULL) NOT VALID` 2) `VALIDATE CONSTRAINT` 3) `SET NOT NULL` | `SET NOT NULL` directo (toma `ACCESS EXCLUSIVE` y escanea) |
| Índice | `CREATE INDEX CONCURRENTLY` (fuera de transacción) | `CREATE INDEX` en tabla caliente |
| Renombrar columna | Crear nueva + backfill + switch; o vista con el nombre viejo | `ALTER TABLE RENAME` con código viejo desplegado |
| Cambiar tipo | Columna nueva + backfill + switch | `ALTER TYPE` con rewrite |
| Borrar columna | Fase 4, tras confirmar que ningún código la lee | Borrar en el mismo deploy del switch |

**Ventana de compatibilidad N-1**: el servidor debe soportar el esquema viejo y el nuevo durante al menos un deploy. Si no, un rollback de la app con el esquema nuevo produce errores en cascada.

**Backfill**: siempre por lotes con `WHERE id > :last ORDER BY id LIMIT 1000`, idempotente, con métrica de progreso y posibilidad de pausar. Un `UPDATE` de 5 millones de filas en una sola transacción genera un WAL enorme, bloquea y puede tumbar el servidor modesto.

### 8.2 Versionado de API

| Esquema | Ejemplo | Pro | Contra |
|---|---|---|---|
| **Path** | `/v1/deals` | Simple, cacheable, visible en logs | Versiones gruesas |
| **Fecha** | Stripe `Stripe-Version: 2024-06-20`; HubSpot `/crm/objects/2026-09/...` | Cambios incrementales, adopción gradual | Muchas versiones vivas a la vez |
| **Media type** | `Accept: application/vnd.crm.v2+json` | Ortodoxo | Nadie lo usa en la práctica |

Recomendación para un CRM nuevo: **path `/v1` + `api_version` por fecha dentro del payload de webhooks**, con política de soporte de **12–24 meses** y telemetría obligatoria: qué versión usa cada token, cuántas requests, qué endpoints. Sin telemetría, no puedes retirar nada.

Cabeceras de deprecación: `Deprecation: true`, `Sunset: Wed, 01 Jul 2027 00:00:00 GMT`, `Link: <docs>; rel="deprecation"`. Y un correo al contacto técnico del token, no al admin comercial.

### 8.3 Rate limits: números reales y diseño

| Producto | Límite | Detalle |
|---|---|---|
| **HubSpot** (privado) | 100 req/10 s (Free/Starter), 190 (Pro/Ent) **por app** | Diario **compartido por cuenta**: 250k / 625k / 1M **[verificado + https://developers.hubspot.com/docs/developer-tooling/platform/usage-guidelines]** |
| **HubSpot** (OAuth marketplace) | 110 req/10 s por cuenta instalada | El add-on de límites **no** lo aumenta **[verificado + misma fuente]** |
| **HubSpot** Search | **5 req/s** por cuenta, **10.000 resultados** por query | Headers de rate limit **ausentes** en search **[verificado + https://developers.hubspot.com/docs/api-reference/latest/crm/search-the-crm]** |
| **Pipedrive** | Presupuesto diario de **tokens**: `30.000 × multiplicador × seats` | Lite ×1, Growth ×2, Premium ×5, Ultimate ×7. Burst 20–120 req/2 s según plan y tipo de token **[verificado + https://pipedrive.readme.io/docs/core-api-concepts-rate-limiting]** |
| **Pipedrive** (costo) | GET single 2 tokens, GET lista 20, UPDATE 10, DELETE 6, **search 40** | El costo depende de la operación, no solo del conteo **[verificado + misma fuente]** |
| **Salesforce** | `100.000 + (licencias × llamadas por tipo)` por 24 h rodantes | Enterprise: 1.000 por licencia Salesforce; Unlimited: 5.000. Developer: 15.000 **[verificado + https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/limits.htm]** |
| **Salesforce** (por transacción) | 100 SOQL / 150 DML / 50.000 filas / 100 callouts / 120 s de callouts | Síncrono **[verificado + cheatsheet de governor limits]** |

Diseño que se recomienda copiar:

1. **Límite por token Y por cuenta.** El de cuenta evita que una integración consuma el presupuesto de todos.
2. **Costo por endpoint**, no solo conteo (Pipedrive). Un `search` que escanea 10.000 filas no puede costar lo mismo que un `GET` por id.
3. **Headers siempre**: `X-RateLimit-Limit`, `-Remaining`, `-Reset`, y `Retry-After` en el `429`. HubSpot **no** los manda en search — es una decisión que hay que evitar.
4. **`429` con cuerpo explicable**: `policyName` (`TEN_SECONDLY_ROLLING` vs `DAILY`) para que el cliente sepa si esperar 10 segundos o hasta mañana **[verificado + misma fuente de HubSpot]**.
5. **Presupuesto por integración** en la UI: "tu app usó 78 % del presupuesto diario".

### 8.4 Observabilidad

**Logs** estructurados en JSON, con: `ts`, `level`, `request_id`, `tenant_id`, `actor_id`, `route`, `status`, `duration_ms`, `db_queries`, `cache_hit`. **Nunca** el cuerpo completo del request (PII) ni tokens.

**Trazas** (OpenTelemetry) con un span por capa:

```
POST /v1/deals                       [========================] 182 ms
 ├─ auth.authenticate                [==]  4 ms
 ├─ acl.resolve (cached)             [=]   1 ms
 ├─ db.select deals                  [=============] 61 ms   (9 rows, 1 query)
 ├─ automation.run (3 rules)         [========] 38 ms
 │   ├─ rule.assign_owner            [==]  9 ms
 │   └─ rule.notify_manager          [===] 14 ms   (encolado, no enviado)
 ├─ outbox.insert                    [=]   2 ms
 └─ serialize                        [==]  6 ms
```

El span de `db.select` con **conteo de queries** es el detector de N+1 en producción: si `db_queries` crece con el tamaño de la página, hay un loop.

**Métricas** (las cuatro familias):

| Familia | Ejemplos | Alerta |
|---|---|---|
| RED (requests) | `http_requests_total{route,status}`, `http_duration_seconds` (p95/p99) | p99 > 1 s, 5xx > 1 % |
| DB | conexiones activas, queries lentas (>200 ms), bloat, locks | connections > 80 % del pool |
| Colas | `job_pending{queue}`, `job_oldest_age_seconds`, `job_failed_total` | `oldest_age` > 5 min |
| Negocio | deals creados/min, webhooks fallidos/consecutivos, reglas disparadas | caída abrupta a 0 = algo se rompió |

**Cardinalidad**: `user_id` **no** va en labels de métricas (explota la serie temporal). Va en logs y trazas. Labels: `route` (normalizada, sin ids), `status_class`, `queue`.

### 8.5 Respaldos y restauración

| Nivel | Qué | Frecuencia | RPO | RTO |
|---|---|---|---|---|
| PITR | WAL archivado | continuo | < 5 min | 30–60 min |
| Snapshot | `pg_basebackup` / dump comprimido | diario | 24 h | 1–2 h |
| Lógico | `pg_dump` por esquema | semanal | 7 días | 4 h |
| Off-site | copia cifrada en storage externo | diario | 24 h | depende de bajada |

Reglas:

- **Réplica ≠ backup.** Un `DELETE` se replica instantáneamente.
- **Restore probado** cada trimestre, cronometrado, en una máquina distinta. Un backup no probado es una hipótesis.
- **`pg_dump` + WAL** en el mismo host no protege de perder el host.
- **Cifrado en reposo y en tránsito**, y la clave **no** en el mismo servidor.
- **Prueba de restore parcial**: recuperar un solo tenant o un solo objeto, que es lo que realmente se pide en un incidente ("borré 400 contactos").

---

## 9. Cinco errores de diseño caros y comunes

### Error 1 — Row-level como post-filtro en la aplicación

```python
rows = query("SELECT * FROM deals WHERE tenant_id=%s", t)   # trae 50.000
rows = [r for r in rows if can_read(user, r)]               # filtra en Python
```
**Consecuencia concreta:** un listado de 50.000 registros tarda 20–40 s y consume RAM; el `can_read` hace una query por registro (N+1); y en cuanto alguien escribe un endpoint nuevo (export, PDF, búsqueda, webhook) **se olvida del filtro y hay fuga de datos**. Es el bug que convierte un CRM en un incidente de privacidad.
**Antídoto:** una sola función `visibility_clause(principal, object)` que **devuelve SQL** y que todo camino de lectura está obligado a usar; test que verifica que `search ⊆ list`.

### Error 2 — Field-level security solo en la interfaz

La UI oculta el campo "salario"; la API lo devuelve.
**Consecuencia concreta:** cualquiera con el token de un vendedor, o con las devtools abiertas, lee la banda salarial de todo el equipo. En un CRM con datos de compensación o datos personales (Ley 21.719, GDPR), es una brecha notificable, no un bug.
**Antídoto:** FLS en la proyección SQL (§3.4), `describe` por usuario, y un test que intente leer un campo prohibido y espere `403`/campo ausente.

### Error 3 — Auditar todo, en la misma base, para siempre

Se registra cada `GET`, cada payload completo, sin particiones ni retención.
**Consecuencia concreta:** la tabla de auditoría crece 20–40 GB/año y se come el disco; el autovacuum no da abasto; los backups tardan horas; y cuando el disco se llena, **lo primero que se borra es la auditoría** — es decir, se pierde justo el registro que era obligatorio conservar. El intento de cumplimiento produce el incumplimiento.
**Antídoto:** tres auditorías separadas (§4.1), solo campos marcados, partición mensual, `BRIN`, archivo a storage frío, y topes explícitos de campos auditables por objeto (20/200, como Salesforce).

### Error 4 — Sincronizar lo que debería ser asíncrono, y encolar lo que no debe serlo

Dos caras del mismo error:
- Enviar el correo y el webhook **dentro** del request de guardado → timeout a los 30 s, el usuario reintenta, se duplica el correo, y el webhook se pierde.
- Encolar **la creación de la venta** → worker caído = ventas que existen en la UI de alguien y no en la base; o el cliente ve "guardado" y no hay nada.
**Consecuencia concreta:** duplicados en el mejor caso; pérdida silenciosa de ingresos en el peor.
**Antídoto:** la tabla de §6.1 como regla de revisión de código; outbox transaccional para los efectos; nunca diferir una decisión de negocio.

### Error 5 — Automatización sin control de bucles ni de propiedad de campos

Tres workflows que se disparan entre sí, dos que escriben `stage`.
**Consecuencia concreta:** el registro oscila entre estados, cada edición dispara 40 ejecuciones, la CPU se va al 100 %, y el reporte de pipeline da cifras que nadie puede explicar. En Odoo esto es posible porque el motor no bloquea la recursión (§5.3); en EspoCRM los bucles por fórmula también se documentan como posibles **[verificado + https://docs.espocrm.com/administration/workflows/]**.
**Antídoto:** profundidad máxima, set de reglas visitadas, `transition_only`, y **validación de "un dueño por campo"** al guardar la regla.

### Bonus 1 — Campos personalizados con `ALTER TABLE` en caliente

El admin crea un campo y la app hace `ALTER TABLE deals ADD COLUMN c_123 text` sobre una tabla de 5 M filas. Con `DEFAULT` no nulo o un índice, bloquea escrituras minutos.
**Consecuencia:** el CRM se congela porque alguien agregó "Teléfono alternativo".
**Antídoto:** híbrido `JSONB` para campos de usuario, columnas físicas solo para campos promovidos con migración planificada.

### Bonus 2 — Cursor de paginación no determinista

Ordenar por `updated_at` sin desempate y paginar por cursor.
**Consecuencia:** páginas duplicadas o filas que nunca aparecen; en el caso de Twenty, **pérdida del 95 % de las filas** con `hasNextPage: false` **[verificado + https://github.com/twentyhq/twenty/pull/24354]**. Un export "completo" que en realidad exporta la mitad.
**Antídoto:** `(sort_key, id)` siempre, `NULL` explícito, y test de paginación exhaustiva sobre un dataset con duplicados y nulos.

---

## 10. Qué haría distinto en un servidor modesto (2–4 GB RAM)

Un SaaS multinacional puede permitirse Elasticsearch, Kafka, microservicios y réplicas de lectura. En 2–4 GB **cada componente compite por la misma RAM** y no hay escala horizontal: si el diseño hace N+1, no hay "agregar otra instancia" que lo salve.

### 10.1 Decisiones, una por una

| Necesidad | En SaaS | En 2–4 GB | Por qué |
|---|---|---|---|
| Búsqueda | Elasticsearch / OpenSearch | **PostgreSQL FTS** (`tsvector` + GIN) + `pg_trgm` para "contiene" | Elastic son 1–2 GB de heap mínimo |
| Cola | Kafka / SQS / RabbitMQ | **Tabla `job` + `FOR UPDATE SKIP LOCKED`** | Cero procesos extra, transaccional con el negocio |
| Cache | Redis / Memcached | **LRU en proceso** (permisos, metadata, settings) | Redis son 100–300 MB para ahorrar 5 ms |
| Sesiones | Redis | **Tabla** (o cookie firmada) | Mismo motivo |
| Búsqueda de texto + filtros | Índice dedicado | Un solo `WHERE` con FTS + visibilidad | Una sola fuente de verdad de permisos |
| Tareas programadas | Scheduler distribuido | **`cron` + un worker con lock de advisory** (`pg_advisory_lock`) | Evita ejecuciones duplicadas sin coordinador |
| Multi-tenancy | Shard por tenant | **`tenant_id` en todas las tablas + índices compuestos** (o schema por tenant si son <20) | Un schema por tenant multiplica el catálogo y el vacuum |
| Reportes | Data warehouse | **Vistas materializadas** refrescadas por job + límite de rango | Un `GROUP BY` sobre 5 M filas bloquea el OLTP |
| Archivos | S3 + CDN | **Disco + `sendfile`**, o S3 si hay salida | Los adjuntos en la DB matan el backup |
| Observabilidad | Datadog | **Logs a archivo con rotación + `/metrics` Prometheus** | Un agente pesado no cabe |
| Webhooks | Cola dedicada | Cola `webhook` con worker propio y límite de concurrencia | Aísla al cliente caído |

### 10.2 Presupuesto de memoria (ejemplo 4 GB)

| Componente | RAM | Nota |
|---|---|---|
| SO + page cache | 512 MB | El page cache es lo que hace rápido a Postgres: **no lo ahogues** |
| PostgreSQL `shared_buffers` | 1 GB | ~25 % de la RAM total |
| PostgreSQL `work_mem` | 4–16 MB por operación | Ojo: se multiplica por conexiones y por nodos del plan |
| App (1 proceso, 4–8 threads) | 512–768 MB | Monolito; sin JVM |
| Worker (concurrencia 4) | 256 MB | Mismo binario, otro modo |
| Margen / picos | 512 MB | Sin margen, el OOM killer decide por ti |

Ajustes que importan más que el hardware: `shared_buffers` 25 %, `effective_cache_size` 50–60 %, `work_mem` bajo, `max_connections` 40–60 con **pool en la app**, `random_page_cost=1.1` en SSD, `checkpoint_timeout` alto, `wal_compression=on`.

### 10.3 Lo que NO intentaría en 2 GB

- **Kafka, Elasticsearch, Temporal, Keycloak**: cada uno pide más RAM que toda la app.
- **Microservicios**: el overhead de N runtimes + N pools de conexión es mayor que el problema que resuelven.
- **GraphQL público sin límites**: una query anidada puede generar un plan de 40 JOINs. Si se ofrece, con `depth limit`, `complexity limit` y *persisted queries*.
- **Event sourcing completo**: el `event store` crece sin techo y la reconstrucción de estado necesita RAM.
- **Multi-región / HA real**: con un solo servidor, la HA es "restaurar rápido", no "failover". Hay que decirlo y ensayarlo.
- **Búsqueda con `LIKE '%texto%'`** sobre millones de filas: usa `pg_trgm` con GIN o FTS.

### 10.4 Modo degradado explícito

Cuando la carga sube, el sistema debe degradarse **en orden conocido**, no al azar:

```
1. Pausar jobs de mantenimiento (backfill, reindex, reportes programados)
2. Bajar concurrencia de workers de import/export
3. Servir listados desde cache de 30 s
4. Rechazar exports > 50.000 filas con 429 + "usa el export asíncrono"
5. Pausar webhooks salientes (se acumulan en outbox, no se pierden)
6. NUNCA degradar: autenticación, autorización, escritura de la venta
```

Ese orden se documenta, se prueba con carga, y se expone en `/health` como `degraded_mode: 3`.

### 10.5 La consecuencia de arquitectura más importante

En SaaS, un N+1 se arregla con hardware y caching. **En 2–4 GB, un N+1 es un outage.** Por eso, en este contexto:

1. La visibilidad **tiene** que ser una cláusula SQL (§3.6): es la única forma de que un listado de 500 filas sea 1 query.
2. La paginación **tiene** que ser keyset: un `OFFSET 40000` en 4 GB es muerte por I/O.
3. La auditoría **tiene** que estar particionada y con retención: no hay disco infinito.
4. Los exports **tienen** que ser asíncronos: un `SELECT *` de 200.000 filas en un request mata el pool.
5. La búsqueda **tiene** que compartir la cláusula de visibilidad: no hay un segundo sistema que puedas permitirte mantener sincronizado.

---

## 11. Apéndice

### 11.1 DDL mínimo del núcleo de permisos

```sql
-- Tenant y principal
CREATE TABLE tenant (id uuid PRIMARY KEY, name text NOT NULL, created_at timestamptz DEFAULT now());

CREATE TABLE app_user (
  id uuid PRIMARY KEY,
  tenant_id uuid NOT NULL REFERENCES tenant(id),
  email citext NOT NULL,
  is_active boolean NOT NULL DEFAULT true,
  manager_id uuid REFERENCES app_user(id),
  hierarchy_path text NOT NULL,                 -- '/root/<id>/' materialized path
  created_at timestamptz DEFAULT now(),
  UNIQUE (tenant_id, email)
);
CREATE INDEX ON app_user (tenant_id, hierarchy_path text_pattern_ops);

-- RBAC
CREATE TABLE role (id uuid PRIMARY KEY, tenant_id uuid NOT NULL, name text NOT NULL, is_baseline boolean DEFAULT false);
CREATE TABLE role_permission (
  role_id uuid REFERENCES role(id) ON DELETE CASCADE,
  object text NOT NULL,
  action text NOT NULL,                          -- read|create|edit|delete|view_all|modify_all
  granted boolean NOT NULL DEFAULT true,
  PRIMARY KEY (role_id, object, action)
);
CREATE TABLE user_role (user_id uuid REFERENCES app_user(id), role_id uuid REFERENCES role(id), PRIMARY KEY (user_id, role_id));

-- FLS
CREATE TABLE field_permission (
  role_id uuid REFERENCES role(id) ON DELETE CASCADE,
  object text NOT NULL,
  field text NOT NULL,
  can_read boolean NOT NULL DEFAULT true,
  can_edit boolean NOT NULL DEFAULT false,
  PRIMARY KEY (role_id, object, field)
);

-- Equipos y pertenencia
CREATE TABLE team (id uuid PRIMARY KEY, tenant_id uuid NOT NULL, name text NOT NULL);
CREATE TABLE team_member (team_id uuid REFERENCES team(id), user_id uuid REFERENCES app_user(id), PRIMARY KEY (team_id, user_id));

-- Row-level: owner en el registro + shares explícitos para el resto
CREATE TABLE record_share (
  object text NOT NULL,
  record_id uuid NOT NULL,
  grantee_type text NOT NULL,                    -- user|team|role
  grantee_id uuid NOT NULL,
  access text NOT NULL,                          -- read|edit
  row_cause text NOT NULL,                       -- manual|rule|team
  created_at timestamptz DEFAULT now(),
  PRIMARY KEY (object, record_id, grantee_type, grantee_id)
);
CREATE INDEX ON record_share (grantee_type, grantee_id, object, record_id);
```

### 11.2 Reglas de visibilidad: la tabla que hay que completar antes de escribir código

| Caso | Predicado | Ejemplo |
|---|---|---|
| Propio | `owner_id = :me` | siempre |
| Equipo | `team_id = ANY(:my_teams)` | EspoCRM `team` |
| Jerarquía | `owner_path LIKE :my_prefix` | Salesforce role hierarchy |
| Compartido | `EXISTS (record_share ...)` | manual sharing |
| Criterio | `region = :my_region` | criteria-based rule |
| Público | `true` (o `false` si el objeto es privado) | OWD Public Read |
| Válvula | `:has_view_all` → `true` | View All Records |

**Precedencia documentada:** `view_all` > (`own` OR `team` OR `hierarchy` OR `shared` OR `criteria` OR `public`). Y una restricción adicional (restriction rule) que **sí puede cerrar**, aplicada como `AND` al final.

### 11.3 Checklist de revisión de diseño

**API**
- [ ] ¿Existe `fields=` y se intersecta con FLS antes del SQL?
- [ ] ¿La paginación es `(sort_key, id)` con `NULL` explícito y cursor validado?
- [ ] ¿Hay `Idempotency-Key` en los `POST` que crean registros?
- [ ] ¿Los errores traen `code` + `request_id`?
- [ ] ¿Hay endpoint `describe` con `readable`/`editable` por usuario?

**Auth**
- [ ] ¿Cada token es un principal con scopes, no un usuario prestado?
- [ ] ¿Se valida el array de scopes **otorgados**, no los pedidos?
- [ ] ¿SCIM soporta `filter=userName eq` y `PATCH active:false`?
- [ ] ¿La baja en el IdP desactiva el acceso en < 5 min?

**Autorización**
- [ ] ¿Existe una sola función de visibilidad reutilizada por list/detail/search/export/webhook?
- [ ] ¿El test de conteo de queries pasa con 100 registros?
- [ ] ¿Un campo restringido en `order_by` da `400`?
- [ ] ¿Un usuario sin rol ve cero?
- [ ] ¿La impersonación registra `on_behalf_of`?

**Auditoría**
- [ ] ¿Está particionada y con retención definida?
- [ ] ¿Solo campos marcados, con `old`/`new`?
- [ ] ¿Sobrevive al borrado del usuario (sin FK)?
- [ ] ¿Se puede consultar por registro y por actor en < 200 ms?

**Automatización**
- [ ] ¿Hay profundidad máxima y set de reglas visitadas?
- [ ] ¿Cada regla declara qué campos escribe, y se rechaza el conflicto?
- [ ] ¿Los cambios hechos por automatización quedan auditados con `automation_run_id`?
- [ ] ¿El escalamiento por tiempo usa `next_action_at` indexado, no un scan?

**Async / webhooks**
- [ ] ¿La venta se commitea en el request (no se encola)?
- [ ] ¿Los eventos salen por outbox en la misma transacción?
- [ ] ¿Firma HMAC sobre cuerpo crudo con timestamp y comparación constante?
- [ ] ¿DLQ tiene replay?
- [ ] ¿El orden no se asume, y el consumidor reconcilia por `version`?

**Operación**
- [ ] ¿Las migraciones son expand/contract, sin locks largos?
- [ ] ¿El restore se probó este trimestre y se midió el tiempo?
- [ ] ¿Los límites de tasa devuelven headers y `Retry-After`?
- [ ] ¿El modo degradado está definido y probado?
- [ ] ¿Los logs no contienen PII ni tokens?

### 11.4 Fuentes consultadas

| Tema | URL |
|---|---|
| Salesforce — record-level security | https://help.salesforce.com/s/articleView?id=platform.security_data_access.htm&language=en_US&type=5 |
| Salesforce — sharing rules (300/50) | https://help.salesforce.com/s/articleView?id=platform.security_about_sharing_rules.htm&language=en_US&type=5 |
| Salesforce — sharing rows y row cause | https://developer.salesforce.com/docs/platform/salesforce-record-access-under-the-hood/guide/uth-entity.html |
| Salesforce — una fila por grupo, no por miembro | https://developer.salesforce.com/blogs/engineering/2013/10/behind-the-scenes-of-record-ownership-in-salesforce |
| Salesforce — recálculo asíncrono de sharing | https://help.salesforce.com/s/articleView?id=005103630&language=en_US&type=1 |
| Salesforce — FLS y precedencia sobre sharing | https://developer.salesforce.com/docs/platform/apex-dev/references/apex-dev/apex-classes-perms-enforcing.html |
| Salesforce — governor limits | https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_apexgov.htm |
| Salesforce — límites de API por 24 h | https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/limits.htm |
| Salesforce — Field Audit Trail / Field History | https://developer.salesforce.com/docs/atlas.en-us.field_history_retention.meta/field_history_retention/field_audit_trail.htm |
| Salesforce — retención de Event Monitoring | https://help.salesforce.com/s/articleView?id=000396295&language=en_US&type=1 |
| Salesforce — Setup Audit Trail (180 días) | https://help.salesforce.com/s/articleView?id=xcloud.shr_set_up_audit_trail.htm&language=en_US&type=5 |
| Salesforce — recursión en record-triggered flow | https://architect.salesforce.com/docs/architect/decision-guides/guide/record-triggered |
| HubSpot — rate limits y headers | https://developers.hubspot.com/docs/developer-tooling/platform/usage-guidelines |
| HubSpot — CRM Search API (10k, 5 rps, 200/pág) | https://developers.hubspot.com/docs/api-reference/latest/crm/search-the-crm |
| HubSpot — scopes granulares y gobernanza | https://developers.hubspot.com/blog/build-with-confidence-app-governance |
| Odoo — access rights y record rules | https://www.odoo.com/documentation/17.0/developer/reference/backend/security.html |
| Odoo — triggers y recursión (tests) | https://github.com/odoo/odoo/blob/f399f99d/addons/test_base_automation/tests/test_flow.py |
| EspoCRM — roles, niveles, FLS | https://docs.espocrm.com/administration/roles-management/ |
| EspoCRM — workflows y safeguard de bucles | https://docs.espocrm.com/administration/workflows/ |
| Twenty — bugs de paginación por cursor | https://github.com/twentyhq/twenty/pull/24354 |
| Twenty — orden multi-campo y tie-breaker | https://github.com/twentyhq/twenty/issues/20520 |
| Pipedrive — rate limits por tokens | https://pipedrive.readme.io/docs/core-api-concepts-rate-limiting |
| Stripe — paginación | https://docs.stripe.com/api/pagination |
| Stripe — idempotencia | https://docs.stripe.com/api/idempotent_requests |
| Stripe — firma de webhooks | https://docs.stripe.com/webhooks/signature |
| Stripe — reintentos, orden, duplicados | https://docs.stripe.com/webhooks |
| SCIM 2.0 — protocolo | https://www.rfc-editor.org/rfc/rfc7644 |
| SCIM — requisitos de Okta | https://developer.okta.com/docs/api/openapi/okta-scim/guides/scim-20 |
| SCIM — requisitos de Entra ID (grupos/PATCH) | https://learn.microsoft.com/en-us/entra/identity/app-provisioning/use-scim-to-provision-users-and-groups |
| SCIM — API de Entra con cursor (RFC 9865) | https://learn.microsoft.com/en-us/entra/identity/app-provisioning/entra-id-scim-api-reference |
| Temporal — activities, retry, idempotencia | https://docs.temporal.io/standalone-activity |
| Temporal — idempotencia y ejecución durable | https://temporal.io/blog/idempotency-and-durable-execution |
| Transactional outbox | https://microservices.io/patterns/data/transactional-outbox.html |
| n8n — workflows | https://docs.n8n.io/workflows/ |

---

*Fin del documento. Las secciones 1, 3 y 6 son las que definen si el backend sobrevive al tercer cliente; las demás se pueden iterar.*
