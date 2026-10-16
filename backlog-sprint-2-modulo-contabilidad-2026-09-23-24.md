# Backlog técnico — Sprint 2

## Módulo Contabilidad — Renaceris

**Proyecto Linear:** [Módulo Contabilidad (REN)](https://linear.app/sprinta/project/modulo-contabilidad-9d9e4c0af62b)  
**Sprint:** 23/09/2026 — 24/09/2026  
**Deadline:** 24/09/2026, fin del día  
**Objetivo:** llevar a producción las correcciones de la revisión del 28/08 con el modelo acordado el 23–24/09: **cuentas con responsables**, **alcance por sede** para admin_sede y contador, y **transferencias internas**.  
**Sprint anterior:** [backlog-sprint-1-modulo-contabilidad-2026-08-25-27.md](backlog-sprint-1-modulo-contabilidad-2026-08-25-27.md)  
**Checklist general del módulo:** [TODO-modulo-contabilidad.md](TODO-modulo-contabilidad.md)

---

## 1. Objetivo del sprint

Al cierre del sprint debe estar en producción:

1. El registro de egresos corregido tras la revisión del 28/08 (REN-27 a REN-33).
2. Cada caja o banco operado solo por sus **responsables asignados**, con un catálogo administrable por el admin (REN-34).
3. El `contador` y el `admin_sede` limitados a las **sedes asignadas** en `user_locations`.
4. Las **transferencias internas** entre cuentas, fuera del total de egresos y con validación (REN-35).
5. Las migraciones aplicadas en Supabase y las credenciales de VerificaPE configuradas.

---

## 2. Decisiones del 23 y 24/09

| Tema | Decisión |
|---|---|
| Superusuario contable | **Se retira** (mig. 039 eliminada). El alcance lo dan el rol y sus sedes en `user_locations`. |
| Quién opera una cuenta | Solo sus responsables (`accounting_account_users`, muchos a muchos; normalmente 1). Traspaso de caja al cambiar el responsable. |
| Sede de la cuenta | Obligatoria, salvo cuentas de proyecto (CAJA BALSA, CAJA EVENTOS). |
| Códigos | Genéricos (`CTA-0001`); nombre y sede editables por el admin. |
| `admin` | Opera CAJA RONALD, CAJA CHICA (nueva), cajas generales de Huánuco, Lima y Pucallpa, y bancos de esas sedes. |
| `admin_sede` | Ve toda su sede; registra y corrige solo desde su caja; filtro "Solo míos". |
| `contador` | Ve cuentas y movimientos de sus sedes y los proyectos; **no registra ni elimina**; corrige y valida en sus sedes. |
| Sedes de usuarios | Solo el admin las asigna a admin (incluido él mismo), admin_sede y contador. |
| Destino del egreso | Desaparece "sede o proyecto". **Egreso corriente**: concepto escrito a mano y sede tomada de la cuenta de origen. **Transferencia interna**: cuenta destino. |
| Proyectos de destino | **Se retira** el catálogo `accounting_projects`. |
| Transferencias | Misma validación en dos etapas que los egresos. |
| Quién valida | `admin` cualquiera; `contador` si el movimiento está en sus sedes; nunca uno propio. |
| Proveedores | Por RUC **o por DNI** (proveedores sin RUC). |

**Por defecto (no confirmado; documentado para revisar con el cliente):**

- Las cuentas de proyecto y sus movimientos los ven el admin y todo contador; el admin_sede solo si la tiene asignada.
- CAJA RONALD, CAJA CHICA y BCP PUBLICIDAD quedan **sin sede** en el seed: no se pueden usar hasta que el admin les asigne sede desde "Cuentas".

---

## 3. Tickets de Linear incluidos

| Ticket | Uso en este sprint |
|---|---|
| [REN-27](https://linear.app/sprinta/issue/REN-27) | Alcance por sede de admin_sede y contador; retiro del superusuario. |
| [REN-28](https://linear.app/sprinta/issue/REN-28) | Campos operativos del egreso, sin destino ni proyectos; concepto obligatorio. |
| [REN-29](https://linear.app/sprinta/issue/REN-29) | Catálogo de 20 cuentas con códigos genéricos. |
| [REN-30](https://linear.app/sprinta/issue/REN-30) | Validación en dos etapas con filtro de sede para el contador. |
| [REN-31](https://linear.app/sprinta/issue/REN-31) | Evidencias y duplicados; el emisor se identifica por RUC o DNI. |
| [REN-32](https://linear.app/sprinta/issue/REN-32) | Proveedores por RUC o DNI; keys de VerificaPE por scope. |
| [REN-33](https://linear.app/sprinta/issue/REN-33) | Pagos al personal vinculados a usuarios. |
| [REN-34](https://linear.app/sprinta/issue/REN-34) | **Nuevo:** cuentas con responsables y catálogo administrable. |
| [REN-35](https://linear.app/sprinta/issue/REN-35) | **Nuevo:** transferencias internas con validación. |

## 4. Fuera del compromiso

| Ticket | Motivo |
|---|---|
| [REN-11](https://linear.app/sprinta/issue/REN-11) | Pagos mixtos: requiere desglose de pagos. Sprint 3. |
| [REN-12](https://linear.app/sprinta/issue/REN-12) | Inconsistencias y pagos sin comprobante. Sprint 3. |
| [REN-15](https://linear.app/sprinta/issue/REN-15) / [REN-21](https://linear.app/sprinta/issue/REN-21) | Cuadre de entrega diaria de efectivo. Sprint 4. |
| [REN-19](https://linear.app/sprinta/issue/REN-19) | Inventario: falta decidir frontera con SIBI. Sprint 5. |
| — | Evidencias (voucher) en transferencias. Sprint 3. |

---

## 5. Modelo implementado

### 5.1 Datos (migraciones reescritas; ninguna aplicada aún)

| Migración | Contenido |
|---|---|
| 037 | `accounting_accounts.is_project`, códigos `CTA-NNNN` por secuencia, nombre único entre cuentas activas, tabla `accounting_account_users`, seed de 20 cuentas y asignación al admin activo. Se eliminan las cuentas temporales del Sprint 1, sin egresos asociados. |
| 038 | Medio de pago Efectivo/Transferencia/Yape y número de operación. Se elimina `accounting_expenses.location_id`; la sede del egreso se obtiene de la cuenta de origen. |
| 040 | Validación en dos etapas de egresos. |
| 041 | Comprobante, clave de duplicado y evidencias en bucket privado. |
| 042 | `accounting_suppliers` con `document_type` (`ruc`/`dni`) + `document_number`. |
| 043 | Beneficiario (usuario) en pagos al personal. |
| 044 | `accounting_transfers` con validación en dos etapas. |

Validadas en un PostgreSQL 17 local, con stubs de los objetos de Supabase: sintaxis, orden, idempotencia del seed y restricciones.

### 5.2 Reglas de acceso (la API es la frontera)

| Acción | admin | admin_sede | contador |
|---|---|---|---|
| Ver cuentas | todas | las de sus sedes | sus sedes + proyectos |
| Ver egresos y transferencias | todos | sus sedes (transferencia: origen o destino) | sus sedes + proyectos |
| Registrar egreso o transferencia | desde cuentas asignadas | desde cuentas asignadas | ✗ |
| Corregir egreso (pendiente o rechazado) | todos | origen asignado | en su alcance |
| Eliminar | todos | origen asignado | ✗ |
| Validar o rechazar | todos, salvo propios | ✗ | en su alcance, salvo propios |
| Catálogo de cuentas y responsables | ✓ | ✗ | ✗ |
| Asignar sedes a admin, admin_sede y contador | ✓ | ✗ | ✗ |

Reglas centralizadas en `renaceris_backend/src/accounting/accounting-access.ts`.

### 5.3 Endpoints nuevos o cambiados

- `GET /accounting/accounts`: cuentas visibles, con `configured`, `assignedToMe` y `canOperate`.
- `GET /accounting/accounts/transfer-destinations`: cuentas configuradas que pueden recibir una transferencia.
- `GET|POST /accounting/account-catalog`, `PATCH /accounting/account-catalog/:id`, `PUT /accounting/account-catalog/:id/holders`, `GET /accounting/account-catalog/holders` (solo admin).
- `GET|POST /accounting/transfers`, `PATCH|DELETE /accounting/transfers/:id`, `POST /accounting/transfers/:id/review`.
- `GET /accounting/expenses?mine=true`.
- `GET /accounting/suppliers/lookup/:documentType/:documentNumber`.
- Retirados: `/accounting/projects`, `/accounting/permissions*`.

---

## 6. Tareas

### ✅ ACC-19 — Retiro del superusuario contable

Backend `bc6f7ef`, frontend `5b05c68`.

### ✅ ACC-20 — Asignación de sedes por el admin

**Relacionado:** REN-27  
`Users.tsx`: el botón **Sedes** aparece para admin_sede, contador y el propio admin cuando quien opera es admin; el formulario de alta permite sedes para contador. Backend: `PUT /users/:id/locations` exige rol `admin` si el usuario destino es admin, admin_sede o contador, y audita cada cambio.

### ✅ ACC-21 — Alcance por sede y cuentas asignadas en la API

**Relacionado:** REN-27, REN-34  
`SCOPED_ROLES = admin_sede, contador`, `resolveViewer()` (sedes + cuentas asignadas), operar solo desde cuentas asignadas y configuradas, contador corrige sin registrar ni eliminar, validación con filtro de sede. `ParseUUIDPipe` en `PATCH/DELETE /accounting/expenses/:id`.

### ✅ ACC-22 — Quién valida y quién elimina

**Relacionado:** REN-30  
Valida el admin o el contador de la sede, nunca el autor. Eliminan el admin y el admin_sede responsable de la cuenta de origen.

### ✅ ACC-23 — Pantalla de contabilidad

**Relacionado:** REN-27, REN-34, REN-35  
Formulario con modalidad **Egreso corriente / Transferencia interna** y el origen primero; lista de egresos y transferencias con estado, "Solo míos" y revisión; tarjetas por cuenta con entradas, salidas y egresos; pestaña **Cuentas** para el admin; banners por rol.

### ✅ ACC-24 — Tests del modelo

Backend 294 tests. Destacan `accounting.scope.spec.ts`, `accounting.transfers.spec.ts`, `accounting.account-catalog.spec.ts`, `accounting.roles.spec.ts` (matriz de endpoints) y `users.locations.spec.ts`. Frontend: build y 51 tests.

### ✅ ACC-27 — Catálogo de cuentas y traspaso de caja

**Relacionado:** REN-29, REN-34  
Migración 037 reescrita, `AccountingAccountsService` y pestaña **Cuentas**.

### ✅ ACC-28 — Transferencias internas

**Relacionado:** REN-35  
Migración 044, `AccountingTransfersService`. No suman a egresos, mueven el saldo de ambas cuentas, piden nº de operación si hay banco y pasan por validación.

### ✅ ACC-29 — Proveedores por DNI

**Relacionado:** REN-32  
Migración 042 con tipo y número de documento; consulta de DNI directa a VerificaPE (`lookupDniPerson`); aviso si la persona ya existe con RUC 10.

---

### ⏳ ACC-25 — Despliegue

**Prioridad:** P0 · **Día:** 24/09

1. Verificar en producción que los nombres de `locations` coinciden con la semilla de la 037 (`huánuco/huanuco`, `pucallpa`, `lima`, `puerto fest`).
2. Aplicar en el SQL editor, en orden: **037 → 038 → 040 → 041 → 042 → 043 → 044**. Si PostgREST no ve las tablas nuevas: `NOTIFY pgrst, 'reload schema';`.
3. Regenerar `supabase-schema.ts` desde el esquema aplicado.
4. Configurar en Railway `VERIFICAPE_DNI_API_KEY` y `VERIFICAPE_RUC_API_KEY`; agregarlas a `.env.example`.
5. Push y despliegue: **backend primero, luego frontend**.
6. Desde **Cuentas**: asignar sede a CAJA RONALD, CAJA CHICA y BCP PUBLICIDAD, y asignar CAJA MIGUEL, CAJA ADARLIN y CAJA CRISTIAN a sus administradores de sede.
7. Desde **Usuarios**: asignar sedes a cada admin_sede y a cada contador.

**Criterio de aceptación:** `GET /accounting/me`, `/summary`, `/expenses` y `/transfers` responden 200 en producción para admin, admin_sede y contador.

### ⏳ ACC-26 — Validación E2E con datos reales

**Prioridad:** P0 · **Día:** 24/09

| Caso | Resultado esperado |
|---|---|
| admin_sede registra un egreso desde su caja | Pendiente de validación; sede = la de su caja |
| admin_sede intenta registrar desde la caja general o un banco de su sede | 403 |
| admin_sede usa "Solo míos" | Solo sus egresos |
| Transferencia CAJA HUANUCO → CAJA MIGUEL | No cambia el total de egresos; saldo −/+ en ambas cuentas |
| Transferencia con banco sin nº de operación | Rechazada con mensaje |
| Contador con 3 sedes | Ve las 3 sedes y los proyectos; corrige; no registra ni elimina |
| Contador valida un movimiento de otra sede | 403 |
| Validar un movimiento propio | Bloqueado |
| Traspaso de CAJA MIGUEL a otro admin_sede | El nuevo opera; el anterior recibe 403 |
| Cuenta sin sede como origen | Rechazada hasta configurarla |
| Mismo comprobante dos veces | Bloqueado; se registra solo con motivo |
| Proveedor por DNI real | Completa el nombre; dirección manual |
| RUC real nuevo | Completa razón social y dirección |
| Pago a personal | Exige usuario registrado de la sede |

---

## 7. Plan diario

### Martes 23/09

- ✅ Retiro del superusuario (ACC-19).
- ✅ Decisiones de modelo con el cliente.

### Miércoles 24/09

- ✅ ACC-20 a ACC-24, ACC-27 a ACC-29 (backend `069d881`, frontend `9b83153`).
- ⏳ ACC-25 despliegue.
- ⏳ ACC-26 validación E2E.
- Actualizar Linear y el [TODO](TODO-modulo-contabilidad.md).

---

## 8. Definiciones técnicas no negociables

- El alcance por sede y las cuentas asignadas se deciden en NestJS; el frontend solo lo refleja.
- Se registra solo desde una cuenta asignada y configurada (con sede, o de proyecto).
- Un egreso pertenece a la sede de su cuenta de origen.
- Una transferencia nunca cuenta como egreso.
- El contador nunca recibe datos personales de pacientes.
- Nadie valida un movimiento que registró; uno validado no se edita ni se elimina.
- Las migraciones se aplican antes de desplegar el backend que las usa.

## 9. Entregables al deadline

- Migraciones 037–044 (sin 039) aplicadas en producción.
- Backend y frontend desplegados con REN-27 a REN-35.
- Cuentas configuradas (sedes y responsables) y sedes de usuarios asignadas.
- Credenciales de VerificaPE por scope en Railway.
- Validación E2E registrada en Linear.

## 10. Riesgos y decisiones pendientes

1. **Cuentas sin sede:** CAJA RONALD, CAJA CHICA y BCP PUBLICIDAD no se pueden usar hasta que el admin les asigne sede.
2. **Responsables de cajas de sede:** falta la lista definitiva de qué admin_sede recibe cada caja; hoy CAJA MIGUEL, ADARLIN y CRISTIAN quedan sin responsable.
3. **Visibilidad de proyectos:** por defecto la tienen admin y todo contador; confirmar con el cliente.
4. **Contador que corrige y luego valida** el mismo egreso (no lo registró): hoy está permitido; confirmar si se bloquea.
5. **Nombres de sedes:** si `locations` no coincide con la semilla, las cuentas quedan sin sede.
6. **API de RUC:** el endpoint `/ruc/{ruc}` y sus campos se asumieron; se confirman con una consulta real.
7. **Saldos por cuenta:** los cobros aún no registran la cuenta receptora; el saldo solo incluye transferencias y egresos (Sprint 3).
8. **Truncamiento a 1000 filas:** resumen, egresos, transferencias y el puente SIBI no paginan (Sprint 3).
