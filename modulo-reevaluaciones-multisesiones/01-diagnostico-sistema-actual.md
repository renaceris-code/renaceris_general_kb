# Diagnóstico del sistema actual — Reevaluaciones y multisesiones

Estado del código al 03/09/2026. Este documento describe **cómo está modelado hoy** el ciclo consulta → proforma → orden → sesión, y dónde exactamente se rompe cada uno de los seis puntos del modelo de negocio. El modelo propuesto está en [02-modelo-funcional-propuesto.md](02-modelo-funcional-propuesto.md).

## 1. Qué se revisó

- Migraciones `006` a `022` en `renaceris_backend/migrations/`.
- Módulos backend `appointments`, `proformas`, `service-orders`, `services`, `accounting`, `reports`.
- Pantallas `Appointments.tsx`, `NewAppointment.tsx`, `AppointmentDetailModal.tsx`, `Services.tsx`, `PatientProfile.tsx`.
- Schema tipado `renaceris_backend/src/interfaces/supabase-schema.ts`.

## 2. Piezas del modelo vigente

| Entidad | Rol real hoy | Campos que gobiernan el comportamiento |
|---|---|---|
| `appointments` | Acto físico en la agenda | `appointment_type`, `category`, `service_id`, `service_order_id`, `treatment_group_id`, `total_amount`, `amount_paid`, `procedure_status` |
| `proformas` | Cotización clínica de un tratamiento | `appointment_id`, `treatment_service_id`, `price_no_modality`, `price_modality_1`, `price_modality_2` |
| `proforma_sales_quotes` | Cotización comercial libre (modalidad personalizada) | `amount`, `description` |
| `service_orders` | Contrato económico del tratamiento | `proforma_id`, `service_id`, `modality`, `total_amount`, `paid_amount`, `status`, `source`, `pricing_status` |
| `payments` | Cobro confirmado | `appointment_id` **o** `service_order_id`, `confirmed`, `payment_date` |
| `services` + `service_prices` | Catálogo y precio por tipo de cita | `appointment_type`, `referential_price`, índice único `(service_id, appointment_type)` |

Enumeraciones vigentes:

- `appointment_type_enum`: `consultation`, `treatment`, `direct_procedure`. Es el eje que decide precio, permiso para emitir proforma y ruta del dinero.
- `appointment_category`: `doctor_consultation`, `treatment`, `laser`, `reevaluation`, `pre_surgery_exam`, `dr_heider`. **Hoy es únicamente color de agenda**, sin ninguna regla de negocio detrás.
- `service_modality_enum`: `Sin modalidad`, `Modalidad 1`, `Modalidad 2`, `Cotización personalizada`.

Un servicio tiene además una naturaleza implícita, derivada de qué filas de `service_prices` tiene: **consulta + tratamiento** (flujo con proforma) o **procedimiento directo** (precio único, sin proforma ni orden). Se configura en `Services.tsx` mediante el campo `service_nature`.

## 3. Las siete reglas duras que definen el sistema

| # | Regla vigente | Dónde vive |
|---|---|---|
| R1 | Una proforma solo puede nacer de una cita de tipo `consultation` | `proformas.service.ts:183`, repetida en el RPC `create_service_order_from_proforma` de la migración `020` |
| R2 | La consulta debe estar `in_progress` o `concluded` para emitir proforma | `proformas.service.ts:186` |
| R3 | Máximo 3 proformas por consulta y una sola por tratamiento | `proformas.service.ts:201`, trigger `enforce_appointment_proforma_limit`, índice único `(appointment_id, treatment_service_id)` |
| R4 | Una proforma admite como máximo una orden de servicio no cancelada | índice único parcial `uq_service_orders_active_proforma`, migración `007` |
| R5 | Una cita de tratamiento **exige** `service_order_id`, y el servicio de la cita debe ser idéntico al de la orden | `appointments.service.ts:164` y `:185` |
| R6 | En una cita de tratamiento el dinero vive en la orden: la cita se guarda con `total_amount = null` y `amount_paid = 0` | `appointments.service.ts`, construcción de `appointmentPayload` |
| R7 | La interfaz permite un solo pago por cita de sesión; el resto se registra en Gestión de Órdenes | `AppointmentDetailModal.tsx:291` |

R1 y R5 son los dos candados que explican casi todo lo que falta.

## 4. Lectura punto por punto del modelo de negocio

### Punto 3 — Varias sesiones con una sola proforma (masajes) · parcialmente soportado

Lo que **ya funciona**: nada impide que varias citas apunten al mismo `service_order_id`. El índice antiduplicados solo bloquea la tripleta paciente + servicio + fecha-hora exacta, así que un paquete de diez masajes puede agendarse como diez citas de tratamiento contra una única orden. Los pagos se acumulan correctamente en la orden vía `recalculateServiceOrderPaidAmount`.

Lo que **falta**:

1. No existe el concepto de paquete de N sesiones. La orden no sabe cuántas sesiones vendió.
2. No hay contador de sesiones usadas ni restantes, ni bloqueo al agendar la sesión 11 de un paquete de 10.
3. No hay vigencia del paquete ni política para sesiones no consumidas.
4. Una sesión cancelada o no-show no se distingue de una consumida.
5. No hay agendamiento en bloque: recepción crea diez citas a mano.
6. El catálogo de servicios no declara que un servicio sea multisesión, así que la interfaz no puede adaptarse sola.
7. Contablemente todo el ingreso del paquete se reconoce contra la orden en la fecha de pago. No hay forma de saber cuántas sesiones quedan **por entregar**, que es una obligación real de la clínica.

### Punto 4 — Varias sesiones con varias proformas (láser CO₂, tatuajes) · bloqueado

Este es el caso que el sistema **no puede representar**. La sesión 2 necesita su propia proforma, pero R1 exige que toda proforma cuelgue de una cita de consulta, y la sesión 2 no es una consulta. Hoy solo hay dos salidas, ambas malas:

- **Fingir una consulta** para poder cotizar. Infla el conteo de consultas en reportes, aplica el precio de consulta y contamina cualquier lógica futura de vigencia.
- **Registrar la sesión como tratamiento** de la orden anterior. No permite cotizar ni cobrar la sesión nueva a un precio distinto.

Además R5 obliga a que el servicio de la cita sea el mismo de la orden, y R3 limita a tres proformas por cita. Ambas reglas fueron calibradas para consultas y quedan mal dimensionadas para una serie de láser de seis u ocho sesiones.

### Punto 5 — Reevaluaciones puras (cirugía) · no existen como tipo

`reevaluation` solo vive en `appointment_category`, que es decorativa. No hay un tipo de cita que signifique "evalúo, no cobro, no cotizo, no ejecuto". Para agendar un control post-operatorio hoy hay que elegir entre:

- Crear una **consulta**, con su precio de consulta asociado y su impacto en reportes.
- Crear un **tratamiento**, que por R5 exige una orden. Esto es literalmente lo que hace el sistema para pacientes operados antes de su implantación: se crea una orden `legacy_manual` solo para tener dónde colgar la reevaluación. En `NewAppointment.tsx:1113` se fija `category = 'reevaluation'` y se precarga la nota "Reevaluación de tratamiento iniciado antes del sistema".

Tampoco existe la serie de controles planificados de una cirugía (7 días, 1 mes, 3 meses), ni un vínculo formal entre la reevaluación y la cirugía que la originó. `treatment_group_id` existe en `appointments` y `proformas` pero está prácticamente sin uso: solo lo consultan la resolución de proformas y el listado de órdenes.

### Punto 6 — Una reevaluación que termina generando otro tratamiento · solo dentro de una consulta

La capacidad de cotizar un tratamiento distinto al de la cita **sí existe**: la migración `020` agregó `proformas.treatment_service_id` y permite hasta tres tratamientos cotizados en la misma cita, con la restricción de que el tratamiento pertenezca a la sede de la cita. Eso es lo que el equipo identificó como ya contemplado.

Pero está atado a R1. El caso real del negocio, rinoplastia → reevaluación a los cuatro meses → bótox, no se puede registrar porque esa reevaluación no es una consulta. La funcionalidad existe; le falta el punto de entrada.

### Punto 2 — Servicios de una sola sesión (criolipólisis) · soportado con fricción

El primer ciclo funciona completo: consulta → proforma → orden → cita de tratamiento. La fricción aparece en la segunda sesión un mes después: como necesita su propia proforma, vuelve a chocar con R1 y obliga a registrar otra consulta. Es el mismo bloqueo del punto 4, con menor frecuencia.

La alternativa actual es configurar el servicio como **procedimiento directo**, que cobra en la cita y no genera proforma ni orden. Resuelve el agendamiento pero pierde la trazabilidad de la cotización, las modalidades de precio y los pagos en cuotas.

### Punto 1 — Consulta inicial y vigencia · inexistente

No hay ningún rastro de este concepto en el código. El precio de consulta se resuelve por servicio desde `service_prices` y cada cita de consulta se cobra por separado. No existe:

- un crédito o vigencia de consulta a nivel de paciente,
- un motivo de exoneración auditable cuando no se cobra,
- ni una marca que distinga "consulta gratuita por vigencia" de "consulta gratuita por campaña" o de "consulta impaga".

Como el beneficio es **del paciente y no del servicio** —paga consulta por rinoplastia y meses después consulta por bótox— no puede modelarse sobre `services` ni sobre `service_prices`. Necesita una entidad propia a nivel de paciente.

## 5. Deuda conceptual: `appointment_category` mezcla cuatro ejes

Los valores actuales no pertenecen a la misma dimensión:

| Valor | Qué es en realidad |
|---|---|
| `doctor_consultation` | Tipo de cita |
| `treatment` | Tipo de cita |
| `laser` | Área clínica o equipo |
| `reevaluation` | Propósito de la visita |
| `pre_surgery_exam` | Propósito de la visita |
| `dr_heider` | Persona o recurso |

Cualquier regla de negocio construida sobre este campo será frágil. La recomendación es dejar `category` como etiqueta visual de agenda y mover el propósito de la visita al eje de tipo de cita, que es el que ya gobierna precio, proforma y dinero.

## 6. Efectos colaterales que hoy no se ven y se activarán

- **Reportes**: `reports.service.ts` cuenta atenciones por `appointment_type`. Introducir reevaluaciones sin tocar esa lógica las contará como consultas o las dejará fuera del conteo.
- **Contabilidad**: el módulo contable reconoce ingreso por pago confirmado contra `payments`. Con paquetes multisesión aparece una obligación nueva, sesiones cobradas y no entregadas, que hoy no se puede reportar.
- **Cuentas por cobrar**: una serie de láser abandonada a mitad de camino deja órdenes con saldo pendiente que nunca se cobrará y que inflan el resultado de `GET /accounting/receivables`.
- **Productividad por especialista**: el ingreso de un paquete se registra en la fecha de pago, no en la fecha de cada sesión, así que la sesión ejecutada por un especialista estético no tiene ingreso atribuible.
- **Duración y capacidad de agenda**: `services` solo tiene `consultation_duration_minutes` y `full_service_duration_minutes`. Una reevaluación de diez minutos y una sesión de láser de cuarenta y cinco competirían por el mismo slot mal dimensionado.
