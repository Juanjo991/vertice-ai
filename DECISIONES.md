# Decisiones cerradas — Vértice AI

Registro de definiciones de producto y comerciales ya acordadas. Si algo no está aquí, no está decidido.

---

## D01 — Dos arquetipos, un solo desembarco

**Arquetipo A — Autoempleado.** Trabaja por cuenta propia y opera solo de punta a punta: vende, atiende, cobra, compra y anota. Su control vive en la cabeza, en un cuaderno o en un Excel. No tiene a nadie que le diga **qué hacer** con la información que ya tiene.

> *"No tiene datos; tiene registros sueltos y ninguna interpretación."*

**Arquetipo B — Pyme con empleados.** Personal a cargo, volumen diario de transacciones y capacidad de pago instalada. Ya pasó la etapa del cuaderno, pero sigue sin una foto confiable y única de su operación.

> *"Tiene los datos, pero repartidos entre personas y sistemas que no se hablan. Paga por registrar y sigue decidiendo a ciegas."*

**Producto:** empezamos por A. B llega cuando el núcleo esté validado.

---

## D02 — La brecha de escala

> El valle del software no es un problema de funcionalidad, es un problema de **precio**. La Pyme es demasiado grande para el Excel y demasiado pequeña para el software a medida que existía hasta ahora: millones de pesos de implementación, un semestre de desarrollo, un equipo dedicado. Ese precio la deja fuera del mercado, no su tamaño.

**Vértice es a medida — a precio de genérico.** A medida en el resultado (su base de datos, sus parámetros, su contexto, sus módulos); estandarizado en la construcción (librería reutilizable de módulos + desarrollo asistido por IA).

Por eso la ruptura con el mercado no es funcional, es económica: **no competimos por funcionalidad, competimos por acceso.**

---

## D03 — La IA es el pilar, no una capa opcional

**El código es la oficina y el reglamento; la IA es el empleado.**

- El código hace la aritmética, controla el stock, consulta y barre patrones. No inventa nada y no cuesta tokens.
- La IA interpreta los datos, decide, redacta y conversa con el dueño.

**Prohibido en toda la documentación:** decir que el sistema puede operar sin IA, o que su costo es despreciable. El costo de IA es real, mensual y variable según cuánto use el cliente al asistente.

---

## D04 — Escalera de segmentos

- **A nunca se deja de vender.** Es motor de volumen, validación y confianza.
- **B se suma cuando el núcleo está probado.** Es motor de margen.
- No es abandono del micro: es escalera.

---

## D05 — Precios

| Cliente | Implementación | Mensual |
|---|---|---|
| Cercano / validación | ~$20 USD simbólico | — |
| Autoempleado (A) | $80 USD base → $120–150 con módulos | $9.900 CLP **voluntario** |
| Pyme con empleados (B) | $500 → $800 → $1.000 USD | $79.900 – $220.000 CLP (obligatorio) |

El precio final se fija en la **Auditoría**, según el diagnóstico del flujo de caja del cliente.

**La Línea Directa de A es acceso prioritario, no un arriendo de software:** canal directo, prioridad de cola, 15% de descuento en la siguiente implementación y biblioteca de formación. Sin garantía de disponibilidad. Se detalla en D08.

---

## D06 — Stack del cliente (Arquetipo A)

| Concepto | Monto aproximado | Nota |
|---|---|---|
| Plan de IA | ~$20.000/mes | Obligatorio — es el empleado |
| VPS | $10.000 – 13.000/mes | Servidor propio del cliente |
| Línea Directa | $9.900/mes | Voluntaria |
| **Total** | **~$40.000 – 43.000/mes** | |

**Proveedores de IA ofrecidos:** ChatGPT plan básico (recomendación estándar), Claude (más capaz, más costoso), o alternativa de pago por uso a pedido explícito. Con los protocolos bien escritos, cualquier modelo se comporta correctamente dentro del sistema: no tiene que construir nada, solo interpretar.

**Servidor y datos son del cliente.** Ese es el modelo: el sistema no se arrienda, se instala.

---

## D07 — Clasificación interna de módulos (no comercial)

Los módulos se agrupan **por área de trabajo** para efectos de diseño y roadmap: captura, caja y margen, inventario, clientes, agente, interfaz y conexiones externas. Se detalla en [`04-MODULOS.md`](04-MODULOS.md).

**Esta clasificación es interna.** No es una escalera de compra: el cliente no adquiere áreas por separado. Ver D10.

**La web con pasarela de pago no es Vértice.** Es un servicio complementario que Vértice puede orquestar.

---

## D08 — Línea Directa

**Qué es:** acceso prioritario. No es una licencia de software ni un contrato de mantención. El cliente compra **posición en la fila y un canal directo**.

**Precio:** $9.900 CLP/mes (≈ $10 USD). Voluntario, mensual, cancelable en cualquier momento.

**Regla de oro:** no se promete disponibilidad ni tiempo de resolución. Se promete **prioridad de atención**.

| | Con Línea Directa | Sin Línea Directa |
|---|---|---|
| Canal | Directo: número o correo personal | Formulario automatizado con triage |
| Atención | Se atiende apenas se recibe, en horario hábil | Por lotes, en una ventana semanal |
| Velocidad relativa | Rápida | Significativamente más lenta |
| Descuento en expansión | 15% en la siguiente implementación | Sin descuento |
| Biblioteca de formación | Incluida | No incluida |
| Garantía de disponibilidad | Ninguna | Ninguna |

**Incluye:** canal directo sin triage · prioridad de cola · 15% de descuento en la siguiente implementación · biblioteca de formación.

**No incluye:** garantía de *uptime* · soporte 24/7 · desarrollos nuevos a medida · costos de VPS o de IA · hardware.

**Cancelación:** cualquier mes, sin penalidad. El cliente conserva sus datos y recibe exportación completa.

**Por qué se vende:** nadie paga por "mantención" que cree que nunca va a necesitar. Sí paga por escribirle directo al que construyó el sistema, y por un 15% en lo que venga después.

---

## D09 — Biblioteca de formación

**Qué es:** material de iniciación para clientes, incluido en Línea Directa. Privado, no público.

**Justificación interna:** el cliente no entiende qué compró. Confunde el modelo con la empresa, el agente con una persona, el token con una palabra. Esa confusión genera tickets que no deberían existir. **La formación es prevención de soporte, no un regalo.**

| Módulo | Contenido |
| :-: | :--- |
| 0 | Qué compraste y qué no: el sistema no piensa, predice. No es una persona |
| 1 | Qué es un modelo de lenguaje · qué es un token · por qué se equivoca |
| 2 | Qué es un agente · qué es una herramienta · qué es una API · qué es una base de datos · dónde viven tus datos |
| 3 | Operación diaria: registrar una venta, leer un aviso del radar, pedirle cosas al asistente |
| 4 | Límites y seguridad: qué nunca debe hacer, privacidad, respaldos |

**Formato:** primero PDF/ebook por módulo. Videos de 30-45 min en YouTube privado cuando exista margen de tiempo.

---

## D10 — Un solo producto

**El cliente no compra por partes.** Recibe Vértice completo desde el día 1. No existe una escalera de cinco pagos para llegar al producto.

**Versión 1 incluye:**
- Ventas · compras · inversiones · costos
- Stock automático (descuenta al vender, suma al comprar)
- Flujo de caja: cuánto entró, cuánto salió, cuánto queda a fin de mes
- Margen unitario por producto
- Fichas de clientes (se activan solo si el cliente las necesita)
- **Interpretación de IA sobre todos esos datos desde el primer día**

**Capacidad que se agrega después** (no es requisito para tener Vértice, y es la segunda y tercera venta sobre el mismo cliente):
- Interfaz web con dashboard — la v1 opera con el agente y Telegram
- Mayor inteligencia comercial: radar de recompra, fechas clave, match de demanda vs. stock
- Mayor seguimiento de otros datos: documentos (PDF/OCR), pagos con tarjeta, otros canales
- Mayor personalización por rubro
- Conexiones externas: redes sociales, correo electrónico, campañas
- Multiusuario y roles

**Precio:** una sola implementación por cliente. El monto lo fija la Auditoría según el alcance de sus datos, no un menú de niveles.

**Regla de oro:** el producto es uno. Lo que varía por cliente es el alcance de sus datos, no el número de compras.
