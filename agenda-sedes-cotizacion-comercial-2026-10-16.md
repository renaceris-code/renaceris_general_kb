# Agenda por sede, cotización comercial y fin previsto

**Estado:** implementado el 16/10/2026 · **Migraciones:** 051 (cotización comercial) y 052 (fin previsto)
**Commits:** backend `3aebcc0`, `1c9cf91` · frontend `dab6614`, `c18b2de`, `287da1b` (rama `main-version/KCE`)

---

## 1. Reglas

### Sedes

1. El paciente se atiende en cualquier sede. Su sede de primera atención (la de la HC o, sin HC, la de su primera cita) solo filtra la navegación. Ver [historia-clinica-numeracion-y-secciones.md](historia-clinica-numeracion-y-secciones.md).
2. **Admin** y **supervisor** ven todas las sedes. **Admin de sede**, **ventas**, **médicos** y **especialistas estéticos** ven la agenda solo de sus sedes asignadas (`user_locations`); médicos y especialistas, además, solo sus propias citas.
3. Sin sedes asignadas se muestra "Sin sede de trabajo asignada" y listados vacíos. Un error de carga tampoco abre acceso global.
4. Con una sede se preselecciona y se oculta el selector; con varias, "Todas" abarca solo las asignadas. El acceso directo por id y las sedes recibidas por navegación se validan.
5. Al agendar, los pacientes se filtran por defecto por la sede de la cita, con las opciones "Todos los pacientes" y "Sin primera atención". La búsqueda acepta nombre, apellidos, DNI, teléfono e HC.
6. Solo se asignan especialistas activos con la sede de la cita. El backend lo valida al crear, en lotes y al reasignar; una edición que no toca la asignación conserva al especialista histórico.

### Cotización comercial

| Acción | Ventas | Admin | Admin de sede | Recepción |
|---|---|---|---|---|
| Ver proforma desde Órdenes | Sus sedes | Todas | Sus sedes | Igual que hoy |
| Crear o actualizar cotización comercial | Sus sedes | Todas | Sus sedes | No |
| Generar orden de servicio | **No** | Sí | Sus sedes | Sí |

7. Una cotización comercial vigente por proforma, separada de los precios médicos. Cada cambio efectivo crea una versión inmutable con importe, descripción, usuario, rol y fecha. Las cotizaciones anteriores a 051 quedan con rol "no registrado".
8. Con una orden no cancelada, la cotización no se modifica.
9. Al generar la orden se elige **una** alternativa (precio médico o cotización comercial); los importes no se suman. La orden guarda la versión aceptada, su importe y condiciones; `created_by` de la orden es quien la genera, distinto de quien cotizó.
10. Si la cotización cambió mientras se generaba la orden, el backend responde 409 y la pantalla recarga las opciones. Guardar la cotización y generar la orden bloquean la misma proforma, así que no se cruzan.
11. La modalidad pública sigue siendo "Cotización personalizada" para no tocar órdenes existentes.

### Fin previsto

12. `appointments.visual_duration_minutes`: duración visual del calendario, editable en ambos formularios. NULL usa la duración de consulta del servicio o 30 minutos.
13. Debe ser positiva y terminar antes de la medianoche del día de la cita. Al reprogramar se conserva; puede restablecerse.
14. No toca `start_time`, `procedure_started_at` ni `procedure_ended_at`, ni cambia cupos o disponibilidad.

### Observaciones

15. El calendario muestra `notes` de la cita en tarjetas con espacio; en las demás, un indicador accesible abre el texto completo en la vista previa (día, semana y mes). Las notas clínicas no se incluyen.

## 2. APIs

| Endpoint | Uso |
|---|---|
| `GET/PATCH /proformas/:id/commercial-quote` | Cotización vigente con autor, rol y versión |
| `GET /proformas/:id/commercial-quote/versions` | Historial de versiones |
| `GET/PATCH /proformas/:id/sales-quote` | Alias temporal, mismas reglas de rol y sede |
| `POST /service-orders/from-proforma` | Acepta `commercial_quote_version`; sin ella toma la vigente |
| `GET /service-orders/pending-proformas` | Incluye `commercial_quote`; admite proformas con solo cotización comercial |

## 3. Despliegue

1. Ejecutar `renaceris_backend/scripts/audit_sede_assignments.sql` (solo lectura) y asignar sedes a quien aparezca. Al 15/10 faltaban los 4 usuarios de ventas y 5 especialistas de Huánuco (aplicado el 16/10).
2. Aplicar `051_commercial_quotes_and_versions.sql` y `052_appointment_visual_duration.sql` (aplicadas el 16/10).
3. Desplegar backend y frontend juntos: el backend anterior consulta `proforma_sales_quotes`, que deja de existir.
4. Verificar: cotizar desde ventas y desde admin, generar una orden con cotización comercial, agenda de un usuario con una sede y de uno sin sedes.

## 4. Verificación realizada

- Backend: 473 pruebas; frontend: 86 pruebas y build.
- `test/commercial-quotes.integration.sql` sobre PostgreSQL 18 local con esquema sintético: renombrado sin pérdida, migración reentrante, versiones, bloqueo por orden y orden con versión aceptada.
- Concurrencia probada con dos sesiones en ambos sentidos: la segunda operación espera el bloqueo y se rechaza.
- Pendiente: verificación sobre una copia de datos reales.
