# Corrección de cierres POS y pagos de recepción — 07/10/2026

La primera versión de 048 ya había creado las tablas. Agregar columnas dentro de `CREATE TABLE IF NOT EXISTS` no modifica esas tablas: el backend actualizado consultaba `pos_cash_closures.surplus_reason` y `accounting_incomes.kind`, pero esas columnas no existían en la instalación anterior.

El SQL incremental está en `renaceris_backend/scripts/049_repair_pos_and_reception_payments.sql`. Agrega las columnas, restricciones e índices, actualiza las funciones de venta/cierre/validación y la RPC de órdenes, y recarga el esquema de PostgREST. La 048 también admite ahora instalaciones anteriores sin estas dos columnas.

## Cambios implementados

- Voucher privado JPG, PNG, WEBP o PDF, hasta 5 MB, en cita nueva, detalle de cita y pagos/anticipos de Gestión de Órdenes. Vista previa de imágenes y enlaces firmados en el historial. Un fallo al subir el archivo no duplica el pago: se puede adjuntar después.
- Los pagos iniciales de paquetes y tratamientos se vinculan a su cita y orden. Los anticipos de aceptación de proforma guardan el cobrador en la misma transacción que crea el pago.
- La UI usa la sede real: Yape Huánuco → BCP Huánuco; Plin y tarjeta → BBVA de la sede; transferencia exige elegir BCP o BBVA de la sede; efectivo → caja general de la sede y sin código de transacción. Las transferencias históricas sin banco permanecen por definir.
- Los destinos confirmados se asignan al validar el cierre. Si el banco de la sede no existe o es ambiguo, la validación falla sin generar ingresos parciales. Las reasignaciones incompatibles se rechazan, conservando un audit correcto.
- Los vouchers de citas y órdenes son opcionales y pueden adjuntarse después. El cierre exige vouchers para ventas de productos con Yape, Plin o transferencia. Los reportes históricos conservan sus importes; SQL bloquea editar o borrar pagos incluidos en cierres cerrados/validados.
- Los formularios de `Appointments.tsx`, `NewAppointment.tsx`, detalle y órdenes comparten métodos separados y destino bancario. Los vouchers seleccionados y guardados muestran imágenes grandes y responsivas (o PDF), con apertura en tamaño completo.
- Recepción puede editar o eliminar sus propios pagos hasta 30 minutos después de `created_at`. El backend y la migración 050 verifican el plazo; la UI desactiva la edición cuando vence. Administración conserva su acceso, sujeto al bloqueo de cierres. Adjuntar un voucher después no altera monto, método ni fecha de registro.
- Lectura de vouchers de recepción limitada a sus propios cobros; administración y contabilidad acceden según sede. Conteos paginados para evitar truncamiento de evidencias.

## Qué cobros no indicaban quién cobró

El flujo defectuoso identificado es el **anticipo o pago inicial registrado al aceptar una proforma y generar una orden de servicio**. La RPC insertaba `payments` sin `created_by`. El backend asignaba el creador únicamente a la orden, después de ejecutar la RPC. Los pagos posteriores desde `addPayment` de citas y de Gestión de Órdenes ya registraban actor.

La corrección evita nuevos casos. No se atribuyeron históricos al creador de la orden porque eso no demuestra quién recibió el dinero. No fue posible consultar los IDs reales: la conexión de lectura a Supabase falló también fuera del sandbox.

Consulta para enumerar todos los pagos confirmados sin cobrador, incluidos los históricos de otros flujos:

```sql
SELECT p.id AS payment_id, p.service_order_id, p.appointment_id,
       p.created_at, p.amount, p.payment_method, p.notes,
       l.name AS sede
FROM public.payments p
LEFT JOIN public.appointments a ON a.id = p.appointment_id
LEFT JOIN public.service_orders o ON o.id = p.service_order_id
LEFT JOIN public.services s ON s.id = o.service_id
LEFT JOIN public.locations l ON l.id = COALESCE(a.location_id, s.location_id)
WHERE p.created_by IS NULL AND p.confirmed IS TRUE AND p.amount > 0
ORDER BY p.created_at DESC, p.id;
```

## Despliegue y verificación

1. Con 048 ya instalada, ejecutar **049_repair_pos_and_reception_payments.sql** en el SQL editor de Supabase. No es necesario borrar tablas ni reaplicar todo el esquema.
2. Ejecutar **050_card_destination_and_payment_edit_window.sql** después de 049: destino BBVA de tarjeta y guardia SQL del plazo de edición. Requiere la trazabilidad de pagos de la migración 047 (`updated_by` y RPC de eliminación con actor).
3. Desplegar/reiniciar el backend y desplegar el frontend, ambos compilados.
4. Comprobar que la consulta `SELECT surplus_reason FROM public.pos_cash_closures LIMIT 1` y `SELECT kind FROM public.accounting_incomes LIMIT 1` funcionan; registrar pagos de prueba en cada sede, adjuntar voucher, cerrar y validar.

Verificación realizada: compilaciones backend/frontend; pruebas de citas, órdenes, POS, ingresos y permisos de vouchers; pruebas del carrito frontend; SQL ejecutado en PostgreSQL 18 desechable, con instalación antigua simulada, reparación repetida, anticipo con actor, normalización de métodos, rutas BCP/BBVA, rechazo de banco incorrecto y bloqueo de edición/eliminación de cobros cerrados.

Verificación del ajuste posterior: 90 pruebas backend y 11 frontend aprobadas, ambas compilaciones correctas; 050 ejecutada dos veces en PostgreSQL 18 y comprobado el destino BBVA de tarjeta, edición a los 29 minutos, rechazo de edición/eliminación a los 31 minutos y excepción administrativa. Caso reproducible: `renaceris_backend/test/payment-edit-window.integration.sql`.

Las migraciones 049 y 050 **no se aplicaron al Supabase remoto**: esta sesión no dispone de conexión SQL a ese proyecto y la consulta REST falló. El error de la instalación reportada desaparecerá después del paso 1 y del reinicio del backend.
