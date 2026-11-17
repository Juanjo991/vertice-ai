# 13 — La memoria: el segundo producto

**Estado:** borrador de trabajo (16 de noviembre de 2026). Propuesta para discutir, no decisión cerrada.
**Qué es:** cómo Vértice guarda lo que el dueño dice, cómo lo devuelve, y por qué eso es un servicio vendible por sí solo.

---

## La tesis

> **El agente no tiene memoria. Tiene la memoria del negocio.**
> **Y la memoria del negocio es la memoria del dueño.**

Un CRM guarda lo que pasó. Un ERP guarda lo que costó. Ninguno guarda **lo que el dueño decidió y por qué**. Ese es el hueco más grande del mercado y es el más barato de llenar, porque el dueño ya lo está diciendo todo el día — solo que hoy se pierde en una conversación de WhatsApp consigo mismo.

La promesa comercial no es "tenemos inteligencia artificial". Es:

> **"No vuelvas a explicarle tu negocio a nadie. Ni a un empleado nuevo, ni a un socio, ni a ti mismo en seis meses."**

Eso se vende sin demo técnica. Se demuestra mostrando lo que el sistema recuerda de una conversación de hace tres meses.

---

## Las cuatro capas (un hecho, un dueño)

El error clásico es guardar lo mismo en dos lados: la memoria dice que subiste el precio 10% y la base dice otra cosa. Cuando se contradicen, se pierde la confianza y no hay segunda oportunidad.

**Regla:** cada cosa tiene **una sola casa**. Si aparece en otra capa, es un puntero, no una copia.

| Capa | Qué guarda | Dónde vive | Quién escribe | Se puede editar |
|---|---|---|---|---|
| **1. Hechos** | la plata, el stock, las ventas, los gastos | SQL (`negocio.db`) | solo el núcleo determinístico | **no**: solo se agrega, se corrige con un asiento contrario |
| **2. Conversación** | lo que el dueño dijo, cuándo y en qué contexto | Engram (`memoria.db`, SQLite + FTS5) | el agente, automático | sí, se corrige y se olvida |
| **3. Decisiones** | el **por qué**: cambié el precio porque… | tabla `decisiones` en SQL | el agente, con confirmación | **no**: se revierte con otra decisión, no se borra |
| **4. Doctrina** | cómo funciona este negocio: reglas, trato, ciclo | `.md` versionados en git | el agente, con confirmación explícita | sí, con diff y commit |

### Por qué las decisiones van en SQL y no solo en Engram

Porque hay una consulta que Engram no puede hacer y que es la más valiosa del producto:

```
"Subí el precio el 3 de marzo. ¿Qué pasó con las ventas después?"
```

Eso es un **join** entre una decisión y una serie de números. Engram busca texto; no cruza con la caja. Por eso la decisión tiene fila propia, y la conversación completa queda además en Engram como narrativa.

```sql
decisiones(
  id,
  fecha,
  tipo,              -- precio | proveedor | surtido | horario | regla | inversion | cliente
  titulo,            -- "Subí las pulseras de plata de 9.000 a 10.000"
  motivo,            -- el por qué, en palabras del dueño
  esperaba,          -- qué esperaba que pasara  ← esto es oro
  entidad, entidad_id,
  estado,            -- propuesta | vigente | revertida
  fecha_revision,    -- cuándo revisar si funcionó
  decision_previa_id -- linaje: a qué reemplaza
)
```

**El campo `esperaba` es el que convierte la memoria en algo que se revisa solo.** A los 30 días el radar pregunta: *"Dijiste que subir el precio iba a bajar el volumen pero subir el margen. Bajó el volumen 12% y el margen subió 4%. ¿Resultó?"* Ningún CRM hace eso, porque ningún CRM guarda la expectativa.

---

## Qué se guarda en cada capa, con ejemplos

### Capa 2 — Conversación (Engram)

Se guarda **todo lo que el dueño dice sobre su negocio**, sin que él haga nada. Con `topic_key` estable para que un tema evolucione en vez de multiplicarse:

| Ejemplo de lo que dice | `topic_key` |
|---|---|
| "Le vendí dos pulseras a la Sra. Rojas, quedó encantada" | `clientes/rojas` |
| "Estoy pensando en subir el precio de las pulseras" | `precios/pulseras-plata` |
| "El proveedor de plata tardó dos semanas" | `proveedores/plata` |
| "Los sábados no doy horas, estoy con mi hija" | `reglas/horario` |
| "Me gustaría empezar con uñas semipermanentes" | `negocio/lineas-producto` |

Lo que **no** se guarda acá: cifras (viven en SQL), RUT, teléfonos, secretos.

### Capa 4 — Doctrina (los `.md`)

Archivos cortos, en git, que el agente lee al empezar. Son la diferencia entre un asistente genérico y uno que sabe cómo funciona *este* negocio:

```
cerebro/
  01_duenio.md        -- quién es, cómo habla, qué le importa, qué no soporta
  02_negocio.md       -- qué vende, a quién, cómo cobra, qué márgenes maneja
  03_clientes.md      -- reglas de trato, segmentos, quién es quién
  04_reglas.md        -- horarios, políticas, lo que nunca se hace
  05_interfaz.md      -- cómo le gusta que le hablen y le avisen
  06_compliance.md    -- qué datos no se tocan, qué se declara
  07_evolucion.md     -- qué cambió y cuándo (bitácora, no diario)
```

**Tope de tamaño:** cada archivo bajo ~80 líneas. La memoria que no se poda se vuelve ruido, y el ruido se ignora. Si crece, se consolida (ver protocolo).

**El `.md` no guarda números.** Guarda la regla: *"el ciclo de recompra de pulseras es 45 días"*. El número de clientes en riesgo lo calcula SQL.

---

## El protocolo de auto-actualización de la doctrina

Esto es lo que pediste: el agente puede modificar el segundo cerebro, pero **protocolado**. La regla que lo ordena:

> **El agente propone, el dueño dispone. Nunca escribe la doctrina en silencio.**

### Tres tipos de entrada, tres comportamientos

| Lo que dice el dueño | Cómo se clasifica | Qué hace el agente |
|---|---|---|
| "Hoy me fue mal con una clienta" | **anécdota** | solo Engram. Nunca toca el `.md` |
| "No trabajo los domingos" | **preferencia estable** | Engram, y propone al `.md` si se repite dos veces o si dice "guárdalo" |
| "Subí todos los precios 10%" | **cambio estructural** | propone el `.md` de inmediato, con diff y confirmación |

### El procedimiento (4 pasos, siempre igual)

1. **Detecta.** El agente reconoce que lo dicho cambia algo durable, no una anécdota.
2. **Muestra el diff exacto**, en lenguaje del dueño, con el archivo y la línea:
   > *"Voy a cambiar `04_reglas.md`: de `90 días sin volver → reactivar` a `45 días`. ¿Confirmas?"*
3. **Escribe solo con un "sí" explícito.** Y en el mismo acto: fila en `decisiones` + `mem_save` en Engram + **commit de git** con mensaje legible.
4. **Avisa qué cambió**, en una línea, sin ceremonia: *"Anotado. Desde hoy el aviso de recompra sale a los 45 días."*

### Seis guardarraíles

1. **Nunca borra: anota.** `~~90 días~~ 45 días (16/11/2026)`. La historia de las reglas es parte de la memoria.
2. **Un commit por cambio**, con mensaje que el dueño entendería. Si algo se rompe, se revierte a mano.
3. **`AGENTS.md` tiene candado aparte.** Cambiar cómo se comporta el agente no es cambiar un dato del negocio: requiere confirmación distinta y explícita (*"esto me cambia cómo te comportas, ¿seguro?"*).
4. **Tope de tamaño.** Si un archivo pasa de ~80 líneas, el agente propone consolidar antes de seguir agregando.
5. **Nada de PII en la doctrina.** Nombres de clientes en la ficha SQL; en el `.md`, solo reglas de trato.
6. **Todo cambio es auditable.** El dueño puede pedir *"¿qué cambiaste este mes?"* y recibir la lista con fechas.

### Por qué esto no es peligroso (y cómo se prueba)

El riesgo real es que el agente "recuerde" algo que el dueño nunca dijo, o que se auto-modifique las reglas y empiece a comportarse raro. Se ataca con tres pruebas, que se escriben antes del código:

- **Prueba de invención:** conversación con ambigüedad → el agente debe preguntar, no completar.
- **Prueba de silencio:** pedirle que guarde una regla y responder "no" → el archivo no cambia ni un byte.
- **Prueba de reversión:** cambiar una regla, revertir el commit, verificar que el comportamiento vuelve.

---

## Engram como pieza, no como marca

Es la herramienta correcta para la capa 2: binario Go único, sin dependencias, SQLite + FTS5, MCP y HTTP, con `engram setup opencode` ya resuelto. Corre entero dentro del VPS del cliente.

Lo que hay que tener claro antes de casarse:

| A favor | A vigilar |
|---|---|
| Un binario, una base, cero infraestructura extra | **La marca es de su autor.** Licencia MIT para el código, pero no se puede llamar "Engram" al producto ni insinuar afiliación |
| `topic_key` estable: un tema evoluciona, no se duplica | Es memoria para **agentes de código**, pensada para decisiones técnicas. Nuestro caso es negocio: hay que escribir el protocolo de qué se guarda (arriba) |
| TUI para que el dueño **vea** lo que el sistema recuerda | La memoria que no se lee es un cementerio: sin recuperación en el momento correcto, no vale nada |
| Export a Obsidian → el dueño se lleva su memoria | Crece sin techo: necesita consolidación y poda, o el contexto se llena de ruido |
| Conflicto detectado entre memorias relacionadas | Hay que decidir quién gana cuando dos memorias se contradicen |

**Decisión de nombre:** la capa se llama **Memoria** o **Bitácora** en el producto. Engram es la pieza de abajo, igual que SQLite.

---

## Lo que nunca entra a la memoria

No es una lista de estilo, es la Ley 21.719 y la confianza del cliente:

- **RUT, teléfonos, direcciones.** Viven en SQL y se enmascaran antes de cualquier llamada al modelo.
- **Datos sensibles** (salud, embarazo, situación familiar, deudas personales) sin base legal explícita. "La clienta está embarazada" no se guarda.
- **Secretos, claves, tokens.**
- **Lo que el dueño pide olvidar.** Olvido real: se borra la fila, no se marca como borrada.

**Ventaja de privacidad que hay que saber vender:** la memoria es local. La conversación no viaja al modelo; viaja **el fragmento recuperado**, nada más. Un CRM en la nube manda todo.

---

## La base de datos: qué hacer con tu intuición

Dijiste: *una base más grande para las transacciones y SQLite para los clientes*. La intuición es correcta en el fondo —**separar el dinero de la relación**— pero la separación tiene que ser por **dominio y semántica**, no por archivo. Tres razones concretas:

1. **Atomicidad.** En modo WAL, una transacción de SQLite **no puede ser atómica entre dos archivos**. Registrar una venta toca dinero, stock y ficha del cliente al mismo tiempo. Partirlo en dos archivos abre la puerta a una venta guardada sin stock descontado.
2. **El tamaño no es el problema.** Un negocio chico hace ~100 ventas al día: 36.500 filas al año. SQLite maneja millones sin despeinarse. Lo que hoy está mal no es la capacidad, es el **esquema** (ver `PENDIENTES.md`).
3. **Postgres no arregla un esquema malo.** Solo lo hace más caro de administrar, y rompe dos promesas: "el cliente se lleva su base en un archivo" y "no hay que administrar un motor".

### La propuesta: un archivo, dos dominios

```
negocio.db  (un archivo por cliente, un solo servicio que escribe)
├── DINERO      append-only: ventas, venta_items, movimientos_stock,
│               gastos, pagos.  Nunca UPDATE: se corrige con asiento contrario.
└── CONTEXTO    mutable: clientes, interacciones, productos, decisiones,
                fechas_clave, notas
memoria.db   (Engram, archivo aparte: no necesita transacciones cruzadas)
```

**Por qué funciona:** el libro de dinero queda pequeño, estable y fácil de auditar y respaldar seguido. El contexto puede crecer, deteriorarse y reconstruirse sin tocar un peso. Y todo sigue siendo un solo archivo portable que el cliente puede llevarse.

### Cuándo sí migrar a Postgres (el disparador, escrito)

No cuando haya muchos datos. Cuando pase **una** de estas:

- un mismo servidor atienda a **más de un cliente** con un servicio compartido;
- exista **más de un escritor concurrente real** (varios empleados escribiendo a la vez);
- una base supere ~5 GB o el respaldo en caliente deje de caber en la ventana diaria.

Antes de eso, no. Y cuando pase, se migra el dominio `DINERO` primero: es el que exige garantías.

---

## El CRM: qué es, qué tomamos, qué no

**Qué es un CRM.** Software que gestiona las **interacciones** con el cliente: quién es, en qué etapa está, qué le dijimos, cuándo hay que volver a hablarle. Su unidad es el **contacto**; su promesa es *no perder el seguimiento*.

**Qué es Vértice.** Su unidad es la **transacción** más la **relación**. Su promesa es *saber qué hacer*.

No construiste un CRM sin saberlo. Construiste la mitad que **nadie más tiene**: la verdad de los números. Lo que falta —y ahora sabés cómo se llama— es la capa de relación, que es la parte fácil y la que hace que el sistema se sienta vivo.

### Lo que hay que tomar del CRM (y hoy no tenemos)

1. **La interacción como entidad de primera clase.** Hoy solo existe la venta. "Le mandé el catálogo", "no me contestó", "vino y no compró", "me reclamó por una uña" **no tienen dónde vivir**. Sin eso no hay gestión de relación, solo contabilidad.
2. **El estado del cliente.** Nuevo · activo · tibio · en riesgo · dormido · perdido, calculado con la **fecha del último contacto**, no solo la última compra.
3. **El próximo paso, con fecha.** Todo cliente tiene uno. Es la mecánica central del CRM y es lo que hace que el sistema trabaje solo.
4. **El historial de comunicación.** Qué le mandé, qué respondió, por qué canal.

### Lo que no hay que tomar

- **Embudo de etapas y pipeline de ventas.** Asume un vendedor con tiempo para mover tarjetas. Nuestro dueño no lo tiene.
- **Campañas y envíos masivos.** Otro producto, otro riesgo legal (consentimiento).
- **Campos obligatorios.** El CRM muere porque exige disciplina. Nosotros capturamos hablando.

### El vocabulario, ordenado

- **Transacción** = movimiento de plata. Vive en el dominio `DINERO`.
- **Interacción** = contacto con un cliente (venta, cotización, mensaje, visita, reclamo, no-venta). Vive en `CONTEXTO`.
- **Decisión** = cambio durable en cómo opera el negocio, con su por qué.
- **Memoria** = todo lo anterior, más lo que el dueño dijo, en una sola cosa consultable.

---

## Por qué esto es un servicio y no una función

Una función se incluye y nadie la valora. Un servicio se cobra porque resuelve un dolor que el cliente reconoce. Acá el dolor es concreto y tiene nombre: **volver a explicar**.

| Lo que el dueño vive hoy | Lo que vive con memoria |
|---|---|
| Explica su negocio a cada persona nueva | Le entrega la doctrina: cómo se atiende, qué se cobra, quién es quién |
| Se olvida de por qué subió un precio | Pregunta y recibe el motivo, la fecha y qué pasó después |
| Pierde la idea que tuvo manejando | La dictó y quedó guardada con fecha |
| No sabe qué prometió | El sistema le avisa antes de que quede mal |
| Si vende el negocio, se va todo con él | Se lleva la memoria: la doctrina en `.md`, los hechos en SQL, la conversación exportable |

**Cómo se cobra sin prometer magia:** el servicio no es "memoria ilimitada". Es **tres preguntas respondidas con hechos del propio negocio**, y una cuarta que ningún otro sistema puede responder: *"¿qué decidí y cómo me fue?"*.

**Lo que hay que decir en voz alta:** la memoria crece con el uso. El mes 1 recuerda poco; el mes 6 recuerda lo que el dueño ya no recuerda. Si eso no se dice al vender, el cliente espera magia el primer día.

---

## Lo que hay que construir (orden de menor a mayor riesgo)

| # | Pieza | Depende de | Riesgo |
|:-:|---|---|---|
| 1 | Engram instalado por cliente, con `engram setup opencode` y protocolo de qué se guarda | nada | bajo |
| 2 | Tabla `decisiones` + la pregunta *"¿resultó?"* a los 30 días | esquema de dinero | medio |
| 3 | Tabla `interacciones` + estado del cliente + próximo paso | esquema | medio |
| 4 | Los `.md` de doctrina por cliente, con git y el protocolo de 4 pasos | 1 | medio |
| 5 | TUI / vista web para que el dueño vea y corrija lo que se recuerda | 1 | bajo |
| 6 | Consolidación y poda automática | 1-4 | alto (es donde se rompe la confianza) |

**Lo que no se construye en la v1:** consolidación automática, memoria compartida entre clientes, búsqueda semántica con vectores (FTS5 alcanza y es determinístico).

---

## Las preguntas que abre este bloque

Están en `12-PREGUNTAS.md`, bloque C. Son las únicas del catálogo que **no se pueden responder hoy de ninguna forma** —ni con Excel, ni con un CRM, ni con ChatGPT— porque dependen de que alguien haya estado escuchando.
