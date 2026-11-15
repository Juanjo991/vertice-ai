# 10 — Operación

Cómo se mantiene funcionando una cartera de clientes con una sola persona a cargo.

---

## El principio

**No se promete disponibilidad. Se promete prioridad.**

El servidor es del cliente. Si su VPS se cae, se cae su sistema, y no hay contrato que cambie eso. Lo que sí se puede controlar es **a quién se atiende primero** — y eso es exactamente lo que se vende.

---

## Los dos canales

| | **Línea Directa** | **Canal normal** |
|---|---|---|
| Acceso | Número o correo personal | Formulario automatizado |
| Triage | Ninguno: llega directo | Automatizado, con preguntas previas |
| Atención | Apenas se recibe, en horario hábil | Por lotes, en una ventana semanal |
| Prioridad | Primera | Después de las de Línea Directa |
| Garantía de tiempo | **Ninguna** | **Ninguna** |

**La diferencia es real, no decorativa.** Un ticket del canal normal pasa por preguntas automáticas que filtran lo trivial antes de llegar a una persona. Eso lo hace más lento por diseño, y es lo que justifica pagar por el otro.

**Horario hábil:** lunes a viernes. Fuera de ese horario, un incidente crítico se atiende según disponibilidad, en ambos canales.

---

## Clasificación de tickets

| Tipo | Ejemplo | Quién lo resuelve |
|---|---|---|
| **Duda de uso** | "¿cómo registro una compra?" | Agente de auto-ayuda, documentado en bitácora |
| **Dato mal cargado** | "puse $45.000 y eran $54.000" | El cliente, desde el chat. Si no puede, se corrige |
| **Falla funcional** | "no me está descontando el stock" | Se diagnostica, se corrige, se documenta |
| **Falla de infraestructura** | "el servidor no responde" | Depende del proveedor del cliente; se guía |
| **Petición nueva** | "quiero que además haga X" | No es soporte: es una expansión, se cotiza |

**Regla:** el agente de auto-ayuda absorbe el primer tipo. Si esa capa falla, el soporte humano se come el día.

---

## Tope de trabajo

| Tipo de cliente | Tope |
|---|---|
| **Arquetipo A** | **1 día completo** en total: implementación, ingesta, entrega y acompañamiento del primer mes |
| **Arquetipo B** | Según lo pactado en la implementación |

**Si el alcance no cabe, se reduce el alcance. No se extiende el plazo ni se baja el precio.**

Un cliente A que consume una semana destruye el negocio, porque su techo de ingreso es la implementación de US$80-150 más una membresía voluntaria.

---

## Cohortes

Los clientes del Arquetipo A se implementan **en grupos de 4 a 6**, en la misma ventana de tiempo.

**Por qué:** cada implementación exige cambiar de contexto (entender un rubro, cargar datos distintos, parametrizar protocolos). Hacerlo de a uno, intercalado con reuniones de Pyme, multiplica el tiempo perdido.

**Cómo:** se acumulan clientes cerrados hasta completar la cohorte, y se ejecuta una semana de implementación en bloque.

---

## Respaldos

| Aspecto | Definición |
|---|---|
| Frecuencia | Diaria, automática |
| Retención | 30 días rotativos |
| Ubicación | Fuera del servidor del cliente |
| Verificación | Chequeo de integridad semanal |
| Restauración | Probada antes de la entrega y cada 90 días |

**Un respaldo que nunca se restauró no es un respaldo.**

---

## Incidentes

### Si el sistema del cliente no responde

1. Verificar si el VPS responde (es responsabilidad del proveedor del cliente).
2. Revisar logs de la aplicación.
3. Revisar integridad de la base.
4. Restaurar desde el respaldo si es necesario.
5. Documentar en la bitácora técnica del cliente.

### Si hay un error en un dato

1. Corregir con el cliente presente por el chat.
2. Si el error afecta cálculos, revisar si el origen es de captura o de lógica.
3. Si es de lógica, es un bug: se corrige en el código y se replica a todos los clientes.

### Comunicación

- **Incidencia propia:** se informa antes de que el cliente pregunte.
- **Incidencia del proveedor del cliente:** se guía, no se resuelve.
- **Nunca** se promete una hora de restablecimiento si no se controla la causa.

---

## Actualizaciones

| Tipo | Cuándo | A quién |
|---|---|---|
| **Corrección de error** | Inmediata | A todos los afectados |
| **Mejora de versión** | En ventanas planificadas | A todos, con aviso previo |
| **Módulo nuevo** | Solo si se contrata | Al cliente que lo contrata |

**Regla:** nunca se actualiza un cliente sin respaldo verificado y sin posibilidad de reversa.

---

## Salud de la cartera

Cinco indicadores, revisados una vez al mes:

| Indicador | Señal de alarma |
|---|---|
| Ventas registradas por cliente | Menos de 20 al mes |
| Consultas de soporte por cliente | Suben mes a mes |
| Tiempo desde la entrega hasta la venta 20 | Más de 45 días |
| Adopción de Línea Directa | Baja o decreciente |
| Clientes sin actividad en 14 días | Cualquiera |

**El cliente que deja de registrar es el cliente que se va.** No hay que esperar a que avise: hay que escribirle.

---

## Capacidad real

Con una persona y el tope de un día por cliente A:

| Actividad | Consumo mensual estimado |
|---|---|
| Implementaciones A (cohorte de 5) | 5 días |
| Implementaciones B | 4-6 días por cliente |
| Soporte de cartera activa | 3-4 días |
| Desarrollo de producto | 6-8 días |

**Conclusión honesta:** la cartera crece hasta que el desarrollo de producto se detiene. El límite operativo no es la venta, es la implementación. Ese es el argumento real para profesionalizar el despliegue antes de escalar el número de clientes.
