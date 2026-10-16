Módulo: Reevaluaciones y Sesiones

Cambios implementados:
Día 10/09
* Nuevo tipo de cita "Reevaluación": el paciente vuelve a control sin que haya que registrarle una consulta ni una orden de mentira. Se agenda desde la agenda y desde el perfil del paciente.
* Línea de tratamiento: todo lo de un mismo tratamiento queda agrupado —sus órdenes, sus pagos y sus sesiones— y se ve en el perfil del paciente.
* Cotizar en la reevaluación: el especialista genera la proforma durante el control, sin abrir una consulta nueva. Antes solo se podía cotizar desde una consulta.
* Tratamiento el mismo día: al generar la orden, recepción marca "El tratamiento se realiza hoy, en esta misma atención" y la cita en curso queda como la sesión. No se duplica la cita en la agenda.
* Cobro por sesión: en Láser y tatuajes cada sesión se cotiza en su propia visita y genera su propia orden y su propio pago, con precio distinto cada vez.
* Paquetes de sesiones: una orden puede vender varias sesiones. El sistema lleva la cuenta de cuántas se entregaron y cuántas quedan, y no deja agendar la sesión 11 de un paquete de 10.
* Al agendar, recepción ve a qué tratamiento corresponde la cita: el desplegable muestra el servicio, el código de la orden, su fecha y "quedan X de Y sesiones". El botón "Preparar restantes" propone las fechas de las sesiones que faltan según el intervalo del servicio.
* Paquetes de procedimiento directo: también generan su orden y descuentan sesiones, aunque no pasen por médico ni por proforma.
* Configuración por servicio: cada servicio define si se vende como sesión única, paquete o serie por sesión; cuántas sesiones trae, cada cuántos días y si su reevaluación es gratuita o cobrada.
* Reportes: las reevaluaciones dejan de contarse como consultas.
* Tratamientos hechos antes del sistema: "Gestionar tratamiento previo" sigue disponible. Al registrarlo, la agenda salta directo a una reevaluación con ese tratamiento ya seleccionado.
* Lo anterior al cambio no se toca: las órdenes y citas que ya existen siguen funcionando igual, sin contador de sesiones y sin bloqueos nuevos.

Cambios pendientes:
* Ampliar un paquete: hoy, cuando se agotan las sesiones, el sistema bloquea. Falta poder vender sesiones adicionales sobre la misma línea de tratamiento.
* Controles automáticos: que al concluir una cirugía el sistema proponga solo las fechas de los controles.
* Aviso al agendar una sesión antes del intervalo mínimo configurado en el servicio.
* Ventana de seguimiento: definir cuántos meses de control van incluidos por tipo de cirugía y desde cuándo se cobra. Hoy depende solo de un interruptor por servicio: gratuita o cobrada.
* Vigencia de consulta (12 meses): pendiente de definición del negocio.
* Temporal: se retiró del desplegable la opción "Sin tratamiento previo en el sistema". El paciente operado en otra clínica pasa primero por consulta, y de esa consulta sale su orden. Se puede reactivar cuando haga falta.
