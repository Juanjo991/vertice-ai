# 12 — Las preguntas que responde Vértice

**Estado:** borrador de trabajo (16 de noviembre de 2026). No es una decisión cerrada.
**Qué es:** el catálogo de preguntas que el sistema debe poder responder, ordenado en tres bloques: lo que ya da un CRM (línea base), lo que solo Vértice responde sobre los números, y lo que solo Vértice puede responder sobre el tiempo (la memoria). Es la materia prima de la venta, del diseño de la base y de las pruebas del agente.

---

## Por qué las preguntas y no las funciones

Al cliente no se le vende "un módulo de inventario". Se le vende **la respuesta a una pregunta que hoy no puede contestar**. Cada pregunta del catálogo es una decisión que el dueño toma y que hoy toma a ciegas.

De ahí salen tres consecuencias prácticas:

1. **La captura se ordena sola.** El bot solo debe captar los datos que alguna pregunta necesita. Si un campo no alimenta ninguna pregunta, no se pide.
2. **La venta se mide.** "En treinta días te muestro cuánta plata tienes parada en stock" es una promesa verificable; "inteligencia artificial para tu negocio" no lo es.
3. **Las pruebas se escriben solas.** Cada pregunta es un caso de prueba: mismos datos, misma pregunta, misma respuesta esperada.

---

## Cómo se calcula cada respuesta (no es una escalera de compra)

Toda respuesta del catálogo trae una marca de **cómo se produce**, no de qué plan la incluye. El producto es uno (D10, D13) e incluye el agente desde el día 1 (D08).

| Marca | Qué significa | Costo |
|---|---|---|
| **N1** | El núcleo determinístico responde el número. Sin modelo. | $0 de tokens |
| **N2** | El núcleo detecta un patrón con reglas (el radar). | $0 de tokens |
| **N3** | El agente interpreta lo que el núcleo ya calculó y lo convierte en conversación. | Costo real de IA |

**Regla dura:** el agente nunca calcula. Toda cifra que aparezca en una respuesta N3 viene de una consulta N1 o N2. Esto es lo que separa a Vértice de un chatbot.

---

## BLOQUE A — La línea base: lo que ya responde cualquier CRM

Este bloque **no diferencia**. Si el cliente compara con un CRM de $30.000 al mes, aquí empatamos o perdemos. Su función es otra: es el piso de confianza. Si estas respuestas están mal, el cliente no cree en ninguna de las del Bloque B.

Se incluyen igual, por tres razones: son las preguntas que el dueño hace primero, son la prueba de que los datos están completos, y son la puerta de entrada a la conversación donde aparece el Bloque B.

### A1. Ventas y actividad

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| A1.1 | ¿Cuánto vendí hoy, esta semana, este mes? | N1 | ventas por fecha |
| A1.2 | ¿Cuántas ventas hice? | N1 | ventas |
| A1.3 | ¿Cuál es mi ticket promedio? | N1 | monto total / cantidad de ventas |
| A1.4 | ¿Qué vendí más, por unidades y por plata? | N1 | ítems de venta |
| A1.5 | ¿Cómo va este mes contra el anterior? | N1 | ventas por período |
| A1.6 | ¿Cuánto vendí por canal (local, Instagram, WhatsApp, feria)? | N1 | canal de origen |
| A1.7 | ¿Cuánto entró en efectivo, transferencia y tarjeta? | N1 | forma de pago |

### A2. Clientes

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| A2.1 | ¿Cuántos clientes tengo? | N1 | clientes |
| A2.2 | ¿Quiénes son mis mejores clientes por monto? | N1 | ventas por cliente |
| A2.3 | ¿Cuándo fue la última vez que me compró X? | N1 | última compra por cliente |
| A2.4 | ¿Qué le vendí a X y cuánto lleva gastado conmigo? | N1 | historial por cliente |
| A2.5 | ¿Cuántos clientes nuevos entraron este mes? | N1 | primera compra |
| A2.6 | ¿Cuántos me han comprado más de una vez? | N1 | recurrencia |
| A2.7 | ¿Cuál es el teléfono o correo de X? | N1 | ficha |
| A2.8 | ¿Quién no me compra desde hace N días? | N2 | última compra vs. ciclo del rubro |

### A3. Producto y stock

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| A3.1 | ¿Cuánto stock me queda de X? | N1 | movimientos de stock |
| A3.2 | ¿Qué productos tengo? | N1 | catálogo |
| A3.3 | ¿Cuánto vale mi inventario, a costo y a precio de venta? | N1 | stock × costo / precio |
| A3.4 | ¿Cuánto me costó X y a cuánto lo vendo? | N1 | producto |
| A3.5 | ¿A qué proveedor le compro cada cosa? | N1 | proveedores × productos |
| A3.6 | ¿Cuánto vendí de X y con cuánto stock quedé? | N1 | ventas + stock |

### A4. Caja y gastos

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| A4.1 | ¿Cuánto gasté este mes? | N1 | gastos |
| A4.2 | ¿En qué gasté? | N1 | categorías |
| A4.3 | ¿Cuánto tengo en caja hoy? | N1 | entradas − salidas |
| A4.4 | ¿Cuánto entró y cuánto salió? | N1 | flujo del período |
| A4.5 | ¿Cuánto me deben y cuánto debo? | N1 | cuentas por cobrar / pagar |
| A4.6 | ¿Cuánto voy a pagar de IVA? | N1 | ventas netas + IVA (ver bloque de límites) |

### A5. Operación

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| A5.1 | ¿Qué tengo agendado esta semana? | N1 | agenda (solo rubros con cita) |
| A5.2 | ¿Qué me quedó pendiente? | N1 | tareas / pasos abiertos |

**Nota de honestidad:** A5.1 y A5.2 solo aplican a rubros con agenda (servicios, estética, consultas). En compra-venta de productos no existen y no hay que prometerlas.

---

## BLOQUE B — Lo que solo Vértice responde

Ninguna de estas preguntas la contesta un CRM, un ERP contable ni un Excel, porque todas necesitan dos cosas que el mercado no tiene juntas: **los datos conectados en una sola base** y **alguien que los interprete**.

### B1. ¿Cuánto gané de verdad?

La pregunta que abre la conversación (caso de uso central del producto).

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| B1.1 | ¿Cuánto gané limpio este mes, de verdad? | N3 sobre N1 | ventas, costo de lo vendido, gastos |
| B1.2 | ¿Cuál es mi margen por producto? ¿cuál me deja más plata aunque venda menos? | N3 sobre N1 | costo congelado por transacción |
| B1.3 | ¿Qué vendo mucho y me deja poco? | N2 | margen × volumen |
| B1.4 | ¿Cuánto de lo que vendí es plata que ya tengo y cuánto está en la vitrina? | N3 sobre N1 | ventas vs. compras de stock |
| B1.5 | ¿Me comí el margen cuando cambié el precio de X? | N2 | historial de precios |
| B1.6 | ¿Cuánto me queda después de reponer lo que vendí? | N1 | resultado − costo de reposición |

**Por qué no lo responde un CRM:** un CRM suma ventas. No sabe que comprar mercadería no es un gasto —es un cambio de forma de la plata— y por eso confunde "vendí mucho" con "gané mucho".

### B2. La plata dormida

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| B2.1 | ¿Cuánta plata tengo parada en stock que no se mueve? | N1 | stock × costo, sin venta en N días |
| B2.2 | ¿Qué producto lleva más tiempo sin venderse y cuánto capital tengo ahí? | N2 | rotación por producto |
| B2.3 | ¿Qué me conviene liquidar y cuánto puedo recuperar? | N3 sobre N2 | rotación + costo + margen |
| B2.4 | ¿Qué se me acaba esta semana? | N2 | stock vs. velocidad de venta |
| B2.5 | ¿Cuándo se me acaba X si sigo vendiendo a este ritmo? | N1 | días de cobertura |
| B2.6 | ¿Qué compré de más? | N2 | stock vs. rotación |
| B2.7 | ¿Cuánta plata necesito para reponer lo que se viene? | N3 sobre N1 | proyección de venta + costo |

### B3. El cliente como relación, no como fila

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| B3.1 | ¿Qué clientes puedo recuperar este mes y cuánta plata representan? | N3 sobre N2 | inactividad + ticket histórico |
| B3.2 | ¿Quién dejó de comprar y por qué? | N3 sobre N2 | inactividad + motivo de pérdida |
| B3.3 | ¿A quién estoy a punto de perder? | N2 | ciclo del rubro, aún activo |
| B3.4 | ¿Qué le gusta a X? ¿qué compró, qué rechazó, qué conversamos? | N3 | historial + memoria cualitativa |
| B3.5 | ¿Qué tienen en común los que compraron una vez y no volvieron? | N3 sobre N2 | cohortes de compra única |
| B3.6 | ¿Cuánto vale un cliente en el tiempo? | N1 | ticket × frecuencia × vida |
| B3.7 | ¿Quién está comprando más seguido y quién menos que antes? | N2 | cambio de frecuencia |
| B3.8 | ¿A quién le escribo esta semana y qué le digo? | N3 | radar + memoria + stock |
| B3.9 | ¿Quién debería volver por el siguiente paso del ciclo? | N2 | ciclo por rubro (uña, mantención, retoque) |

**Por qué no lo responde un CRM:** el CRM tiene el dato de contacto y la fecha de la última compra. No tiene el **por qué** ni el **qué le gusta**, y no sabe cuál es el ciclo natural de recompra de ese rubro en particular.

### B4. La venta que no ocurrió

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| B4.1 | ¿Cuánto vendí y cuánto dejé de vender? | N1 | ventas + ventas perdidas |
| B4.2 | ¿Por qué no cerré? (precio, sin stock, lo pensará, se fue a otro) | N1 | motivo de pérdida |
| B4.3 | ¿Cuánto me costó no tener stock de X? | N1 | ventas perdidas por motivo |
| B4.4 | ¿Qué me piden y no tengo? | N2 | demanda insatisfecha |
| B4.5 | Ya llegó lo que me pidieron: ¿a quién le aviso? | N2 | match demanda ↔ stock nuevo |
| B4.6 | ¿Qué canal me trae clientes que compran y cuál solo consulta? | N1 | conversión por canal |

**Por qué no lo responde un CRM:** los CRM modernos registran "oportunidad perdida" solo si el vendedor la carga a mano. El dueño solo no lo hace. Un botón de un toque en Telegram, sí.

### B5. El futuro

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| B5.1 | ¿Cuánto voy a cerrar este mes si sigo así? | N1 | ritmo actual + días restantes |
| B5.2 | ¿Puedo pagar X el próximo mes? | N3 sobre N1 | compromisos vs. caja proyectada |
| B5.3 | ¿Qué fechas vienen y a quién tengo que contactar? | N2 | tabla de fechas clave |
| B5.4 | ¿Qué postventa tengo pendiente y con quién? | N2 | garantías, mantenciones, recambios |
| B5.5 | ¿Cuál fue mi mejor y mi peor mes, y qué pasó ahí? | N3 sobre N1 | historia + contexto |

### B6. Qué hago esta semana

El grupo que más diferencia: el sistema pasa de informar a **recomendar**.

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| B6.1 | ¿Qué hago esta semana? | N3 sobre N1+N2 | todos los hallazgos, priorizados por plata |
| B6.2 | ¿Qué es lo más urgente por plata, no por tiempo? | N3 | severidad económica del hallazgo |
| B6.3 | ¿Qué dejé de hacer que me está costando plata? | N3 sobre N2 | postventa vencida, stock parado, cliente dormido |
| B6.4 | ¿Qué pasa si subo el precio de X un 10%? | N3 sobre N1 | margen, volumen, elasticidad histórica |
| B6.5 | ¿Qué pasa si dejo de comprarle a este proveedor? | N3 sobre N1 | costos, plazos, alternativas |

**Regla de redacción:** la recomendación siempre dice **cuánta plata está en juego** y **qué se hace**, en ese orden. "Te quedan 2 pulseras y vendes 3 por semana: si no repones el jueves, pierdes ~$45.000" es una recomendación. "Tu stock está bajo" es un tablero.

### B7. Comparado conmigo mismo

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| B7.1 | ¿Cómo voy comparado conmigo mismo, no con el mercado? | N1 + N3 | series históricas propias |
| B7.2 | ¿Este mes es raro o es normal para mi negocio? | N2 | desviación sobre su propia historia |
| B7.3 | ¿Tengo estacionalidad? ¿cuáles son mis meses buenos? | N3 sobre N2 | 12+ meses de historia |
| B7.4 | ¿Qué cambió desde el mes pasado y qué lo explica? | N3 | comparación + hallazgos del radar |
| B7.5 | ¿Por qué vendí menos? | N3 | ventas + stock + clientes + canal |

**Advertencia comercial:** este grupo **crece con el tiempo**. Con dos semanas de datos no hay comparación posible. Hay que decirlo al vender, para no prometer magia el primer día.

**Advertencia comercial:** este grupo **crece con el tiempo**. Con dos semanas de datos no hay comparación posible. Hay que decirlo al vender, para no prometer magia el primer día.

> La memoria del negocio (qué conversamos con este cliente, por qué subí el precio, qué decidí el mes pasado) no vive acá: es el **Bloque C**, con su propia arquitectura y su propio tiempo de maduración.

---

## BLOQUE C — La memoria: la persona y el negocio

Este bloque no depende de tener buenos datos. Depende de haber estado escuchando. **Ninguna** de estas preguntas la responde un CRM, un ERP, un Excel ni un chat suelto, porque todas necesitan una memoria persistente de lo que el dueño dijo.

La arquitectura que las sostiene está en [`13-MEMORIA.md`](13-MEMORIA.md): conversación en Engram, el por qué en la tabla `decisiones`, la doctrina en `.md` versionados.

### C1. La memoria de la persona

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| C1.1 | ¿Qué sé de este cliente, más allá de lo que compró? | N3 | memoria por cliente |
| C1.2 | ¿Qué me contó la última vez? ¿qué le gusta y qué no? | N3 | memoria + ficha |
| C1.3 | ¿Cómo le gusta que lo traten? | N3 | reglas de trato |
| C1.4 | ¿Qué le prometí y no cumplí? | N2 | compromisos con fecha |
| C1.5 | ¿Hace cuánto no hablo con él? (contacto, no compra) | N1 | última interacción |

### C2. La memoria de las decisiones

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| C2.1 | ¿Por qué subí el precio de X? | N3 | tabla `decisiones` |
| C2.2 | ¿Qué decidí el mes pasado y cómo me fue? | N3 sobre N1 | decisiones + resultado |
| C2.3 | ¿Qué esperaba que pasara y no pasó? | N2 | campo `esperaba` + revisión |
| C2.4 | ¿Qué cambié y después revertí? | N2 | linaje de decisiones |
| C2.5 | ¿Estoy por decidir algo que ya decidí antes? | N3 | búsqueda por similitud de título |

### C3. La memoria del negocio

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| C3.1 | ¿Cómo funciona mi negocio, según todo lo que te he contado? | N3 | doctrina `.md` |
| C3.2 | ¿Qué cambió este año? | N3 | bitácora de cambios |
| C3.3 | ¿Qué reglas tengo y de dónde salieron? | N2 | doctrina + decisiones |
| C3.4 | ¿Qué contradice lo que creo hoy? | N3 | detección de conflicto entre memorias |
| C3.5 | ¿Qué te dije que iba a probar y no probé? | N2 | intenciones sin cierre |
| C3.6 | ¿Qué se me está olvidando? | N3 | memoria sin acción asociada |

### C4. La memoria como servicio

| # | Pregunta del dueño | Marca | Datos que necesita |
|:-:|---|:-:|---|
| C4.1 | ¿Qué recuerdas de mí que yo ya había olvidado? | N3 | todo lo anterior |
| C4.2 | Si mañana entra alguien a trabajar conmigo, ¿qué leería? | N1 | doctrina exportable |
| C4.3 | ¿Qué parte de mi negocio no está escrita en ninguna parte? | N3 | huecos de doctrina |

**Por qué este bloque es el más defendible:** las preguntas del bloque B se pueden copiar. Un competidor puede construir el mismo radar de stock en tres meses. Una memoria de dos años de conversaciones **no se copia**: se construye escuchando, un día a la vez. Es el único activo del producto que se vuelve más fuerte con el tiempo y que el cliente no puede llevarse a otro lado sin perder historia.

**Advertencia de venta:** C1 a C3 empiezan vacías y se llenan hablando. C4.1 no tiene respuesta el primer mes. Prometerla el día 1 es la forma más rápida de perder al cliente el día 30.

---

## Resumen del catálogo

| Bloque | Grupo | Preguntas |
|---|---|:-:|
| A | Ventas y actividad | 7 |
| A | Clientes | 8 |
| A | Producto y stock | 6 |
| A | Caja y gastos | 6 |
| A | Operación | 2 |
| **A total** | **Línea base (no diferencia)** | **29** |
| B | Cuánto gané de verdad | 6 |
| B | La plata dormida | 7 |
| B | El cliente como relación | 9 |
| B | La venta que no ocurrió | 6 |
| B | El futuro | 5 |
| B | Qué hago esta semana | 5 |
| B | Comparado conmigo mismo | 5 |
| **B total** | **Diferenciación sobre los números** | **43** |
| C | La memoria de la persona | 5 |
| C | La memoria de las decisiones | 5 |
| C | La memoria del negocio | 6 |
| C | La memoria como servicio | 3 |
| **C total** | **Diferenciación sobre el tiempo** | **19** |
| | **Total** | **91** |

---

## Las preguntas que hoy no se pueden responder bien

No es una lista de deseos: es lo que impide responder con verdad algunas preguntas del catálogo. Sale de `PENDIENTES.md` y de `analisis/01-auditoria-datos.md`.

| Pregunta del catálogo | Por qué hoy falla | Qué falta |
|---|---|---|
| A4.6 y B1.1 (IVA y utilidad real) | `monto` no distingue bruto de neto; el margen sale ~19% corrido en cliente formal | separar neto + IVA + total |
| A3.1 y B2.* (stock y rotación) | `stock` es una columna que se sobrescribe, no un historial | libro de movimientos con saldo derivado |
| A1.4 y B1.2 (margen por producto) | venta y línea de venta están fusionadas: con descuentos, el total no cuadra | separar `ventas` y `venta_items` |
| A1.1 y B7.* (ventas de la noche) | fecha con offset mal interpretada: una venta de 22:30 cae al día siguiente | `fecha_local` sin offset + UTC aparte |
| Cualquiera con respaldo | copiar la base en caliente produce archivos vacíos que pasan el chequeo de integridad | `VACUUM INTO` + restauración probada |

**Regla:** mientras estas no se corrijan, esas preguntas **no se prometen en la venta**. Se responde lo que se puede sostener.

---

## La versión 1: tres preguntas de números y una de memoria

Criterio de elección: **se ven en pesos, piden pocos datos y el dueño puede actuar esta semana.**

| Elegida | Por qué | Depende de |
|---|---|---|
| **B1.1** — ¿Cuánto gané limpio este mes, de verdad? | Es la pregunta que abre toda conversación y la que el cliente no puede contestar hoy | arreglo de IVA y de costo de lo vendido |
| **B2.1** — ¿Cuánta plata tengo parada en stock que no se mueve? | Se ve en pesos, no requiere que el cliente registre nada nuevo, y se puede mostrar en la primera semana | libro de movimientos de stock |
| **B3.1** — ¿Qué clientes puedo recuperar y cuánta plata representan? | Convierte la base de clientes en una lista de llamadas con monto | fecha de última compra + ticket |
| **C2.1** — ¿Por qué subí el precio de X? | Es la demo más barata y más difícil de olvidar: se responde con una conversación de hace tres meses | Engram instalado + protocolo de guardado |

Las tres primeras comparten una virtud: **no necesitan seis meses de historia**. Se pueden demostrar con el histórico que el cliente entrega en la implementación.

La cuarta es distinta y por eso importa: **es la única que no depende de los datos, sino de haber escuchado.** Se puede mostrar el día 2 del piloto, no el mes 6, y no requiere arreglar el esquema de dinero. Es la que se enseña en la reunión 2 del embudo.

Quedan fuera de la v1, a propósito: todo el grupo B7 y C3 (necesitan historia), B6.4 y B6.5 (simulación), y C4 (la memoria como servicio se promete, no se demuestra el día 1).

---

## Lo que no hay que prometer

Para no perder la pelea equivocada:

- **No competir como CRM.** Si el cliente quiere pipeline, etapas, embudo de ventas y campañas, no es nuestro terreno. Ahí hay productos de $30.000 al mes con diez años de desarrollo.
- **No competir como contabilidad.** El sistema no reemplaza al contador ni emite documentos tributarios.
- **No prometer "cero errores".** El código no se equivoca en la aritmética; el agente puede interpretar mal. Lo que se promete es que **toda cifra viene de una consulta**, y que el agente dice de dónde salió.
- **No prometer el grupo B7 el primer mes.** Crece con la historia acumulada.

---

## Cómo se usa este documento

1. **Para vender:** el catálogo B es la lista de lo que se ofrece. Se eligen 3 y se prometen por escrito.
2. **Para diseñar la captura:** cada campo que el bot pide tiene que alimentar al menos una pregunta del catálogo.
3. **Para probar:** cada pregunta es un caso de regresión con datos fijos.
4. **Para comparar modelos:** las mismas diez preguntas del Bloque B sobre los mismos datos a tres modelos distintos, y se comparan respuestas, costo y dónde quedan los datos.
5. **Para el brief de la demo:** solo se construyen las consultas de las tres preguntas elegidas.
