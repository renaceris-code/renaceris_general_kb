# TODO — Módulo Contabilidad (Renaceris)

**Propósito:** checklist por sprint de lo que falta para cerrar el alcance de [`modulo-contabilidad-final.md`](modulo-contabilidad-final.md), basado en una auditoría del código de punta a punta (migración → backend → frontend).
**Fecha:** 23/09/2026
**Proyecto Linear:** [Módulo Contabilidad (REN)](https://linear.app/sprinta/project/modulo-contabilidad-9d9e4c0af62b)

**Leyenda:** `- [x]` hecho y verificado en código · `- [ ]` pendiente · **Completo** / **Parcial** / **Falta** / **Fuera de alcance** = estado de cada feature frente al doc final.

---

## Estado actual por sección (auditoría 23/09)

| # | Sección del doc final | Estado | Evidencia / gap principal |
|---|---|---|---|
| 1a | Reportes de ingresos (fecha, sede, forma de pago, servicio, doctor) | **Completo** (con matices) | `accounting.service.ts` `getSummary`/`buildIncomeEntries`, filtros en `Accounting.tsx`. Filtra por `payments.created_at` y no por `payment_date`; sin fallback de citas pagadas sin fila en `payments`; "tipo de servicio" = servicio puntual, no categoría |
| 1b | Cuentas por cobrar | **Parcial** | `getReceivables` sí filtra por sede/servicio/doctor, pero solo trae órdenes/citas **creadas dentro del rango** (por defecto, hoy): no es el saldo vigente a una fecha de corte |
| 1c | Pagos sin comprobante + inconsistencias | **Falta** | Sin endpoint ni UI. [REN-12](https://linear.app/sprinta/issue/REN-12) bloqueado |
| 1d | Ventas mixtas (efectivo + transferencia) | **Falta** | `payments.payment_method` es un solo texto. [REN-11](https://linear.app/sprinta/issue/REN-11) bloqueado |
| 2 | Egresos categorizados | **Completo** (sin desplegar) | CRUD + validación + evidencias + proveedores RUC/DNI + beneficiario (mig. 012, 014, 038, 040–043). Lo registran admin/admin_sede desde sus cuentas asignadas, **no recepción** |
| 3a | Separación caja chica / bancos | **Parcial** | 19 cuentas sembradas (mig. 037, sin aplicar). Los cobros **no registran la cuenta receptora** → saldo por caja/banco solo muestra egresos (`accounting.service.ts:1474`) |
| 3b | Caja grande → caja chica, saldos por cuenta | **Parcial** (sin desplegar) | Transferencias internas con validación (mig. 044, [REN-35](https://linear.app/sprinta/issue/REN-35)). Falta saldo inicial por cuenta |
| 3c | Balance diario | **Parcial** | Neto del periodo (ingresos − egresos, efectivo/banco); no hay vista por día ni por sede |
| 3d / Res. | Entrega/cierre diario de efectivo | **Falta** | [REN-15](https://linear.app/sprinta/issue/REN-15) y [REN-21](https://linear.app/sprinta/issue/REN-21) bloqueados |
| 4 | Roles y accesos | **Completo** (sin desplegar) | admin global; admin_sede y contador por `user_locations`; operar solo desde cuentas asignadas ([REN-34](https://linear.app/sprinta/issue/REN-34)); contador sin datos de paciente. La exportación SIBI sigue sin estar disponible para contador ni admin_sede |
| 5 | Comprobantes referenciales | **Completo** | Mig. 015, `createReferentialDocument` (anula con NC), pestaña "Comprobantes" |
| 6a | Exportación puente SIBI (Excel/CSV) | **Completo** (con matices) | `reports.service.ts:517`, botones en `Reports.tsx:1248`. Solo exporta pagos con comprobante referencial; `@Roles('admin','supervisor')` |
| 6b | Excel/CSV del listado de pagos | **Falta** | `Accounting.tsx` no tiene exportación |
| 6c | PDF resumen de caja / balance diario por sede | **Falta** | El único PDF es el volcado de la tabla SIBI en Reportes |
| 7 | Inventarios (2 almacenes) | **Falta** | Sin tablas ni código. [REN-19](https://linear.app/sprinta/issue/REN-19) bloqueado, falta decidir frente al kardex de SIBI |

---

## Sprint 1 — 25/08 al 27/08 (completado)

- [x] Módulo `AccountingModule` + ruta `/accounting` con `AuthGuard` + `RolesGuard` ([REN-16](https://linear.app/sprinta/issue/REN-16))
- [x] Resumen de ingresos por fecha, sede, medio de pago, servicio y doctor, calculado en backend ([REN-9](https://linear.app/sprinta/issue/REN-9))
- [x] Cuentas por cobrar de órdenes y citas con pago parcial, sin duplicar ([REN-10](https://linear.app/sprinta/issue/REN-10))
- [x] Egresos manuales categorizados: alta, edición, baja y auditoría en `audit_logs` (mig. 012) ([REN-13](https://linear.app/sprinta/issue/REN-13))
- [x] Cuentas caja/banco y balance ingresos − egresos con split efectivo/banco (mig. 014) ([REN-14](https://linear.app/sprinta/issue/REN-14))
- [x] Roles `contador` (solo lectura, sin datos de paciente) y `admin_sede` (mig. 013, 016) ([REN-16](https://linear.app/sprinta/issue/REN-16))
- [x] Comprobantes referenciales: nota de venta, boleta y nota de crédito que anula (mig. 015) ([REN-17](https://linear.app/sprinta/issue/REN-17))
- [x] Exportación puente SIBI en XLSX/CSV/PDF desde Reportes ([REN-18](https://linear.app/sprinta/issue/REN-18))
- [x] Tests unitarios de resumen, cuentas por cobrar, comprobantes y matriz de roles

---

## Sprint 2 — 23/09 y 24/09

**Objetivo:** llevar a producción las correcciones de la revisión del 28/08 con cuentas por responsable, alcance por sede y transferencias internas. Detalle en [backlog-sprint-2](backlog-sprint-2-modulo-contabilidad-2026-09-23-24.md).

**Correcciones del 28/08 (backend y frontend en `main` local, sin push ni migraciones aplicadas)**
- [x] Catálogo de 20 cuentas con códigos genéricos `CTA-NNNN`, incluida la nueva CAJA CHICA (mig. 037) ([REN-29](https://linear.app/sprinta/issue/REN-29))
- [x] Nº de operación, medio Efectivo/Transferencia/Yape, origen y concepto de 50 caracteres; sin destino ni proyectos (mig. 038) ([REN-28](https://linear.app/sprinta/issue/REN-28))
- [x] Validación en dos etapas: pendiente/validado/rechazado, sin autovalidación (mig. 040) ([REN-30](https://linear.app/sprinta/issue/REN-30))
- [x] Evidencia en bucket privado + detección de comprobante duplicado por emisor (mig. 041) ([REN-31](https://linear.app/sprinta/issue/REN-31))
- [x] Proveedores por RUC **o DNI** con autocompletado de VerificaPE (mig. 042) ([REN-32](https://linear.app/sprinta/issue/REN-32))
- [x] Pagos al personal vinculados a usuarios registrados (mig. 043) ([REN-33](https://linear.app/sprinta/issue/REN-33))

**Modelo del 23–24/09**
- [x] Superusuario contable retirado; mig. 039 eliminada (`bc6f7ef`, `5b05c68`)
- [x] Cuentas operadas solo por sus responsables (`accounting_account_users`), traspaso de caja y pestaña **Cuentas** para el admin ([REN-34](https://linear.app/sprinta/issue/REN-34))
- [x] admin_sede y contador limitados a sus sedes; contador corrige y valida pero no registra ni elimina ([REN-27](https://linear.app/sprinta/issue/REN-27))
- [x] Filtro "Solo míos" para quien registra
- [x] Transferencias internas con validación, fuera del total de egresos (mig. 044) ([REN-35](https://linear.app/sprinta/issue/REN-35))
- [x] Solo el admin asigna sedes a admin, admin_sede y contador (Usuarios + API)
- [x] `ParseUUIDPipe` en `PATCH/DELETE /accounting/expenses/:id`
- [x] Tests: backend 294, frontend 51 + build; migraciones probadas en PostgreSQL 17 local (`069d881`, `9b83153`)

**Decisión con cliente**
- [ ] Lista de qué admin_sede es responsable de CAJA MIGUEL, CAJA ADARLIN y CAJA CRISTIAN
- [ ] Sede de CAJA RONALD, CAJA CHICA y BCP PUBLICIDAD (hoy quedan sin sede y no se pueden usar)
- [ ] Confirmar que admin y todo contador ven las cuentas de proyecto (Balsa, Eventos)
- [ ] Confirmar si un contador puede validar un egreso que corrigió

**Despliegue**
- [ ] Revisar en producción que los nombres de `locations` coinciden con la semilla de 037 (huánuco/huanuco, pucallpa, lima, puerto fest)
- [ ] Aplicar migraciones en orden: 037 → 038 → 040 → 041 → 042 → 043 → 044 (039 descartada), **antes** de desplegar el backend
- [ ] Push de `main` backend y frontend + deploy (backend primero, luego front)
- [ ] Configurar `VERIFICAPE_DNI_API_KEY` y `VERIFICAPE_RUC_API_KEY` en Railway y agregarlas a `.env.example`
- [ ] Configurar cuentas (sedes y responsables) desde **Cuentas** y sedes de usuarios desde **Usuarios**
- [ ] E2E con datos reales de REN-27..35 (casos en el backlog, ACC-26)

**Gaps chicos de alto valor (si entra en los 2 días)**
- [ ] Paginar con `.range()` los `select` de `getSummary`, `getExpenses`, transferencias, `getReceivables` y `getSibiSalesExport` (PostgREST corta en 1000 filas, ver `plan-truncacion-supabase-paginacion-indices.md`)
- [ ] Mostrar el medio de pago de ingresos en español (la tabla de ingresos muestra `cash`/`bank`)

---

## Sprint 3 — Ingresos confiables

**Objetivo:** que cada cobro diga cómo y a qué cuenta entró, y que la contadora vea las inconsistencias.

- [ ] Diseñar el modelo de desglose de pagos (p. ej. `payment_splits`: método + monto + cuenta) compatible con los pagos históricos ([REN-11](https://linear.app/sprinta/issue/REN-11))
- [ ] Registrar la **cuenta receptora** (caja/banco) de cada cobro en recepción y órdenes de servicio
- [ ] UI de cobro mixto (efectivo + transferencia/Yape) en citas y órdenes ([REN-11](https://linear.app/sprinta/issue/REN-11))
- [ ] Resumen contable: ingresos por cuenta → saldo real por caja/banco (reemplaza "Egresos por caja y banco")
- [ ] Reporte de pagos confirmados sin comprobante referencial ([REN-12](https://linear.app/sprinta/issue/REN-12))
- [ ] Detección de inconsistencias: atención completada sin pago, medio de pago dudoso, monto pagado ≠ total ([REN-12](https://linear.app/sprinta/issue/REN-12))
- [ ] Cuentas por cobrar como saldo vigente a una fecha de corte (no solo órdenes creadas en el rango)
- [ ] Ingresos por `payment_date` (fallback `created_at`) y definir el fallback de citas pagadas sin fila en `payments`
- [ ] Categorías de ingreso definitivas según términos de Ronald

**Dependencias externas:** términos de categorización de ingresos — Ronald · reporte de junio SIBI para validar el mix — Ronald · análisis de plantilla SIBI [REN-20](https://linear.app/sprinta/issue/REN-20) (en revisión).

---

## Sprint 4 — Caja diaria y exportaciones

**Objetivo:** cuadre de entrega diaria de efectivo en versión ligera, más los exportables que pide la contadora.

- [ ] Cerrar la investigación del proceso real de entrega/cierre de caja ([REN-21](https://linear.app/sprinta/issue/REN-21))
- [ ] Entrega diaria de efectivo: recepción registra lo entregado, el sistema calcula lo esperado (cobros en efectivo − egresos en efectivo) y marca la diferencia ([REN-15](https://linear.app/sprinta/issue/REN-15))
- [ ] Confirmación del administrador sobre la entrega recibida ([REN-15](https://linear.app/sprinta/issue/REN-15))
- [ ] Obligatoriedad del cierre configurable por sede, sin apertura ni bloqueo tipo POS ([REN-15](https://linear.app/sprinta/issue/REN-15))
- [ ] Definir cómo entran los egresos en efectivo que paga recepción (hoy recepción no tiene acceso a contabilidad)
- [ ] Saldo inicial por cuenta (las transferencias internas ya se entregaron en el Sprint 2)
- [ ] Evidencia (voucher) adjunta a transferencias internas
- [ ] Vista de balance diario por sede (día a día dentro del rango)
- [ ] Exportar a Excel/CSV el listado de pagos desde Contabilidad: fecha, comprobante, monto, forma de pago, paciente/DNI, sede, servicio (paciente oculto para contador)
- [ ] PDF de resumen de caja / balance diario por sede
- [ ] Mover el puente SIBI al guard contable (admin, contador y admin_sede por sede) y decidir si el contador ve DNI/teléfono ahí (la plantilla los exige y choca con "sin datos personales")
- [ ] Revisar si `supervisor` debe seguir exportando SIBI con DNI y teléfono (`ReportsController` `@Roles('admin','supervisor')`)
- [ ] Puente SIBI: oficina desde la sede de la cita (hoy usa la sede del servicio) y serie/correlativo reales en vez de `FE01/BE01/NV01` fijos

**Dependencias externas:** investigación de caja — Serghio · decisión "prescindir si el efectivo es marginal" con el reporte de junio — Ronald · requiere Sprint 3 (cuenta receptora + pagos mixtos) para que el esperado cuadre.

---

## Sprint 5 — Inventario clínico

**Objetivo:** decidir la frontera con SIBI y, si aplica, construir el inventario de dos almacenes.

- [ ] Decisión con cliente: inventario propio vs. apoyarse en el kardex de SIBI ([REN-19](https://linear.app/sprinta/issue/REN-19))
- [ ] Modelo: almacén de venta al público (fajas, proteínas) y almacén de insumos de cirugía ([REN-19](https://linear.app/sprinta/issue/REN-19))
- [ ] Control de stock con movimientos (entrada, salida, ajuste) y auditoría ([REN-19](https://linear.app/sprinta/issue/REN-19))
- [ ] Salida de insumos por cirugía ligada a la cita/orden y reingreso de sobrantes ([REN-19](https://linear.app/sprinta/issue/REN-19))
- [ ] Permisos por sede, igual que el resto del módulo

**Dependencias externas:** credenciales de SIBI para revisar su inventario — Ronald · decisión de alcance del cliente.

---

## Sprint 6 — Limpieza de pagos históricos y endurecimiento

**Objetivo:** dejar los pagos históricos consistentes y el módulo cubierto por tests. No hay egresos anteriores que migrar: las cuentas temporales se eliminan en 037 y `accounting_expenses.location_id` se elimina en 038.

- [ ] Normalizar los `payments.payment_method` históricos en texto libre (hoy se normalizan solo en la respuesta)
- [ ] Revisar pagos históricos de junio: equivalencia entre `appointments` y `payments`
- [ ] `getDocumentById` devuelve 404 en vez de 500; quitar `UpdateReferentialDocumentDto`, que no se usa
- [ ] Tests de front (navegación por rol, filtros, formulario de egreso) y E2E del flujo egreso → validación
- [ ] Test de truncamiento: resumen con más de 1000 pagos

**Dependencias externas:** muestras de junio y confirmación de cuentas reales — Ronald.

---

## Fuera de alcance

- No emite boletas/facturas electrónicas ni se integra automáticamente con SUNAT (queda en SIBI; el puente es por exportación).
- No emite ni anula comprobantes: solo los registra de forma referencial.
- No reemplaza al contador ni los libros contables.
- No calcula ni declara impuestos (IGV/ICBPER van en 0 en el puente).
- No genera asientos contables (partida doble).
- No hace conciliación bancaria automática.
- No impone un cierre de caja rígido tipo POS que bloquee la operación.
