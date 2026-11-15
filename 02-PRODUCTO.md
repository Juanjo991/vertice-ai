# 02 — El producto

## Qué es Vértice

**Un empleado digital para el negocio.** Captura lo que pasa en la operación y lo convierte en decisiones concretas: a quién llamar, qué dejar de comprar, cuánto estás ganando de verdad.

Técnicamente es un sistema agéntico instalado en el servidor del cliente, compuesto por:

- un **núcleo determinístico** en Python + SQLite que hace toda la aritmética y el control de datos;
- un **agente de IA** que interpreta, dialoga y redacta;
- un **conjunto de protocolos** (el *arnés*) que le dicen al agente cómo comportarse con los datos de ese negocio en particular.

---

## La metáfora que explica todo

> **El código es la oficina y el reglamento. La IA es el empleado.**

Una oficina sin empleado no trabaja: tiene escritorios y archivos, y nadie los usa. Un empleado sin oficina improvisa: anota en servilletas, se equivoca en las sumas, olvida cosas.

Vértice es un empleado con oficina:

| Sin IA | Con IA sin arnés | Vértice |
|---|---|---|
| Excel automático: captura datos y no dice nada | Asistente que inventa números y alucina | Empleado que interpreta sobre datos verificados |

**Consecuencia de diseño:** el sistema **no puede operar sin IA**. Si se le quita el agente, queda un registrador. Y si se le quita el núcleo determinístico, queda un chatbot que opina sin saber.

---

## Qué recibe el cliente el día 1

**Un solo producto. No hay escalera de pagos para llegar a Vértice.**

### Captura de la operación
- **Ventas** — monto, producto, cantidad, cliente, canal de origen, fecha
- **Compras e inversiones** — mercadería, insumos, equipamiento
- **Costos y gastos** — fijos y variables, con su categoría
- **Stock automático** — descuenta al registrar una venta, suma al registrar una compra

### Lectura del negocio
- **Flujo de caja** — cuánto entró, cuánto salió y cuánto queda a fin de mes
- **Margen unitario** por producto — precio menos costo, sin comisiones escondidas
- **Fichas de clientes** — historial, última compra, canal de origen. Se activan solo si el cliente las necesita
- **Informe del período** — el estado del negocio en una página

### La capa que lo hace distinto
- **Interpretación de IA sobre todos esos datos, desde el primer día.** El sistema no solo muestra la cifra: dice qué significa, qué se movió y qué conviene mirar.

> Ejemplos reales de lo que el agente puede decir en la primera semana:
> *"Ganaste $180.000 este mes, pero el 40% salió de un solo producto. Te quedan 3 unidades y se venden 2 por semana: si no repones, se te corta."*
> *"Tres clientes compraron dos veces en 60 días y no volvieron. Los tres compraron el mismo producto."*

### Cómo se usa
- **Conversación en lenguaje natural** con el agente. El dueño se explaya hablando y el sistema desglosa y enruta.
- **Botones rápidos** en Telegram para captura sin fricción: `+Venta`, `+Compra`, `+Gasto`, `+Stock`.
- **Notificaciones** en Telegram para lo que requiere atención.

---

## Qué NO incluye el día 1

Decirlo explícitamente evita expectativas falsas:

- **Interfaz web con dashboard.** La v1 opera con el agente y Telegram. La web llega después.
- **Conexión automática con Mercado Pago u otras pasarelas.** Los pagos se registran manualmente o por mensaje.
- **Lectura de documentos** (facturas y boletas en PDF o foto). El gasto se carga a mano.
- **Radar automático de recompra** y fechas clave recordatorias.
- **Multiusuario con roles.** La v1 asume un solo operador.
- **Conexiones con redes sociales, correo o campañas.**

---

## Qué se agrega después

Nada de esto es requisito para "tener Vértice". El cliente ya lo tiene. Esto es **capacidad nueva** que se vende sobre el mismo cliente, cuando su uso real lo justifica:

| Capacidad | Qué resuelve |
|---|---|
| **Interfaz web con dashboard** | Ver el negocio en una pantalla, sin depender de Telegram |
| **Mayor inteligencia comercial** | Radar de recompra, fechas clave, match de demanda insatisfecha contra stock |
| **Mayor seguimiento de datos** | Documentos (PDF/OCR), pagos con tarjeta, otros canales de venta |
| **Mayor personalización** | Ajustes específicos por rubro: relojería, estética, reventa, servicios |
| **Conexiones externas** | Redes sociales, correo electrónico, campañas |
| **Multiusuario** | Roles y permisos para que varias personas trabajen sin pisarse |

---

## Los tres principios no negociables

### 1. La IA es el pilar, no una capa opcional
El código hace la aritmética y el control. La IA interpreta, decide y redacta. **Nunca se describe el producto como algo que funciona sin IA**, porque sin ella es exactamente el Excel que vinimos a reemplazar.

### 2. Determinístico para lo predecible, IA para lo ambiguo
Toda cuenta —caja, margen, stock, plazos— se calcula en Python y SQL. Cero tokens, cero alucinación. El modelo solo entra cuando hay lenguaje natural que interpretar o un hallazgo que explicar.

### 3. Los datos y el servidor son del cliente
El sistema se instala en su infraestructura. No hay arriendo de software ni acceso perpetuo: el cliente es dueño del sistema y de sus datos. Si deja de pagar, nada se apaga.

---

## En qué se diferencia de lo que ya existe

| | Excel | ERP contable | Vértice |
|---|---|---|---|
| Registra | Sí, a mano | Sí | Sí, sin fricción |
| Calcula caja | A mano | Sí | Sí |
| Calcula margen real por producto | No | Parcial | Sí |
| Sabe qué hacer con el dato | No | No | **Sí** |
| Se adapta al negocio | No | Parcial | Sí |
| Cuesta millones | No | Sí, en el tiempo | No |
| Le habla al dueño | No | No | **Sí** |

---

## Un día en la vida del cliente

**8:40.** Vende dos piezas. Le manda un mensaje al asistente: *"le vendí dos pulseras de plata a la Sra. Rojas, $45.000"*. El sistema registra la venta, descuenta el stock y actualiza la caja.

**13:00.** Llega mercadería. Foto o mensaje: *"entraron 10 pulseras, me costaron $12.000 cada una"*. El stock sube, el costo queda cargado, el margen se recalcula.

**21:30.** El asistente le escribe: *"Hoy vendiste $45.000. Esta semana vas $312.000, 18% arriba de la anterior. Ojo: te quedan 2 pulseras de plata y vendes 3 por semana."*

**Fin de mes.** En vez de abrir el Excel, pregunta: *"¿cómo me fue este mes?"* Y recibe el número, el margen por producto y el capital que tiene parado en stock.

Eso es el producto. No es un tablero bonito: es alguien que mira los números con él y le dice qué hacer.
