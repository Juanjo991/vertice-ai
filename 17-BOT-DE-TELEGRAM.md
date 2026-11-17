# 17 — El bot de Telegram: plan de construcción

**Qué es:** el plan completo del bot que captura y muestra. Determinístico, por menús, con confirmación antes de escribir.
**Estado:** plan aprobado para ejecutar. Nada de esto usa un modelo de lenguaje.

---

## 1. Lo que encontré (y lo que descarté)

| Pieza | Versión | Qué aporta | Decisión |
|---|---|---|---|
| **python-telegram-bot** | **22.8** (jun-2026) | framework, menús, conversaciones, colas de trabajo | **se usa** |
| `ConversationHandler` | incluido | el bucle y el estado por usuario, con tiempo límite | **se usa** |
| `PicklePersistence` | incluido | si el bot se reinicia a mitad de una captura, la conversación sobrevive | **se usa** |
| `JobQueue` (extra `[job-queue]`) | incluido | alertas diarias sin depender de `cron` | **se usa en fase 5** |
| aiogram | 3.32.0 (nov-2026) | alternativa más moderna, con su propio gestor de estados | **se descarta** |
| transitions | 0.9.3 | máquina de estados aparte | **se descarta** |
| pydantic | 2.14.3 | validación de datos | **no hace falta**: el núcleo ya valida |

**Por qué `python-telegram-bot` y no `aiogram`:** es la misma versión que ya corre en el bot de Catalina (22.8), así que hay un solo stack, un solo estilo y un solo lugar donde aprender. Cambiar de librería para ganar elegancia nos costaría días y no cambia nada de lo que el usuario ve.

**Por qué no `transitions`:** `ConversationHandler` ya es una máquina de estados. Sumar otra sería tener dos relojes para la misma hora.

**Dato de la documentación que hay que respetar:** con `ConversationHandler`, `concurrent_updates` debe quedar en `False`. Las conversaciones dependen de que los mensajes se procesen de a uno. Si se activa la concurrencia, el bot se enreda.

**El ejemplo oficial que sirve de patrón:** `nestedconversationbot.py`. Menú principal + conversaciones anidadas que vuelven al menú. Es exactamente lo que queremos, sin inventar.

---

## 2. Los cinco principios del bot

1. **Determinístico.** Ningún modelo. Cada botón es una operación declarada.
2. **Un solo núcleo.** Cada acción llama a una función de `nucleo.py`. El bot no escribe SQL.
3. **Nada se guarda sin confirmación.** Antes de escribir, muestra exactamente qué va a escribir, dónde y con qué consecuencia.
4. **Nunca crea una persona a ciegas.** Busca; si no encuentra, pregunta.
5. **Siempre vuelve al menú.** Ningún callejón sin salida: "cancelar" existe en todos los pasos y el tiempo límite devuelve al inicio.

---

## 3. Los tres grupos de acciones

### Grupo 1 — Registrar (la ingesta)

| Acción | Qué escribe | Función del núcleo |
|---|---|---|
| **Venta** | venta + líneas + pago + stock + evento | `registrar_venta` |
| **Compra / mercadería** | egreso + entradas de stock | `registrar_compra` (falta) |
| **Gasto** | egreso | `registrar_egreso` (falta) |
| **Cobro o pago de una venta** | pago | `registrar_pago` (falta) |
| **Devolución / corrección** | corrección | `registrar_correccion` |
| **Venta perdida** | oportunidad cerrada con motivo | `registrar_venta_perdida` (falta) |
| **Nota / lo que pasó** | actividad | `registrar_actividad` |
| **Próximo paso** | tarea | `crear_tarea` |
| **Decisión** | decisión | `registrar_decision` |
| **Producto y precio** | producto | `crear_producto` / `cambiar_precio` (falta) |

Las seis que faltan se agregan al núcleo en la fase que corresponda. **El bot no inventa nada: si la función no existe, el botón no aparece.**

### Grupo 2 — Consultar (lectura)

| Acción | Qué muestra | Función |
|---|---|---|
| **Cliente** | ficha completa: compras, cadencia, estado, últimas actividades, tareas | `ficha_cliente` |
| **El mes** | facturado, caja, ganancia, margen | `ventas_por_periodo` + `caja_del_mes` + `ganancia_del_mes` |
| **Stock** | unidades, capital, mínimos | `stock_actual` |
| **Márgenes** | por producto, ordenado | `margen_por_producto` |
| **Plata dormida** | capital inmovilizado | `plata_dormida` |
| **Ventas perdidas** | por motivo y monto | `ventas_perdidas` |
| **Decisiones por revisar** | título, motivo, expectativa | `decisiones_por_revisar` |
| **Mis tareas** | pendientes y vencidas | consulta directa |

### Grupo 3 — Ajustes

Ciclo de recompra, umbrales de alertas, datos del negocio. **Fase 5.** No es urgente y no bloquea nada.

---

## 4. El árbol de menús

Dos niveles: menú principal → grupo → acción. Las capturas son asistentes de 4 a 6 pasos.

```
🏠 Principal
├── 📥 Registrar
│   ├── Venta
│   ├── Compra / mercadería
│   ├── Gasto
│   ├── Cobro
│   ├── Devolución
│   ├── Venta perdida
│   ├── Nota
│   ├── Próximo paso
│   ├── Decisión
│   └── Producto
├── 📊 Consultar
│   ├── Cliente
│   ├── El mes
│   ├── Stock
│   ├── Márgenes
│   ├── Plata dormida
│   ├── Ventas perdidas
│   ├── Decisiones
│   └── Tareas
├── 🔔 Alertas          (fase 5)
└── ⚙️ Ajustes          (fase 5)
```

**El menú es un solo diccionario en `menu.py`.** De ahí salen los teclados *y* el enrutamiento. Si se agrega una acción, se agrega una línea y aparece en los dos lados. Nunca se desincronizan.

---

## 5. El flujo de una venta, paso a paso

```
[Venta]
   │
   ├─ 1. "¿A quién le vendiste?"
   │      → busca por nombre o teléfono
   │      ├─ 1 resultado  → "¿Es Camila Rojas (+569…)?"
   │      ├─ varios       → lista numerada
   │      └─ ninguno      → "No encontré a nadie.
   │                         ¿Es cliente nuevo? / Buscar otra vez / Cancelar"
   │
   ├─ 2. "¿Qué le vendiste?"  → busca en el catálogo, igual que las personas
   ├─ 3. "¿Cuántos?"          → 1 / 2 / 3 / otra cantidad
   ├─ 4. "¿Algo más?"         → sí (vuelve a 2) / no
   ├─ 5. "¿Cómo pagó?"        → efectivo / transferencia / tarjeta / mixto
   │
   ├─ 6. CONFIRMACIÓN
   │      ┌──────────────────────────────────────────┐
   │      │ Voy a registrar:                         │
   │      │   Camila Rojas                           │
   │      │   1 reloj automático + 1 correa          │
   │      │   Total $77.000 (neto $64.706 + IVA)     │
   │      │   Pago: transferencia · hoy 16/11        │
   │      │   Stock: automático 3→2 · correa 16→15   │
   │      │                                          │
   │      │ [ Confirmar ] [ Corregir ] [ Cancelar ]  │
   │      └──────────────────────────────────────────┘
   │
   └─ 7. "Listo. Venta #12 guardada." + [ Menú ] [ Otra venta ]
```

La confirmación no dice "¿confirmas?" y nada más: **muestra la consecuencia**. Que el stock baje de 3 a 2 antes de apretar es lo que hace que el dueño confíe.

**Datos que no se preguntan:** la fecha (hoy, y se puede cambiar al final) y el precio (el del catálogo, y se puede cambiar en el paso 2). Preguntar lo obvio es lo que hace lento un bot.

---

## 6. Cómo resuelve personas y productos

Es la única parte que parece "inteligente", y es pura búsqueda:

```
1. Normalizar   (teléfono a +56…, correo a minúsculas, nombre sin tildes)
2. Buscar       por teléfono → correo → nombre (en ese orden)
3. Decidir      exacto: usa ese
                varios: muestra hasta 6 y pregunta
                ninguno: propone crear
4. Nunca        crear sin que el dueño diga "sí, es nuevo"
```

**La persona no es un menú: es un efecto de la venta.** No existe un paso obligatorio de "crear cliente". Si el dueño vende a alguien nuevo, el sistema ofrece crearlo en ese momento. El menú "Nuevo cliente" existe aparte, solo para cuando alguien llega sin comprar (un prospecto, un contacto de la feria).

---

## 7. Arquitectura de archivos

```
bot/
├── bot.py            arranque, handlers, rutas. No tiene lógica.
├── menu.py           el árbol de menús (una sola fuente)
├── pantallas.py      todos los textos que ve el dueño
├── estados.py        la captura en curso (una estructura de datos)
├── flujos/
│   ├── venta.py      paso(estado, texto) → (estado, respuesta)
│   ├── compra.py
│   ├── gasto.py
│   ├── cliente.py
│   ├── consulta.py
│   └── ...
└── tests/
    ├── test_menu.py      todo botón tiene handler, todo handler tiene botón
    └── test_flujos.py    los asistentes, contra una base de prueba
```

**La decisión de diseño que hace todo testeable:** cada flujo es una función pura.

```python
def paso(estado, texto):
    """Recibe el estado y lo que escribió el dueño. Devuelve el estado nuevo
    y qué mostrar. No sabe que existe Telegram."""
    ...
```

El archivo `bot.py` solo traduce: mensaje de Telegram → `paso()` → respuesta de Telegram. Así los flujos se prueban sin Telegram, sin red y en milisegundos.

**Y el mismo flujo lo puede usar el agente después.** El agente no va a necesitar los pasos (él entiende la frase), pero sí la validación y la confirmación.

---

## 8. Lo que se prueba, y qué bug caza

| Prueba | Qué bug caza |
|---|---|
| Todo `callback_data` del menú tiene handler | el botón que no hace nada (el bug clásico) |
| Todo handler del menú tiene botón | código muerto que nadie puede alcanzar |
| Cada flujo: camino feliz | que la venta se guarde con los datos correctos |
| Cada flujo: cancelar en el paso N | que quede un estado colgado |
| Cada flujo: dato inválido | que el bot se caiga en vez de pedir de nuevo |
| Confirmación rechazada | que se escriba sin permiso |
| Venta con cliente nuevo | que cree dos veces a la misma persona |
| Reinicio a mitad de captura | que se pierda el estado (PicklePersistence) |

Regla heredada del bot anterior: **toda prueba tiene que fallar antes del arreglo y pasar después.** Donde no se puede escribir la prueba, no se entendió el problema.

---

## 9. Fases

| Fase | Qué | Entregable |
|:-:|---|---|
| **1** | esqueleto, menú, **venta completa**, resumen del mes | se puede vender por Telegram y ver el mes |
| **2** | gasto, compra, stock | se puede cargar el día completo |
| **3** | cliente, nota, tarea | se puede consultar y seguir a una persona |
| **4** | venta perdida, decisión, corrección, producto | el ciclo completo, incluido el "no" |
| **5** | alertas con JobQueue, ajustes, systemd | el bot vive solo y avisa |

**La fase 1 es la que valida todo.** Si vender por Telegram se siente rápido y claro, el resto es repetir el patrón. Si se siente lento, hay que cambiar el flujo antes de escribir diez más.

---

## 10. Lo que no va en esta versión

- **Fotos y voz.** Primero texto y botones.
- **Editar a mitad de la confirmación.** Se cancela y se empieza de nuevo: es más simple y más difícil de romper.
- **Varias personas escribiendo a la vez.** Un solo chat autorizado.
- **Alertas por Telegram.** Fase 5, y con la discusión de destinos que quedó pendiente.
- **Cualquier cosa con IA.** El bot no interpreta: el dueño elige.

---

## 11. La diferencia con el agente, dicha una vez

| | Bot | Agente |
|---|---|---|
| Cómo elige la operación | el dueño aprieta un botón | la infiere de la frase |
| Quién valida | el núcleo | el núcleo (el mismo) |
| Quién confirma | el dueño, en pantalla | el dueño, en el chat |
| Qué escribe | por las mismas funciones | por las mismas funciones |

**El bot es la implementación de referencia.** Cuando llegue el agente, no se programa de nuevo: se le enseña a elegir la operación que el bot ya sabe ejecutar.

---

## 12. Alcance decidido: el bot solo toca números

Decisión del fundador, y es correcta: **Telegram se usa para lo numérico**. Lo
que necesita conversación se deja para el agente.

**Entra al bot:**

| Acción | Por qué |
|---|---|
| Venta | es el hecho central del negocio |
| Compra / mercadería | sin esto el stock miente |
| Gasto | es la diferencia entre facturar y ganar |
| Cobro de una venta | una venta en cuotas no entra completa el día uno |
| Devolución | corrige sin reescribir |
| Ajuste de stock | merma, conteo, error |
| Producto y precio | el catálogo tiene que poder crecer |
| **Venta perdida** (monto + motivo) | es plata con una razón; un toque y queda |

**Queda fuera, para el agente:**

| Acción | Por qué |
|---|---|
| Próximo paso (tarea) | sin conversación se convierte en una lista que nadie mira |
| Decisión (el porqué) | una decisión a medias es peor que ninguna |
| Nota larga | es texto libre: un botón no alcanza |

**Consecuencia que hay que asumir:** si el bot no crea tareas, la alerta de
"cliente en riesgo" es solo un aviso. El que va a proponer el próximo paso es
el agente, con contexto. Mientras tanto, el aviso cumple: el dueño sabe a quién
llamar.

---

## 13. La clasificación del dinero

Son cuatro cosas distintas y van a lugares distintos:

| Qué es | Ejemplo | Dónde va | ¿Mueve la caja? | ¿Mueve la ganancia? |
|---|---|---|---|---|
| **Mercadería** | reponer 6 relojes | `egresos` categoría `mercaderia` + entrada de stock | sí | **no** (la plata cambió de forma) |
| **Inversión** | vitrina, balanza, notebook | `egresos` categoría `inversion` | sí | **no** (es un bien que dura) |
| **Gasto** | arriendo, luz, publicidad, IA | `egresos` categoría `gasto_fijo` / `marketing` / `insumos` | sí | **sí** |
| **Flete de una compra puntual** | el despacho de esos 6 relojes | se suma al **costo unitario** del producto | sí | sí, pero **dentro del margen** |

**La regla que el dueño puede recordar:**

> ¿Lo puedo volver a vender? → **mercadería**.
> ¿Me sirve para trabajar y dura años? → **inversión**.
> ¿Se consume este mes? → **gasto**.

**Por qué importa el flete:** si el flete de la mercadería se manda a gasto, el
margen del producto queda inflado. Es chico, pero se acumula y hace que el
dueño crea que gana más de lo que gana. Si es identificable, va al costo del
producto; si es general de varias compras, se prorratea o se manda a gasto, y
se dice.

**Verificación con los datos del prototipo, septiembre:**

| | |
|---|---|
| Entró a la cuenta | $110.000 |
| Salió (mercadería 165.000 + inversión 120.000 + publicidad 20.000) | $305.000 |
| **Caja del mes** | **−$195.000** |
| **Ganancia del mes** | **+$8.437** |

Compró una vitrina de $120.000 y repuso mercadería: la cuenta quedó en rojo y el
negocio ganó plata. Las dos cosas son verdad porque están en lugares distintos.

---

## 14. Las reglas de experiencia

1. **Botones siempre que la respuesta posible se conozca.** Pago, cantidad,
   sí/no, categoría, elegir entre candidatos. Escribir solo para lo que no se
   puede adivinar: un nombre, un producto, un monto.
2. **Un paso, una pregunta.** Nunca "dime el producto, la cantidad y el pago".
3. **La confirmación muestra la consecuencia**, no solo el total: el stock antes
   y después, la fecha que se va a usar, el mes que se va a mover.
4. **Cierre explícito.** Al terminar, el dueño ve "✅ guardada" **y** el total del
   mes, para saber que el dato ya está contando.
5. **Cancelar en todos los pasos**, y `/menu` siempre disponible.
6. **Si se queda a medias 15 minutos, la captura se descarta.** Es mejor
   empezar de nuevo que adivinar qué estaba haciendo.
7. **Autónomo en lo obvio:** la fecha es la del día de la conversación (en la
   zona del negocio); los botones de cantidad no ofrecen más de lo que hay; el
   catálogo pone el precio.
8. **Nunca más de seis pasos** para lo que se hace todos los días.

---

## 15. Lo que cambió respecto al plan

**No se usa `ConversationHandler`: se usa un despachador con estado explícito.**

El plan original lo proponía. Al escribirlo quedó claro que agrega una capa de
magia —estados anidados, `entry_points`, `fallbacks`— que un auditor tiene que
reconstruir mentalmente. En cambio:

- el estado es **una estructura de datos** (`estados.py`);
- el avance es **una función pura** (`flujos/venta.py`);
- el cableado es **un solo archivo** (`bot.py`).

Se conserva lo importante —el bucle, el tiempo límite, la persistencia entre
reinicios— sin la parte que no se puede leer de arriba abajo. `PicklePersistence`
sigue haciendo que una captura sobreviva a un reinicio.

---

## 16. Estado actual

| Fase | Qué | Estado |
|:-:|---|---|
| **1** | menú, venta completa, el mes, stock | **hecho** — 19 pruebas en verde |
| 2 | gasto, compra, ajuste de stock | pendiente |
| 3 | cliente, ficha, historial | pendiente |
| 4 | venta perdida, devolución, producto | pendiente |
| 5 | alertas, ajustes, systemd | pendiente |
