# 01 — El problema

## En una frase

El dueño de un negocio pequeño **trabaja con información que ya tiene pero no puede leer**. Anota, cobra, compra y vende todos los días; lo que no puede hacer es convertir esos registros sueltos en una decisión.

No le falta esfuerzo. Le falta interpretación.

---

## Dos arquetipos, dos problemas distintos

Vértice atiende dos tipos de cliente. **No son el mismo problema en distinto tamaño**: son problemas diferentes, y confundirlos es el error más común al diseñar software de gestión.

### Arquetipo A — El autoempleado

> Persona que trabaja por cuenta propia y **opera sola de punta a punta**: vende, atiende, cobra, compra el insumo y anota. No tiene empleados ni procesos formales.

**Su control vive en tres lugares:** la cabeza, un cuaderno, o un Excel. A veces los tres a la vez, y ninguno coincide con los otros dos.

**Lo que le pasa concretamente:**

- Sabe cuánto vendió, pero no cuánto **ganó**. Confunde facturar con ganar.
- No sabe cuánto capital tiene inmovilizado en stock. Compra por intuición.
- Atiende al cliente que le escribe. Al que no le escribió, lo olvida — y ese cliente se va sin que nadie lo note.
- Cuando no vende, no sabe por qué: si fue precio, si no había stock, si el cliente "lo iba a pensar".
- Si le preguntan cuánto ganó el mes pasado, abre el Excel, suma, y aun así duda.

**Por qué no lo resuelve solo:** automatizar un Excel requiere una habilidad que no tiene y que no tiene por qué tener. Su oficio es otro. Y las planillas que hereda de un conocido o de YouTube no están pensadas para su negocio, así que las abandona en dos semanas.

> **Frase canónica:** *"No tiene datos; tiene registros sueltos y ninguna interpretación."*

### Arquetipo B — La Pyme con empleados

> Negocio con personal a cargo, volumen diario de transacciones y capacidad de pago instalada. Ya pasó la etapa del cuaderno.

**Aquí el problema cambia de naturaleza.** Ya no es desorden de una persona: es **descoordinación entre varias**. Uno anota las ventas, otro compra el stock, el contador recibe las boletas, el dueño mira el saldo del banco. Cada uno tiene un pedazo, y los pedazos no se hablan.

**Los tres sub-casos** (esto define qué se le vende):

| Sub-caso | Qué tiene hoy | Qué necesita |
|---|---|---|
| **B0 — Desnudo** | Nada. Todo en papel o en la cabeza del dueño | Implementación completa. El mejor caso: no hay nada que migrar ni desinstalar |
| **B1 — Excel compartido** | Planillas que se pisan entre personas | Una fuente única de verdad |
| **B2 — ERP subutilizado** | Paga licencia y usa 6 de 30 módulos | La capa de interpretación que su ERP no tiene |

> **Frase canónica:** *"Tiene los datos, pero repartidos entre personas y sistemas que no se hablan. Paga por registrar y sigue decidiendo a ciegas."*

---

## La brecha de escala

Este es el argumento central del producto, y hay que decirlo con precisión porque es fácil decirlo mal:

> **El problema no es que la Pyme sea chica. Es que el software a medida es caro.**

La Pyme queda atrapada en el medio:

- Es **demasiado grande para el Excel**: el volumen rompe la planilla, varias personas la pisan, los datos se contradicen.
- Es **demasiado pequeña para el software a medida que existía hasta ahora**: implementaciones de millones de pesos y semestres de desarrollo, con un equipo dedicado.

Ese precio la deja fuera del mercado. **Su tamaño no.**

### Y entonces, ¿Vértice no es a medida?

Sí lo es. Y esa es justamente la ruptura:

> **Vértice es a medida — a precio de genérico.** A medida en el resultado: su base de datos, sus parámetros, su contexto, sus módulos. Estandarizado en la construcción: una librería reutilizable de módulos más desarrollo asistido por IA.

Por eso cuesta cientos de miles y no millones, y por eso se entrega en semanas y no en un semestre.

**La ruptura con el mercado no es funcional, es económica.** Vértice no compite por tener más funciones que un ERP. Compite por **acceso**.

---

## Los cuatro dolores que el cliente reconoce

Estos son los que se pueden decir en voz alta en una reunión, porque el dueño los ha pensado antes:

1. **Ceguera de caja.** Llega a fin de mes sin saber con exactitud cuánto dinero limpio ganó.
2. **Capital dormido.** Tiene plata parada en stock que no rota y no lo sabe.
3. **Venta silenciosa perdida.** El cliente que no volvió, el que preguntó por algo que no había, el que "lo iba a pensar". Nadie los persigue porque nadie los registra.
4. **Trabajo administrativo ingrato.** Horas semanales en tareas repetitivas que no producen nada y que se podrían automatizar.

---

## Por qué fallan las alternativas

| Alternativa | Por qué falla en la Mipyme |
|---|---|
| **Cuaderno / Excel / Google Sheets** | Guarda números mudos, exige carga manual que se abandona, no hace seguimiento de nadie. Es estático, manual y no habla: no te dice qué está pasando |
| **Software contable / ERP** | Diseñado para el contador. Registra el pasado de forma pasiva y pierde todo el contexto humano de la venta. Se cobra como acceso mensual perpetuo y entrega 30 módulos donde se usan 6 |
| **CRM / agenda** | Silos cerrados, desconectados de los costos reales, de las facturas de proveedores y del margen en caja |
| **Automatizaciones sueltas (n8n, Make) o bots simples** | Frágiles al escalar, sin memoria contextual de largo plazo, sin arquitectura mantenible, con riesgo legal sobre los datos |
| **Desarrollo a medida tradicional** | Funciona, pero cuesta millones y toma meses. Es la razón por la que la Pyme no lo tiene |

**Lo que ninguna de estas hace:** decirle al dueño **qué hacer** con lo que ya sabe.

---

## Lo que este problema NO es

Para evitar vender lo que no somos:

- **No es un problema de contabilidad.** No reemplazamos al contador ni hacemos libros. Hacemos gestión comercial.
- **No es un problema fiscal.** El pre-cálculo de IVA es un subproducto útil, no el producto.
- **No es un problema de marketing.** Vértice no hace crecer el tráfico; hace que el negocio existente deje de perder plata.
- **No es un problema de digitalización.** El cliente no necesita "estar en digital". Necesita saber qué está pasando en su negocio.

---

## Cómo se comprueba que el problema existe

Tres preguntas que cualquier dueño responde sin dudar y que demuestran el dolor en 60 segundos:

1. *"¿Cuánto ganaste limpio el mes pasado, sin contar lo que vas a gastar?"*
2. *"¿Cuánta plata tienes parada en stock que no se ha movido en dos meses?"*
3. *"¿Cuántos clientes te compraron una vez y no volvieron?"*

Si las tres respuestas son "no sé" o requieren abrir un Excel, el problema está instalado. **No hay que convencerlo: hay que mostrarle el número que no sabía.**
