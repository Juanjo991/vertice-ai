# Plan de construcción por bloques

**Fecha:** 16 de noviembre de 2026
**Qué es:** la investigación dividida en bloques que se pueden pensar, decidir y construir de a uno. Cada bloque tiene hechos, una inferencia y un artefacto. No se avanza al siguiente sin cerrar el anterior.

---

## Cómo se trabaja cada bloque

Cuatro pasos, siempre igual:

1. **Hechos.** Lo que la investigación verificó. No se discute: se cita.
2. **Inferencia.** Qué significa eso para nosotros. Acá es donde se decide.
3. **Decisión.** Qué se adopta, escrito en una línea.
4. **Artefacto.** Lo que queda: un plano, una regla, una pantalla, una prueba.

Si un bloque no produce un artefacto, no estaba bien definido.

---

## El mapa

Once bloques en cuatro etapas. El orden no es temático: es de dependencia.

### Etapa 1 — Cimientos (lo que no se puede cambiar después)

| # | Bloque | Qué contiene | Qué inferimos | Artefacto |
|:-:|---|---|---|---|
| **1** | **La forma de la casa** | cuántas secciones tiene el sistema y dónde vive cada dato | son siete, no seis: el CRM tiene seis y le falta el libro de plata; y cada sección tiene **un escritor distinto** | el plano de las siete secciones |
| **2** | **El dinero y el tiempo** | cómo se guarda un monto y un momento | hay cuatro datos que, si no están desde el día uno, no se pueden agregar después | las cuatro reglas |
| **3** | **La historia** | qué se guarda de "qué cambió" | la etapa es obligatoria y barata; el resto es caro (8×) y selectivo | la lista de 5-6 campos vigilados |
| **4** | **El edificio por dentro** | dónde vive todo y cómo se respalda | un cuaderno por cliente simplifica el 70 % del sistema | la decisión de forma + el respaldo |

### Etapa 2 — Vida (lo que el usuario toca)

| # | Bloque | Qué contiene | Qué inferimos | Artefacto |
|:-:|---|---|---|---|
| **5** | **La entrada del dato** | las cinco puertas por donde entra la información | el crudo hay que guardarlo: el proveedor lo borra en días | el recorrido de entrada, puerta por puerta |
| **6** | **La pantalla que manda** | la lista, la ficha, la línea de tiempo | el sistema se diseña desde la lista hacia atrás | las cinco pantallas en orden |
| **7** | **El trabajo del día** | el próximo paso, el "no", los avisos | el "no" con motivo y fecha es la palanca de plata | la mecánica del seguimiento |

### Etapa 3 — Verdad (lo que se cobra)

| # | Bloque | Qué contiene | Qué inferimos | Artefacto |
|:-:|---|---|---|---|
| **8** | **Los números** | los cinco reportes y la foto mensual | cinco bastan el día uno; el resto es vanidad | las cinco preguntas con su cálculo |
| **9** | **Quién ve qué** | permisos y auditoría | esconder en pantalla no protege: la regla va en la consulta | la regla única de visibilidad |

### Etapa 4 — Memoria y entrega

| # | Bloque | Qué contiene | Qué inferimos | Artefacto |
|:-:|---|---|---|---|
| **10** | **La memoria** | el porqué de las decisiones y la doctrina del negocio | nadie lo guarda; es el único activo que crece con el tiempo | el registro de decisiones y su revisión |
| **11** | **La casa se mantiene** | respaldos, actualizaciones, entrega al cliente | la copia en caliente produce archivos vacíos que *pasan* el chequeo | el procedimiento probado |

---

## Dependencias

```
1 ── 2 ── 3 ── 4 ── 5 ── 6 ── 7
                    │
                    └── 8 ── 9
                              └── 10 ── 11
```

- **1 y 2 no dependen de nada.** Se pueden cerrar hoy.
- **3 depende de 1**, porque la historia se cuelga de las secciones.
- **5 y 6 dependen de 1-4**: no se puede capturar ni mostrar lo que no está definido.
- **7 y 8 dependen de 5-6**, porque necesitan datos entrando.
- **9 se puede posponer** mientras haya un solo usuario.
- **10 y 11 son transversales**, pero se construyen al final: la memoria sin datos es un cuaderno vacío, y el respaldo sin datos no tiene nada que proteger.

---

# BLOQUE 1 — La forma de la casa

## 1.1 Hechos

- Todo CRM comercial cabe en seis secciones: personas/empresas, lo que se quiere vender, lo que pasó, el responsable, el próximo paso y el resultado.
- Contacto y empresa se relacionan **muchos a muchos**, no uno a muchos. Y el cargo pertenece a la relación, no a la persona.
- La etapa de un negocio tiene que ser una fila con orden y probabilidad, no una etiqueta de texto.
- Ningún CRM guarda **por qué** se tomó una decisión: guardan el qué.
- Ningún CRM tiene libro de plata ni mercadería: eso vive en el ERP, en otro sistema, desconectado.

## 1.2 Inferencia

**El CRM tiene seis secciones y le falta la séptima, que es justo donde vive la verdad.** Por eso existen dos mercados separados: uno te dice a quién llamar y el otro cuánto ganaste. Nadie te dice las dos cosas con el mismo dato.

Y hay una segunda inferencia, que es de interfaz y no de datos: **cada sección tiene un escritor distinto.** No es lo mismo:

| Sección | Quién la escribe de verdad |
|---|---|
| Personas | una persona, a mano o por formulario |
| Lo que se quiere vender | una persona, hablando |
| Lo que pasó | **las máquinas**: correo, calendario, botón |
| El próximo paso | el sistema solo, y la persona confirma |
| La venta | **el núcleo**, cuando el dueño confirma |
| El porqué | el asistente, con confirmación |
| El libro de plata | **solo el núcleo**, nunca una persona |

Eso significa que la interfaz no es "un chat". Son **cuatro puertas distintas** según quién escribe. Un chat que pretenda ser las cuatro va a ser malo en las cuatro.

## 1.3 Decisión propuesta

Siete secciones, un dueño por dato, y ninguna sección copia lo que ya vive en otra: solo lo apunta.

## 1.4 Artefacto: el plano

### Sección 1 — Personas y empresas
**Qué guarda:** quién es, cómo se le habla, en qué empresa trabaja y con qué cargo.
**Qué no guarda:** plata, historia, opiniones. Todo eso apunta acá, no se copia.
**Detalle fino:** el cargo va en el vínculo entre persona y empresa, porque la misma persona puede tener dos.

### Sección 2 — Lo que se quiere vender
**Qué guarda:** qué está en juego, en qué punto va, cuánto vale y para cuándo.
**Qué no guarda:** los productos (eso es la cotización) ni los pagos (eso es la venta).
**Detalle fino:** una empresa puede tener varias cosas en juego a la vez, y cada una avanza por su cuenta.

### Sección 3 — Lo que pasó
**Qué guarda:** cada llamada, mensaje, visita, reunión o nota, con la fecha en que ocurrió.
**Qué no guarda:** qué hacer después. Eso es la sección 4.
**Detalle fino:** "cuándo pasó" no es lo mismo que "cuándo lo anoté". Una llamada de ayer anotada hoy es de ayer.

### Sección 4 — El próximo paso
**Qué guarda:** qué hay que hacer, con quién, para cuándo y quién es el responsable.
**Detalle fino:** todo cliente debería tener uno. Si no tiene, el sistema lo dice en vez de callarse.

### Sección 5 — La venta
**Qué guarda:** el hecho: cuánto, cuándo, cómo se pagó, qué se llevó.
**Qué no guarda:** la intención. Eso es la sección 2.
**Detalle fino:** la venta es el puente entre el CRM y el libro de plata. Es el único lugar donde los dos mundos se tocan.

### Sección 6 — El porqué
**Qué guarda:** qué se decidió, por qué, qué se esperaba que pasara, y cuándo revisar si resultó.
**Detalle fino:** es la única sección que no existe en ningún CRM del mercado.

### Sección 7 — El libro de plata y mercadería
**Qué guarda:** la plata que entra y sale, lo que hay en stock, lo que se compró, lo que se debe.
**Quién escribe:** solo el núcleo. Ninguna persona, ningún asistente.
**Detalle fino:** solo se agrega, nunca se tacha. Un error se corrige con un asiento al revés, como en contabilidad.

---

## 1.5 Lo que queda abierto de este bloque

1. **¿El "responsable" es una sección o un dato?** En un negocio de una persona no hay a quién asignar. Propongo que sea un dato de las secciones 1, 2 y 4, no una sección propia.
2. **¿La cotización es una sección?** Propongo que no en la primera versión: se guarda como una actividad con monto, y cuando haya volumen se separa.
3. **¿La empresa existe si el negocio vende a personas?** Propongo que sí, siempre, aunque sea "Particular": evita migrar después y permite agrupar.
