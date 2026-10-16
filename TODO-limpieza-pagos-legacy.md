# TODO — Limpieza de pagos legacy en citas (Renaceris)

**Propósito:** dejar `payments` como única fuente de verdad de lo cobrado y retirar las tablas y columnas de pago que quedaron del modelo anterior a la migración 007.
**Fecha:** 28/09/2026
**Origen:** [REN-36](https://linear.app/sprinta/issue/REN-36): una cita sin pagos aparecía como "con pagos" porque el formulario leía `appointments.amount_paid`.

**Leyenda:** `- [x]` hecho · `- [ ]` pendiente · **Eliminar** / **Eliminar tras backfill** / **Mantener** / **Renombrar** = veredicto.

---

## Fuente de verdad

| Dato | Dónde vive |
|---|---|
| Pagos de una cita | `payments` con `appointment_id` |
| Pagos de una orden de servicio | `payments` con `service_order_id` |
| Precio de la cita | `appointments.total_amount` (+ `total_amount_updated_by/at` y `appointment_amount_changes`) |
| Precio y saldo de una orden | `service_orders.total_amount` / `paid_amount` |
| Quién tocó cada pago | `payments.created_by/updated_by` y `payment_changes` |

> ⚠️ **No recalcular `appointments.amount_paid` desde `payments` de forma masiva.** Antes de la 007, una cita pagada al contado guardaba el pago **solo en la cita** (`amount_paid` + `payment_confirmed` + `payment_method`…); `appointment_payments` era únicamente para cuotas. Un recálculo masivo deja esas citas en 0 y borra el único registro de esos cobros. Primero va el backfill (Fase 1).

---

## Veredicto por elemento

| Elemento | Veredicto | Por qué | Se puede quitar cuando… |
|---|---|---|---|
| Tabla `appointment_payments` | **Eliminar** | La 007 copió todo a `payments` (mismo `id`). Ningún código la usa y ya no aparece en los tipos generados, así que puede que ya no exista. | La consulta 0.1 devuelve 0 filas faltantes |
| Tabla `appointment_payments_backup_temp` | **Eliminar** | Respaldo temporal. Solo aparece en la lista RLS de la 015. | La consulta 0.1 devuelve 0 filas faltantes |
| `appointments.amount_paid` | **Eliminar tras backfill** (último) | Es un acumulado que se guarda aparte y quedó corrompido por el formulario de edición antiguo. Hoy lo leen la lista, el calendario, el dashboard, `paid/list`, el ingreso de caja, las cuentas por cobrar, el perfil del paciente y la RPC `link_appointment_to_service_order` (B6); la RPC `create_direct_sale_order_for_appointment` lo escribe. | Fases 1 a 3 cerradas |
| `appointments.payment_confirmed` | **Eliminar tras backfill** | Solo tenía sentido para el pago al contado legacy; hoy vive en `payments.confirmed`. Además causa el bug B2: el dashboard pierde los pagos registrados después de crear la cita. | Fase 2 cerrada |
| `appointments.payment_method`, `billing_type`, `billing_number`, `transaction_code` | **Eliminar tras backfill** | Duplican datos de `payments`: `create()` los sigue copiando a la cita al registrar el pago inicial. Solo aportan información en las citas legacy, que se trasladan con el backfill. | Fases 1 y 3 cerradas |
| `appointments.payment_type` (`full`/`installments`) | **Eliminar** | Ninguna pantalla lo escribe; solo lo usa el filtro legacy de `getCashIncome`. | Fase 2 cerrada |
| `CreateAppointmentDto.amount_paid` + `payment_*`/`billing_*`/`transaction_code` | **Renombrar** | En la creación `amount_paid` significa "pago inicial", no "total pagado". Pasar a un objeto `initial_payment { amount, payment_method, … }`. | Fase 3 |
| `UpdateAppointmentDto.amount_paid`, `payment_method`, `payment_confirmed`, `billing_*`, `transaction_code` | **Eliminar** | `amount_paid` ya se ignora (REN-36). El resto reescribe datos legacy desde el formulario de edición. Se mantienen solo para no romper clientes viejos (`forbidNonWhitelisted`). | El frontend deja de enviarlos (Fase 3) |
| `formData.amount_paid` en `Appointments.tsx` | **Renombrar** a `total_amount` | Ese campo del formulario es el "Monto Total a Percibir"; el nombre causó el bug original. | Fase 3 |
| Prop `onPaymentConfirm` de `AppointmentDetailModal` | **Eliminar** | Nadie la pasa ni la usa. | Ya |
| `ModernCalendar.tsx` | **Eliminar** | Ningún archivo lo importa, y muestra `amount_paid` como precio. | Ya |
| `appointments.total_amount` | **Mantener** | Fuente de verdad del precio de la cita. | — |
| `service_orders.paid_amount` | **Mantener** (por ahora) | También es un acumulado de `payments`, pero siempre se recalcula y no tiene historial corrupto. Se puede evaluar una vista más adelante. | — |

---

## Bugs de "origen equivocado" encontrados

- [x] **B0** — El formulario "Editar Planificación" guardaba el monto total en `amount_paid` y usaba `amount_paid > 0` para bloquear la edición. Ahora consulta `payments` (REN-36).
- [x] **B1** — `getCashIncome` contaba dos veces las citas creadas con pago inicial: una vez por su fila en `payments` y otra por `payment_confirmed` + `amount_paid`. Ahora el bloque de pagos al contado ignora las citas que tienen filas en `payments` (REN-36, test en `reports.service.spec.ts`).
- [ ] **B2** — El dashboard (`getDashboardStats`) calcula `todayRevenue` y `weekRevenue` con `appointments.payment_confirmed = true`, pero `addPayment` nunca marca ese campo. Los pagos registrados después de crear la cita no cuentan, y los de tratamientos tampoco (van a la orden). Hay que decidir si el ingreso se cuenta por fecha del pago (lo correcto contablemente) o por fecha de la cita.
- [ ] **B3** — En `getCashIncome`, el saldo pendiente excluye las citas con `payment_confirmed = true` aunque tengan saldo (por ejemplo, pago inicial parcial).
- [ ] **B4** — La lista de citas (`Appointments.tsx`, vista tarjeta y tabla) y `Calendar.tsx` muestran `S/ amount_paid` como si fuera el precio de la cita.
- [ ] **B6** — La RPC `link_appointment_to_service_order` (versión vigente en la 036) impide ejecutar el tratamiento en la visita si `total_amount > amount_paid`. Con un `amount_paid` corrupto bloquea o deja pasar cuando no corresponde. Debe sumar `payments` confirmados de la cita.
- [ ] **B5** — Contabilidad (ingresos) solo lee `payments`, así que no incluye los pagos al contado legacy. Se resuelve con el backfill (ya anotado en `TODO-modulo-contabilidad.md`, sección 1a).

---

## Fase 0 — Diagnóstico (solo lectura)

- [ ] **0.1** Verificar si las tablas legacy siguen en la base y si están completamente copiadas:
  ```sql
  select to_regclass('public.appointment_payments') as appointment_payments,
         to_regclass('public.appointment_payments_backup_temp') as backup_temp;

  -- Repetir con appointment_payments si existe. Debe dar 0.
  select count(*) as faltantes
  from appointment_payments_backup_temp b
  where not exists (select 1 from payments p where p.id = b.id);
  ```
- [ ] **0.2** Listar las citas cuyo `amount_paid` no coincide con `payments`, clasificadas:
  ```sql
  with real as (
    select a.id, a.created_at, a.payment_confirmed, a.amount_paid, a.total_amount,
           count(p.id) as filas_payments,
           coalesce(sum(p.amount) filter (where p.confirmed), 0) as pagado_real
    from appointments a
    left join payments p on p.appointment_id = a.id
    group by a.id
  )
  select *,
         case
           when filas_payments = 0 and payment_confirmed then 'legacy_contado'   -- candidato a backfill
           when filas_payments = 0 then 'sin_pagos_amount_paid_sucio'            -- probable bug del formulario
           else 'desincronizado'
         end as clasificacion
  from real
  where amount_paid is distinct from pagado_real
  order by created_at;
  ```
- [ ] **0.3** Separar las citas corrompidas por el formulario de edición de los pagos legacy reales. Son las ediciones cuyo `audit_logs` cambió `amount_paid`:
  ```sql
  select entity_id as appointment_id, created_at, user_id,
         changes->'before'->>'amount_paid' as antes,
         changes->'after'->>'amount_paid'  as despues
  from audit_logs
  where entity_type = 'appointment' and action = 'UPDATE'
    and changes->'before'->>'amount_paid' is distinct from changes->'after'->>'amount_paid'
  order by created_at;
  ```
  Una cita `legacy_contado` que aparece aquí **no** es un cobro real: el valor lo escribió el formulario. Validar con recepción los casos dudosos.

## Fase 1 — Backfill y saneamiento de datos

- [ ] **1.1** Migración `048`: pasar a `payments` los pagos al contado legacy **revisados** en la Fase 0. Se fechan con la cita para que los reportes por fecha los ubiquen en su día:
  ```sql
  insert into payments (appointment_id, amount, payment_method, transaction_code,
                        billing_type, billing_number, notes, confirmed, payment_date, created_at)
  select a.id, a.amount_paid, a.payment_method, a.transaction_code,
         a.billing_type, a.billing_number,
         'Pago al contado migrado desde la cita (previo a payments)', true, a.datetime, a.datetime
  from appointments a
  where a.id = any(:ids_revisados)
    and not exists (select 1 from payments p where p.appointment_id = a.id);
  ```
  El trigger `payments_track_changes` registra cada fila como `create` con `changed_by` NULL; así queda identificado el backfill.
- [ ] **1.2** Solo después de 1.1, recalcular `amount_paid` de las citas con la clasificación `sin_pagos_amount_paid_sucio` o `desincronizado`. Con eso, 0.2 debe devolver 0 filas.

## Fase 2 — Que todos los lectores usen `payments`

- [ ] Lista y calendario de citas (B4): mostrar `total_amount` como precio y lo pagado desde `payments` (o `getPayments`).
- [ ] `getDashboardStats` (B2): ingresos desde `payments` (incluye pagos a órdenes), filtrados por sede de la cita o de la orden.
- [ ] `findPaidAppointments` (`GET /appointments/paid/list`): **Eliminar**. El frontend no lo usa y filtra por `payment_confirmed`.
- [ ] RPC `link_appointment_to_service_order` (B6): nueva migración que calcule el saldo con `payments` confirmados de la cita en lugar de `amount_paid`.
- [ ] `getCashIncome`: eliminar el bloque de pagos al contado (paso 2) y la condición `payment_confirmed` del pendiente (B3). Tras el backfill, todo está en `payments`.
- [ ] `accounting.service.ts` `getReceivables`: calcular lo pagado de las citas desde `payments`.
- [ ] `patients.service.ts` `getFullProfile`: quitar `amount_paid`/`payment_confirmed` del select, o reemplazarlos por el total de `payments`.

## Fase 3 — Dejar de escribir en los campos legacy

- [ ] `appointments.service.ts` `create()`: no copiar `amount_paid`, `payment_confirmed`, `payment_method`, `billing_*` ni `transaction_code` a la cita. El pago inicial solo va a `payments`.
- [ ] Reemplazar esos campos de `CreateAppointmentDto` por `initial_payment { amount, payment_method, transaction_code, billing_type, billing_number }`. Ajustar `Appointments.tsx` y `NewAppointment.tsx`.
- [ ] Formulario de edición: dejar de enviar `payment_method`, `payment_confirmed`, `billing_*` y `transaction_code`; después quitarlos de `UpdateAppointmentDto`.
- [ ] Renombrar `formData.amount_paid` a `formData.total_amount` en `Appointments.tsx`.
- [ ] Quitar `onPaymentConfirm` de `AppointmentDetailModal` y borrar `ModernCalendar.tsx`.

## Fase 4 — Eliminar del esquema

- [ ] Migración que elimina `appointment_payments` y `appointment_payments_backup_temp` (depende de 0.1).
- [ ] Recrear `create_direct_sale_order_for_appointment` (versión vigente en la 032) sin `amount_paid = 0`, en la misma migración o antes.
- [ ] Migración que elimina `appointments.amount_paid`, `payment_confirmed`, `payment_method`, `billing_type`, `billing_number`, `transaction_code` y `payment_type`. Antes, confirmar que ninguna otra RPC o vista los use (`grep` en `migrations/`, `select * from pg_views` y `pg_proc.prosrc`). Hoy los usan: 029/032 (`create_direct_sale_order_for_appointment`) y 034/036 (`link_appointment_to_service_order`).
- [ ] Regenerar `src/interfaces/supabase-schema.ts` y quitar los campos de las interfaces del frontend (`Appointment`, `DetailModalAppointment`, `Calendar`, `DoctorPanel`).
- [ ] Desplegar en este orden: 1) frontend y backend que ya no leen ni escriben los campos; 2) la migración de borrado.
