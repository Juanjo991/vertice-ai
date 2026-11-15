# 03 — Arquitectura

## Vista general

```
[ENTRADAS]
  Botones rápidos (Telegram)  ·  Lenguaje natural (agente)  ·  Ingesta de histórico
  ·  Documentos (posterior)   ·  Pagos automáticos (posterior)
        │
        ▼
[PROCESAMIENTO DETERMINÍSTICO — Python]
  Valida · normaliza · calcula · enruta
  Seudonimiza datos personales antes de cualquier llamada externa
        │
        ▼
[ALMACENAMIENTO DUAL]
  ├─ CAPA A: SQLite relacional  → verdad de los datos duros
  └─ CAPA B: Memoria + protocolos → contexto cualitativo del negocio
        │
        ▼
[INTELIGENCIA]
  ├─ Consultas determinísticas (caja, margen, stock, plazos) → costo 0 tokens
  └─ Agente de IA (interpreta, decide, redacta) → solo cuando aporta
        │
        ▼
[SALIDAS]
  Respuesta en el chat · Notificación · Informe · Alerta de patrón
```

---

## Las dos capas de almacenamiento

Esta separación es una de las decisiones más importantes del diseño, porque resuelve un error común: **pedirle a una memoria de texto que recuerde datos que deben ser exactos.**

### Capa A — SQLite relacional (fuente única de verdad)

Todo dato duro, exacto y auditable vive acá:

| Tabla | Qué guarda |
|---|---|
| `transactions` | Ingresos, costos, gastos e inversiones: monto, fecha, categoría, canal de origen, duración |
| `products` | Productos o servicios: nombre, costo unitario, precio, stock, margen calculado |
| `clients` | Fichas: identificador, nombre, contacto, fecha de primera y última compra, canal |
| `providers` | Proveedores: costos históricos, plazos de reposición, calidad |
| `lost_sales` | Ventas no cerradas: producto buscado, motivo, fecha |
| `fechas_clave` | Hitos futuros: cliente, tipo, fecha, días de anticipación, estado |
| `appointments` | Citas y servicios con agenda (cuando aplique) |

**Regla:** toda cifra que el cliente vaya a usar para decidir se calcula desde acá. Nunca se le pide al modelo que sume.

### Capa B — El "segundo cerebro" del negocio

El conocimiento que no cabe en una tabla:

- **Archivos de protocolo** (`.md`) que describen cómo funciona ese negocio: qué vende, cómo atiende, qué considera una venta perdida, cuál es su ciclo natural de recompra.
- **Memoria conversacional** con búsqueda de texto completo: lo que el dueño contó en una conversación y vale la pena recordar (*"este cliente prefiere que lo contacten por WhatsApp"*, *"las piezas de colección las compra un cliente que viaja en diciembre"*).

**Regla:** la memoria cualitativa **no dispara eventos**. No es la que recuerda avisar en una fecha. Eso vive en `fechas_clave`.

---

## El radar

Un proceso que corre a diario y revisa la base buscando patrones:

- clientes inactivos según el ciclo propio del rubro;
- hitos de `fechas_clave` próximos a vencer;
- productos con stock en cero que tienen demanda registrada;
- ventas perdidas por falta de stock que ahora sí hay;
- postventas pendientes (garantías, mantenciones).

**Dos salidas distintas, a propósito:**

| Tipo de hallazgo | Qué hace el sistema | Costo |
|---|---|---|
| Aviso operativo simple (*"mañana vence la garantía de X"*) | Python notifica directo, sin modelo | 0 tokens |
| Oportunidad comercial (*"12 clientes dejaron de comprar"*) | Despierta al agente, que consulta el contexto y redacta la recomendación | Tokens, solo acá |

---

## El agente y su arnés

El agente no tiene acceso libre a la base. Interactúa a través de **herramientas cerradas**: funciones determinísticas, ya probadas, que reciben parámetros y devuelven resultados.

Esto significa que el modelo:

- **no escribe SQL libre** contra la base del cliente;
- **no hace aritmética**;
- **no inventa** qué pasó: recibe el resultado ya calculado y lo interpreta.

El *arnés* son los protocolos en lenguaje natural que le dicen cómo comportarse:

- cómo hablarle al dueño de ese negocio (tono, vocabulario del rubro);
- qué preguntar cuando falta un dato;
- qué considerar urgente y qué no;
- qué nunca debe hacer (dar consejos tributarios, inventar cifras, prometer plazos).

**Por qué funciona con cualquier modelo:** como el agente no tiene que construir nada ni razonar sobre aritmética, un modelo mediano se comporta bien. Eso le da al cliente libertad de elegir proveedor y mantiene el costo bajo control.

---

## Flujo completo: una venta

```
1. El dueño escribe: "le vendí 2 pulseras a la Sra. Rojas, $45.000"
2. El agente identifica la intención y extrae: producto, cantidad, monto, cliente
   └─ Si falta algo, pregunta. No adivina.
3. Llama a la herramienta determinística `registrar_venta(...)`
4. Python: valida · inserta en `transactions` · descuenta stock en `products`
   · actualiza última compra en `clients`
5. Python devuelve el estado nuevo: stock restante, caja del día, margen
6. El agente interpreta ese estado y responde en una frase útil
   "Registrado. Te quedan 2 pulseras de plata y esta semana vas 18% arriba."
```

**Lo importante:** el paso 6 solo puede decir lo que el paso 5 le entregó. No hay margen para inventar.

---

## Flujo completo: un hallazgo del radar

```
1. El radar detecta: 12 clientes sin comprar hace más de 2× su ciclo
2. Python construye el hallazgo con identificadores seudonimizados (CLT_014, CLT_032…)
3. Despierta al agente con el hallazgo y el contexto del negocio
4. El agente prioriza: quién compró más, quién compró hace más tiempo, quién tiene
   una preferencia registrada
5. Redacta tres mensajes concretos con nombre real (reconstruido localmente)
6. El dueño recibe: "Estos tres primero. Al CLT_014 le vendiste 5 piezas en marzo
   y le gustan las de colección: hay una nueva que le calza."
```

---

## Decisiones de diseño y su razón

| Decisión | Por qué |
|---|---|
| **SQLite y no PostgreSQL** | Un archivo por cliente. Aislamiento total, respaldo simple, cero administración de servidor de base de datos, funciona en un VPS pequeño |
| **Un archivo `.db` por cliente** | Si algo se compromete, se compromete un cliente, no la cartera entera. Además simplifica el borrado legal |
| **Python determinístico para la aritmética** | Cero alucinación en dinero. Es la única forma de que el dueño confíe en la cifra |
| **Instalación en servidor del cliente** | El cliente es dueño del sistema y los datos. Elimina el costo de infraestructura para nosotros y elimina su miedo a "perder el acceso" |
| **Herramientas cerradas en vez de SQL libre** | El modelo no puede romper la base ni inventar consultas |
| **Telegram como interfaz inicial** | Cero desarrollo de interfaz, funciona en el teléfono que el dueño ya usa, notificaciones nativas. La web llega después |
| **Protocolos en `.md`** | El comportamiento del agente es editable y auditable sin tocar código |

---

## Límites conocidos

Ser honesto sobre esto evita promesas que después se rompen:

- **Depende de que el cliente registre.** Un sistema que no se alimenta no interpreta nada. La adopción es el riesgo principal del producto.
- **La calidad de la interpretación depende de la calidad de los datos.** Si el cliente no carga costos, el margen no existe.
- **Un solo operador en la v1.** Sin roles ni permisos.
- **Sin integraciones automáticas en la v1.** Pagos y documentos son manuales.
- **El servidor es del cliente.** Si su VPS se cae, el sistema se cae. No hay promesa de disponibilidad.
