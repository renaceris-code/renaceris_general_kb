# Catálogo de escenarios — Reevaluaciones y multisesiones

Casos que el modelo debe resolver, incluidos los que no aparecieron en la conversación original. Cada fila indica qué ocurre hoy, la regla propuesta y si necesita una definición del cliente. Las decisiones marcadas con **D** están detalladas en [05-decisiones-pendientes.md](05-decisiones-pendientes.md).

## A. Paquetes multisesión con una sola proforma · punto 3

| # | Escenario | Hoy | Regla propuesta | Decisión |
|---|---|---|---|---|
| A1 | Paquete de 10 masajes vendido en la consulta | Se puede agendar N citas contra una orden, pero nadie sabe que son 10 | `sessions_total = 10` en la orden; la agenda muestra "sesión k de 10" | — |
| A2 | Se intenta agendar la sesión 11 | Se permite en silencio | Bloqueo con mensaje; un administrador puede ampliar el paquete creando una orden adicional en el mismo episodio | — |
| A3 | La paciente cancela una sesión | Indistinguible de una entregada | Cancelada no consume sesión | — |
| A4 | La paciente no asiste sin avisar | Igual que cancelada | No consume por defecto; política de penalidad configurable por sede | **D** |
| A5 | Reprogramación de una sesión | Cambio de fecha en la cita | Sin efecto sobre el contador; se conserva el historial de reprogramaciones | — |
| A6 | Agendar las 10 sesiones de una vez | Manual, cita por cita | Acción de agendamiento en bloque con periodicidad, respetando capacidad y feriados | — |
| A7 | Paquete que vence con sesiones sin usar | No existe vencimiento | `valid_until`; al vencer se bloquean y requieren reactivación con motivo y usuario responsable | **D** |
| A8 | Paciente abandona a la mitad con saldo pendiente | La orden queda en cuentas por cobrar para siempre | Cierre del episodio como `abandoned`; la orden sale del reporte de cobranza activa y pasa a un estado de incobrable revisable | **D** |
| A9 | Paciente compra sesiones adicionales al terminar | Habría que inventar otra consulta | Nueva orden en el mismo episodio, sin repetir consulta | — |
| A10 | Masajes incluidos en el precio de la cirugía | Se registran como orden aparte o no se registran | Orden con `included_in_order_id` apuntando a la cirugía y total 0; las sesiones se controlan igual | **D** |
| A11 | Masajes proformados durante el proceso de la cirugía | Posible como segunda proforma de la consulta, con el límite de 3 | Se mantiene; si un caso real necesita más de tres tratamientos cotizados, se eleva el límite por configuración | **D** |
| A12 | Dos paquetes activos del mismo servicio a la vez | La UI obliga a elegir orden, sin criterio | Consumo por orden más antigua con vencimiento más próximo, con opción de cambio manual | — |
| A13 | Sesión ejecutada sin cita previa | No se puede registrar | Registro retroactivo de cita ya completada, con auditoría | — |
| A14 | Sesiones en una sede distinta a la de la orden | El servicio está atado a una sede | Permitir la ejecución en otra sede registrando la sede real de atención, para no distorsionar la productividad por sede | **D** |
| A15 | Cambio de servicio a mitad de paquete, por ejemplo masajes a drenaje | No contemplado | Cierre parcial de la orden y nueva orden por el remanente, dentro del mismo episodio | **D** |
| A16 | Transferencia del paquete a otra persona | No contemplado | Prohibido; el episodio pertenece al paciente | — |
| A17 | Distintos especialistas ejecutan distintas sesiones | Se registra el especialista de cada cita | Sin cambios; la productividad se calcula por sesión entregada, no por orden | — |
| A18 | Sesión parcial, se aplicó solo una zona | No contemplado | Se registra como entregada con nota clínica; si corresponde compensación, se agrega una sesión al paquete con motivo | **D** |

## B. Series con proforma por sesión · punto 4

| # | Escenario | Hoy | Regla propuesta | Decisión |
|---|---|---|---|---|
| B1 | Sesión 2 de láser CO₂ con precio propio | Bloqueado, exige fingir una consulta | La reevaluación emite proforma y orden | — |
| B2 | La sesión se ejecuta en la misma cita de la reevaluación | No representable | La cita queda vinculada como sesión de la orden que acaba de crear | — |
| B3 | Precio distinto en cada sesión | No hay dónde registrarlo | Cada proforma es independiente y no hereda precios | — |
| B4 | Intervalo clínico mínimo entre sesiones | Sin control | `min_days_between_sessions` con advertencia al agendar, no bloqueo duro | **D** |
| B5 | En la reevaluación se decide no aplicar la sesión por contraindicación | No contemplado | La cita se cierra como reevaluación sin tratamiento, sin cobro, y se reprograma | — |
| B6 | La paciente no vuelve después de la sesión 3 | Órdenes pendientes eternas | Episodio `abandoned`, órdenes sin saldo se cierran, órdenes con saldo se marcan incobrables | **D** |
| B7 | Se cotizó la sesión pero la paciente no la toma ese día | La orden queda activa sin sesión | Orden `pending` con vencimiento de cotización; se puede anular sin afectar el episodio | **D** |
| B8 | Serie con número de sesiones desconocido al inicio | No contemplado | El episodio no exige total de sesiones; solo las órdenes de paquete lo requieren | — |
| B9 | Paquete de láser vendido cerrado, por ejemplo 6 sesiones con descuento | Ambiguo | Se modela como paquete, `package_single_quote`, no como serie | **D** |
| B10 | Cambio de equipo o parámetros entre sesiones | Nota clínica libre | Sin cambio estructural; queda en la nota de la sesión | — |

## C. Reevaluaciones · puntos 5 y 6

| # | Escenario | Hoy | Regla propuesta | Decisión |
|---|---|---|---|---|
| C1 | Control post-operatorio a 7, 30 y 90 días | Se agenda a mano como consulta o como tratamiento con orden inventada | Serie propuesta automáticamente al completar la cirugía, según `followup_schedule` | **D** |
| C2 | Reevaluación de un tratamiento previo al sistema | Se crea una orden `legacy_manual` como percha | Episodio con `origin = legacy`, sin necesidad de orden ficticia | — |
| C3 | Paciente operada en otra clínica que viene a controlarse | No contemplado | Episodio con `origin = external`; la reevaluación probablemente se cobra | **D** |
| C4 | Reevaluación fuera del periodo de seguimiento incluido | Todo es gratis o todo se cobra | `reevaluation_pricing` por servicio y ventana de seguimiento incluida; fuera de ella se cobra | **D** |
| C5 | Reevaluación que deriva en retoque o cirugía de revisión | No contemplado | Proforma nueva con precio cero o precio de garantía, y motivo obligatorio | **D** |
| C6 | Reevaluación atendida por un médico distinto al que operó | Permitido | Sin cambios; se registra el médico real de la atención | — |
| C7 | Reevaluación virtual | El chequeo de modalidad virtual depende del tipo de cita | Habilitar la modalidad virtual para reevaluaciones según `supports_virtual_consultation` | **D** |
| C8 | Reevaluación que solo genera venta de producto, por ejemplo una faja | No contemplado en este módulo | Sale por inventario y venta de producto, no por proforma de servicio | — |
| C9 | Dos reevaluaciones de episodios distintos el mismo día | Posible, sin relación entre ellas | Cada cita apunta a su episodio; se permite | — |
| C10 | Reevaluación que descubre una complicación y requiere procedimiento inmediato | No contemplado | Igual que reevaluación y tratamiento, con motivo clínico de complicación | — |
| C11 | Reevaluación que se convierte en consulta de otra especialidad | Hoy obligaría a otra cita | Se permite cotizar otro tratamiento desde la reevaluación; si además debe cobrarse consulta, aplica la política de vigencia del punto 1 | **D** |
| C12 | El tratamiento cotizado no existe en la sede de la cita | Bloqueado por regla vigente | Se mantiene el bloqueo, con mensaje que sugiera la sede donde sí está disponible | — |

## D. Frontera clínica y de agenda

| # | Escenario | Hoy | Regla propuesta | Decisión |
|---|---|---|---|---|
| D1 | La reevaluación dura 10 minutos y ocupa un slot de consulta | Un solo slot por servicio | `reevaluation_duration_minutes` propio | — |
| D2 | Sesión de láser de 45 minutos con slot de tratamiento genérico | `full_service_duration_minutes` único por servicio | Suficiente por ahora; revisar si aparecen sesiones de duración variable | — |
| D3 | Especialista no asignado al crear la sesión | Permitido, se exige antes de registrar la llegada | Sin cambios | — |
| D4 | Capacidad de sala o equipo, por ejemplo un solo láser | No modelado | Fuera de alcance de este módulo; se registra como riesgo | — |
| D5 | Recordatorios por sesión | Existe el sistema de recordatorios | Extenderlo a sesiones y reevaluaciones, con texto distinto | — |
| D6 | Reprogramación masiva de un paquete | No existe | Deseable en la fase de agendamiento en bloque | — |

## E. Comercial y contable

| # | Escenario | Hoy | Regla propuesta | Decisión |
|---|---|---|---|---|
| E1 | Paquete pagado por adelantado y entregado en tres meses | El ingreso se reconoce completo al pagar | Se mantiene el criterio de caja y se agrega el indicador de sesiones cobradas no entregadas | — |
| E2 | Pago por sesión dentro de un paquete | La UI permite un pago por cita de sesión | Se mantiene, con el resto en Gestión de Órdenes | — |
| E3 | Comprobante por paquete o por sesión | El comprobante se registra por pago | Depende de cómo emita SIBI; documentar la práctica real antes de tocar el puente | **D** |
| E4 | Descuento por paquete, 10 sesiones al precio de 8 | Se puede reflejar en el total de la orden | Sin cambios; el precio unitario derivado es solo informativo | — |
| E5 | Sesiones no usadas al vencer un paquete pagado | No contemplado | Definir si se pierde, se extiende o se devuelve | **D** |
| E6 | Devolución parcial | No contemplado | Registro de la anulación en el módulo contable y nota de crédito en SIBI | **D** |
| E7 | Comisión de ventas por paquete o por sesión entregada | No modelado | Fuera de alcance; se deja el dato disponible para calcularla después | — |
| E8 | Cambio de precio de lista a mitad de paquete | El total de la orden ya está fijado | El precio pactado en la orden no cambia | — |
| E9 | Atribución de ingreso a la fecha de la sesión | Todo el ingreso queda en la fecha de pago | Reporte operativo de sesiones entregadas por fecha, separado del reporte de ingresos | — |
| E10 | Cuentas por cobrar de episodios abandonados | Inflan el saldo pendiente | Excluirlos del pendiente activo y mostrarlos en un bloque aparte | **D** |

## F. Datos, migración y concurrencia

| # | Escenario | Riesgo | Mitigación |
|---|---|---|---|
| F1 | Agregar un valor al enum `appointment_type_enum` | Ya se rehizo una vez en la migración `008`; hay un índice único sobre `(service_id, appointment_type)` en `service_prices` y RPCs que comparan el tipo | Agregar el valor sin recrear el tipo; revisar cada punto donde se compara con `'consultation'` |
| F2 | Citas históricas con `category = 'reevaluation'` y tipo `treatment` | Quedarían clasificadas como sesiones | Migración de reclasificación a `reevaluation` cuando cuelgan de órdenes `legacy_manual` sin sesión real |
| F3 | Órdenes `legacy_manual` creadas solo como percha de reevaluación | Ensucian el reporte de tratamientos | Convertirlas en episodios `legacy` y conservar la orden solo si tuvo dinero |
| F4 | Dos recepcionistas agendan la última sesión disponible a la vez | Sobreventa del paquete | Bloqueo de fila sobre la orden al insertar la cita, igual que hace el trigger de proformas |
| F5 | `treatment_group_id` con datos parciales | Ambigüedad entre grupo y episodio | Migrar a `episode_id` y dejar la columna anterior en desuso antes de eliminarla |
| F6 | Relajar la regla de proforma solo en el servicio y olvidar el RPC | El flujo falla en el paso de crear la orden | La restricción vive en tres lugares: `proformas.service.ts`, el RPC `create_service_order_from_proforma` y la UI. Los tres se cambian juntos |
| F7 | `getForAppointment` devuelve lista vacía si la cita no es consulta | La reevaluación no vería sus proformas | Incluir el nuevo tipo en esa consulta |
| F8 | Reportes y contabilidad que agrupan por `appointment_type` | Aparece un tipo nuevo no contemplado | Actualizar `reports.service.ts` y el resumen contable en la misma entrega |
| F9 | Roles con permiso para emitir proformas | Hoy médico y administrador | Confirmar si el especialista estético puede cotizar en una reevaluación de láser | **D** |
| F10 | Índice antiduplicados de citas activas | Bloquea la misma tripleta paciente, servicio y hora | Compatible con multisesión; no requiere cambio |

## G. Consulta inicial y vigencia · punto 1, prioridad menor

| # | Escenario | Regla propuesta | Decisión |
|---|---|---|---|
| G1 | Vigencia por año calendario o 12 meses desde la primera consulta | Recomendado: 12 meses desde la consulta pagada, para no perjudicar a quien consulta en noviembre | **D** |
| G2 | Consulta pagada en una sede y segunda consulta en otra | Recomendado: crédito global del paciente, registrando la sede de cada atención | **D** |
| G3 | Consulta con cirujano y luego con dermatología o nutrición | Recomendado: crédito global, no por especialidad | **D** |
| G4 | Consulta virtual | Definir si otorga y si consume vigencia | **D** |
| G5 | Consulta pagada y no asistida | Definir si consume el crédito o se reprograma | **D** |
| G6 | La consulta se descuenta del tratamiento si la paciente opera | Práctica frecuente en el rubro; confirmar si aplica | **D** |
| G7 | Consulta gratuita por campaña, referido o convenio | Motivo de exoneración obligatorio y trazable, distinto de la exoneración por vigencia | — |
| G8 | El precio de consulta cambia durante la vigencia | El crédito exonera el cobro, no fija un precio | — |
| G9 | Paciente con tratamiento activo y vigencia vencida | Las reevaluaciones del tratamiento activo nunca cobran consulta | — |
| G10 | Fichas duplicadas del mismo paciente | El DNI único ya está garantizado por la migración `018`; el crédito sigue al paciente | — |
| G11 | Historia previa al sistema | Definir fecha de corte desde la cual cuenta la vigencia | **D** |
| G12 | Reporte de consultas exoneradas | Indicador propio, separado de las consultas cobradas | — |
