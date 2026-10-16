Cambios implementados:
Día 28/09
* Edición del monto de la cita: recepción puede corregir el monto a cobrar desde el detalle de la cita o desde "Editar Planificación". Debe indicar un motivo. Si la cita ya tiene pagos, solo un administrador puede cambiar el monto, y nunca por debajo de lo ya pagado.
* Trazabilidad: el sistema registra quién cambió el monto, cuándo, el valor anterior, el nuevo y el motivo. En el detalle de la cita se ve "Monto editado por…" con su historial. Cada pago también muestra quién lo registró o editó, y los pagos eliminados quedan en el historial.
* Corrección: al editar una cita, el monto que se ingresaba se guardaba como si fuera lo pagado. Por eso algunas citas sin pagos aparecían como "con pagos" y el sistema no permitía editarlas. Ya está corregido.
* Reporte de ingresos de caja: las citas creadas con un pago inicial se contaban dos veces. Ahora se cuentan una sola vez.

Cambios pendientes:
* Resumen de ingresos del inicio: los pagos registrados después de crear la cita no se suman a "ingresos de hoy" ni "de la semana". Se corregirá para que cuente todos los pagos.
* Listado y calendario de citas: el monto que se muestra se ajustará para que refleje el precio de la cita y lo pagado.
* Limpieza de datos: se revisarán las citas pagadas antes del sistema actual de pagos para trasladarlas y corregir los montos que quedaron mal por el error de edición.

---

Reunión de contabilidad — acuerdos y pendientes:

Objetivo:
* Recoger el catálogo de cuentas, los formatos de estados financieros y las reglas contables que la contadora necesita que respete el sistema. Con estos insumos se definirán la clasificación de egresos, los reportes y las relaciones de base de datos.

Información solicitada a la contadora:
* Excel de cuentas contables: debe incluir el código de la cuenta y la denominación de la cuenta. Si existen cuentas principales y subcuentas, indicar su jerarquía y cuáles se pueden utilizar para registrar movimientos. Los códigos deben conservarse como texto para evitar perder ceros iniciales.
* Excel de estados financieros: entregar los formatos que se necesitan generar e indicar cómo se relacionan sus rubros con las cuentas contables. Por cada rubro, señalar las cuentas o rangos de cuentas que lo alimentan, las agrupaciones, los subtotales y las fórmulas que correspondan. Puede entregarse en un mismo archivo con hojas separadas para los formatos y su relación con las cuentas.
* Reglas contables en sus propias palabras: explicar qué debe permitir, exigir o impedir el sistema al clasificar un egreso y generar los reportes. No es necesario utilizar términos técnicos de programación ni de base de datos; el equipo de desarrollo traducirá esas reglas a requisitos técnicos.

Guía para describir las reglas:
* Indicar qué situación se presenta, cómo debe resolverse y si existen excepciones.
* Acompañar cada regla con un ejemplo concreto: cuenta utilizada, operación y resultado esperado.
* Ejemplo por confirmar: «Cada cuenta contable pertenece a una sola clasificación, como Activo o Pasivo; varias cuentas pueden pertenecer a la misma clasificación».
* Otros ejemplos para validar: «Un egreso puede distribuirse entre varias cuentas», «Solo determinadas subcuentas permiten registrar movimientos» o «Un período cerrado no puede modificarse sin autorización». Estos ejemplos son preguntas de levantamiento, no acuerdos ya aprobados.

Aclaración para el diseño técnico:
* Separar la clasificación contable (por ejemplo, Activo, Pasivo o Patrimonio) del estado financiero y del rubro donde se presenta una cuenta. La contadora debe precisar estos conceptos y sus relaciones antes de definir el modelo de datos.
* Si cada cuenta pertenece a una clasificación y una clasificación agrupa varias cuentas, la relación es de muchas cuentas a una clasificación (N:1), o de una clasificación a muchas cuentas (1:N). No sería una relación 1:1.
* Una relación 1:1 solo correspondería si ambos extremos fueran exclusivos: cada cuenta tiene un único registro relacionado y ese registro pertenece únicamente a esa cuenta.
* La relación entre cuentas y rubros de los estados financieros se definirá a partir del Excel y las reglas confirmadas. Debe verificarse si una cuenta puede intervenir en varios reportes o si requiere distribuirse entre rubros según la operación.

Preguntas pendientes para la contadora:
* ¿Qué estados financieros necesitan generar y con qué periodicidad?
* ¿Qué clasificación, estado financiero y rubro corresponde a cada cuenta? ¿Una misma cuenta puede participar en más de un reporte?
* ¿La denominación se toma siempre del catálogo? ¿Quién puede crear cuentas o modificar sus nombres?
* ¿Un egreso se registra en una sola cuenta o puede dividirse entre varias? ¿Cómo se manejan impuestos, anticipos y pagos de deudas previamente registradas?
* ¿Cómo deben registrarse las operaciones pendientes de cobro o pago y las transferencias entre cuentas propias?
* ¿Qué saldos iniciales, ajustes y datos adicionales necesitan para preparar los estados financieros?
* ¿Quién valida y contabiliza las operaciones? ¿Cómo se corrigen asientos y se cierran o reabren períodos?

Próximos pasos:
* Contadora: entregar ambos Excel y explicar las reglas con ejemplos y excepciones.
* Equipo de desarrollo: revisar los insumos, documentar los requisitos funcionales y proponer las relaciones de base de datos.
* Validación conjunta: confirmar las reglas y comprobar con operaciones de ejemplo que los saldos y rubros coinciden con los formatos entregados.
* Pendiente de definición: fechas de entrega, formatos definitivos y alcance de los estados financieros. Este reporte documenta el levantamiento; no confirma funcionalidades implementadas.
