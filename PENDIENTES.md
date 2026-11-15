# Puntos a mirar

Cosas detectadas en una revisión del **diseño documentado** que todavía no están corregidas. No son errores de un sistema en producción — no hay sistema en producción. Son **huecos del diseño escrito** que conviene resolver antes de construir sobre ellos.

Estado: **sin corregir, a propósito.** Se dejan visibles para no perderlas.

---

## 1. El esquema no contempla IVA
**Dónde:** `06-DATOS-Y-SEGURIDAD.md` → tabla `transactions`.

`monto` no distingue bruto de neto. Si el negocio es formal, el margen se calcula sobre un precio que incluye un impuesto que no es ingreso. El pre-cálculo de IVA no puede construirse sobre este esquema.

**A decidir:** `monto_neto` + `iva` + `total`, o una sola columna con una bandera de tipo.

---

## 2. Venta y líneas de venta están fusionadas
**Dónde:** `06-DATOS-Y-SEGURIDAD.md` → `transactions`.

Una transacción es a la vez cabecera y línea. Con descuentos repartidos en varias líneas, el total no coincide con la suma de las partes por redondeo, y el precio unitario con descuento se pierde.

**A decidir:** separar `ventas` y `venta_items`.

---

## 3. El stock es un número que se modifica, no un historial
**Dónde:** `06-DATOS-Y-SEGURIDAD.md` → `products.stock`.

El mismo documento dice tres líneas más arriba que guardar resultados calculados es la forma más común de tener datos que se contradicen. El stock es exactamente eso.

**A decidir:** libro de movimientos de stock (entrada, salida, merma, ajuste) con el saldo derivado.

---

## 4. El formato de fecha propuesto rompe las ventas de la noche
**Dónde:** `06-DATOS-Y-SEGURIDAD.md` → notas de `transactions.fecha`; `05-STACK.md` → convenciones.

Guardar la fecha como texto ISO con zona horaria incluida hace que SQLite interprete mal el cambio de día: una venta a las 22:30 queda en el día siguiente.

**A decidir:** guardar `fecha_local` (texto sin offset, ya convertida) y por separado la marca UTC cuando se necesite.

---

## 5. El método de respaldo propuesto produce copias vacías
**Dónde:** `10-OPERACION.md` → Respaldos.

Copiar el archivo de la base mientras está en uso puede generar un archivo sin datos, y el chequeo de integridad igual lo declara correcto.

**A decidir:** usar el mecanismo de respaldo en caliente del propio motor (`VACUUM INTO` o la API de respaldo), y **restaurar de verdad** antes de dar por bueno el procedimiento.

---

## 6. El producto vende cosas que el roadmap pone después
**Dónde:** `07-COMERCIAL.md` (Línea Directa) contra `09-ROADMAP.md` (V1.1) y `08-IMPLEMENTACION.md`.

El **cierre de mes** y la **corrección de registros** se ofrecen como parte de lo que el cliente recibe, pero están agendados para una versión posterior. Un cliente que paga por el cierre de mes y no lo recibe el mes 2 cancela.

**A decidir:** mover ambas a la primera versión, o dejar de ofrecerlas hasta que existan.

---

## 7. "Un día por cliente" no cierra con las tareas listadas
**Dónde:** `08-IMPLEMENTACION.md` → Tiempos y capacidad.

Sumando todo lo que el mismo documento exige (reuniones, investigación, ingesta, despliegue, entrega, acompañamiento del mes 1), el total pasa de un día. El tope mezcla horas de trabajo con citas de calendario.

**A decidir:** definir el tope en horas reales, y qué se recorta cuando no cabe.

---

## 8. La primera versión es de un solo usuario, pero el segmento grande no
**Dónde:** `01-PROBLEMA.md` (arquetipo B) contra `03-ARQUITECTURA.md` (un solo operador).

La Pyme con empleados se define justamente por tener varias personas sobre el mismo dato. La primera versión no tiene roles ni permisos. Si el peso del negocio está puesto en ese segmento, hay una contradicción de fondo.

**A decidir:** adelantar el multiusuario, o sostener la primera etapa con el otro arquetipo.

---

## Nota

Estos puntos salieron de una revisión del diseño escrito, no de pruebas sobre un sistema real. Varios fueron verificados en SQLite con casos concretos.

Si alguno se corrige, se borra de esta lista y se registra en [`DECISIONES.md`](DECISIONES.md).
