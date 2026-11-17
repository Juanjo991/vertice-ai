# 16 — El motor: cómo se producen y dónde viven las respuestas

**Qué es:** el recorrido completo de una respuesta, desde la pregunta hasta el texto que ve el dueño. Y cómo conversan las dos bases.
**Estado:** implementado en `prototipo_relojes/` (`nucleo.py`, 25 funciones, 14 vistas, 5 triggers).

---

## 1. Las dos bases

| | `negocio.db` | `memoria.db` |
|---|---|---|
| Qué guarda | los hechos y la plata | lo que se dijo y se aprendió |
| Quién escribe | solo el núcleo | solo el asistente |
| Se puede reconstruir | **no** | sí |
| Se respalda | todos los días | cada semana |
| Qué pasa si se pierde | se pierde la verdad | se pierde el contexto |

**Por qué separadas:** porque tienen ciclos de vida distintos. La memoria crece sin control, se reindexa, se puede perder y se rehace. La plata no. Mezclarlas obliga a respaldar y auditar todo con el mismo rigor, y eso encarece lo que no lo necesita.

---

## 2. Cómo conversan

No comparten transacción ni llave foránea. Se hablan de tres maneras:

**a) Por número, con una convención.**
`memoria.episodios.persona_id` guarda el número de `negocio.personas.id`. No hay llave foránea porque son archivos distintos. La integridad se verifica con un chequeo periódico (huérfanos = episodios que apuntan a una persona que ya no existe).

**b) Por evento, en una sola dirección.**

```
negocio.db                          memoria.db
  venta registrada
  ├── INSERT venta            ┐
  ├── INSERT venta_items      │ una sola transacción
  ├── INSERT movimiento       │
  ├── INSERT pago             │
  └── INSERT evento ──────────┘
                    │
                    ▼  (procesar_eventos)
            solo lo que trae palabras:
            decisiones y actividades
                    │
                    ▼
              INSERT episodio
```

**La regla que evita el desastre:** el evento se escribe **en la misma transacción** que el hecho. Si la venta se guarda y el evento no, el aviso se pierde para siempre. Si el evento se guarda y la venta no, el consumidor lee algo que no existe. [verificado en la investigación: patrón *transactional outbox*]

**c) Por consulta, cuando se necesita cruzar.**
El asistente lee una vista de `negocio.db` y busca el contexto en `memoria.db`. El cruce ocurre **en el código**, nunca en una consulta entre bases.

**Lo que no se copia:** los hechos. La memoria no guarda "vendió $71.000". Guarda "dijo que el dorado le gustaba más". Si copiara los hechos, habría dos verdades y una se caería.

---

## 3. El recorrido de una respuesta

```
  ¿Cuánta plata tengo parada?
        │
        ▼
  interfaz (bot / agente)          ← pone las palabras
        │
        ▼
  núcleo: plata_dormida(45)        ← decide qué mirar
        │
        ▼
  vista: v_plata_dormida           ← ya sabe calcular
        │
        ▼
  {producto: "cuarzo", capital: 154000, dias: 75}
        │
        ▼
  interfaz: "Tienes $154.000 dormidos en 7 cuarzos…"
```

**Tres reglas del recorrido:**

1. **La respuesta no se guarda.** Se calcula cada vez. Guardarla obligaría a mantenerla sincronizada, y eso es exactamente lo que produce números que se contradicen.
2. **La función devuelve números, no frases.** `154000`, no `"$154.000"`. El formato es de la interfaz, porque el bot escribe distinto que el agente.
3. **La vista es el contrato.** Si mañana cambia el cálculo, cambia la vista y todos los que la usan quedan actualizados. Si el cálculo estuviera copiado en tres funciones, habría tres versiones.

---

## 4. Dónde se guardan las respuestas

Hay tres lugares, y cada uno tiene su motivo:

| Se guarda en | Qué | Cuándo | Por qué |
|---|---|---|---|
| **nada** | las respuestas comunes (caja, margen, stock) | al preguntar | se recalculan; guardarlas las desincroniza |
| **`alertas`** | lo que hay que avisar | diario | necesitan estado (enviada, resuelta): no son una consulta, son una cola |
| **`cierre_mensual`** | el mes cerrado | una vez por mes | el pasado no se recalcula |

**La alerta es el caso interesante.** No es "una consulta que devuelve cosas". Es una **cola con estado**: nace pendiente, se envía una vez, se resuelve sola cuando la condición desaparece. Sin estado, el bot repetiría el mismo aviso todos los días hasta que el dueño lo odie.

### El pasado se protege de dos maneras

**1. Un mes cerrado no acepta escrituras.** Si alguien intenta registrar una venta con fecha de agosto cuando agosto ya está cerrado, la función la rechaza:

> `El mes 2026-08 está cerrado. Registra la venta con su fecha real o usa una corrección.`

**2. Una corrección no edita: agrega.** Una devolución es un hecho nuevo, con su motivo y su fecha, que resta en el mes en que ocurre. La venta original no se toca.

Así se comporta el sistema con los datos de ejemplo:

| | |
|---|---|
| Agosto en la foto | **149.000** |
| Agosto calculado hoy | **149.000** |
| Devolución de 12.000 sobre una venta de agosto, registrada hoy | cae en **noviembre** |
| Noviembre después de la devolución | **12.000** (antes 24.000) |

El mes cerrado no se movió. La devolución quedó donde ocurrió, con su motivo, y el número de noviembre dice la verdad. **Eso es trazabilidad: cada cambio tiene fecha, dueño y razón.**

---

## 5. Las diez alertas, con su umbral

Todas son reglas. Ninguna usa un modelo.

| Código | Qué detecta | Umbral | Plata en juego |
|---|---|---|---|
| AL01 | Cliente que dejó de comprar | ciclo × 1,5 | ticket promedio |
| AL02 | Cotización sin movimiento | fecha esperada vencida | monto cotizado |
| AL03 | Plata dormida en stock | 45 días sin venta | capital inmovilizado |
| AL04 | Stock bajo | bajo el mínimo | capital a reponer |
| AL05 | Se va a acabar | menos de 15 días de cobertura | margen que queda |
| AL06 | Decisión que pide revisión | fecha de revisión vencida | — |
| AL07 | Tarea vencida | fecha pasada | — |
| AL08 | Venta perdida sin volver a contactar | fecha de reciclaje llegada | monto perdido |
| AL09 | Caída de ventas | proyección < 70% del mes anterior | diferencia |
| AL10 | Concentración de clientes | top 3 > 60% (con 6+ clientes) | ingreso concentrado |

**Los umbrales viven en `config`, no en el código.** Cambiar el ciclo de recompra de 30 a 45 días es un `UPDATE`, no un despliegue.

**Cómo funciona el motor (y por qué no molesta):**

1. Calcula todas las candidatas.
2. Inserta con `INSERT OR IGNORE` sobre una clave única (`AL01:persona:4`): no duplica.
3. **Lo que ya no aplica se marca resuelto solo.** Si Paula vuelve a comprar, la alerta se cierra sin que nadie la toque.

Resultado con los datos de ejemplo: **6 alertas**, ordenadas por severidad y plata. Ninguna inventada.

---

## 6. Los triggers: solo para invariantes

Los triggers no calculan negocio. Solo impiden que entre algo imposible.

| Trigger | Qué impide |
|---|---|
| `trg_oportunidad_etapa_ins/upd` | que un cambio de etapa no quede registrado en el historial |
| `trg_venta_inmutable` | que se reescriba una venta (hay que corregir con asiento inverso) |
| `trg_stock_no_negativo` | que un movimiento deje el stock bajo cero |
| `trg_pago_no_excede` | que los pagos superen el total de la venta |

**Por qué tan pocos:** la investigación es clara en que la automatización sin control de bucles es el fallo número uno de los sistemas con reglas. Un trigger que calcula o que dispara otro cálculo encadena y termina oscilando.

**Regla:** el trigger **impide**; la función **calcula**; la vista **responde**. Cada cosa en su lugar.

---

## 7. Qué corre y cuándo

| Momento | Qué corre |
|---|---|
| Al registrar una venta | núcleo (una transacción): venta, líneas, stock, pago, evento |
| Al llegar un mensaje | núcleo: buscar/crear persona, registrar actividad |
| Cada mañana | `generar_alertas()` + el bot envía las pendientes |
| Cada 15 min | `procesar_eventos()`: pasa a memoria lo que trae palabras |
| Fin de mes | `cerrar_mes(periodo)`: congela la foto |
| Cada domingo | respaldo de las dos bases + chequeo de huérfanos |

---

## 8. El contrato: veinticinco funciones

**Lectura (14):** `resumen_cliente`, `ficha_cliente`, `clientes_en_riesgo`, `stock_actual`, `plata_dormida`, `dias_cobertura`, `caja_del_mes`, `ganancia_del_mes`, `margen_por_producto`, `ventas_por_periodo`, `oportunidades_frias`, `decisiones_por_revisar`, `ventas_perdidas`, `alertas_pendientes`.

**Acción (11):** `buscar_persona`, `crear_persona`, `registrar_venta`, `registrar_actividad`, `crear_tarea`, `registrar_decision`, `registrar_correccion`, `generar_alertas`, `procesar_eventos`, `cerrar_mes`, `periodo_cerrado`.

Cada una tiene una descripción de una línea. **Esa línea es lo que el agente va a leer para decidir cuál usar**: el catálogo de herramientas del asistente se escribe acá, dos años antes de que exista.

---

## 9. Lo que no se automatiza

- **El texto que ve el dueño.** Se escribe en la interfaz.
- **La decisión de avisar o no.** Las alertas se generan; el envío lo decide el bot según horario y estado.
- **El cierre de un mes antes de que termine.** La foto se toma cuando el mes terminó, no antes.
- **La corrección de un dato.** El sistema propone; el dueño confirma.

---

## 10. Lo que falta para el bot

1. Un `bot.py` que use estas funciones y no sepa SQL.
2. Pantallas (textos y teclados) separadas de la lógica.
3. Un identificador de chat autorizado, en `config`.
4. Estado de la conversación en una tabla aparte, o en el propio bot.
5. Un `cron` que corra alertas cada mañana y `procesar_eventos` cada 15 minutos.

Ninguna de esas cinco cosas toca la base. Esa es la señal de que la base está bien: **lo que viene encima no la modifica.**
