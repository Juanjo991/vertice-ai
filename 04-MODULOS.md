# 04 — Módulos

El catálogo agrupado **por área de trabajo**, no por niveles de compra. Todos los módulos de la **versión 1** llegan juntos: el cliente no compra por partes.

**Estados:**
- **V1** — va en la primera versión, desde el día 1
- **V1.x** — capacidad nueva que se agrega sobre un cliente que ya opera
- **Posterior** — capacidad arquitectónica; no se promete en el corto plazo

---

## Área 1 — Captura

### 1.1 Registro de ventas · V1
**Qué hace:** registra monto, producto, cantidad, cliente, canal de origen y fecha. Descuenta stock y actualiza la caja en el mismo acto.
**Qué gana el cliente:** deja de anotar en dos lugares. Una venta registrada sirve para caja, stock y ficha del cliente a la vez.

### 1.2 Registro de compras e inversiones · V1
**Qué hace:** suma stock y registra el egreso de caja. Distingue mercadería de insumo y de equipamiento.
**Qué gana el cliente:** sabe cuánto capital entró al negocio y en qué forma.

### 1.3 Registro de costos y gastos · V1
**Qué hace:** clasifica egresos fijos y variables por categoría.
**Qué gana el cliente:** el gasto deja de ser un número global y pasa a tener origen.

### 1.4 Captura por botones · V1
**Qué hace:** comandos rápidos `+Venta`, `+Compra`, `+Gasto`, `+Stock` en Telegram, con preguntas de completitud cuando falta un dato.
**Qué gana el cliente:** registra en 10 segundos, sin abrir nada.

### 1.5 Captura conversacional · V1
**Qué hace:** el dueño escribe o dicta en lenguaje natural; el agente desglosa, pregunta lo que falta y enruta.
**Qué gana el cliente:** no aprende un sistema. Habla.

### 1.6 Ingesta del histórico · V1 (dentro de la implementación)
**Qué hace:** carga las planillas, exportaciones y listas antiguas que el cliente entrega como material de arranque.
**Qué gana el cliente:** el sistema no arranca vacío. Conoce su historia desde el primer día.

### 1.7 Captura documental · V1.x
**Qué hace:** extrae montos, proveedor e ítems de facturas y boletas en PDF, y de fotos de boletas físicas.
**Qué gana el cliente:** deja de digitar gastos.

### 1.8 Conciliación de pagos · V1.x
**Qué hace:** recibe webhooks de pasarelas de pago y crea la transacción; si falta un dato, pregunta.
**Qué gana el cliente:** ningún pago con tarjeta se le pierde.

---

## Área 2 — Caja y margen

### 2.1 Flujo de caja · V1
**Qué hace:** calcula entradas, salidas, saldo y resultado del período. Proyecta el cierre de mes con lo registrado.
**Qué gana el cliente:** sabe cuánto va a tener a fin de mes sin abrir Excel.

### 2.2 Margen unitario · V1
**Qué hace:** precio menos costo, por producto. Sin comisiones escondidas.
**Qué gana el cliente:** distingue lo que más vende de lo que más le deja.

### 2.3 Capital inmovilizado y rotación · V1
**Qué hace:** valoriza el stock parado (unidades × costo) y calcula cuánto demora en rotar cada producto.
**Qué gana el cliente:** sabe cuánta plata tiene dormida en la vitrina.

### 2.4 Informe del período · V1
**Qué hace:** una página con lo que pasó: caja, margen, stock, clientes.
**Qué gana el cliente:** llega a fin de mes con el número listo.

### 2.5 Motivo de pérdida y conversión por canal · V1.x
**Qué hace:** registra por qué no se cerró una venta (`precio`, `sin stock`, `lo pensará`, `compró en otro lado`) y cruza la tasa de conversión por canal de origen.
**Qué gana el cliente:** sabe por qué pierde y de dónde le llega el cliente que sí compra.

### 2.6 Comparador de proveedores · V1.x
**Qué hace:** historial de costos de compra, plazos de reposición y calidad por proveedor.
**Qué gana el cliente:** sabe a quién comprarle y cuándo.

### 2.7 Pre-cálculo de IVA · V1.x
**Qué hace:** estimación del IVA del período a partir de lo registrado.
**Qué gana el cliente:** llega a la conversación con el contador con el número adelantado.

---

## Área 3 — Inventario

### 3.1 Stock automático · V1
**Qué hace:** descuenta al vender, suma al comprar, alerta cuando llega a cero.
**Qué gana el cliente:** nunca más cuenta a mano.

### 3.2 Producto y precio · V1
**Qué hace:** catálogo con costo, precio y margen por ítem; historial de cambios de precio.
**Qué gana el cliente:** el precio deja de ser memoria.

### 3.3 Match de demanda insatisfecha · V1.x
**Qué hace:** cruza las ventas perdidas por falta de stock con las nuevas entradas de mercadería y avisa a quién ofrecerle.
**Qué gana el cliente:** convierte una venta perdida en una venta futura.

---

## Área 4 — Clientes

### 4.1 Ficha de cliente · V1 (se activa si el cliente la necesita)
**Qué hace:** historial de compras, primera y última compra, canal de origen, notas cualitativas.
**Qué gana el cliente:** tiene la historia de cada relación en un solo lugar.

### 4.2 Radar de inactividad · V1.x
**Qué hace:** detecta clientes fuera del ciclo natural de recompra de su rubro. El ciclo se parametriza por negocio: una estética y una relojería no se comportan igual.
**Qué gana el cliente:** le avisan a quién escribirle antes de perderlo.

### 4.3 Fechas clave y postventa · V1.x
**Qué hace:** hitos futuros por cliente (garantía, mantención, recambio, aniversario) con días de anticipación.
**Qué gana el cliente:** no se le olvida la postventa que trae la segunda compra.

### 4.4 Segmentación de clientes · V1.x
**Qué hace:** clasifica en compra única, recurrente y dormido.
**Qué gana el cliente:** distingue al que vuelve del que no, y sabe cuánto vale cada uno.

---

## Área 5 — El agente

### 5.1 Asistente conversacional · V1
**Qué hace:** responde preguntas sobre el negocio usando herramientas determinísticas. Interpreta, no calcula.
**Qué gana el cliente:** tiene a quién preguntarle.

### 5.2 Segundo cerebro · V1
**Qué hace:** guarda el contexto del negocio (protocolos y memoria de conversación) y lo usa para personalizar la interpretación.
**Qué gana el cliente:** el asistente aprende cómo funciona su negocio, no el promedio.

### 5.3 Agente de auto-ayuda · V1
**Qué hace:** resuelve dudas de uso y fallas menores, y documenta cada intervención en una bitácora técnica.
**Qué gana el cliente:** no depende de una persona para lo básico.

---

## Área 6 — Interfaz

### 6.1 Centro de mando en Telegram · V1
**Qué hace:** botones de captura y canal de notificaciones.
**Qué gana el cliente:** el sistema cabe en su teléfono.

### 6.2 Centro de Comando Web · V1.x
**Qué hace:** dashboard de caja, tareas, calendario y consola del agente en una pantalla.
**Qué gana el cliente:** ve todo junto y puede mostrarle el negocio a un socio o a su contador.

### 6.3 Multiusuario y roles · V1.x
**Qué hace:** permisos por persona y registro de quién hizo qué.
**Qué gana el cliente:** su gente trabaja sin pisarse los datos.

---

## Área 7 — Conexiones externas · Posterior

Capacidad arquitectónica, **no prometida en el corto plazo**. Depende de APIs de terceros y consume horas de desarrollo que hoy no existen.

| Módulo | Qué resolvería |
|---|---|
| Atribución de redes sociales | Cruzar el alcance en redes con las ventas cerradas |
| Publicación automática | Publicar catálogo o contenido desde el sistema |
| Gestión de campañas | Administrar anuncios con presupuesto del cliente |
| Conexión con correo | Enviar y registrar comunicaciones con clientes |

**Regla:** existen en el roadmap técnico y en la descripción de la arquitectura. **No aparecen en ninguna propuesta comercial** hasta que se construyan.

---

## Fuera del producto

**Web con pasarela de pago:** es un servicio complementario, no una función de Vértice. Si un cliente la necesita, se cotiza aparte y con otro alcance.

---

## Resumen de estados

| Estado | Cantidad | Qué significa |
|---|---|---|
| **V1** | 16 módulos | Entregados desde el día 1 |
| **V1.x** | 11 módulos | Capacidad nueva sobre clientes que ya operan |
| **Posterior** | 4 módulos | Capacidad arquitectónica, no prometida |
| **Fuera** | 1 | Servicio complementario |
