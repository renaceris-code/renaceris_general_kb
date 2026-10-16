# Decisiones pendientes — Reevaluaciones y multisesiones

Definiciones que necesita el cliente para cerrar el módulo. Cada una trae una recomendación por defecto, para que la ausencia de respuesta no detenga la implementación. Están ordenadas por la fase que condicionan, según [04-plan-de-implementacion.md](04-plan-de-implementacion.md).

## Bloque 1 — Paquetes multisesión · condiciona la fase 1

| # | Pregunta | Opciones | Recomendación | Si no se responde |
|---|---|---|---|---|
| 1.1 | ¿Un no-show consume sesión del paquete? | Sí / No / Configurable por sede | No consume, con registro del no-show | Se implementa "no consume" y queda configurable |
| 1.2 | ¿Los paquetes vencen? ¿En cuánto tiempo? | Sin vencimiento / 6 meses / 12 meses | 12 meses desde la primera sesión | Se implementa el campo sin activarlo |
| 1.3 | Sesiones pagadas y no usadas al vencer | Se pierden / Se extienden con autorización / Se devuelven | Se bloquean y se pueden reactivar con motivo y responsable | Se implementa la reactivación con motivo |
| 1.4 | Masajes incluidos en el precio de una cirugía | Orden aparte con total 0 vinculada a la cirugía / Incluidos sin orden | Orden con total 0 vinculada, para poder controlar las sesiones | Se implementa la orden vinculada |
| 1.5 | ¿Se pueden ejecutar sesiones en una sede distinta a la de la venta? | Sí / No | Sí, registrando la sede real de atención | Se permite y se registra |
| 1.6 | ¿Se permite cambiar de servicio a mitad de paquete? | Sí / No | Sí, cerrando la orden por lo entregado y abriendo otra por el remanente | Queda fuera de la fase 1 |
| 1.7 | Sesión aplicada parcialmente | Cuenta como entregada / Se compensa con una sesión extra | Cuenta como entregada, con compensación manual autorizada | Se implementa así |
| 1.8 | ¿Cuántos tratamientos puede cotizar una misma cita? | Se mantiene 3 / Se eleva | Mantener 3 y volverlo configurable si aparece un caso real | Se mantiene 3 |

## Bloque 2 — Reevaluaciones · condiciona las fases 2 y 3

| # | Pregunta | Opciones | Recomendación | Si no se responde |
|---|---|---|---|---|
| 2.1 | ¿Cuánto dura el seguimiento incluido de una cirugía? | 3 meses / 6 meses / 12 meses / Por tipo de cirugía | Por tipo de cirugía, con un valor por defecto de 6 meses | Todas las reevaluaciones son gratuitas |
| 2.2 | ¿Se cobra la reevaluación fuera de esa ventana? | Sí, precio de consulta / Sí, precio propio / No | Precio propio de reevaluación por servicio | No se cobra |
| 2.3 | Paciente operada en otra clínica que viene a controlarse | Se cobra consulta / Se cobra reevaluación / Gratis | Se cobra como consulta, porque no hay episodio propio | Se cobra consulta |
| 2.4 | Retoque o cirugía de revisión por garantía | Proforma en 0 / Precio de garantía / Caso a caso | Proforma con motivo obligatorio de garantía y precio definido caso a caso | Caso a caso con motivo |
| 2.5 | ¿Existen reevaluaciones virtuales? | Sí / No | Sí, para los servicios que ya admiten consulta virtual | Se habilita según el servicio |
| 2.6 | Esquema de controles post-operatorios | 7, 30 y 90 días / Otro / Por tipo de cirugía | 7, 30 y 90 días como valor por defecto, configurable por servicio | Se usa ese esquema como sugerencia |
| 2.7 | ¿Quién puede cotizar en una reevaluación? | Solo médico y administrador / También especialista estético | Mantener médico y administrador | Se mantiene el permiso actual |
| 2.8 | Intervalo mínimo entre sesiones de láser | Advertencia / Bloqueo | Advertencia, la decisión clínica es del médico | Advertencia |
| 2.9 | ¿El láser se vende también como paquete cerrado con descuento? | Sí / No | Si existe, se modela como paquete y no como serie | Se modela solo como serie |
| 2.10 | Vigencia de una cotización no tomada | Sin vencimiento / 30 días / 60 días | 30 días, con posibilidad de recotizar | Sin vencimiento |

## Bloque 3 — Comercial y contable · condiciona la fase 4

| # | Pregunta | Opciones | Recomendación | Si no se responde |
|---|---|---|---|---|
| 3.1 | Comprobante de un paquete | Uno por el paquete / Uno por sesión | Seguir la práctica actual de SIBI; documentarla antes de tocar el puente | Sin cambio en el puente contable |
| 3.2 | Episodios abandonados con saldo | Siguen en cuentas por cobrar / Pasan a un bloque aparte | Bloque aparte de saldos incobrables revisables | Siguen sumando al pendiente |
| 3.3 | Devoluciones parciales de sesiones no usadas | Sí / No | Registrarlas en el módulo contable con nota de crédito en SIBI | Fuera de alcance |
| 3.4 | ¿Se necesita atribuir ingreso a la fecha de cada sesión? | Sí / No | No para contabilidad; sí como reporte operativo de sesiones entregadas | Solo reporte operativo |

## Bloque 4 — Consulta inicial y vigencia · condiciona la fase 5

Este bloque es el que estaba explícitamente pendiente con Ronald y es el de menor prioridad de implementación.

| # | Pregunta | Opciones | Recomendación | Si no se responde |
|---|---|---|---|---|
| 4.1 | Cómo se calcula la vigencia | Año calendario / 12 meses desde la consulta pagada | **12 meses desde la consulta pagada.** El año calendario castiga a quien consulta en noviembre o diciembre, que fue justamente la objeción levantada | La fase 5 no se implementa |
| 4.2 | Alcance del beneficio | Global del paciente / Por sede / Por especialidad | Global, registrando la sede de cada atención | — |
| 4.3 | ¿La consulta virtual otorga vigencia? | Sí / No | Sí, si se cobró | — |
| 4.4 | ¿La consulta virtual consume vigencia? | Sí / No | No, si existe vigencia activa se atiende sin cobro | — |
| 4.5 | Consulta pagada a la que la paciente no asiste | Se pierde / Se reprograma / Otorga vigencia igual | Otorga vigencia, porque el ingreso ya se percibió | — |
| 4.6 | ¿La consulta se descuenta del tratamiento si opera? | Sí / No | Confirmar. Es práctica común del rubro y cambia el precio final de la orden | Se asume que no se descuenta |
| 4.7 | Pacientes anteriores al sistema | Fecha de corte / Sin vigencia previa | Fecha de corte igual a la primera consulta registrada en el sistema | — |
| 4.8 | ¿Renueva la vigencia cada consulta pagada? | Sí / No | Sí, la última consulta pagada reinicia los 12 meses | — |

## Cómo llevar esto a la reunión

Tres preguntas resuelven la mayor parte del diseño y conviene cerrarlas primero:

1. ¿El no-show y el vencimiento consumen sesiones de un paquete? Define el control de sesiones de la fase 1.
2. ¿Cuánto dura el seguimiento incluido de una cirugía y qué pasa después? Define si la reevaluación es siempre gratuita.
3. ¿La vigencia es de 12 meses desde la consulta pagada? Define toda la fase 5.

El resto puede avanzar con las recomendaciones por defecto y ajustarse después por configuración, sin cambios de esquema.
