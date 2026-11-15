# 09 — Roadmap

## Cómo se ordena el trabajo

No se planifica por módulos sueltos, sino por **versiones que un cliente puede usar**. Cada versión debe dejar a alguien operando, no a medias.

La regla que ordena todo: **primero lo que el cliente usa todos los días, después lo que le gusta ver, al final lo que automatiza.**

Razón: un sistema que se usa produce datos, y sin datos la inteligencia no existe. La inteligencia comercial es lo último en la fila porque **depende de que haya algo que interpretar**.

---

## Versión 1 — El núcleo completo

**Objetivo:** que un cliente pueda operar de punta a punta sin Excel.

| Capacidad | Estado |
|---|---|
| Registro de ventas, compras, costos e inversiones | A construir |
| Stock automático con descuento y reposición | A construir |
| Flujo de caja con proyección de cierre | A construir |
| Margen unitario por producto | A construir |
| Fichas de clientes | A construir |
| Agente conversacional con herramientas cerradas | A construir |
| Captura por botones en Telegram | A construir |
| Notificaciones | A construir |
| Segundo cerebro (protocolos + memoria) | A construir |
| Agente de auto-ayuda y bitácora | A construir |
| Ingesta de histórico | A construir |
| Informe del período | A construir |

**Criterio de salida:** un cliente real registra 20 ventas en un mes sin ayuda.

---

## Versión 1.1 — Confianza operativa

**Objetivo:** que el sistema aguante el uso real sin supervisión.

| Capacidad | Por qué |
|---|---|
| Respaldos automáticos con restauración probada | Sin esto, cualquier falla es pérdida total |
| Migraciones versionadas con reverso | Permite actualizar clientes sin miedo |
| Registro de errores y auditoría de accesos | Sin esto no se puede diagnosticar nada |
| Corrección de registros desde el chat | El error de tipeo es la consulta número uno |
| Cierre de mes generado automáticamente | Es el entregable de la Línea Directa |

**Criterio de salida:** dos clientes operando 60 días sin intervención manual.

---

## Versión 1.2 — Segunda venta

**Objetivo:** activar las capacidades que hacen que el cliente quiera pagar más.

| Capacidad | A quién se le vende |
|---|---|
| Radar de inactividad parametrizado por rubro | A (es su mayor dolor) |
| Fechas clave y postventa | A |
| Motivo de pérdida y conversión por canal | B |
| Match de demanda insatisfecha vs. stock | A |
| Comparador de proveedores | B |
| Pre-cálculo de IVA | B |

**Criterio de salida:** tres clientes con una segunda compra cerrada.

---

## Versión 2 — Interfaz web

**Objetivo:** que el negocio se vea en una pantalla y que más de una persona pueda usarlo.

| Capacidad | Por qué ahora y no antes |
|---|---|
| Dashboard de caja, margen y stock | Recién hay datos que valga la pena mostrar |
| Consola del agente en la web | Recién hay volumen de conversación |
| Multiusuario con roles | Aparece cuando el cliente contrata gente |
| Reportes visuales y exportación | Para compartir con el contador o un socio |

**Criterio de salida:** un cliente con dos usuarios activos.

**Nota técnica:** el multiusuario no es un interruptor. SQLite con un solo escritor obliga a rediseñar la capa de acceso. Es la deuda técnica más grande del proyecto y hay que tomarla en serio.

---

## Versión 2.x — Captura automática

**Objetivo:** sacarle al cliente el trabajo de digitar.

| Capacidad | Dependencia |
|---|---|
| Lectura de facturas y boletas en PDF | Librería de extracción |
| Foto de boleta física (OCR / visión) | API de visión |
| Conciliación de pagos por pasarela | Webhooks del proveedor |

**Criterio de entrada:** al menos cinco clientes pidiéndolo, no uno.

---

## Versión 3 — Ecosistema extendido

**Objetivo:** conectar el negocio con su entorno.

| Capacidad | Riesgo |
|---|---|
| Conexión con correo electrónico | Bajo |
| Atribución de redes sociales | Alto: depende de APIs de terceros y de permisos |
| Publicación automática | Alto: es otro negocio, el de contenido |
| Gestión de campañas | Alto: consume horas y exige presupuesto de medios |

**Regla:** estas capacidades **no se prometen a ningún cliente** hasta que estén construidas y probadas. Existen como dirección de diseño, no como compromiso.

---

## Cómo se decide qué construir después

Tres filtros, en orden:

1. **¿Lo está pidiendo más de un cliente?** Una petición es un caso particular. Tres son un patrón.
2. **¿Desbloquea una segunda compra?** Si no genera ingreso nuevo ni reduce soporte, no es prioridad.
3. **¿Se puede entregar en menos de dos semanas?** Si no, se parte por la mitad y se entrega algo usable.

**Lo que nunca se hace:** construir por entusiasmo técnico. El proyecto ya tiene suficiente superficie; cada módulo nuevo es soporte futuro.

---

## Lo que está explícitamente fuera

- Aplicación móvil nativa (el bot y la web cubren el caso).
- Facturación electrónica propia (se integra con el sistema del cliente, no se construye).
- Contabilidad formal (no reemplazamos al contador).
- Generación de contenido o gestión de comunidades.
- E-commerce con pasarela: es un servicio complementario, no parte del producto.

---

## Riesgo principal del roadmap

**La dispersión.** Es el patrón de fracaso más común en productos de gestión: cada cliente pide algo distinto y el producto se convierte en veinte productos.

**Antídoto:** el alcance de un cliente se ajusta con **parámetros y datos**, no con código. Si una petición exige código nuevo, se evalúa con los tres filtros de arriba. Si es solo para ese cliente, se cobra aparte como desarrollo específico y no entra al producto base.
