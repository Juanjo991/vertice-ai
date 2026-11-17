# 14 — Arquitectura de datos

**Qué es:** las reglas que hacen que la base aguante años, clientes y una capa de IA encima. No son preferencias de estilo: cada una previene un error concreto que después no se puede corregir.
**Estado:** vigente como criterio. Se aplica sobre el prototipo de `prototipo_relojes/`.

---

## La distinción que ordena todo

**Compartimento** = la idea, la función, la promesa. Se decide una vez y no cambia.
**Tabla** = el mueble, el almacenamiento. Cambia cuando la máquina lo necesita.

Un compartimento puede tener varias tablas. Una tabla puede no ser un compartimento (`config`). Confundirlos hace que cada optimización rompa el producto.

---

## Las diez reglas

### 1. Un hecho, un dueño

Cada dato vive en una sola tabla. Lo demás lo apunta por su id.

**Previene:** dos verdades que se contradicen (el teléfono en la persona y en cada venta; el stock como columna y como movimientos).

**Excepción legítima:** las fotos. El costo y el precio se copian en la línea de la venta **a propósito**, porque el de hoy no es el de marzo. Una foto no es un duplicado: es una decisión.

### 2. El dinero no se reescribe

Las tablas de plata solo crecen. Un error se corrige con un asiento al revés, con fecha y motivo.

**Previene:** que el reporte de marzo cambie en abril y nadie sepa por qué.

### 3. La fecha del negocio y el instante son dos cosas

`fecha` = el día en que ocurrió para el negocio. `ocurrio_en` = el instante exacto. `registrada_en` = cuándo se anotó.

**Previene:** la venta de las 22:30 que cae al día siguiente, y la línea de tiempo que miente.

### 4. Lo calculado no se guarda como verdad

El stock es la suma de movimientos. El total del cliente es la suma de sus ventas. Si por velocidad se guarda una copia, se marca como copia y se recalcula.

**Previene:** el número que se desactualiza en silencio y contradice a su fuente.

### 5. Nada se borra

`eliminado_en` para lo que deja de existir, asiento inverso para la plata.

**Previene:** perder historia y no poder explicar un cambio.

### 6. La historia de etapa, desde el día uno

Cada cambio de etapa es una fila con fecha de entrada y salida.

**Previene:** no poder responder nunca "cuánto demoré en cerrar". Es el único dato que no se puede reconstruir después.

### 7. Los eventos salen en la misma transacción que el hecho

Cuando se registra una venta, se guarda también el aviso en una bandeja de salida, en la misma operación.

**Previene:** que el asistente o el notificador se enteren tarde, o nunca. Si el proceso muere entre el hecho y el aviso, el aviso se pierde para siempre.

### 8. La IA no toca las tablas de dinero

El asistente **lee vistas ya calculadas** y **escribe solo en su propia memoria** (`decisiones`, `actividades`, `memoria.db`). Nunca en `ventas`, `pagos` ni `movimientos_stock`.

**Previene:** que el modelo invente una cifra o corrompa la caja. La aritmética es del código; el modelo interpreta.

### 9. El esquema se versiona y se migra

Cada cliente tendrá su copia en su servidor. Si el esquema cambia, hay un archivo de migración numerado y una tabla que dice en qué versión va cada base.

**Previene:** islas incompatibles. Actualizar a veinte clientes sin esto es imposible.

### 10. Los nombres se explican solos

`movimientos_stock`, no `ms`. `fecha_revision`, no `fr`. Y un diccionario que diga, en una línea, qué significa cada campo.

**Previene:** que el asistente —y el que programe en dos años— tenga que adivinar. El esquema es la interfaz de la IA: si no se entiende, no se puede usar.

---

## Las tres zonas

```
┌─────────────────────────────────────────────┐
│  negocio.db        LA VERDAD                │
│  personas, ventas, stock, egresos            │
│  Escribe: solo el núcleo. Nunca el modelo.   │
└──────────────┬──────────────────────────────┘
               │ vistas ya calculadas
               │ (v_cliente_resumen, v_stock, …)
┌──────────────▼──────────────────────────────┐
│  VISTAS            EL CONTRATO              │
│  respuestas listas, con nombre y explicación │
│  Es lo único que el asistente consulta.      │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│  memoria.db        LO QUE SE DIJO           │
│  conversación, decisiones, doctrina          │
│  Escribe: el asistente. Apunta a negocio.db. │
└─────────────────────────────────────────────┘
```

**Por qué las vistas y no las tablas:** porque el asistente no debe escribir consultas. Le das respuestas con nombre —`v_cliente_resumen`, `v_plata_dormida`, `v_decisiones_vencidas`— y él solo elige cuál mirar. Es la diferencia entre un empleado que sabe leer un informe y uno que tiene la llave de la bóveda.

**Y por qué la memoria va aparte:** porque crece sin control, se reindexa, se puede perder y se reconstruye. La plata, no.

---

## La frontera de escritura

| Escribe | Dónde | Cómo se impone hoy | Cómo se impondrá |
|---|---|---|---|
| Núcleo determinístico | `negocio.db` | un solo servicio escribe | rol de base de datos |
| Botones / formularios | `negocio.db` vía núcleo | la API del núcleo | igual |
| Asistente | `memoria.db` + `decisiones` + `actividades` | regla de código | permisos por tabla |
| Nadie | `ventas`, `pagos`, `movimientos_stock` | regla de código | permisos por tabla |

Hoy la frontera está en el código. Cuando migremos a un motor con usuarios (Postgres), pasa a ser **un permiso real de la base**. Esa es la razón técnica más fuerte para migrar algún día: no el tamaño, la separación de poderes.

---

## Lo que hay que agregar al prototipo (barato hoy, caro después)

| # | Qué | Por qué ahora |
|:-:|---|---|
| 1 | Tabla `_migraciones` + archivos numerados | sin esto no se puede actualizar a un cliente sin recrear su base |
| 2 | Tabla `eventos` (bandeja de salida) | es la que permite que el asistente reaccione sin revisar todo cada vez |
| 3 | Vistas de respuesta (`v_*`) | es el contrato con la IA; definirlas ahora fija qué se puede preguntar |
| 4 | `eliminado_en` en las tablas principales | el borrado real llega tarde y sin historia |
| 5 | `creado_por` / `actualizado_en` | sin esto no hay auditoría posible, y la auditoría no se puede inventar después |

---

## Lo que NO vamos a hacer

- **No replicar el modelo de Salesforce.** Su complejidad viene de vender a empresas de diez mil empleados, no de la idea.
- **No poner `tenant_id` en cada tabla.** Un archivo por cliente: el archivo **es** el inquilino. Repetir el campo en 15 tablas sería complejidad sin beneficio.
- **No guardar el resultado de cada consulta.** Se guarda la foto mensual, que es la que no se puede recalcular.
- **No dejar que la IA escriba SQL libre.** Consulta vistas; si necesita algo nuevo, se agrega una vista.
- **No auditar todo.** Cinco o seis campos con historia; el resto, no. Guardar todo cuesta ocho veces más y nadie lo lee.
