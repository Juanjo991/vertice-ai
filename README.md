# Vértice AI — Documentación maestra

**Vértice AI** es un ecosistema agéntico de captación de datos e inteligencia comercial para micro y pequeñas empresas. Su corazón es un **asistente de IA que funciona como un empleado digital**: centraliza la información del negocio y le dice al dueño **qué hacer con ella**.

> **En una línea:** un ERP guarda números mudos. Vértice convierte esos números en la historia viva de cada relación comercial — y en la siguiente acción concreta.

---

## Cómo leer este repositorio

Este repositorio es **documentación, no código**. Describe qué es el producto, por qué existe, cómo está construido, cómo se vende y cómo se implementa.

Si es tu primera vez, lee en este orden:

1. **[01 — Problema](01-PROBLEMA.md)** — a quién le duele qué, y por qué las alternativas fallan.
2. **[02 — Producto](02-PRODUCTO.md)** — qué es Vértice y qué recibe el cliente.
3. **[03 — Arquitectura](03-ARQUITECTURA.md)** — cómo está construido por dentro.
4. **[07 — Comercial](07-COMERCIAL.md)** — cómo se vende.

Los demás documentos son de consulta.

---

## Índice

| # | Documento | Qué contiene |
|:-:|---|---|
| 01 | [Problema](01-PROBLEMA.md) | El dolor, los dos arquetipos, la brecha de escala, por qué falla la competencia |
| 02 | [Producto](02-PRODUCTO.md) | Qué es Vértice, el empleado digital, qué entra el día 1 y qué viene después |
| 03 | [Arquitectura](03-ARQUITECTURA.md) | Pipeline, capas, almacenamiento dual, el radar, el agente y su arnés |
| 04 | [Módulos](04-MODULOS.md) | Catálogo completo por área, con estado y valor para el cliente |
| 05 | [Stack](05-STACK.md) | Tecnologías, proveedores, entornos y costos del cliente |
| 06 | [Datos y seguridad](06-DATOS-Y-SEGURIDAD.md) | Esquema de tablas, privacidad, cumplimiento legal |
| 07 | [Comercial](07-COMERCIAL.md) | Arquetipos, precios, Línea Directa, embudo de venta, expansión |
| 08 | [Implementación](08-IMPLEMENTACION.md) | Cómo se instala un cliente, de la reunión a la entrega |
| 09 | [Roadmap](09-ROADMAP.md) | Versiones, fases y qué se construye cuándo |
| 10 | [Operación](10-OPERACION.md) | Soporte, prioridades, cohortes, respaldos |
| 11 | [Ingeniería](11-INGENIERIA.md) | Cómo trabajamos, roles, estándares y revisión de código |
| — | [Decisiones](DECISIONES.md) | Registro de lo ya decidido. Si no está ahí, no está decidido |

---

## El principio que ordena todo

**El código determinístico es el arnés; la IA es el empleado.**

- El **código** hace la aritmética, controla el stock, consulta la base y barre patrones. No inventa nada y no consume tokens.
- La **IA** interpreta, decide, redacta y conversa. **Es el pilar del producto.**

El código no reemplaza a la IA: le da un piso confiable para que no improvise. Sin IA, el sistema es un Excel automático que captura datos y no dice nada. Con IA sin arnés, es un asistente que inventa números. Vértice es la combinación.

---

## El producto, en corto

Un solo producto, entregado completo desde el día 1:

- Ventas · compras · inversiones · costos
- Stock automático
- Flujo de caja
- Margen unitario por producto
- Fichas de clientes
- **Interpretación de IA sobre todo lo anterior, desde el primer día**

Lo que viene después —web, radar de recompra, documentos, pagos, conexiones externas— es **capacidad nueva que se agrega a un cliente que ya tiene Vértice**, no un requisito para alcanzarlo.

---

## Estado de este repositorio

Documentación en construcción activa. Las decisiones cerradas están en [`DECISIONES.md`](DECISIONES.md) y no se reabren sin dejarlas registradas ahí.
