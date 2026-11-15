# 05 — Stack

## Principio de selección

Cada tecnología se eligió por una razón operativa concreta, no por moda. El criterio dominante: **el sistema tiene que funcionar en un VPS barato, administrado por una sola persona, sin equipo de operaciones.**

---

## Núcleo

| Componente | Tecnología | Por qué |
|---|---|---|
| Lenguaje | **Python 3.11+** | Ecosistema de librerías, legibilidad, velocidad de desarrollo |
| Base de datos | **SQLite** | Un archivo por cliente, cero administración, respaldo por copia de archivo |
| Búsqueda de texto | **SQLite FTS5** | Memoria conversacional sin servidor adicional |
| Agente | **OpenCode** | Runtime agéntico con herramientas cerradas, protocolos en `.md` y control de contexto |
| Control de versiones | **Git** | Historial auditable del código y de los protocolos |

### Por qué SQLite y no PostgreSQL

- Un archivo `.db` por cliente = **aislamiento físico real**. Un incidente afecta a un cliente, no a la cartera.
- Respaldo = copiar un archivo. Restauración = reemplazarlo. Sin `pg_dump`, sin usuarios, sin permisos de servidor.
- Funciona en un VPS de 2 GB de RAM sin tuning.
- El borrado legal (derecho de supresión) es literalmente eliminar un archivo.

**Costo de esta decisión:** no hay concurrencia real de escritura. Es aceptable mientras la v1 sea de un solo operador, y es la razón por la que el multiusuario llega con un rediseño, no con un flag.

---

## Interfaces

| Canal | Estado | Uso |
|---|---|---|
| **Telegram (bot)** | V1 | Captura por botones y notificaciones |
| **Chat con el agente** | V1 | Conversación en lenguaje natural |
| **Web (dashboard + consola)** | V1.x | Ver el negocio en una pantalla |
| **Correo electrónico** | Posterior | Comunicación y notificaciones formales |

### Por qué Telegram primero

- El dueño ya lo tiene instalado y ya lo usa todos los días.
- Notificaciones nativas, sin construir infraestructura de push.
- Cero desarrollo de interfaz: el esfuerzo se concentra en el núcleo.
- Registro en 10 segundos sin abrir una aplicación nueva.

**Lo que se pierde:** no hay dashboard. Por eso la web es la primera expansión natural.

---

## Modelo de IA

El cliente elige proveedor. El sistema no está casado con ninguno.

| Opción | Perfil | Cuándo recomendarla |
|---|---|---|
| **ChatGPT (plan básico)** | Estándar | Recomendación por defecto |
| **Claude** | Más capaz, consume más | Cliente que quiere más profundidad y puede costearlo |
| **Pago por uso (DeepSeek / Qwen)** | Bajo costo variable | Solo si el cliente lo pide explícitamente |

**Por qué funciona con cualquier modelo:** el agente no construye nada. Las cuentas las hace Python, los datos se los entrega una herramienta cerrada, y su trabajo es interpretar y redactar. Con protocolos bien escritos, un modelo mediano se comporta correctamente.

**Advertencia honesta:** el costo de IA **no es despreciable y varía con el uso**. Un cliente que conversa mucho con el asistente gasta más. Por eso no se promete un costo fijo: se recomienda un plan mensual con límites.

---

## Infraestructura del cliente

| Componente | Especificación típica | Costo aproximado |
|---|---|---|
| VPS Linux | 2 vCPU / 4 GB RAM / disco SSD | $10.000 – 13.000 CLP/mes |
| Dominio | Opcional en v1 | ~$10.000 CLP/año |
| Plan de IA | Según proveedor elegido | ~$20.000 CLP/mes |

**Regla:** el servidor y los datos son del cliente. Vértice no arrienda infraestructura ni aloja información de terceros en sus propios nodos para este segmento.

---

## Librerías previstas

| Necesidad | Librería |
|---|---|
| Bot de Telegram | `python-telegram-bot` |
| Cálculo y tablas | `pandas` (ingesta y reportes), `sqlite3` (núcleo) |
| PDFs | `pypdf` / `pdfplumber` |
| Documentos e informes | `reportlab` o generación de HTML a PDF |
| Fechas y zonas horarias | `zoneinfo` (nativo), `dateutil` |
| Variables de entorno | `python-dotenv` |
| Pruebas | `pytest` |
| Calendario (V1.x) | `google-api-python-client` |
| Pagos (V1.x) | SDK de la pasarela correspondiente |

---

## Entornos

| Entorno | Propósito | Datos |
|---|---|---|
| **Desarrollo** | Construir y probar | Sintéticos |
| **Pruebas** | Validar con escenarios reales | Anonimizados |
| **Producción del cliente** | Operación real | Reales del cliente |

**Nunca** se copian datos de producción de un cliente a un entorno de desarrollo sin anonimizar. Ver [06 — Datos y seguridad](06-DATOS-Y-SEGURIDAD.md).

---

## Convenciones técnicas

- **Credenciales siempre en `.env`**, nunca en el código ni en el repositorio.
- **Un repositorio por instancia de cliente** cuando se despliega en su servidor; el código base vive aparte y se versiona.
- **Migraciones de esquema** versionadas: cada cambio de tabla tiene su script y su reverso.
- **Idempotencia** en las herramientas de registro: registrar dos veces la misma venta no debe duplicar el stock.
- **Zona horaria explícita** en toda marca de tiempo (`America/Santiago`), nunca hora local implícita.
- **Moneda en CLP** como unidad base, con enteros cuando sea posible para evitar errores de punto flotante en dinero.

---

## Deuda técnica asumida

Se declara para que nadie la descubra después:

| Deuda | Motivo | Cuándo se resuelve |
|---|---|---|
| Sin concurrencia de escritura | SQLite y un solo operador | Con el multiusuario |
| Sin cola de trabajos | El volumen no lo exige | Cuando haya más de 30 clientes |
| Sin monitoreo centralizado | Cada cliente es independiente | Cuando exista operación delegada |
| Sin tests de carga | No hay carga todavía | Antes del cliente 20 |
