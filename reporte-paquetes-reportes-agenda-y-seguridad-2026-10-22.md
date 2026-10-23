Cambios implementados:

Días 21 y 22/10/2026

* Paquetes con un precio acordado: al crear un procedimiento de varias sesiones, recepción indica cuántas sesiones incluye y cuánto se cobrará por todo el paquete. El sistema sugiere un precio como referencia, pero permite ajustarlo al acuerdo con el paciente.
* Pagos a lo largo del tratamiento: el paciente puede pagar todo al inicio o hacer abonos en distintas citas. Cada pago reduce la deuda del paquete completo. Por ejemplo, si el paquete cuesta S/ 1.000 y paga S/ 300, quedan S/ 700 pendientes. Si ya pagó todo, puede seguir asistiendo a sus sesiones sin generar una nueva deuda.
* Número de cita visible: al agendar y al consultar la agenda se muestra «Cita 4/20», por ejemplo, para reconocer en qué sesión está el paciente. Cambiar la fecha o la hora conserva ese número. También se evita asignar el mismo número a dos citas o reservar más sesiones de las acordadas.
* Sesiones disponibles al eliminar una cita: el espacio de esa sesión vuelve a quedar libre, incluso si la atención ya había comenzado o terminado. Las demás citas mantienen sus números y se actualiza la cantidad de sesiones realizadas.
* Datos del paciente a la mano: desde la vista previa de la agenda y el detalle de la cita se pueden consultar la historia clínica, el DNI, la edad y el teléfono, con botones para copiar los datos correspondientes. La vista previa permanece abierta al mover el cursor sobre ella, para poder usar esos botones.
* Reporte con el formato de la clínica: el Excel conserva la hoja BASE y las columnas acordadas, con el periodo al inicio. Incluye fecha, recepción, área, profesional que atendió, DNI y nombre del paciente, categoría de cita, servicio, cobro y comprobante. El personal estético aparece en el área TÉCNICOS y los doctores en MÉDICOS.
* COSTO muestra lo realmente cobrado: cada fila suma los pagos confirmados de esa cita y muestra sus comprobantes. Si un paquete de S/ 1.000 recibe S/ 300 en la primera cita y S/ 50 en la segunda, el reporte muestra 300 y 50. Si no hubo pago, muestra cero. Así se evita repetir el precio del paquete como si se hubiera cobrado nuevamente.
* Los mismos filtros para Excel y PDF: desde «Exportar citas» se eligen la categoría y el área, además de las fechas y la sede. Esa selección se utiliza en ambos formatos. Se puede obtener un reporte de personal estético, de médicos o de ambos. Se retiraron las exportaciones SIBI y el menú adicional que repetía las opciones de área en Excel.
* Eliminación con información clara: el administrador general y el administrador de sede pueden eliminar citas dentro de sus permisos. Si es una cita creada por error y no tiene información relacionada, se muestra una confirmación breve. Si tiene pagos, comprobantes o una atención iniciada o terminada, el aviso explica qué se borrará y cómo afectará al saldo antes de ofrecer «Eliminar de todas maneras».
* Qué se conserva al eliminar: permanecen el paciente, sus otras atenciones y la orden del paquete. Se borran los pagos y documentos clínicos exclusivos de la cita eliminada; los demás pagos se conservan y la deuda se recalcula. Si alguno de los pagos estaba incluido en un cierre de caja, el aviso lo indica y se conserva el registro histórico de ese cierre. Cancelar una cita sigue siendo una opción distinta.
* Historia clínica después de eliminar: si era la única cita del paciente, se retiran su número de historia clínica y su sede de origen. Si quedan otras citas, se mantienen. Los números retirados no se asignan a otro paciente. Si la información cambia mientras se revisa la confirmación, el sistema pide revisar el aviso actualizado antes de eliminar.
* Revisión antes de publicar: se incorporaron controles automáticos para comprobar los cambios antes de ponerlos en uso. La aprobación y la publicación de las actualizaciones continúan bajo control manual.

Verificación realizada:

* Se probaron paquetes con un precio distinto al sugerido, varios abonos, cambios de horario, sesiones disponibles y reservas simultáneas. También se revisaron los cobros y comprobantes del reporte y los permisos para eliminar citas.
* Las comprobaciones automáticas de esta entrega finalizaron correctamente. Se utilizaron datos de prueba; queda pendiente la revisión final con el equipo de la clínica al publicar la actualización.

Cambios pendientes:

* Publicar la actualización y realizar el ajuste de la base de datos que la acompaña. Estas acciones no se ejecutaron durante esta sesión.
* Comprobar con el equipo de la clínica un paquete de 20 sesiones, sus pagos, las exportaciones por área y los avisos de eliminación antes de comenzar a operar con la actualización.
* Evaluar con los dueños si el reporte debe incluir más adelante el precio total del procedimiento y el saldo pendiente. Por ahora se mantiene únicamente COSTO como importe cobrado en cada cita.

---

Seguridad del sistema — revisión y pendientes:

* Se revisó el aviso de seguridad de Railway, el proveedor donde funciona el sistema, y las claves de acceso disponibles en la configuración local. Se identificó una clave con acceso amplio a la información de Supabase, donde se guarda la base de datos. No se confirmó que la base de datos haya sido consultada por terceros ni se pudo comprobar qué claves estaban cargadas en Railway.
* Se recomendó reemplazar de inmediato las claves privadas que hayan estado en Railway y dejar de aceptar las anteriores, aunque sea necesario un periodo de mantenimiento. Apagar la aplicación por sí solo no bloquea una clave que permita entrar directamente a la base de datos. También corresponde revisar las claves de consulta de DNI y RUC de VerificaPE y cualquier otra credencial privada utilizada allí.
* Google y correo solo requieren esta revisión si sus claves estaban configuradas en Railway. Que aparezcan como opciones en el archivo de ejemplo no confirma que se estén usando.
* Quedó pendiente ejecutar el cambio de claves, revisar los registros de actividad y comprobar el funcionamiento del sistema. Durante esta sesión no se cambiaron credenciales ni se detuvieron servicios.
