# Backlog técnico — Sprint 1

## Módulo Contabilidad — Renaceris

**Proyecto Linear:** [Módulo Contabilidad (REN)](https://linear.app/sprinta/project/modulo-contabilidad-9d9e4c0af62b)  
**Sprint:** 25/08/2026 — 27/08/2026  
**Deadline:** 27/08/2026, fin del día  
**Objetivo:** entregar un vertical slice funcional con avance real de backend y frontend.  
**Acceso del sprint:** únicamente `admin`.

---

## 1. Objetivo del sprint

Al cierre del sprint debe existir una primera pantalla de contabilidad accesible solo por administradores, conectada al backend, que permita:

1. Consultar ingresos consolidados por rango de fechas.
2. Filtrar por sede y forma de pago; dejar preparada la extensión a servicio y doctor.
3. Consultar cuentas por cobrar de órdenes de servicio y citas con pago parcial.
4. Registrar egresos manuales categorizados.
5. Visualizar un balance básico: ingresos confirmados menos egresos registrados.

Este sprint no busca cerrar todo el proyecto contable. Busca validar la arquitectura, el control de acceso y el primer flujo financiero de extremo a extremo.

---

## 2. Tickets de Linear incluidos

| Ticket | Uso en este sprint |
|---|---|
| [REN-9](https://linear.app/sprinta/issue/REN-9) | Reportes de ingresos: primera versión funcional. |
| [REN-10](https://linear.app/sprinta/issue/REN-10) | Cuentas por cobrar: primera versión funcional. |
| [REN-13](https://linear.app/sprinta/issue/REN-13) | Registro de egresos categorizados. |
| [REN-14](https://linear.app/sprinta/issue/REN-14) | Balance básico y separación conceptual de efectivo/banco. |
| [REN-16](https://linear.app/sprinta/issue/REN-16) | En este sprint se restringe a `admin`; el perfil contador queda posterior. |

## 3. Tickets explícitamente fuera del compromiso

| Ticket | Motivo |
|---|---|
| [REN-11](https://linear.app/sprinta/issue/REN-11) | El desglose de pagos mixtos requiere un cambio de modelo de datos y compatibilidad con pagos históricos. Se diseñará antes de implementarlo. |
| [REN-12](https://linear.app/sprinta/issue/REN-12) | Requiere definir catálogo de inconsistencias y corrección de pagos sin comprobante. |
| [REN-15](https://linear.app/sprinta/issue/REN-15) | El cuadre de entrega diaria sigue condicionado a la investigación operativa. |
| [REN-18](https://linear.app/sprinta/issue/REN-18) | La plantilla exacta de SIBI aún es una dependencia externa. |
| [REN-19](https://linear.app/sprinta/issue/REN-19) | Inventario requiere definición de frontera con el inventario de SIBI y no pertenece al vertical financiero inicial. |
| [REN-20](https://linear.app/sprinta/issue/REN-20) | Se mantiene como dependencia para exportación y categorización definitiva. |
| [REN-21](https://linear.app/sprinta/issue/REN-21) | Investigación operativa, no bloqueante para el dashboard inicial. |

---

## 4. Diagnóstico de la infraestructura actual

### 4.1 Acceso a Supabase

`renaceris_front/src/lib/supabase.ts` crea un cliente browser con:

- `VITE_SUPABASE_URL`.
- `VITE_SUPABASE_ANON_KEY`.
- Realtime limitado a `eventsPerSecond: 10`.

El frontend no debe consultar ni mutar directamente las tablas contables con este cliente. La migración `015_enable_rls_backend_only.sql` revoca el acceso de `anon` y `authenticated` a las tablas de negocio y establece el backend como frontera de autorización.

La integración contable debe seguir este flujo:

```text
Accounting.tsx
  -> api.ts
  -> Bearer access_token
  -> NestJS AuthGuard
  -> NestJS RolesGuard (@Roles('admin'))
  -> AccountingService
  -> SupabaseService.getClient()
  -> Supabase service_role
```

No se debe importar `supabase` desde la nueva pantalla para leer ingresos, egresos o saldos.

### 4.2 Backend actual

- NestJS organiza cada dominio como módulo, controller y service.
- `SupabaseService.getClient()` devuelve el cliente server-only con `SUPABASE_SERVICE_ROLE_KEY`.
- `AuthGuard` valida el Bearer token y asigna `request.user`.
- `RolesGuard` valida los roles declarados con `@Roles(...)`.
- `ReportsController` actualmente usa `@Roles('admin', 'supervisor')`; no debe reutilizarse para exponer el módulo contable porque este módulo será solo de administrador.
- La API frontend centraliza las llamadas en `renaceris_front/src/lib/api.ts`, incluye refresh automático de tokens y manejo común de errores.

### 4.3 Modelo de datos existente relevante

El schema tipado de `renaceris_backend/src/interfaces/supabase-schema.ts` confirma estas fuentes:

- `payments`: cobros confirmados, monto, fecha, medio de pago, comprobante, cita y/o orden de servicio.
- `appointments`: monto total, monto pagado, confirmación de pago, sede, servicio, doctor y paciente.
- `service_orders`: tratamientos en cuotas, `total_amount`, `paid_amount`, estado, servicio y paciente.
- `services`: servicio y sede asociada.
- `locations`: sedes.
- `patients`: datos del paciente para el detalle administrativo.

Actualmente no existen tablas de egresos, cuentas financieras ni caja contable. Estas tablas deben agregarse mediante migración versionada.

### 4.4 Riesgo de duplicidad de ingresos

El reporte existente `ReportsService.getCashIncome()` combina:

1. Registros de `payments`.
2. Citas completas con `payment_confirmed = true`.

La lógica contable no debe sumar ambos registros cuando representan el mismo cobro. Para este sprint se define:

- Fuente primaria: pagos confirmados de `payments`.
- Fallback legado: una cita pagada directamente solo se incluye si no existe un `payments` asociado.
- Una fila de `payments` se cuenta una sola vez aunque tenga `appointment_id` y `service_order_id`.
- Fechas: usar `payment_date` cuando exista; de lo contrario, `created_at`.
- Rangos de fecha: límite inferior inclusivo y límite superior exclusivo, usando la zona horaria `America/Lima`, consistente con `ReportsService.resolveRange()`.

---

## 5. Tareas comprometidas

### ACC-01 — Crear frontera del módulo y acceso exclusivo para `admin`

**Prioridad:** P0  
**Tipo:** Backend + Frontend  
**Día:** 25/08  
**Relacionado:** REN-16

#### Backend

- Crear `AccountingModule`, `AccountingController` y `AccountingService`.
- Registrar `AccountingModule` en `AppModule`.
- Proteger el controller con `@UseGuards(AuthGuard, RolesGuard)`.
- Aplicar `@Roles('admin')` a nivel de controller o endpoint.
- No permitir `supervisor`, `recepcionista`, `medico`, `especialista_estetico` ni `ventas`, aunque puedan acceder a otros reportes.
- Responder `401` sin token y `403` con token válido pero rol no autorizado.
- No confiar en el rol enviado desde el frontend; el rol efectivo debe salir de `AuthGuard`/`AuthService`.

#### Frontend

- Crear lazy route `/accounting` con `ProtectedRoute allowedRoles={['admin']}`.
- Mostrar la opción “Contabilidad” en la navegación únicamente cuando `user?.role === 'admin'`.
- Si otro rol intenta navegar manualmente a `/accounting`, mostrar el comportamiento estándar de acceso denegado.
- No exponer la pantalla durante la carga de sesión si el usuario aún no está validado.

#### Criterios de aceptación

- Admin puede cargar `/accounting`.
- Cualquier otro rol recibe `403` desde API y no puede visualizar la información mediante la UI.
- No existe endpoint contable sin `AuthGuard` y `RolesGuard`.

---

### ACC-02 — Implementar endpoints de lectura financiera

**Prioridad:** P0  
**Tipo:** Backend  
**Día:** 25–26/08  
**Relacionado:** REN-9, REN-10

#### Contrato propuesto

```text
GET /accounting/summary
GET /accounting/receivables
```

Query params comunes:

```text
startDate=YYYY-MM-DD
endDate=YYYY-MM-DD
locationId=<uuid>       opcional
paymentMethod=<string>  opcional
serviceId=<uuid>        opcional
doctorId=<uuid>         opcional
```

#### `GET /accounting/summary`

Debe devolver, como mínimo:

```ts
{
  period: { start: string; end: string; timezone: 'America/Lima' },
  totals: {
    income: number,
    expenses: number,
    net: number,
    confirmedPaymentCount: number,
  },
  byPaymentMethod: Array<{ key: string; label: string; amount: number; count: number }>,
  byLocation: Array<{ locationId: string; locationName: string; amount: number; count: number }>,
  entries: Array<{
    id: string,
    paymentDate: string,
    amount: number,
    paymentMethod: string | null,
    billingType: string | null,
    billingNumber: string | null,
    appointmentId: string | null,
    serviceOrderId: string | null,
    serviceName: string | null,
    locationName: string | null,
  }>,
}
```

Reglas:

- Incluir únicamente pagos confirmados.
- No incluir citas canceladas como ingreso nuevo.
- No duplicar cobros entre `payments` y el fallback legado de `appointments`.
- Normalizar el medio de pago solo en la respuesta; no alterar todavía los valores históricos de la base de datos.
- Mantener el valor original para trazabilidad.
- La categorización definitiva queda pendiente de la plantilla y reporte de SIBI de REN-20.
- Implementar paginación o un límite seguro para `entries`; los totales no deben depender de un límite visual.

#### `GET /accounting/receivables`

Debe consolidar:

- `service_orders` no canceladas con `total_amount > paid_amount`.
- Citas sin `service_order_id` con `total_amount > amount_paid` y que no estén canceladas.
- No duplicar una cita de tratamiento cuyo saldo ya está representado por `service_orders`.

Respuesta mínima:

```ts
{
  totalPending: number,
  count: number,
  items: Array<{
    source: 'service_order' | 'appointment',
    sourceId: string,
    patientId: string,
    patientName: string,
    patientDni: string | null,
    serviceName: string | null,
    locationName: string | null,
    totalAmount: number,
    paidAmount: number,
    pendingAmount: number,
    lastPaymentDate: string | null,
  }>,
}
```

#### Criterios de aceptación

- Los totales se calculan en backend, no en React.
- Un filtro de fecha usa días completos de Lima sin desfase UTC.
- Un pago ligado a cita y orden aparece una sola vez.
- El saldo de una orden se calcula como `max(totalAmount - paidAmount, 0)`.
- Los endpoints tienen pruebas unitarias para duplicidad, rango de fechas y filtros.

---

### ACC-03 — Crear persistencia y API de egresos

**Prioridad:** P0  
**Tipo:** Supabase migration + Backend  
**Día:** 25–26/08  
**Relacionado:** REN-13, REN-14

#### Migración requerida

Crear una migración con `supabase migration new` antes de editar SQL. No modificar manualmente el archivo generado por el schema tipado como sustituto de una migración.

Tablas mínimas:

```sql
accounting_accounts
-------------------
id uuid primary key default gen_random_uuid()
code text unique not null
name text not null
account_type text not null check (account_type in ('cash', 'bank'))
location_id uuid null references locations(id)
active boolean not null default true
created_at timestamptz not null default now()

accounting_expenses
------------------
id uuid primary key default gen_random_uuid()
expense_date date not null
amount numeric(12,2) not null check (amount > 0)
category text not null
account_id uuid not null references accounting_accounts(id)
description text null
receipt_reference text null
location_id uuid null references locations(id)
created_by uuid not null references users(id)
created_at timestamptz not null default now()
updated_at timestamptz not null default now()
```

Índices mínimos:

- `accounting_expenses(expense_date)`.
- `accounting_expenses(account_id, expense_date)`.
- `accounting_expenses(location_id, expense_date)`.
- `accounting_expenses(created_by)`.

Seguridad Supabase:

- Habilitar RLS en ambas tablas.
- No abrir grants para `anon` ni `authenticated`; el backend seguirá siendo la frontera de acceso.
- No usar `SECURITY DEFINER` para resolver accesos.
- Actualizar `supabase-schema.ts` desde la fuente de schema después de aplicar la migración.
- Crear cuentas iniciales de prueba solo en entorno local/staging; no insertar cuentas productivas sin confirmación de nombres BCP/BBVA y sede.

#### API

```text
GET    /accounting/expenses?startDate&endDate&locationId&accountId&category
POST   /accounting/expenses
PATCH  /accounting/expenses/:id
DELETE /accounting/expenses/:id
```

Payload de alta/edición:

```ts
{
  expenseDate: '2026-08-25',
  amount: 150.50,
  category: 'servicios',
  accountId: '<uuid>',
  description?: 'Compra operativa',
  receiptReference?: 'R-001',
  locationId?: '<uuid>',
}
```

Reglas de negocio:

- El monto debe ser positivo y tener máximo dos decimales.
- La cuenta debe existir, estar activa y ser de tipo `cash` o `bank`.
- `created_by` debe tomarse de `request.user.id`, nunca del body.
- El backend debe validar fechas y UUIDs con DTOs `class-validator`.
- No permitir eliminación silenciosa de un egreso ya usado en un periodo cerrado; en este sprint, si no existe cierre formal, registrar la operación en `audit_logs`.
- El balance se calcula como `ingresos confirmados - egresos registrados`.
- No confundir este balance con conciliación bancaria ni con libros contables.

#### Criterios de aceptación

- Admin puede registrar, listar, editar y eliminar un egreso.
- Cada registro conserva usuario creador y fecha.
- Un egreso en efectivo y uno en banco se reflejan en cuentas distintas.
- Un egreso registrado aparece en el balance del periodo sin recargar manualmente la página.
- Un rol distinto de admin no puede ejecutar ninguna operación.

---

### ACC-04 — Construir pantalla administrativa de contabilidad

**Prioridad:** P0  
**Tipo:** Frontend  
**Día:** 26–27/08  
**Relacionado:** REN-9, REN-10, REN-13, REN-14

#### Implementación

- Crear `renaceris_front/src/pages/Accounting.tsx`.
- Agregar `api.accounting` en `renaceris_front/src/lib/api.ts`; usar `fetchWithRetry` y `handleResponse` existentes.
- No llamar a Supabase desde la página.
- Estado de filtros en React: `startDate`, `endDate`, `locationId`, `paymentMethod`.
- Valor inicial del periodo: día actual en formato local de Lima; evitar construir fechas con `new Date('YYYY-MM-DD')` si eso introduce conversión UTC.
- Cargar resumen y cuentas por cobrar en paralelo cuando cambien los filtros.
- Cancelar o ignorar respuestas obsoletas si el usuario cambia rápidamente los filtros.
- Mostrar estados de `loading`, `error`, `empty` y datos válidos.

#### Componentes funcionales mínimos

1. Tarjetas de ingresos, egresos, neto, pagos confirmados y cuentas por cobrar.
2. Filtros de fecha, sede y medio de pago.
3. Tabla de ingresos con monto, fecha, medio, servicio, sede y comprobante.
4. Tabla de cuentas por cobrar con paciente, servicio, total, pagado y saldo.
5. Modal o panel para registrar egresos.
6. Tabla de egresos con edición y eliminación.
7. Resumen separado de efectivo y banco a partir de `accounting_accounts`.

#### Criterios de aceptación

- La pantalla responde a los filtros sin cálculos financieros duplicados en el frontend.
- Los montos se muestran con dos decimales y moneda `S/`.
- Un error del backend no deja datos anteriores aparentando ser actuales.
- Tras crear o editar un egreso, se invalidan y recargan resumen/listado.
- La pantalla no aparece en navegación para usuarios no admin.

---

### ACC-05 — Integración, pruebas y endurecimiento

**Prioridad:** P0  
**Tipo:** Backend + Frontend  
**Día:** 27/08  
**Relacionado:** todos los tickets comprometidos

#### Backend

- Unit tests de `AccountingService` con mock de Supabase para:
  - pagos confirmados;
  - fallback de cita pagada sin fila en `payments`;
  - prevención de doble conteo;
  - saldo de órdenes y citas parciales;
  - filtro por sede y fechas Lima;
  - validación de egresos;
  - rechazo de cuenta inactiva;
  - auditoría del usuario creador.
- Tests de roles para `admin`, `supervisor`, `recepcionista`, `medico`, `especialista_estetico` y `ventas`.
- Verificar que no existan endpoints contables registrados sin guardas.
- Verificar respuesta consistente de errores `401`, `403`, `404` y `400`.

#### Frontend

- Validar navegación admin-only.
- Validar que un rol no admin no pueda acceder por URL directa.
- Validar carga inicial, cambio de filtros, formulario inválido, error de API y actualización posterior a un egreso.
- Ejecutar `npm run build` en frontend y `npm test`/tests focalizados en backend.

#### Smoke test de aceptación

Con datos controlados:

```text
Ingreso confirmado: S/ 500.00
Egreso en cash:      S/  80.00
Egreso en bank:      S/ 120.00
Neto esperado:      S/ 300.00
```

El dashboard debe mostrar esos valores sin duplicar el ingreso ni mezclar las cuentas.

---

## 6. Plan diario

### Martes 25/08 — Fundación y persistencia

- ACC-01: módulo, ruta y autorización admin-only.
- Crear estructura de `AccountingModule` y contratos iniciales.
- Crear migración de cuentas y egresos.
- Implementar DTO y alta/listado de egresos.
- Dejar disponible un primer endpoint verificable desde Swagger/Postman.

### Miércoles 26/08 — Lógica financiera y primer frontend

- Implementar `GET /accounting/summary`.
- Implementar `GET /accounting/receivables`.
- Implementar edición/eliminación de egresos y balance.
- Crear `Accounting.tsx`, navegación y métodos de `api.ts`.
- Conectar tarjetas, filtros y tabla principal.

### Jueves 27/08 — Integración y cierre

- Conectar cuentas por cobrar y egresos.
- Corregir fechas, duplicidades y estados de carga.
- Ejecutar pruebas backend y build frontend.
- Validar autorización por rol.
- Realizar smoke test financiero.
- Documentar pendientes de SIBI, pagos mixtos y caja.

---

## 7. Definiciones técnicas no negociables

- El frontend no accede directamente a tablas contables mediante `supabase.ts`.
- El acceso real se autoriza en NestJS con `AuthGuard` + `RolesGuard` + `@Roles('admin')`.
- La clave `SUPABASE_SERVICE_ROLE_KEY` nunca se expone al navegador.
- No se agregan permisos Data API públicos para las nuevas tablas.
- Los datos financieros se calculan en backend.
- La zona horaria de negocio es `America/Lima`.
- La migración SQL es la fuente de verdad; `supabase-schema.ts` se actualiza después.
- No se crean asientos contables, impuestos, conciliación bancaria automática ni facturación SUNAT.
- No se implementan pagos mixtos modificando `payments` sin una migración y estrategia de compatibilidad histórica.

---

## 8. Entregables al deadline

- Ruta `/accounting` funcional para `admin`.
- API protegida de resumen, cuentas por cobrar y egresos.
- Migración versionada para cuentas y egresos con RLS backend-only.
- Dashboard inicial conectado a datos reales de Supabase vía NestJS.
- Registro y consulta de egresos.
- Balance básico de ingresos menos egresos.
- Pruebas de autorización y lógica financiera principal.
- Lista actualizada de pendientes para Sprint 2: pagos mixtos, exportación SIBI, inconsistencias, cuadre de entrega e inventario.

## 9. Riesgos y decisiones pendientes

1. **Categorización:** usar categorías provisionales hasta recibir términos y reportes de SIBI de REN-20.
2. **Cuentas bancarias:** no sembrar nombres definitivos de BCP/BBVA sin validar la estructura real por sede.
3. **Pagos históricos:** revisar muestras de junio antes de declarar equivalencia total entre `appointments` y `payments`.
4. **Contador:** aunque REN-16 contempla un perfil restringido, el alcance confirmado para este primer sprint es únicamente `admin`.
5. **Caja de recepción:** no se implementa apertura/cierre formal en este sprint; solo se deja la separación de cuentas preparada para REN-15.
