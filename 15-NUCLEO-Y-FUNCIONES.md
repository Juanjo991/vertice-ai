# 15 — El núcleo: jerarquía, tipos y funciones

**Qué es:** cómo se ordenan las tablas, qué se guarda y qué se calcula, qué operaciones puede hacer el código sobre los datos, y cómo se escriben las funciones para que sirvan igual al bot de Telegram y al agente.
**Regla de la etapa:** todo lo de este documento es **determinístico**. Nada de esto usa un modelo de lenguaje.

---

## 1. La jerarquía

Las tablas no están al mismo nivel. Hay cuatro, y la flecha **siempre apunta hacia abajo**:

```
NIVEL 0 — CATÁLOGOS          lo que se elige, no lo que pasa
  config · etapas · productos

NIVEL 1 — IDENTIDADES        de quién estamos hablando
  personas · empresas · vinculos

NIVEL 2 — HECHOS             lo que pasó, con fecha
  oportunidades · actividades · tareas · ventas · venta_items
  pagos · movimientos_stock · egresos · decisiones

NIVEL 3 — DERIVADOS          lo que significan los hechos
  vistas · agregados · fotos mensuales
```

**Regla:** un hecho apunta a una identidad; una identidad apunta a un catálogo; un derivado lee hechos. **Nunca al revés.**

Un hecho nunca apunta a un derivado (una venta no guarda "el total del cliente"). Un catálogo nunca apunta a un hecho (un producto no guarda "cuántas veces se vendió"). Si eso pasa, se rompe el orden y aparecen las contradicciones.

**Y una consecuencia práctica:** el nivel 3 se puede **borrar y reconstruir entero** sin perder nada. Si un reporte está mal, se borra el derivado y se recalcula. Los hechos no se tocan.

---

## 2. Los tipos de datos

Ya están definidos, pero conviene tener la política escrita:

| Dato | Tipo | Por qué |
|---|---|---|
| Montos | entero en pesos (`INTEGER`) | nunca decimal flotante: `0.1 + 0.2` no da `0.3` y en plata eso es una demanda |
| Cantidades | entero | no existen medias pulseras |
| Costos y precios | entero, **congelados en el hecho** | el de hoy no es el de marzo |
| Fecha del negocio | texto `YYYY-MM-DD` | se ordena alfabéticamente y no se corre de zona horaria |
| Momento exacto | texto `YYYY-MM-DD HH:MM:SS` | para "cuándo se anotó", no para el negocio |
| Booleanos | entero `0/1` | SQLite no tiene booleanos |
| Estados | texto con `CHECK` | `CHECK (estado IN ('pendiente','hecha','anulada'))` |
| Texto libre | `TEXT` | con límite en la aplicación, no en la base |
| Claves | `INTEGER` autoincremental | simple, ordenado, barato de indexar |

**Lo que no se usa:** `FLOAT` para dinero (nunca), `DATE` nativo de SQLite (no existe), y `BOOLEAN` (no existe).

---

## 3. ¿Pueden entrar listas?

Sí, pero **casi nunca en una sola celda**. La regla es una pregunta:

> **¿Vas a necesitar contar, sumar o filtrar por los elementos de esa lista?**

- **Si sí:** es una tabla aparte, una fila por elemento. Una venta con tres productos no es una venta con una lista adentro: son tres filas en `venta_items`. Una persona con dos teléfonos son dos filas en una tabla de contactos.
- **Si no:** puede ser un texto con formato (JSON) en una columna, y se acepta que **no se puede consultar**. Sirve para cosas como "los intereses declarados al entrar" o "la configuración de un aviso".

**Por qué importa:** una lista dentro de una celda no se puede sumar, no se puede filtrar, no se puede indexar y no se puede validar. Es el error más común de quien viene de Excel.

**Casos concretos de este sistema:**

| Lista | Cómo se guarda |
|---|---|
| Productos de una venta | tabla `venta_items` |
| Pagos de una venta | tabla `pagos` |
| Teléfonos de una persona | tabla aparte cuando exista el segundo; hoy un campo |
| Etapas de un embudo | tabla `etapas` |
| Etiquetas de un cliente | tabla `etiquetas` + `persona_etiqueta` (cuando se necesite) |
| Preferencias sueltas | columna de texto, sin consultar |

---

## 4. Qué se guarda y qué se calcula

Tres capas, y cada cosa tiene su lugar:

| Capa | Qué es | Ejemplo | Se puede perder |
|---|---|---|---|
| **Hechos** | lo que pasó, escrito una vez | la venta del 12 de agosto por $71.000 | **no** |
| **Respuestas vivas** | lo que se calcula al preguntar | "Camila ha gastado $190.000" | sí, se recalcula |
| **Fotos** | el pasado congelado | el cierre de agosto | **no** |

**El flujo de caja no es una tabla.** Es una respuesta viva: se arma sumando cobros, restando egresos y separando mercadería. Se calcula cada vez que se pregunta. Si se guardara como tabla, habría que mantenerla sincronizada con las ventas, y ese es exactamente el error que produce números que se contradicen.

**Lo que sí se congela:** el cierre de cada mes. Porque el pasado no se puede recalcular — si en diciembre corriges una venta de agosto, el reporte de agosto cambiaría, y con él lo que ya se informó.

---

## 5. Las cuatro familias de operaciones

Todo lo que el código puede hacer con estos datos cae en una de cuatro familias. No hay una quinta.

### Familia A — Contar y sumar

Suma, conteo, promedio, mediana, mínimo, máximo.

> Camila: 5 compras, $190.000 en total, ticket promedio $38.000.
> La correa deja 60% de margen; el reloj automático, 26%.

**Cuidado con el promedio:** con pocos datos miente. La mediana es más honesta y casi nadie la usa.

### Familia B — Cortar por tiempo

Mes, trimestre, semestre, año, semana, rango arbitrario. Y **comparar** dos cortes.

> Ventas de octubre: $164.000, contra $110.000 de septiembre.
> Este noviembre lleva $24.000 y quedan 14 días.

**Cuidado:** "el mes" es una pregunta, no una columna. Y el corte siempre usa la **fecha del negocio**, nunca el momento de registro.

### Familia C — Acumular y ordenar

Acumulado (running total), ranking, primero y último, top N.

> Acumulado del año: $447.000.
> Top 3 clientes concentran el 62% de la facturación.

**Cuidado:** un acumulado sin fecha de corte no significa nada. Siempre "acumulado al 16 de noviembre".

### Familia D — Descubrir patrones

Acá está lo que no es una suma, y es lo que hace que el sistema avise en vez de esperar que le pregunten:

| Patrón | Cómo se calcula | Qué produce |
|---|---|---|
| **Cadencia** | días entre compras consecutivas | "vuelve cada 28 días" |
| **Riesgo de pérdida** | días sin comprar vs. cadencia × 1,5 | "lleva 75 días, su ciclo es 30" |
| **Plata dormida** | stock × costo, sin venta en N días | "$154.000 inmovilizados" |
| **Días de cobertura** | stock ÷ velocidad de venta | "te quedan 12 días de cuarzo" |
| **Conversión por etapa** | cuántos pasan de una etapa a la siguiente | "de 10 cotizados, 3 compran" |
| **Motivo de pérdida** | agrupación de ventas perdidas | "el 70% se pierde por precio" |
| **Concentración** | cuánto del ingreso viene del top 3 | "dependes de 2 clientes" |
| **Estacionalidad** | comparar el mismo mes entre años | "noviembre siempre es tu mejor mes" |
| **Proyección por ritmo** | vendido ÷ días transcurridos × días del mes | "vas a cerrar en $310.000" |
| **Cohorte** | clientes que entraron el mismo mes y cuántos volvieron | "de los 12 de agosto, 4 volvieron" |

**Estas diez son las que se convierten en alertas.** No necesitan IA: son reglas con umbrales. La IA aparece después, para explicar *por qué* y decidir *qué hacer*.

---

## 6. La trampa de los nombres

"Cantidad de clientes al mes" no significa nada hasta definirlo. Son tres preguntas distintas:

| Pregunta | Definición | Resultado hoy |
|---|---|---|
| ¿Cuántos clientes **nuevos**? | primera compra en el período | 1 en agosto |
| ¿Cuántos **activos**? | al menos una compra en el período | 3 en agosto |
| ¿Cuántos **compraron**? | ventas distintas en el período | 3 en agosto |

Y "total comprado por un cliente hasta hoy" **cambia todos los días**: es una respuesta, no un hecho. Si se congela, hay que decir "hasta el 31 de octubre".

**Regla:** toda métrica que se muestre tiene que tener su definición escrita en el diccionario. Si no, dos personas van a discutir números que son los dos correctos.

---

## 7. La capa de funciones: el contrato

Esto es lo que pediste: que lo mismo que hace el bot lo pueda hacer el agente.

```
   TELEGRAM            AGENTE (después)         WEB (mucho después)
   botones             lenguaje natural         formularios
       │                     │                        │
       └──────────┬──────────┴────────────┬───────────┘
                  ▼                       ▼
          ┌───────────────────────────────────────┐
          │   NÚCLEO — funciones del negocio      │
          │   registrar_venta · resumen_cliente   │
          │   clientes_en_riesgo · caja_del_mes   │
          └───────────────┬───────────────────────┘
                          ▼
                    negocio.db
```

**La regla:** ninguna interfaz habla con la base. Todas llaman a las mismas funciones. Si mañana el agente registra una venta, usa exactamente la misma función que el botón del bot.

### Cómo se escribe una función

1. **Nombre en español, en imperativo o sustantivo claro.** `registrar_venta`, `resumen_cliente`, `plata_dormida`.
2. **Entra data, sale data.** Nunca texto formateado. `45000`, no `"$45.000"`.
3. **Devuelve un diccionario con nombres explícitos**, no una tabla cruda.
4. **Valida todo lo que pueda salir mal** y devuelve un error legible, no una excepción.
5. **Es idempotente cuando se puede.** Registrar la misma venta dos veces no debe duplicar.
6. **No sabe nada de la interfaz.** No dice "presiona el botón".

### Las funciones de lectura (las que el agente va a usar)

| Función | Devuelve |
|---|---|
| `resumen_cliente(persona_id)` | compras, total, ticket promedio, cadencia, días sin comprar, estado |
| `ficha_cliente(persona_id)` | todo lo anterior + últimas 10 actividades + tareas pendientes |
| `clientes_en_riesgo()` | los que pasaron su cadencia × 1,5 |
| `stock_actual(producto_id=None)` | unidades y capital por producto |
| `plata_dormida(dias=45)` | stock sin venta, valorizado |
| `caja_del_mes(periodo)` | entró, salió, saldo |
| `ganancia_del_mes(periodo)` | vendido neto, costo, gastos, margen |
| `margen_por_producto(periodo)` | unidades, ingreso, costo, margen, % |
| `ventas_por_periodo(desde, hasta)` | agrupado por mes o por canal |
| `oportunidades_frias(dias=30)` | cotizaciones sin movimiento |
| `decisiones_por_revisar()` | las que vencieron su fecha |
| `ventas_perdidas(periodo)` | por motivo y monto |

### Las funciones de escritura (las que el bot va a usar)

| Función | Qué hace | Validaciones |
|---|---|---|
| `buscar_persona(telefono, email, nombre)` | devuelve candidatos | normaliza antes de buscar |
| `crear_persona(...)` | crea si no existe | no duplica teléfono ni correo |
| `registrar_venta(persona, items, fecha, pago)` | venta + líneas + stock + pago + evento | no vende sin ítems, no vende stock negativo sin avisar |
| `registrar_compra(items, fecha, proveedor)` | stock + egreso | costo obligatorio |
| `registrar_egreso(...)` | gasto | categoría válida |
| `registrar_actividad(...)` | línea de tiempo | fecha de ocurrencia obligatoria |
| `crear_tarea(...)` | próximo paso | fecha obligatoria |
| `cerrar_oportunidad(motivo)` | venta perdida | motivo obligatorio y reciclar_en |
| `registrar_decision(...)` | el porqué | motivo y expectativa |

**Ocho de escritura y doce de lectura: veinte funciones.** Ese es todo el sistema. Todo lo demás son pantallas.

> Implementadas hoy en `prototipo_relojes/nucleo.py`: **25** (14 de lectura, 11 de acción). Las cinco de más son las que no son ni lectura ni escritura de negocio: `generar_alertas`, `procesar_eventos`, `alertas_pendientes`, `registrar_correccion` y `periodo_cerrado`.

---

## 8. Lo que nunca entra en una función

- **Texto para el usuario.** Eso vive en la interfaz. Si el bot y el agente escriben distinto, está bien: es su voz.
- **SQL libre.** Las funciones tienen su consulta adentro y no aceptan fragmentos.
- **Lógica de presentación.** "Los últimos 10" es decisión de la interfaz; la función devuelve lo que se le pide.
- **Estado de la conversación.** Eso es de la interfaz (el bot tiene su propio estado; el agente tiene su sesión).

---

## 9. Lo que esto habilita

Con esta capa, el agente del futuro **no necesita saber SQL ni conocer las tablas**. Necesita saber que existen veinte funciones y qué devuelve cada una. Su trabajo es elegir cuál llamar y cómo contar el resultado.

Y eso es exactamente lo que queremos: **el modelo interpreta, el código calcula.** La base no se le entrega al modelo; se le entrega la lista de funciones.

---

## 10. Lo que falta decidir

1. **¿Las etiquetas de cliente existen en la v1?** (tabla `etiquetas` + `persona_etiqueta`). Sirven para "mayorista", "regalos", "vip". Yo diría que no en la primera.
2. **¿Se guarda una foto mensual desde el mes 1, o desde que haya datos?** Yo diría desde el mes 1: es barato y el primer cierre es el que más se consulta.
3. **¿El núcleo es una librería o un servicio?** Librería (importada por el bot) es más simple; servicio (HTTP) permite que el agente lo llame igual. Yo empezaría por librería y lo convertiría en servicio cuando entre el agente.
