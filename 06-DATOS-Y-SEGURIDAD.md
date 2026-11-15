# 06 — Datos y seguridad

## Principio

**Los datos son del cliente.** El sistema vive en su servidor, en un archivo que él puede copiar, mover o borrar. Vértice no custodia la información de sus clientes finales: le entrega las herramientas para custodiarla bien.

---

## Esquema de datos

### `transactions`
El corazón del sistema. Toda entrada o salida de dinero.

| Columna | Tipo | Notas |
|---|---|---|
| `id` | INTEGER PK | |
| `fecha` | TEXT (ISO 8601) | Zona horaria `America/Santiago` |
| `tipo` | TEXT | `ingreso` · `costo` · `gasto` · `inversion` |
| `categoria` | TEXT | Insumo, mercadería, arriendo, servicio… |
| `producto_id` | INTEGER FK | Nulo en gastos sin producto asociado |
| `cliente_id` | INTEGER FK | Nulo en gastos |
| `cantidad` | REAL | |
| `monto` | INTEGER | En CLP, sin decimales |
| `costo_unitario` | INTEGER | Congelado al momento de la transacción |
| `canal_origen` | TEXT | Cómo llegó la venta |
| `notas` | TEXT | |

**Regla:** `costo_unitario` se guarda con la transacción, no se lee del producto. Si el costo cambia después, el margen histórico no se altera.

### `products`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | INTEGER PK | |
| `nombre` | TEXT | |
| `sku` | TEXT | Opcional, único si existe |
| `precio` | INTEGER | |
| `costo` | INTEGER | Costo vigente |
| `stock` | REAL | Se actualiza por transacción |
| `stock_minimo` | REAL | Umbral de alerta |
| `activo` | INTEGER | 0/1 |

El **margen** no se guarda: se calcula. Guardar resultados calculados es la forma más común de tener datos que se contradicen.

### `clients`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | INTEGER PK | |
| `token` | TEXT | Identificador seudónimo (`CLT_042`) |
| `nombre` | TEXT | Solo en la base local |
| `telefono` | TEXT | Solo en la base local |
| `email` | TEXT | Solo en la base local |
| `canal_origen` | TEXT | |
| `primera_compra` | TEXT | Derivada, actualizable |
| `ultima_compra` | TEXT | Derivada, actualizable |
| `notas` | TEXT | Contexto cualitativo breve |

### `providers`

`id` · `nombre` · `contacto` · `plazo_reposicion_dias` · `calificacion` · `notas`

### `lost_sales`
Ventas que no se cerraron. Es la tabla que casi nadie tiene y la que más valor genera.

| Columna | Tipo | Notas |
|---|---|---|
| `id` | INTEGER PK | |
| `fecha` | TEXT | |
| `producto_id` | INTEGER FK | Qué buscaba |
| `cliente_id` | INTEGER FK | Nulo si fue consulta anónima |
| `motivo` | TEXT | `precio` · `sin_stock` · `lo_pensara` · `compro_otro` |
| `monto_potencial` | INTEGER | Para dimensionar la pérdida |
| `resuelta` | INTEGER | 0/1 — se marca cuando se recupera |

### `fechas_clave`
Fuente única de verdad de todo evento futuro. **No la memoria conversacional.**

| Columna | Tipo | Notas |
|---|---|---|
| `id` | INTEGER PK | |
| `cliente_id` | INTEGER FK | |
| `tipo` | TEXT | `garantia` · `mantencion` · `recambio` · `aniversario` · `otro` |
| `fecha` | TEXT | |
| `dias_anticipacion` | INTEGER | |
| `estado` | TEXT | `pendiente` · `avisado` · `cumplido` · `descartado` |
| `id_evento_externo` | TEXT | Espejo en calendario, cuando aplique |

### `appointments`
`id` · `cliente_id` · `servicio` · `inicio` · `duracion_min` · `estado` · `monto` · `notas`

### Memoria conversacional (FTS5)
Tabla virtual con búsqueda de texto completo sobre lo que el dueño ha contado. Guarda fragmentos, no transacciones. **No dispara eventos.**

---

## Privacidad y protección de datos

### Roles legales

- **El cliente es el responsable de los datos.** Los clientes finales son suyos.
- **Vértice es el encargado del tratamiento** cuando administra la instancia, y no interviene cuando está autoalojada.

### Las cuatro capas de protección

**1. Seudonimización antes de cualquier llamada externa**

En la base local el cliente es `Juan Pérez` con RUT y teléfono. Cuando el sistema necesita que un modelo externo analice, le envía `CLT_042 compró SKU_12 hace 45 días por $35.000`. La identidad nunca viaja.

El reemplazo se hace localmente y se reconstruye localmente. El modelo responde sobre `CLT_042`; el sistema traduce a `Juan Pérez` antes de mostrarlo.

**2. Política de retención cero en el proveedor**

Se usan conexiones de API con política contractual de no retención y no entrenamiento. Es una condición para elegir proveedor, no una preferencia.

**3. Aislamiento físico por cliente**

Un archivo `.db` por cliente, en su servidor. No hay una tabla compartida con los datos de toda la cartera. Un incidente en un cliente no expone a los demás.

**4. Derechos del titular, automatizados**

| Derecho | Comando | Qué hace |
|---|---|---|
| Acceso | `/exportar_cliente CLT_042` | Genera un archivo con todo lo que el sistema sabe de esa persona |
| Rectificación | edición en ficha | Corrige el dato |
| Supresión | `/borrar_cliente CLT_042` | Elimina el registro y sus vínculos |
| Oposición | campo `no_contactar` | Excluye al cliente de cualquier aviso del radar |

**Nota honesta:** la seudonimización reduce la exposición, no la elimina. Un conjunto de datos seudonimizado sigue siendo dato personal bajo la ley. **No se debe describir el sistema como "anónimo"**, porque no lo es.

---

## Reglas de manejo de datos

1. **Credenciales en `.env`**, nunca en el código, nunca en el repositorio, nunca en un mensaje.
2. **Nunca** datos de producción en desarrollo sin anonimizar.
3. **Nunca** enviar a un modelo externo nombre completo, RUT, teléfono ni dirección.
4. **Nunca** usar los datos de un cliente para otro cliente, ni agregados, ni para entrenar nada.
5. **Siempre** registrar quién accedió a la instancia de un cliente y cuándo.

---

## Respaldos

| Aspecto | Definición |
|---|---|
| Frecuencia | Diaria, automática |
| Retención | 30 días rotativos |
| Ubicación | Fuera del servidor del cliente (almacenamiento externo definido en la implementación) |
| Verificación | Chequeo de integridad semanal |
| Restauración | Probada antes de la entrega, no después |

**Un respaldo que no se ha restaurado nunca no es un respaldo.** Es una carpeta con archivos.

---

## Checklist de cumplimiento antes de entregar

- [ ] `.env` presente, fuera del control de versiones, con permisos restringidos
- [ ] Seudonimización activa y probada con un caso real
- [ ] Comandos de derechos del titular funcionando
- [ ] Campo `no_contactar` respetado por todos los avisos
- [ ] Respaldo automático configurado y **restauración probada**
- [ ] Registro de accesos activo
- [ ] Cliente informado por escrito de dónde viven sus datos y quién los administra
