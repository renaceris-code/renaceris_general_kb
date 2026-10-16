# Módulo de Reevaluaciones y Multisesiones — Renaceris

Diseño del módulo que separa el tipo de cita, la cantidad de sesiones y la generación de proformas. Cubre reevaluaciones puras, reevaluación con tratamiento, paquetes de varias sesiones con una sola proforma y series con una proforma por sesión.

## Documentos

| Documento | Contenido |
|---|---|
| [00-modelo-de-negocio-fuente.md](00-modelo-de-negocio-fuente.md) | Lectura del negocio tal como la planteó la clínica. Insumo original |
| [01-diagnostico-sistema-actual.md](01-diagnostico-sistema-actual.md) | Cómo está modelado hoy y dónde se rompe cada punto, con referencias al código |
| [02-modelo-funcional-propuesto.md](02-modelo-funcional-propuesto.md) | Esquema objetivo: entidades, tipos de cita, configuración por servicio y flujos |
| [03-catalogo-de-escenarios.md](03-catalogo-de-escenarios.md) | Sesenta escenarios con su regla propuesta, incluidos los no previstos en la conversación |
| [04-plan-de-implementacion.md](04-plan-de-implementacion.md) | Cinco fases con migraciones, cambios de backend y frontend, pruebas y riesgos |
| [05-decisiones-pendientes.md](05-decisiones-pendientes.md) | Definiciones que necesita el cliente, con recomendación por defecto |
| [06-modelo-de-negocio-final.md](06-modelo-de-negocio-final.md) | **Cierre.** Modelo de negocio decidido para las dos features, validado contra el esquema. Solo quedan tres preguntas abiertas |
| [07-plan-de-ejecucion.md](07-plan-de-ejecucion.md) | **Plan vigente.** Cinco fases con migraciones, backend, frontend, pruebas y matriz por rol. Sustituye al 04 donde difieran |

## Resumen ejecutivo

**El bloqueo principal.** Una proforma solo puede nacer de una cita de tipo consulta. La regla está escrita tres veces: en el servicio de proformas, en el procedimiento almacenado que crea la orden y en la interfaz. Mientras siga así, las series de láser CO₂ y de eliminación de tatuajes no se pueden representar, y una reevaluación que decide un bótox obliga a registrar una consulta falsa.

**Lo que ya funciona mejor de lo que parece.** Varias citas pueden apuntar a la misma orden de servicio, así que el paquete de masajes es representable hoy en su estructura básica. Lo que falta es que la orden sepa cuántas sesiones vendió y que el sistema controle cuántas se entregaron.

**Lo que no existe.** La reevaluación no es un tipo de cita, solo un color de agenda. Para agendar un control post-operatorio hay que crear una consulta, que cobra y ensucia los reportes, o una orden ficticia que sirva de percha. Tampoco existe ninguna forma de vigencia de consulta.

**Cambio conceptual central.** Se agrega `reevaluation` como cuarto tipo de cita, y "reevaluación y tratamiento" queda como estado derivado, no como tipo propio: una visita empieza siendo una reevaluación y solo a mitad de la atención se sabe si habrá tratamiento.

| Tipo de cita | Evalúa | Ejecuta | Puede cotizar | Exige orden | Precio propio | Cuenta como consulta |
|---|---|---|---|---|---|---|
| Consulta | Sí | Vía orden de la misma visita | Sí, hasta 3 | No | Sí | Sí |
| Reevaluación | Sí | No | Sí | No | No | No |
| Reevaluación y tratamiento | Sí | Sí | Sí | La crea | No | No |
| Sesión de tratamiento | No | Sí | No | Sí | No | No |
| Procedimiento directo | No | Sí | No | No | Sí | No |

## Prioridad

El negocio priorizó del punto 3 en adelante. Las fases siguen ese orden:

1. Episodios y paquetes multisesión, el caso de masajes.
2. Reevaluación como tipo de cita, el caso de las cirugías.
3. Cotización desde una reevaluación, que desbloquea láser, tatuajes y el bótox tras la rinoplastia.
4. Configuración por servicio, para que el comportamiento viva en el catálogo y no en el código.
5. Vigencia de consulta, sujeta a definiciones abiertas con Ronald.

## Tres decisiones que conviene cerrar primero

1. ¿El no-show y el vencimiento consumen sesiones de un paquete?
2. ¿Cuánto dura el seguimiento incluido de una cirugía y qué pasa después de esa ventana?
3. ¿La vigencia de consulta es de 12 meses desde la consulta pagada, en lugar del año calendario?

Las tres están desarrolladas, con recomendación, en [05-decisiones-pendientes.md](05-decisiones-pendientes.md).
