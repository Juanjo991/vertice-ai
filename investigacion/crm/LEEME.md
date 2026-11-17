# Lo que encontramos

**Para leer de corrido, sin decidir nada.**
Unos 25 minutos. Si tienes 10, lee los capítulos 1, 2 y 7.

---

## Antes de empezar

Esto no es un manual ni una propuesta. Es lo que aprendimos mirando cómo están construidos los CRM de verdad: Salesforce, HubSpot, Pipedrive, y varios de código abierto.

La idea es que entiendas **cómo funciona esto por dentro** antes de que tengamos que elegir nada. Cuando lo termines, vas a poder opinar sobre el diseño como si lo hubieras hecho tú.

Un aviso: casi todo lo interesante está en los detalles chicos. Los principios son simples. Lo que cuesta es respetarlos.

---

## 1. Un CRM es un cuaderno con seis secciones y una costumbre

Las seis secciones son:

1. **Quiénes son las personas** — y las empresas donde trabajan.
2. **Qué les quieres vender** — y en qué punto va cada cosa.
3. **Qué pasó con ellas** — cada llamada, mensaje, visita.
4. **Qué hay que hacer después** — con fecha.
5. **Qué plata produjo** — la venta.
6. **Quién es el responsable** — de cada cliente.

Eso es todo. Todo lo demás que ves en las páginas de esas empresas —los mil botones, los embudos de colores, los reportes— está construido encima de esas seis cosas.

Y la costumbre es esta: **el sistema solo sirve si alguien escribe lo que pasó.** Si el vendedor no anota la llamada, la sección 3 queda vacía. Si no anota el motivo del "no", la sección 2 miente. Si no anota la venta, la sección 5 no existe.

Y nadie escribe.

Todos los CRM del mundo mueren de lo mismo, y no es falta de funciones: es que el día a día se come al sistema. Por eso la industria lleva veinte años inventando formas de que el dato entre sin que nadie lo escriba: leer el correo, conectar el teléfono, poner formularios en la web, extensiones en el navegador.

Cuando entiendes eso, entiendes por qué esos sistemas tienen tantas cosas: cada función nueva es un intento de que el usuario escriba sin darse cuenta.

**Y por eso lo nuestro es distinto desde el principio:** no vamos a asumir que el usuario va a ser ordenado. Vamos a asumir que no lo va a ser, y a construir para esa persona. No es una desventaja: es la única forma de que funcione.

---

## 2. Un día en la vida de un dato

Para entender cómo se arma todo, sigamos un dato desde que nace.

**9:12.** Una señora escribe por Instagram preguntando por un reloj. El sistema no sabe nada de ella todavía. Alguien copia el mensaje a un formulario, o le contesta desde el sistema.

**9:12 y un segundo.** El sistema guarda cuatro cosas: que existe una persona llamada así, que llegó por Instagram, que le interesa un reloj, y que esto lo atiende alguien. Si esa persona ya existía —porque compró hace un año— el sistema no la duplica: la encuentra y le agrega el interés nuevo.

**9:15.** Aparece en la lista, arriba, porque es lo más nuevo. Alguien la abre al costado, sin salir de la lista, y ve que hace un año le vendió una correa.

**11:40.** Le contesta. Esa respuesta queda anotada sola —si está conectado el correo— o con un toque, si es por WhatsApp. La línea de tiempo ya tiene dos cosas: "llegó por Instagram" y "le contestamos".

**Al día siguiente.** No compró todavía. Hay una tarea: "llamar el jueves". Está en su ficha y en la lista de pendientes.

**El jueves.** La llamada no contesta. Se anota igual: "no contestó". No es un fracaso, es información. El sistema pone la próxima: "reintentar el lunes".

**El lunes.** Contesta, quiere el reloj, pide precio. Se mueve de "interesada" a "cotizada". Ese movimiento queda guardado con fecha, porque más adelante hay que poder responder "cuánto demoré en cerrar esto".

**El viernes.** Dice que sí. Se registra la venta: qué se llevó, cuánto, cómo pagó. En ese mismo acto pasan tres cosas: se cierra la intención, entra la plata al libro, y baja el stock.

**Fin de mes.** El reloj aparece en los números del mes. Y el margen que muestra es el margen real, porque el costo quedó congelado el día que se vendió.

**Seis meses después.** La señora vuelve a escribir. Alguien abre su ficha y ve todo: que llegó por Instagram, que preguntó una vez y no compró, que se le vendió al quinto intento, y que dijo que el reloj era para su hijo. Eso no lo escribió nadie: se fue guardando solo, hablando.

**Ese último párrafo es el producto.** No el formulario, no la lista, no el reporte. El hecho de que seis meses después no haya que preguntarle nada a nadie.

---

## 3. Los siete compartimentos

El CRM tiene seis. Nosotros necesitamos siete, porque el sexto del CRM —"qué plata produjo"— no dice **dónde vive** esa plata. Eso lo hace el ERP, en otro sistema, desconectado. Y ahí está el hueco.

**1. Personas y empresas.** Quién es, cómo se le habla, dónde trabaja y de qué. Es el ancla: todo lo demás la apunta. No guarda plata ni historia.

**2. Lo que se quiere vender.** La intención. Todavía no pasó, y puede morir. Guarda qué es, cuánto vale, en qué punto va, para cuándo, y de dónde llegó.

**3. Lo que pasó.** Cada llamada, mensaje, visita, reunión. Con la fecha en que ocurrió, no la fecha en que se anotó.

**4. El próximo paso.** Qué hacer, con quién, para cuándo. Una tarea sin fecha no es una tarea, es un deseo.

**5. La venta.** El hecho: qué se llevó, cuánto, cuándo, cómo pagó. Es el puente entre el mundo del cliente y el mundo de la plata.

**6. El porqué.** Qué se decidió, por qué, y qué esperabas que pasara. **Esta no existe en ningún CRM del mercado.**

**7. El libro de plata y mercadería.** La plata que entra y sale, el stock, lo que se compró, lo que se debe. Acá no escribe una persona nunca: solo el núcleo.

La regla que mantiene esto sano: **cada compartimento guarda lo suyo y apunta al resto. Nunca copia.** El teléfono vive en personas. La venta no lo copia: lo apunta. Si lo copia, algún día uno de los dos va a estar mal y nadie va a saber cuál.

---

## 4. Las siete mentiras de los números

Los sistemas de gestión mienten siempre de las mismas formas. No es mala fe: son errores de diseño que se repiten.

**Uno. Comprar mercadería no es un gasto.** Es plata que cambia de forma: estaba en la cuenta, ahora está en la vitrina. Si se anota como gasto, el mes que repones aparece con pérdida y el mes siguiente con una ganancia falsa. Ese ciclo es lo que hace que el dueño deje de confiar en sus números.

**Dos. El dólar de hoy no es el dólar de marzo.** Si el tipo de cambio se lee "al día de hoy", el reporte de marzo cambia en abril. Se congela al escribir, o no hay historia.

**Tres. La venta de las once de la noche cae al día siguiente.** Si se guarda el instante exacto y el reporte usa otra zona horaria, la venta del 15 aparece el 16. El dueño no lo nota hasta que cuadra la caja.

**Cuatro. El pasado se recalcula solo.** Si el precio del producto se guarda en un solo lugar, cambiar el precio hoy reescribe el margen de hace tres meses. El pasado no se consulta: se guarda.

**Cinco. Cambiar de responsable reescribe el mes pasado.** "Las ventas de Juan en marzo" deja de ser cierto si en abril el cliente pasa a otro vendedor. Hay que guardar de quién era en el momento.

**Sexto. Borrar un registro borra la historia.** Si se borra, el mes pasado cambia de número y nadie sabe por qué. Por eso nada se borra: se marca como inactivo, o se corrige con un asiento al revés.

**Siete. Un cruce cuenta dos veces al mismo cliente.** Cualquier reporte que junte dos tablas puede duplicar filas sin avisar. Es el error más común y el más invisible.

La consecuencia es una sola, y es fuerte: **la verdad no se puede arreglar después.** O el sistema guarda las cosas de manera que no se puedan reescribir desde el primer día, o el primer mes que un contador revise, encuentra la diferencia. Y no hay segunda oportunidad con eso.

---

## 5. Por dentro hay tres pisos

Casi todos imaginan que el sistema "calcula todo el tiempo". No es así.

**Piso 1: la operación.** Donde se escribe todo, todo el día. Es pequeño y rápido. Se usa para trabajar.

**Piso 2: los resúmenes.** Cada cierto tiempo, un proceso recorre la operación y deja totales ya masticados: cuánto se vendió por mes, por etapa, por vendedor. Los reportes leen de acá, no de la operación.

**Piso 3: las fotos del pasado.** Al cerrar cada mes se guarda una foto inmutable de cómo quedaron los números. Esa foto no se vuelve a calcular nunca.

¿Por qué tres pisos? Porque medido: preguntar "el embudo completo" sobre diez millones de registros tarda 3,6 segundos. Sobre los resúmenes, cero milisegundos. Y porque el pasado, si se recalcula, miente.

Los CRM serios hacen exactamente esto. HubSpot actualiza sus reportes cada dos horas. Salesforce congela fotos de máximo dos mil filas. Pipedrive ni siquiera guarda por qué una venta cambió de etapa. "En tiempo real" es marketing: los profesionales aceptan que un reporte es **la foto de anoche**.

---

## 6. Las cinco pantallas

El sistema se diseña desde una sola pantalla hacia atrás: **la lista**.

No se diseña para "representar la realidad" ni para "que quede ordenado". Se diseña para que la pantalla donde están todos los clientes, con los filtros puestos, responda al instante. Todo lo demás sale de ahí.

**1. La lista.** Es el producto. Columnas elegidas, filtros guardados con nombre ("los que me deben", "los que no compran hace un mes"), y todo lo que se hace, se hace desde acá.

**2. La ficha.** Se abre al costado, sin salir de la lista. Como hojear un archivador sin cerrar el cajón.

**3. La línea de tiempo.** Todo lo que pasó con esa persona, ordenado por cuándo pasó. Es la memoria de la relación.

**4. El tablero por etapa.** Las mismas tarjetas, agrupadas por el punto en que van. Sirve para ver el embudo de un vistazo. Si no muestra el total de cada columna, es decoración.

**5. Los números.** Va última. Si empiezas por acá, construyes una demo, no un sistema.

---

## 7. Lo que nadie hace

Tres huecos, y son los tres donde está nuestro lugar.

**Nadie guarda el porqué.** Salesforce guarda el historial de veinte campos y nada más. Pipedrive ni siquiera guarda por qué cambió una etapa. Todos guardan el qué. El porqué —"subí el precio porque el proveedor me subió el costo"— no está en ninguna parte. Y es lo único que evita tropezar dos veces con la misma piedra.

**Nadie trabaja el "no".** Cuando un cliente dice "no por ahora", los buenos sistemas guardan el motivo y ponen fecha para volver. Los malos lo dejan tirado en una lista que nadie mira. Es la mecánica que más plata mueve de todo el sistema, y casi nadie la hace bien.

**Nadie dice la verdad.** Los números de un negocio chico mienten por las siete razones de arriba, y el dueño lo sabe. Por eso no confía en ellos y sigue decidiendo de memoria. Un sistema que dice la verdad —y que la sostiene cuando el contador revisa— vale más que diez funciones nuevas.

---

## 8. Glosario mínimo

Las palabras que van a aparecer siempre, en simple:

**Registro.** Una fila. Una persona, una venta, una llamada. Cada cosa que se guarda es un registro.

**Campo.** Un dato dentro de un registro. El nombre, el teléfono, el monto. Como una celda de Excel.

**Duplicado.** El mismo cliente dos veces. Es el enemigo número uno, y se evita limpiando y comparando antes de guardar.

**Crudo.** El dato tal como llegó, sin arreglar. Se guarda aparte porque el proveedor lo borra en días y sin él no se puede reprocesar nada.

**Línea de tiempo.** La lista de lo que pasó con alguien, ordenada por fecha. Es lo que hace que el sistema se sienta vivo.

**Etapa.** El punto en que va una venta: interesado, cotizado, negociando, cerrado. Como las estaciones de un tren.

**Embudo.** Cuánta gente hay en cada etapa. Sirve para ver dónde se cae la gente.

**Foto.** El resumen congelado de un mes. No se recalcula.

**Respaldo.** Una copia de todo. Solo sirve si alguna vez se probó restaurarla.

**Índice.** Como el índice de un libro: permite encontrar rápido sin leer todo. Sin él, todo se vuelve lento a medida que crece.

---

## 9. Los archivos, y qué leer después

Si te quedaste con ganas, este es el orden. Los primeros son para leer; los últimos, para consultar cuando haga falta.

| Archivo | Qué es | Cómo se lee |
|---|---|---|
| **LEEME.md** (este) | la historia completa, en simple | de corrido, 25 minutos |
| `00-SINTESIS.md` | las doce decisiones que la investigación cierra | para decidir, con lápiz |
| `07-plan-de-bloques.md` | cómo se parte el trabajo en once bloques | para trabajar |
| `08-los-compartimentos.md` | el detalle de los siete compartimentos | de consulta |
| `01-modelo-de-datos.md` | cómo se guardan las cosas | de consulta, con SQL |
| `02-captura-e-ingesta.md` | cómo entra el dato | de consulta |
| `03-frontend-y-ux.md` | cómo se ve y se usa | de consulta |
| `04-backend-permisos-automatizacion.md` | la trastienda | de consulta |
| `05-recorrido-completo.md` | el día en la vida del dato, con lujo de detalle | de consulta |
| `06-reportes-busqueda-escala.md` | los números y los límites medidos | de consulta |

No hay que leer los seis informes. Están por si algún día hay que responder "¿y por qué así y no de otra forma?". La respuesta va a estar ahí, con la fuente.

---

*Cuando quieras seguimos. El Bloque 2 —el dinero y el tiempo— es el más aburrido del proyecto y el único que, si se hace mal, no tiene arreglo después. No hay apuro: los apuntes no se vencen.*
