# 08 — Implementación

Cómo se lleva un cliente desde la primera conversación hasta el sistema operando con sus propios datos.

---

## Vista general

```
R1 Diagnóstico  →  R2 Auditoría y cierre  →  [Ingesta]  →  R3 Entrega  →  Operación
     ~30 min            ~60-90 min           3-10 días       ~60 min       continua
```

**Tiempo total típico:** 7 a 14 días desde la firma hasta la entrega, según el volumen de datos históricos.

---

## Fase 0 — Calificación (antes de la primera reunión)

Se decide si vale la pena la reunión. Señales de que sí:

- Vende algo con margen identificable (producto o servicio).
- Tiene volumen: al menos 15-20 operaciones al mes.
- Maneja stock o agenda, o tiene clientes que vuelven.
- Puede sostener su infraestructura (~$30.000 CLP/mes entre servidor y IA).

Señales de que no:

- No tiene actividad comercial recurrente.
- Quiere que el sistema le genere clientes (no es lo que hacemos).
- No puede sostener el costo mensual de su servidor y su IA. **Si no puede, no es el cliente.** No se rebaja el stack para cerrar la venta.

---

## Fase 1 — R1: Diagnóstico

**Duración:** 30 minutos. **Objetivo único:** agendar R2.

**Qué se hace:**
- Conversación sobre su operación, no sobre el producto.
- Se hacen las tres preguntas de dolor (ganancia limpia, capital inmovilizado, clientes que no volvieron).
- Se anota todo lo que no sabe. Eso es el material de la siguiente reunión.

**Qué se entrega:** nada todavía. Se cierra con fecha para R2.

---

## Fase 2 — Investigación previa (entre R1 y R2)

**Duración:** 1-2 horas.

- Revisar sus redes, su catálogo, sus precios públicos, sus reseñas.
- Identificar los puntos ciegos que se pueden nombrar sin sonar invasivo.
- Preparar la demo con datos que se parezcan a su negocio.

**Regla de sutileza:** en R2 se revela el diagnóstico, no antes. La asimetría de información es lo que hace que la reunión valga.

---

## Fase 3 — R2: Auditoría y cierre

**Duración:** 60-90 minutos.

### Estructura

1. **Mapeo del flujo de trabajo** (20 min)
   - Cómo registra hoy · dónde se le pierde la información · cuál es su ciclo de recompra · cuántas personas tocan los datos.
2. **Revelación del diagnóstico** (10 min)
   - Los hallazgos de la investigación previa y los números de R1.
3. **Demo en vivo** (5 min, no más)
   - Registro de una venta real por mensaje, descuento de stock, cambio en la caja, respuesta interpretada del agente.
4. **Propuesta** (15 min)
   - Una página. Alcance, precio, fechas, qué incluye y qué no.
5. **Cierre** (20 min)
   - Contrato y pago.

### La propuesta de una página

Debe caber en una hoja y contener solo esto:

- Qué problema resuelve, en una frase.
- Qué incluye exactamente.
- **Qué no incluye** (tan importante como lo anterior).
- Precio de implementación.
- Fecha de entrega.
- Qué se espera del cliente: el material histórico.

**Nada de 20 páginas.** Una propuesta larga es una propuesta que no se lee.

### El cierre

- **A:** 100% de la implementación al firmar.
- **B:** 50% al firmar **+ 3 meses de mensualidad prepagados**.

**Nunca se trabaja sin pago.** El trabajo es el costo.

---

## Fase 4 — Ingesta

**Duración:** 3-10 días según volumen. **Aquí se gana o se pierde el proyecto.**

### Qué se le pide al cliente

Una sola instrucción: *"mándame todo lo que tengas de tu negocio"*. Planillas, exportaciones, listas de precios, cuadernos fotografiados, facturas.

### Qué se hace

1. **Clasificar** el material recibido por tipo y calidad.
2. **Normalizar** a la estructura de tablas del sistema.
3. **Detectar huecos** y hacer una lista corta de preguntas concretas (no un cuestionario).
4. **Cargar** en `transactions`, `products`, `clients`, `providers`.
5. **Verificar** totales contra el material original: si el cliente decía vender $2.000.000 al mes, la base tiene que reflejarlo.
6. **Parametrizar el negocio**: ciclo de recompra, categorías, motivos de pérdida típicos del rubro, umbrales de stock.

**Regla de oro de la ingesta:** nunca se entrega un sistema con datos que el cliente no reconoce. Si los totales no cuadran, se pregunta antes de la entrega.

### El filtro

Si el cliente no entrega material, **no hay entrega**. Esto se dice en R2 y se sostiene. Un sistema vacío no demuestra nada y el cliente concluye que no funciona.

---

## Fase 5 — Despliegue técnico

**Checklist:**

- [ ] VPS del cliente creado y accesible
- [ ] Usuario sin privilegios de root para la aplicación
- [ ] Python y dependencias instaladas
- [ ] Repositorio clonado, en la versión etiquetada
- [ ] `.env` con credenciales, permisos `600`
- [ ] Base de datos inicializada con el esquema vigente
- [ ] Migraciones aplicadas
- [ ] Bot de Telegram configurado y respondiendo
- [ ] Credenciales de IA del cliente cargadas y probadas
- [ ] Herramientas determinísticas probadas con casos reales
- [ ] Seudonimización activa y verificada
- [ ] Respaldo automático configurado **y restauración probada**
- [ ] Registro de accesos activo
- [ ] Datos históricos cargados y verificados contra el original

---

## Fase 6 — R3: Entrega

**Duración:** 60 minutos.

### Estructura

1. **El sistema vivo**, con sus datos. No una demo: su negocio.
2. **Una decisión real tomada con datos.** Mostrarle algo que no sabía y que puede actuar hoy.
3. **Educación básica** (15 min): cómo registrar, cómo preguntar, qué esperar y qué no.
4. **Entrega de la biblioteca de formación** y del canal de soporte.
5. **Segunda venta:** Línea Directa (A) o la siguiente capacidad (B).

**Lo que se entrega por escrito:**
- Cómo se usa lo esencial (una página).
- Qué datos tiene el sistema y dónde viven.
- Canal de soporte y qué esperar de él.
- Qué está incluido y qué no.

---

## Fase 7 — Primeros 30 días

Los más frágiles. Plan de acompañamiento:

| Cuándo | Qué se hace |
|---|---|
| Día 3 | Verificar que registró al menos 5 operaciones |
| Día 7 | Revisar dudas y corregir vicios de uso |
| Día 15 | Primer cierre quincenal: mostrarle lo que el sistema ya ve de su negocio |
| Día 30 | Cierre de mes + evaluación de adopción |

**Umbral de éxito:** 20 ventas registradas en el primer mes. Si no se llega, el problema es de adopción, y se atiende como tal —no vendiéndole más cosas.

---

## Tiempos y capacidad

| Actividad | Tiempo |
|---|---|
| R1 + R2 + R3 | ~3 horas |
| Investigación previa | 1-2 horas |
| Ingesta | 4-16 horas según volumen |
| Despliegue técnico | 2-6 horas |
| Acompañamiento mes 1 | 2-4 horas |

**Tope operativo para el Arquetipo A: un día de trabajo por cliente**, todo incluido. Si el alcance no cabe en un día, se reduce el alcance — no se extiende el plazo ni se baja el precio.

**Clientes A en cohortes de 4-6.** Vender de a uno, intercalado con reuniones de Pyme, es la forma más rápida de no llegar a fin de mes.

---

## Errores que hay que evitar

| Error | Por qué duele |
|---|---|
| Entregar sin datos cargados | El cliente no ve valor y abandona en dos semanas |
| Vender antes de la ingesta | Se firma con expectativas que la realidad no cumple |
| Prometer disponibilidad | El servidor es del cliente; no se controla |
| Cargar datos sin verificar totales | El cliente detecta el error y pierde la confianza en todo |
| No pedir el material histórico | Se descubre en R3 y ya es tarde |
| Expandir a un cliente que no adopta | Cobra por algo que no usa y lo cuenta |
