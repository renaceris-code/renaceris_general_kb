# Módulo de Contabilidad — Renaceris

Sistema de reportes de ingresos, registro de egresos, control de caja, saldos e inventarios. Complementa el sistema de facturación externo **SIBI** ([sibi.pe](https://sibi.pe)); **no lo reemplaza**. Los balances y egresos se cargan manualmente.

> **Sobre SIBI (sistema externo actual):** POS + facturación electrónica SUNAT (boletas, facturas, guías, notas de crédito/débito), inventario con kardex/código de barras, reporte de caja del día y reportes contables/impuestos, orientado a MiPyMEs. La emisión de comprobantes y el cálculo de impuestos quedan en SIBI. Nuestro módulo aporta lo que SIBI no cubre para la clínica: unir la data de citas/pagos (que vive en nuestro sistema, no en SIBI), manejar ventas mixtas, egresos/caja y un inventario clínico, y **exportar** hacia SIBI/contadora.
> *(En la transcripción el sistema aparece como "CBP/CBI"; es una imprecisión: el sistema real es SIBI.)*

## Qué tendrá

**1. Ingresos y cuentas por cobrar**
- Reportes de ingresos filtrables por fecha, sede, forma de pago, tipo de servicio y doctor.
- Cuentas por cobrar: saldos de pacientes que pagan tratamientos en cuotas.
- Registro de pagos sin comprobante y detección de inconsistencias (atenciones sin pago confirmado, medio de pago mal registrado, montos que no cuadran).
- Ventas mixtas: desglosar un pago en efectivo + transferencia (SIBI hoy no lo permite y genera descuadres).

**2. Egresos / gastos**
- Registro diario de egresos que gestiona recepción, categorizados (pago de personal, servicios, tributos, compras cotidianas, etc.).

**3. Caja y balance diario**
- Separación de **caja chica** (efectivo) y **cuentas bancarias** (transferencias/tarjeta → BCP, BBVA), con saldos manejados por separado.
- Modelo: el efectivo alimenta la caja chica; las transacciones alimentan bancos; de la caja grande se asignan montos a la caja chica para compras cotidianas.
- Balance diario: ingresos totales (efectivo + transferencias) − egresos, cargado manualmente.
- Entrega/cierre diario de efectivo de recepción (ver resolución al final).

**4. Roles y accesos**
- Administrador: acceso completo.
- Contador: acceso restringido a ingresos y egresos, **sin datos personales de pacientes**.

**5. Comprobantes (referencial)**
- Registro referencial de notas de venta (adelantos), boletas y notas de crédito (anulaciones SUNAT), para que la contadora identifique movimientos anulados de periodos anteriores. La emisión sigue en SIBI.

**6. Exportación**
- Excel/CSV: listado de pagos (fecha, comprobante, monto, forma de pago, paciente/DNI, sede, servicio) y reporte "puente" alineado a la plantilla de SIBI.
- PDF: resumen de caja / balance diario por sede.

**Plantilla SIBI validada con el reporte de junio 2026**
- Oficina -> sede del pago/comprobante.
- Usuario -> usuario que creó la cita u orden.
- Tipo de documento / código / serie / correlativo -> normalizados para `Boleta`, `Factura`, `Nota de venta` y `Nota de crédito`.
- Documento del cliente -> DNI.
- Datos del cliente -> nombre y apellido.
- Dirección del cliente -> teléfono de contacto en el puente actual, porque el reporte de SIBI usa ese dato en esa columna.
- Operaciones gravadas / exoneradas / inafectas / gratuitas -> hoy el puente usa exoneradas como total principal y deja los demás en cero.
- Total IGV / ICBPER / otros cargos / descuento global -> cero mientras no exista desglose fiscal en el sistema.
- Estado SUNAT -> derivado del tipo de documento y del estado del registro.
- Tipo de moneda -> `Soles`.
- Observaciones -> código de transacción o nota interna del pago.
- Estado de pago -> `Pagado` o `Pendiente`.
- DAM -> vacío por ahora.

**7. Inventarios**
- Dos almacenes: productos de venta al público (fajas, proteínas) e insumos internos de cirugía.
- Control de stock, salida de insumos por cirugía y reingreso de sobrantes.
- *Nota:* SIBI ya tiene inventario (kardex). Confirmar si se construye propio (por la lógica de cirugía, que SIBI no cubre) o se apoya en SIBI, para evitar duplicar.

## Qué no tendrá

- No emite boletas/facturas electrónicas ni se integra automáticamente con SUNAT (queda en SIBI; el puente es por exportación/manual).
- No reemplaza al contador ni los libros contables.
- No calcula ni declara impuestos.
- No genera asientos contables (partida doble).
- No hace conciliación bancaria automática (los estados de cuenta se llevan por separado).
- No emite ni anula comprobantes (solo los registra de forma referencial).
- No fuerza un cierre de caja rígido tipo POS que bloquee la operación (ver resolución).

## Pendientes / dependencias externas

- Plantilla Excel de facturación de SIBI, para alinear los campos de exportación. — Ronald
- Términos actuales de categorización de ingresos. — Ronald
- Reporte de junio (SIBI) + reportes de ventas/productividad. — Ronald
- Credenciales de acceso a SIBI. — Ronald
- Investigación técnica del proceso real de entrega/cierre de caja del personal. — Serghio

## Resolución: apertura y cierre de caja

**Contexto actualizado.** El efectivo de recepción **no arrastra saldo**: al término del día, recepción cierra en S/ 0 porque entrega el 100% del efectivo al administrador. La caja de recepción es un **nodo de paso**, no un acumulador; el acumulador es la caja chica del administrador.

**Implicancia sistémica.** Como recepción abre y cierra en 0, una *apertura* formal con saldo inicial no aporta casi nada. El valor está en el **cierre**: cuadrar el efectivo esperado (Σ cobros en efectivo − egresos en efectivo del día) contra el efectivo físico realmente entregado al administrador. Ese es el único punto donde el sistema detecta automáticamente los descuadres que ellos mismos reportan como frecuentes.

**Casuística real que cubre.**
- Recepción marca un cobro como "efectivo" pero fue Yape/transferencia → el efectivo entregado es menor al esperado → salta el descuadre.
- Venta mixta (parte efectivo, parte transferencia) mal registrada.
- Error de vuelto o cobro incompleto.
- Egreso menor pagado de la caja sin registrarlo.
- Faltante de efectivo (control de responsabilidad, que el dueño remarcó).

**Resolución (neutral).**
- **Incluir versión mínima:** una **"entrega/cierre diario de efectivo"** = recepción registra el efectivo entregado, el sistema muestra el esperado y marca la diferencia; el administrador confirma la recepción. **Sin apertura formal ni bloqueo tipo POS.**
- **Obligatoriedad opcional/configurable:** la función existe; que sea obligatorio cerrarla cada día se activa/desactiva por sede.
- **Prescindir solo si** los reportes de junio de SIBI muestran que el efectivo es marginal frente a las transacciones bancarias; en ese caso basta el balance diario y el cuadre de efectivo aporta poco.

**En una línea:** no un "apertura/cierre" rígido; sí un **cuadre de entrega diaria de efectivo** en versión ligera y opcional, decisión final sujeta a ver el mix efectivo/banco de junio.
