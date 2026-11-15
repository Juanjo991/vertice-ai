# 11 — Ingeniería

Cómo se construye Vértice, quién hace qué y cómo se revisa.

---

## El modelo de trabajo

**Construcción asistida por IA, validación humana.**

- El fundador diseña la arquitectura y construye a alta velocidad con asistencia de IA.
- Un colaborador de ingeniería de software aporta el **rigor formal**: revisión del código generado, integridad de las bases de datos, seguridad de la infraestructura y mantención.

No es que uno sepa y el otro no. Es que **la velocidad de construcción y el rigor de revisión son dos trabajos distintos**, y hacerlos con la misma cabeza es cómo se escapan los errores.

---

## Roles

| Rol | Responsabilidad |
|---|---|
| **Arquitectura y producto** | Diseño del sistema, pipeline de datos, gobernanza por herramientas, decisiones de alcance |
| **Construcción** | Implementación de módulos, protocolos del agente, despliegue |
| **Revisión técnica** | Lectura crítica del código, modelado de datos, seguridad, pruebas |
| **Mantención** | Correcciones, actualizaciones, soporte de segundo nivel |

---

## Las cuatro reglas no negociables

### 1. Determinístico para lo predecible
Toda aritmética financiera, control de stock y cálculo de márgenes ocurre en Python y SQL. **Nunca** se le pide a un modelo que sume, reste o calcule una fecha.

### 2. Herramientas cerradas
El agente no escribe SQL libre contra la base del cliente. Solo invoca funciones probadas con parámetros validados.

### 3. Credenciales fuera del código
Todo en `.env`. Nunca en el repositorio, nunca en un mensaje, nunca en un log.

### 4. Privacidad antes de la llamada externa
Ningún nombre, RUT, teléfono o dirección sale hacia un proveedor de modelo. Se envía el token seudónimo y se reconstruye localmente.

---

## Flujo de desarrollo

```
1. Definir el alcance     → qué entra, qué no, y cómo se prueba
2. Construir              → código + protocolos + migración de esquema
3. Probar                 → casos reales, no solo caminos felices
4. Revisar                → lectura crítica por el colaborador
5. Desplegar              → primero en un cliente de prueba, después a todos
6. Documentar             → qué cambió y por qué, en la bitácora
```

**Regla:** nada llega a producción de un cliente sin haber corrido antes en un entorno de prueba con datos que se parezcan a los reales.

---

## Estándares de código

| Aspecto | Regla |
|---|---|
| Nombres | En español para el dominio (`registrar_venta`, `calcular_margen`), en inglés para infraestructura |
| Funciones | Una responsabilidad. Si el nombre necesita "y", son dos funciones |
| Dinero | Enteros en CLP. Nunca punto flotante |
| Fechas | ISO 8601 con zona horaria explícita (`America/Santiago`) |
| Errores | Nunca silenciosos. Se registran y se informan al usuario en lenguaje natural |
| Configuración | Todo parámetro del negocio fuera del código (tabla o archivo de protocolo) |
| Idempotencia | Registrar dos veces la misma operación no debe duplicar datos |

---

## Sobre la IA en el código

El código puede generarse con asistencia de IA. Eso no cambia el estándar de revisión: **lo aumenta**.

Quien revisa debe poder responder, para cada función que toca dinero o datos:

1. ¿Qué pasa si recibe un valor nulo?
2. ¿Qué pasa si se ejecuta dos veces?
3. ¿Qué pasa si el dato viene con un formato distinto al esperado?
4. ¿Qué pasa si el cliente tiene 10.000 registros en vez de 10?
5. ¿Qué consulta a la base hace, y usa índice?

Si alguna respuesta es "no sé", no se aprueba.

---

## Checklist de revisión

**Datos**
- [ ] El esquema tiene migración y reverso
- [ ] Los cálculos de dinero son enteros y determinísticos
- [ ] No hay resultados calculados guardados en tablas
- [ ] Las consultas usan índices donde corresponde

**Seguridad**
- [ ] No hay credenciales en el código
- [ ] La seudonimización cubre los campos nuevos
- [ ] Los permisos de archivo son correctos
- [ ] Los logs no filtran datos personales

**Comportamiento**
- [ ] La operación es idempotente
- [ ] Los errores se informan en lenguaje natural
- [ ] El agente no puede inventar una cifra: todo número viene de una herramienta
- [ ] El protocolo del agente está actualizado si cambió el alcance

**Operación**
- [ ] Existe forma de revertir el cambio
- [ ] Quedó registro en la bitácora
- [ ] Se probó con datos parecidos a los de un cliente real

---

## Pruebas

| Nivel | Qué cubre |
|---|---|
| **Unitarias** | Cálculos: caja, margen, stock, fechas |
| **De integración** | Herramientas del agente contra una base real de prueba |
| **De regresión** | Los casos que ya fallaron una vez, para que no vuelvan |
| **Manuales** | Un flujo completo con datos de un negocio real, antes de cada entrega |

**Prioridad:** probar a fondo lo que toca dinero. Un error en el flujo de caja destruye la confianza más rápido que cualquier otra cosa.

---

## Documentación

| Qué | Dónde |
|---|---|
| Decisiones de producto y comerciales | [`DECISIONES.md`](DECISIONES.md) |
| Comportamiento del agente por cliente | Protocolos `.md` en la instancia del cliente |
| Cambios técnicos | Bitácora del cliente |
| Este repositorio | Arquitectura, módulos, comercial, operación |

**Regla:** una decisión que no está escrita no existe. Se vuelve a discutir y se decide distinto.

---

## Deuda técnica conocida

| Deuda | Impacto | Cuándo se paga |
|---|---|---|
| SQLite con un solo escritor | Bloquea el multiusuario | Con la versión web |
| Sin cola de trabajos | Limita procesos largos | Sobre 30 clientes |
| Sin monitoreo centralizado | Diagnóstico manual | Con operación delegada |
| Sin tests de carga | Riesgo al escalar | Antes del cliente 20 |
| Sin CI/CD | Despliegues manuales | Cuando haya más de 10 clientes |

---

## Principio final

**Cada módulo nuevo es soporte futuro.** Antes de construir algo, la pregunta no es "¿se puede?" — casi todo se puede. La pregunta es: *¿quién lo va a mantener, y a quién le vamos a cobrar por eso?*

Si no hay respuesta clara a las dos, no se construye.
