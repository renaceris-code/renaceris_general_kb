Cambios implementados:
Día 25/08
* Búsqueda de DNI: los datos del paciente ahora se completan al primer intento. El proveedor recomendó aumentar el tiempo de espera así que ahora el margen de fallos al consultar DNI será mínimo.
* Citas duplicadas: el sistema ya impide crear la misma cita dos veces por doble clic.
Día 26/09:
* Notificaciones al especialista: si una cita se reasigna o cancela, deja de aparecer en su panel. Ya no descubre el error al iniciar el procedimiento.
* Limpieza: se depuraron/eliminaron las citas generadas por las recepcionistas con duplicados producto del bug.

Cambios pendientes:
* Nuevo tipo de cita "Reevaluación - Tratamiento": el especialista reevalúa al paciente en cada sesión y define ahí mismo el precio de esa sesión. Disponible en Láser e INDIBA (Fecha límite 26/08) y BOTOX (Fecha límite 27/08).
* Cobro por sesión: cada sesión genera su propia orden de servicio y su propio pago, en lugar de un pago único por todo el tratamiento.
* La recepcionista podrá agendar la sesión y ver a qué tratamiento del paciente corresponde.