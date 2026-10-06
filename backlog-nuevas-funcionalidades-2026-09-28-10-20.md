# Backlog técnico — Nuevas funcionalidades

## Secciones de historia clínica, Punto de venta, Vouchers de pago, Ingresos (Sprint 2) y Códigos contables — Renaceris

**Periodo:** 28/09/2026 — 20/10/2026
**Días hábiles:** lunes a viernes. **08/10 es feriado** (Combate de Angamos) y no se planifica trabajo.
**Proyecto Linear:** [Módulo Contabilidad (REN)](https://linear.app/sprinta/project/modulo-contabilidad-9d9e4c0af62b) para ingresos, POS y códigos contables; HC y vouchers como issues sueltos.
**Backlog anterior:** [backlog-sprint-2-modulo-contabilidad-2026-09-23-24.md](backlog-sprint-2-modulo-contabilidad-2026-09-23-24.md)
**Checklist del módulo contable:** [TODO-modulo-contabilidad.md](TODO-modulo-contabilidad.md)

---

## 1. Resumen y cronograma

| # | Funcionalidad | Complejidad | Esfuerzo | Inicio | Deadline | Depende de |
|---|---|---|---|---|---|---|
| F1 | Historia clínica: secciones 01 No quirúrgica / 02 Quirúrgica | Media | 2 días | 28/09 | **30/09** ✅ implementado 29/09 | Servicios marcados por la clínica (D2) |
| F2 | Punto de venta para recepción + cierre de caja (incluye F3b e ingresos mínimos) | Alta | 6 días | 06/10 | **14/10** (QA 15/10) | ✅ implementado 06/10, pendiente de despliegue |
| F3b | Voucher en ventas de productos | Baja | incluido en F2 | 06/10 | **14/10** | ✅ con F2 |
| F3a | Voucher (captura Yape/transferencia) en pagos de citas | Media-baja | 2 días | después de F2 | por definir | Reutiliza `payment_evidences` (columna `payment_id`) |
| F5 | Ingresos Sprint 2 (modal de ingresos externos + contrapartes) | Media | 3 días | después de F2 | por definir | Extiende `accounting_incomes` de la mig. 048; D5 |
| F4 | Códigos contables y conceptos de gasto | Baja | 2 días | 19/10 | **20/10** | **Bloqueado:** catálogo de la clínica (D8) |

**Orden elegido:** F1 y F3a primero porque son independientes y de impacto inmediato en recepción. F5 va antes que F2 porque el cierre de caja del punto de venta, al validarse, se registra como un ingreso en la caja general: F2 reutiliza el modelo de ingresos de F5. F4 queda al final porque depende de información que la clínica aún no envía.

**Cambio de orden (06/10):** la clínica pidió arrancar **solo con el punto de venta** por urgencia. F2 sale primero y trae la versión mínima de `accounting_incomes` (solo `source = 'pos_closure'`) y de `payment_evidences`. F5 y F3a se reordenan después y **extienden** esas tablas en lugar de crearlas.

```
Semana 1  28/09 ─ 02/10   F1 HC secciones ███████  F3a Vouchers citas ███████
Semana 2  05/10 ─ 09/10   F5 Ingresos ███████████  (08 feriado)  F2 POS ███
Semana 3  12/10 ─ 16/10   F2 POS ████████████████████████████ (+F3b, QA)
Semana 4  19/10 ─ 20/10   F4 Códigos contables ███████ (si llega el catálogo)
```

---

## 2. Decisiones pendientes con la clínica

Cada decisión tiene una fecha límite: si no llega, se implementa el valor **por defecto** y se ajusta después.

| # | Pregunta | Por defecto propuesto | Afecta | Responder antes de |
|---|---|---|---|---|
| D1 | ~~Opción A (una HC con secciones) u opción B (dos HC)~~ | ✅ **Resuelto 29/09:** opción A, una sola HC con secciones 01/02. | F1 | — |
| D2 | Lista de servicios quirúrgicos, marcada en cada sede desde **Servicios**. | Hasta que se marquen, todas las citas quedan en la sección 01. Marcar después reclasifica sin renumerar. | F1 | 30/09 |
| D3 | ~~Paciente atendido en una sede y operado en otra~~ | ✅ **Resuelto 29/09:** la sede de la HC solo indica dónde fue la primera cita (quirúrgica o no) y es permanente. El paciente no está afiliado a ninguna sede y conserva su HC: `SHCO-000123-02` aunque se opere en Lima. | F1 | — |
| D4 | ~~Movimientos internos: ¿quién valida?~~ | ✅ **Resuelto 06/10:** valida **el que recibe el dinero o el contador** (basta uno de los dos; el admin siempre puede; nunca el autor). Aplicado ya al cierre del POS: valida el responsable de la caja general de la sede o el contador de la sede. Para transferencias internas se implementa con F5. | F2, F5 | — |
| D5 | Un ingreso externo pagado por transferencia (p. ej. alquiler depositado en BCP Huánuco), ¿se registra en el banco o en la caja general? | Se permite registrar en la caja general **y** en los bancos de las sedes Huánuco, Lima y Pucallpa. | F5 | 02/10 |
| D6 | ~~En el cierre del POS, ¿los cobros por Yape/transferencia van a la caja general o al banco?~~ | ✅ **Resuelto 06/10 (preliminar):** solo el **efectivo** entra a la caja general de la sede (CAJA HUANUCO / PUCALLPA / LIMA). Yape, Plin, transferencia y tarjeta quedan **"por definir"** y el contador les asigna la caja o banco. | F2 | — |
| D7 | ~~Catálogo de productos que vende recepción~~ | ✅ **Resuelto 06/10:** mientras no haya inventario (~2 semanas) se vende con el producto genérico **"Productos diversos"**, precio libre y descripción obligatoria. La venta se guarda con líneas y pagos desde el día 1, así que el inventario se suma sin migrar ventas. | F2 | — |
| D8 | Plan de códigos contables y conceptos de gasto. | Sin valor por defecto: F4 no arranca sin la lista. | F4 | 09/10 |
| D9 | ~~¿El contador puede ver los vouchers?~~ | ✅ **Resuelto 06/10:** sí, el contador ve todo, vouchers incluidos, y es quien resuelve los ingresos por definir (asigna o corrige su destino). | F2, F3 | — |

---

## 3. F1 — Historia clínica: secciones 01 No quirúrgica / 02 Quirúrgica

**Estado:** ✅ implementado el 29/09 (pendiente de despliegue y de D2) · **Complejidad:** Media · **Deadline:** 30/09 · **Migración:** 047
**Decisión (29/09):** opción A, **una sola HC por paciente con dos secciones**. La opción B (dos HC con numeraciones `SHCO-01-…` / `SHCO-02-…`) se descartó porque implicaba renumerar y contradecía la regla de HC única.
**Referencia permanente:** [historia-clinica-numeracion-y-secciones.md](historia-clinica-numeracion-y-secciones.md)

### 3.1 Regla de negocio

- **Una HC por paciente** (`patients.clinical_record_number`), con la numeración existente de 045/046. **No se renumera nada.**
- **La sede de la HC** (prefijo `SHCO`/`SPUC`/`SLIM` y `first_location_id`) **solo indica dónde fue la primera cita elegible**, quirúrgica o no. Es permanente y **no afilia al paciente a esa sede**: se atiende en cualquier sede y conserva su HC. Ninguna regla de acceso ni de agenda debe filtrar pacientes por esa sede.
- **Secciones:** `01` No quirúrgica y `02` Quirúrgica, solo para organizar la misma HC. Código de sección: `SHCO-000123-02`.
- Un paciente atendido en Huánuco y operado después en Lima conserva `SHCO-000123`. Su parte quirúrgica es `SHCO-000123-02`.

### 3.2 Diseño implementado

| Pieza | Detalle |
|---|---|
| `services.is_surgical` | Marca por servicio (y por sede, porque los servicios son por sede). Toggle "Servicio quirúrgico" en **Servicios**. |
| Sección de una cita | **Derivada** del `is_surgical` de su servicio; no se guarda en la cita. Corregir la marca reclasifica todas las citas del servicio y sus documentos, sin tocar números. Por eso no hace falta una carga inicial. |
| Vista `patient_clinical_record_sections` | Secciones abiertas por paciente: al menos una cita **no cancelada** de ese tipo. `security_invoker`; solo `service_role`. |
| API | `GET /patients`, `/patients/my-patients` y `/patients/:id/full-profile` devuelven `clinical_record_sections: ['01','02']`. Las citas del perfil y las líneas de tratamiento traen `services.is_surgical`. |
| Documentos | Formularios y notas clínicas siguen la sección de su cita. Lo que no tiene cita (alergias, antecedentes, notas generales) es común a toda la HC y aparece en cualquier sección. |

Se descartó guardar la sección en la cita (snapshot). Con la sección derivada, la corrección inicial de marcas por parte de la clínica es inmediata. Además, un servicio no cambia de naturaleza en la práctica: si algún día lo hiciera, se crea un servicio nuevo.

### 3.3 Pantallas

- **Servicios:** toggle "Servicio quirúrgico" con aviso de reclasificación al editar; etiqueta "Quirúrgica" en la lista.
- **Pacientes:** etiquetas de sección bajo la HC; filtro "Sección H.C."; búsqueda por número de HC; columnas HC y secciones en el Excel.
- **Perfil del paciente:**
  - Selector **Completa / 01 · No quirúrgica / 02 · Quirúrgica** que filtra el resumen, las líneas de tratamiento, el cronograma, los formularios y las notas.
  - El PDF exporta la sección elegida, con el código `H.C. SHCO-000123-02` en el encabezado.
  - La tarjeta de HC explica qué significa la sede.
- **Nueva cita:** indica en qué sección se registrará la cita y aclara que la sede solo fija el prefijo de forma permanente.

### 3.4 Tareas

| ID | Tarea | Estado |
|---|---|---|
| HC-01 | Migración 047 (`is_surgical`, vista, comentarios de columna, índice). Probada en PostgreSQL 17 local: idempotente, reclasificación, cancelada no abre sección, permisos. | ✅ |
| HC-02 | Backend: `is_surgical` en DTO, listado y actualización de Servicios; secciones en pacientes; `is_surgical` en perfil y líneas de tratamiento. | ✅ |
| HC-03 | Frontend: Servicios, Pacientes, Perfil (selector, filtros, PDF) y Nueva cita. | ✅ |
| HC-04 | Tests: backend `clinical-record-sections.spec.ts` y `services.service.spec.ts`; frontend `clinicalRecordSections.spec.ts`. | ✅ |
| HC-05 | Despliegue: aplicar **047** en el SQL editor (después de 046) → backend → frontend. `NOTIFY pgrst, 'reload schema';` si PostgREST no ve la vista. | ⏳ |
| HC-06 | La clínica marca los servicios quirúrgicos en cada sede (D2). | ⏳ Clínica |

**Criterio de aceptación:**
- Un paciente con una consulta de láser en Huánuco y luego una cirugía en Lima muestra `SHCO-000123` con las secciones `SHCO-000123-01` y `SHCO-000123-02`.
- Un paciente solo quirúrgico muestra solo la 02.
- Ningún número cambia al marcar o desmarcar servicios.

---

## 4. F3 — Vouchers de pago (capturas de Yape y transferencias)

**Complejidad:** Media-baja · **Deadline:** F3a 02/10 · F3b 16/10 · **Migración:** 048 (tentativa)

### 4.1 Situación actual

- `payments` guarda `payment_method` y `transaction_code`, sin archivo adjunto.
- Contabilidad ya tiene evidencias de egresos en bucket privado con URLs firmadas (`accounting-evidence.service.ts`, mig. 041): se reutiliza ese patrón, no el bucket público de `files.service.ts`.

### 4.2 Modelo propuesto

- Tabla `payment_evidences` (`id`, `payment_id` o `pos_sale_id`, `storage_path`, `mime_type`, `size_bytes`, `uploaded_by`, `created_at`), con CHECK de exactamente un padre.
- Bucket privado `payment-evidence`; JPG, PNG, WEBP o PDF, máx. 5 MB, compresión de imágenes en el navegador.
- **Obligatorio** cuando el medio de pago es Yape, Plin o transferencia; opcional en efectivo y tarjeta.
- Visibilidad según D9.

### 4.3 Tareas

| ID | Tarea | Día |
|---|---|---|
| PAG-01 | Migración 048, bucket privado y servicio de evidencias genérico (extraído del de egresos). | 01/10 |
| PAG-02 | Backend: subir, listar y firmar URL en los pagos de citas y órdenes de servicio (`appointments.service.ts`); validar obligatoriedad por medio de pago. | 01/10 |
| PAG-03 | Frontend: campo "Adjuntar captura" con vista previa en el registro de pago de citas; icono de voucher en el historial de pagos. | 02/10 |
| PAG-04 | Tests de obligatoriedad, tamaño/formato y permisos. | 02/10 |
| PAG-05 (F3b) | Mismo componente en el cobro de productos del POS. | 15/10 |

---

## 5. F2 — Punto de venta para recepción

**Complejidad:** Alta · **Deadline:** 14/10 (QA 15/10) · **Migración:** 048 `048_point_of_sale.sql` · **Estado:** ✅ implementado el 06/10, pendiente de despliegue

> **Implementación del 06/10 (reemplaza lo propuesto abajo donde difiera):**
> - **Producto genérico + venta estructurada.** "Productos diversos" (`is_generic`) sin precio referencial y con descripción obligatoria por línea; el precio es editable en todas las líneas. `pos_sales` → `pos_sale_items` (con copia del nombre y la categoría) → `pos_sale_payments` (pagos mixtos, n.º de operación obligatorio si no es efectivo). `pos_products.tracks_stock` queda listo para el inventario.
> - **Vouchers.** Tabla genérica `payment_evidences` (padre `pos_sale_payment_id` o, para F3a, `payment_id`) y bucket privado `payment-evidence`. Obligatorio en Yape, Plin y transferencia: la venta se registra igual, pero **no se puede cerrar caja** con vouchers faltantes.
> - **Cierre.** Uno por recepcionista, sede y día. Esperado = cobros de citas/órdenes que ella registró (`payments.created_by`) ese día en la sede + sus ventas no anuladas. Los cobros incluidos se guardan en `pos_cash_closure_payments` (un cobro entra a un solo cierre). Diferencia ≠ 0 exige observación. Tras un rechazo se corrige y se vuelve a cerrar sobre la misma fila.
> - **Excedente (06/10).** Si se cuenta más efectivo que lo registrado, la recepcionista indica el motivo (vuelto no entregado, cobro no registrado, no identificado u otro) y una observación; el cierre cuadra como "esperado + excedente". Al validar, el excedente entra a la caja de la sede como ingreso aparte (`accounting_incomes.kind = 'excedente_caja'`), separado de los cobros. Un faltante solo exige observación y entra lo contado.
> - **Alcance del reporte.** La recepcionista solo ve sus ventas, cobros, cierres y vouchers (el backend ignora cualquier otro filtro). El admin ve todas las sedes; admin_sede y contador, las suyas; los tres pueden ver toda la sede o elegir una recepcionista.
> - **Validación (D4).** Valida el responsable de la caja general de la sede, el contador de la sede o el admin; nunca quien cerró. RPC `validate_pos_cash_closure` en una transacción: efectivo declarado → `accounting_incomes` asignado a la caja general (`accounting_accounts.accepts_external_income`, editable en **Cuentas**); cada cobro no en efectivo → `accounting_incomes` **por definir**.
> - **Contabilidad.** Pestaña **Por definir** (contador/admin asignan o corrigen la caja o banco de la sede, en bloque, viendo el voucher). El saldo por cuenta suma los ingresos asignados; las ventas de productos entran al resumen de ingresos.
> - **PDF** del reporte y del cierre en el frontend (`jspdf`, igual que Reportes), no en el backend.
> - **Pendiente conocido:** cobros de citas creados por RPC de órdenes/proformas sin `created_by` no entran en ningún cierre (el reporte los muestra como "sin registrar quién cobró").

### 5.1 Alcance

- Módulo nuevo **Punto de venta**, solo para `recepcionista` (y visible para admin, admin_sede y contador), limitado a la sede del usuario (`user_locations`).
- **Sin inventario**: no hay stock ni kardex. Solo catálogo de productos (D7).
- **No toca las cajas contables** mientras no lo valide un contador o admin. Hasta entonces solo sirve para reportes.
- El reporte agrupa por categoría: **Servicios (citas)** —tomados automáticamente de los pagos de citas de la sede y el día, sin volver a digitarlos— y **Productos** —registrados en el POS—, con desglose por medio de pago.
- **Cierre de caja:** la recepcionista declara el efectivo contado. El sistema muestra el esperado, la diferencia (excedente o faltante) y exige una observación si no cuadra. El monto declarado es el que representa el estado real.
- **Validación:** el contador o admin de la sede valida el cierre → se genera un ingreso en la caja general de la sede (CAJA HUANUCO, CAJA LIMA, CAJA PUCALLPA) con el modelo de F5. Si lo rechaza, vuelve a la recepcionista con la observación.
- Exportación a **PDF** del reporte diario y del cierre.

### 5.2 Modelo propuesto

| Tabla | Contenido |
|---|---|
| `pos_product_categories`, `pos_products` | Catálogo sin stock: nombre, categoría, precio referencial, activo. Solo el admin lo edita. |
| `pos_sales` | Venta de productos: sede, fecha, vendedor, cliente opcional (DNI/nombre), total, medio de pago, nº de operación, estado (`registrada`/`anulada` con motivo). |
| `pos_sale_items` | Producto, cantidad, precio unitario (editable, precio referencial por defecto), subtotal. |
| `pos_cash_closures` | Sede, fecha, responsable, totales esperados por medio de pago (servicios + productos), efectivo declarado, diferencia, observación, estado `abierto → cerrado → validado/rechazado`, revisor, `accounting_income_id` al validarse. Un cierre por sede y recepcionista por día. |

Reglas: una venta de un cierre ya cerrado no se edita; una recepcionista no ve otras sedes; nadie valida su propio cierre.

### 5.3 Tareas

| ID | Tarea | Día |
|---|---|---|
| POS-01 | Migración 050 y catálogo de productos (admin). | 09/10 |
| POS-02 | Backend: ventas de productos (alta, anulación con motivo) con alcance por sede. | 12/10 |
| POS-03 | Backend: reporte diario consolidado (pagos de citas + ventas de productos) por categoría y medio de pago. | 12/10 |
| POS-04 | Frontend: pantalla POS (carrito, cobro, lista del día) y reporte por categoría. | 13/10 |
| POS-05 | Cierre de caja: esperado vs. declarado, diferencia y observación obligatoria. | 13/10–14/10 |
| POS-06 | PDF del reporte y del cierre (generado en backend, mismo estilo que el PDF de Reportes). | 14/10 |
| POS-07 | Validación del cierre por contador o admin → ingreso en la caja general de la sede (usa F5). | 15/10 |
| POS-08 | Voucher en venta de productos (F3b / PAG-05). | 15/10 |
| POS-09 | Tests (alcance por sede, cierre, diferencia, validación, no autovalidación) y QA E2E. | 16/10 |

**Criterio de aceptación:** una recepcionista de Pucallpa vende 2 productos, ve sus citas cobradas del día, cierra declarando S/ 10 más de lo esperado con observación, exporta el PDF; el contador de Pucallpa valida y CAJA PUCALLPA muestra el ingreso por el monto declarado.

---

## 6. F5 — Ingresos, Sprint 2

**Complejidad:** Media · **Deadline:** 07/10 · **Migración:** 049 (tentativa)
**Prerrequisito:** migraciones 037–044 aplicadas y Sprint 2 contable desplegado (ACC-25/26 del backlog anterior).

### 6.1 Regla de negocio

- Solo las **cajas generales** HUÁNUCO, PUCALLPA y LIMA reciben ingresos externos (más los bancos de esas sedes si D5 se confirma).
- Las cajas de admin_sede (CAJA MIGUEL, ADARLIN, CRISTIAN) **no** reciben ingresos: solo se alimentan por **transferencia interna** desde otra cuenta.
- En vez de fijar nombres en el código: columna `accounting_accounts.accepts_external_income BOOLEAN DEFAULT FALSE`, activada en la semilla para las 3 cajas generales y editable por el admin en **Cuentas**. La API rechaza un ingreso a una cuenta sin el flag.

### 6.2 Verificación de la lógica actual de movimientos internos

Revisado en `accounting-transfers.service.ts`, `accounting-access.ts`, `accounting.service.ts` y mig. 044:

| Punto | Estado actual | ¿Soporta la inyección interna? |
|---|---|---|
| Caja general → caja de admin_sede | Permitido: cualquier cuenta activa y configurada puede ser destino, incluso de otra sede. | ✅ |
| Caja de admin_sede → caja general (devolución) | Permitido si el admin_sede es responsable de la cuenta de origen. | ✅ |
| Banco involucrado | Exige nº de operación. | ✅ |
| No cuenta como egreso | Se excluye del total de egresos; mueve saldo en ambas cuentas. | ✅ |
| Quién valida | Admin (cualquiera) o contador de la sede de alguna de las cuentas; nunca el autor. **El responsable de la cuenta destino no interviene.** | ⚠️ Ver D4 |
| Saldo mientras está pendiente | Las pendientes **ya suman** al saldo del destino; solo se excluyen las rechazadas (`accounting.service.ts:1376`). El dinero aparece antes de validarse. | ⚠️ |
| Evidencia en transferencias | No existe (estaba planificado para Sprint 3). | ❌ |
| Saldo inicial por cuenta | No existe; el saldo es solo transferencias − egresos del periodo. | ❌ (fuera de este backlog) |

**Conclusión:** la lógica actual ya soporta las inyecciones internas de caja a caja. Faltan tres ajustes, incluidos en este backlog:

1. **Confirmación del receptor (D4).** Recomendación: dos pasos. (1) El responsable de la cuenta destino **confirma que recibió** el dinero, porque es el único que sabe si el efectivo llegó. (2) El contador de la sede o el admin **valida** contablemente. Si el receptor es el mismo admin que valida, basta un paso. Alternativa mínima: mantener la validación actual (contador/admin) y solo notificar al receptor.
2. **Saldo:** las transferencias pendientes se muestran como "por confirmar" y no suman al saldo disponible hasta validarse.
3. **Voucher** opcional en la transferencia, reutilizando F3.

### 6.3 Modal de registro de ingreso (UI)

Un solo modal con secciones que aparecen según lo elegido (no un wizard de varias pantallas), y una frase de resumen al pie que se arma en tiempo real.

```
┌─ Registrar ingreso ───────────────────────────────────────────┐
│ 1. ¿Qué tipo de ingreso es?                                   │
│   [Alquiler de espacio] [Venta de activo] [Devolución/        │
│    reembolso] [Intereses bancarios] [Aporte de socio] [Otro]  │
│                                                               │
│ 2. ¿Quién paga?                                               │
│   (•) Empresa (RUC)  ( ) Persona (DNI)  ( ) Sin identificar   │
│   RUC [20601234567] → "INVERSIONES X S.A.C." (autocompletado) │
│                                                               │
│ 3. Detalle  ← campos según el tipo                            │
│   Alquiler: Área alquilada [Consultorio 3, 2.º piso]          │
│             Periodo [Octubre 2026]  ☐ Es un ingreso mensual   │
│   Venta de activo: Bien vendido [...]                         │
│   Otro: Concepto [texto, 50 caracteres]                       │
│                                                               │
│ 4. ¿Dónde entra el dinero?                                    │
│   Cuenta [CAJA HUANUCO ▾]  (solo cuentas habilitadas)         │
│   Medio [Efectivo|Transferencia|Yape]  Nº operación [...]     │
│   Fecha [28/09/2026]   Monto S/ [1,500.00]                    │
│                                                               │
│ 5. Respaldo: Comprobante emitido (opcional) · Voucher         │
│ ───────────────────────────────────────────────────────────── │
│ "Ingreso de S/ 1,500.00 por alquiler del Consultorio 3        │
│  (octubre 2026) pagado por INVERSIONES X S.A.C.,              │
│  en efectivo a CAJA HUANUCO."        [Cancelar] [Registrar]   │
└───────────────────────────────────────────────────────────────┘
```

Decisiones de diseño:

- **Tipo primero:** define qué campos se piden; así el caso de alquiler no carga campos de otros casos.
- **Agente externo = contraparte:** se reutiliza el catálogo de proveedores por RUC/DNI con VerificaPE (mig. 042), generalizado a **terceros** (`accounting_counterparties` o columna `roles` en `accounting_suppliers`), para que la misma empresa pueda ser proveedor e inquilino.
- **"Es un ingreso mensual":** guarda contrato/periodo y permite ver en el reporte qué meses de alquiler faltan cobrar. La recurrencia automática queda fuera de alcance.
- **Frase de resumen:** permite que quien registra detecte errores antes de guardar.
- Los ingresos externos pasan por la misma validación en dos etapas que egresos y transferencias.
- En el resumen contable se separan **ingresos por atención** (pagos de citas/POS) de **otros ingresos** (este modal).

### 6.4 Modelo y tareas

`accounting_income_types` (catálogo editable por admin) y `accounting_incomes` (`income_date`, `amount`, `to_account_id`, `income_type_id`, `counterparty_id` nulo, `concept`, `detail JSONB` para campos por tipo, `period`, `payment_method`, `operation_number`, receipt, validación en dos etapas, `source` = `manual` | `pos_closure`).

| ID | Tarea | Día |
|---|---|---|
| ING-01 | Migración 049: `accepts_external_income`, tipos de ingreso, `accounting_incomes`, contrapartes; confirmación del receptor en `accounting_transfers` (`received_by`, `received_at`). | 05/10 |
| ING-02 | Backend: CRUD de ingresos con alcance por sede y cuenta habilitada; revisión; resumen con "otros ingresos"; saldo que excluye transferencias pendientes. | 05/10–06/10 |
| ING-03 | Backend: confirmación de recepción en transferencias según D4. | 06/10 |
| ING-04 | Frontend: modal de ingreso (§6.3), lista con estado, flag en **Cuentas**, botón "Confirmar recepción" para el responsable destino. | 06/10–07/10 |
| ING-05 | Tests: cuenta no habilitada → 400; admin_sede no registra ingresos a su caja; transferencia pendiente no suma saldo; confirmación y validación. | 07/10 |

---

## 7. F4 — Códigos contables y conceptos de gasto

**Complejidad:** Baja · **Deadline:** 20/10 · **Estado:** ⛔ bloqueado por D8 · **Migración:** 051 (tentativa)

- Hoy `accounting_expense_categories` (mig. 012) solo tiene nombre y orden.
- Se agrega `code` (p. ej. cuentas del PCGE: `63`, `6311`…), `parent_id` para jerarquía categoría → concepto y `active`.
- El formulario de egreso pasa a elegir **concepto** (que arrastra su código y categoría); el reporte y la exportación SIBI incluyen el código.
- Si la clínica lo pide, el mismo esquema se aplica a los tipos de ingreso de F5.

| ID | Tarea | Día |
|---|---|---|
| ACC-30 | Migración 051 + carga del catálogo recibido; mapeo de egresos existentes a conceptos. | 19/10 |
| ACC-31 | Backend y frontend: selector de concepto, administración del catálogo (admin), código en reporte y exportación. Tests. | 20/10 |

Si el catálogo no llega el 09/10, el deadline se corre dos días hábiles desde su recepción.

---

## 8. Definiciones técnicas no negociables

- Alcance por sede y permisos se deciden en NestJS; el frontend solo los refleja.
- Los números de HC se asignan automáticamente y no se editan. Su sede solo indica la primera cita elegible; nunca restringe dónde se atiende el paciente. La sección (01/02) de una cita se deriva de su servicio.
- El POS no genera movimientos contables hasta que se valide el cierre (responsable de la caja de la sede, contador o admin).
- Nadie valida un movimiento o cierre que registró.
- Los vouchers van en bucket privado con URL firmada; nunca en el bucket público.
- El contador no recibe datos personales de pacientes en reportes; **sí ve los vouchers** (D9, 06/10).
- Las migraciones se aplican antes de desplegar el backend que las usa.

## 9. Riesgos

1. **Secciones y norma:** la opción A mantiene una HC por paciente; conviene que la dirección médica confirme el uso de secciones internas.
2. **Servicios quirúrgicos sin marcar (D2):** mientras la clínica no los marque en las tres sedes, todos los pacientes aparecen solo con la sección 01. Un error se corrige desmarcando/marcando el servicio.
3. **Doble registro en el POS:** las recepcionistas podrían volver a registrar cobros de citas como ventas. El POS solo permite productos; las citas llegan solas al reporte.
4. **Saldos:** cambiar el tratamiento de transferencias pendientes altera los saldos que ya se ven hoy. Comunicar el cambio al desplegar F5.
5. **Cuentas receptoras de cobros:** los pagos de citas aún no registran a qué caja o banco entran. El cierre del POS es el primer puente hacia la caja general; el resto sigue en el Sprint 3 contable.
6. **Truncamiento a 1000 filas** en resumen y reportes (pendiente de Sprint 3): el reporte del POS se pagina desde el inicio.
7. **Feriado 08/10:** sin holgura en la semana 2. Si F5 se atrasa, F2 se corre al 19/10 y F4 al 21–22/10.
