# Base de Conocimiento — Renaceris / Módulo de Contabilidad (Borrador)

> Documento de trabajo. Sirve como **punto de partida** para proponer un nuevo módulo de contabilidad.
> No pretende reemplazar los sistemas de facturación y contabilidad que la clínica ya utiliza.
> Fecha del borrador: 2026-07-20.

---

## 1. Contexto del negocio

**Renaceris** es una clínica estética que opera legalmente en el Perú. Este software representa la **primera digitalización** de sus procesos de:

- **Agendamiento de citas** (a cargo de recepcionistas).
- **Atención médica** (los doctores inician procedimientos, generan órdenes de servicio / proformas y registran la bitácora del paciente).

La clínica **ya cuenta con un sistema propio de facturación y contabilidad**. El presente módulo **no busca reemplazarlo** ni sustituir libros contables, sino **acelerar y complementar** el flujo de información.

### Roles identificados (tabla `users.role`)
- **Recepcionista**: agenda citas, registra pagos y confirma cobros.
- **Doctor**: inicia/finaliza procedimientos, genera proformas y órdenes de servicio, escribe notas clínicas (bitácora), emite recetas.
- **Administración / Contabilidad** (rol destino de este módulo): consumiría reportes y exportaciones.

---

## 2. Entendimiento del software a partir del esquema

El backend (NestJS + Supabase/PostgreSQL) modela el negocio alrededor de **citas → servicios → pagos**. A continuación, las entidades relevantes para lo financiero.

### 2.1 Entidades centrales del flujo de dinero

| Entidad | Rol en el negocio | Campos financieros clave |
|---|---|---|
| **`appointments`** (citas) | Evento raíz: paciente + doctor + sede + servicio en una fecha | `total_amount`, `amount_paid`, `payment_method`, `payment_type`, `payment_confirmed`, `billing_number`, `billing_type`, `transaction_code`, `status`, `procedure_status`, `category`, `appointment_type` |
| **`payments`** (pagos) | Registro de cada cobro; puede ligarse a una cita y/o a una orden de servicio | `amount`, `payment_method`, `payment_date`, `confirmed`, `billing_number`, `billing_type`, `transaction_code`, `notes` |
| **`service_orders`** (órdenes de servicio) | Compromiso de servicio a pagar, normalmente derivado de una proforma | `total_amount`, `paid_amount`, `status`, `modality` |
| **`proformas`** | Cotización que emite el doctor con distintas modalidades de precio | `price_modality_1`, `price_modality_2`, `price_no_modality` |
| **`services`** + **`service_prices`** | Catálogo de servicios y su precio referencial por categoría de cita | `referential_price`, `is_active`, `appointment_category` |
| **`locations`** (sedes) | Sucursales de la clínica; toda cita y servicio pertenecen a una sede | `name`, `city` |
| **`patients`** | Paciente / cliente | `dni`, `name`, `last_name`, `clinical_record_number` |

### 2.2 Cómo se relaciona el dinero (flujo lógico)

```
patient ──> appointment ──(genera)──> proforma ──> service_order
                 │                                      │
                 └──────────── payments ───────────────┘
                        (amount, payment_method,
                         payment_date, confirmed,
                         billing_number/type, transaction_code)
```

- Un **pago** (`payments`) puede referenciar `appointment_id` y/o `service_order_id`.
- La **cita** guarda también su propio resumen de cobro (`total_amount` vs `amount_paid`, `payment_confirmed`).
- La **orden de servicio** lleva su saldo (`total_amount` vs `paid_amount`, `status`), útil para tratamientos en cuotas/modalidades.
- Los campos `billing_number` / `billing_type` sugieren que el **número y tipo de comprobante se registran aquí de forma referencial**, pero la emisión formal ocurre en el sistema externo de facturación.

### 2.3 Enumeraciones relevantes
- **`appointment_category`**: `doctor_consultation`, `treatment`, `laser`, `reevaluation`, `pre_surgery_exam`, `dr_heider`.
- **`appointment_type_enum`**: `consultation`, `treatment`, `direct_procedure`.
- **`service_modality_enum`**: `Sin modalidad`, `Modalidad 1`, `Modalidad 2` (modalidades de precio/pago de un tratamiento).

### 2.4 Entidades de soporte (no financieras, pero relevantes para trazabilidad)
- **`audit_logs`**: registro de acciones (quién, qué, cuándo, cambios). Base para auditar movimientos de dinero.
- **`clinical_notes`** (bitácora), **`prescriptions`**, **`forms`**: operación clínica; fuera del alcance contable pero confirman el contexto.
- Tablas `*_backup_temp` (`appointment_payments_backup_temp`, `services_backup_temp`): respaldos de migración; **no usar como fuente de verdad**.

### 2.5 Observaciones / vacíos detectados (a validar con el cliente)
- No existe una tabla de **egresos/gastos** ni de **caja diaria (apertura/cierre)**. El "flujo de caja" hoy se infiere de `payments`.
- `payment_method` y `billing_type` son texto libre (`string`), no enums → **riesgo de inconsistencia** para reportes (ej. "Efectivo" vs "EFECTIVO" vs "cash").
- No hay tabla de **conceptos contables / centros de costo** ni mapeo hacia el plan de cuentas del sistema externo.
- No se modela explícitamente **anulaciones / notas de crédito / reembolsos** (solo `confirmed` y `status`).
- La relación entre `total_amount` de la cita y la suma de `payments` no está garantizada por el esquema → conviene reconciliar.

---

## 3. Objetivo del módulo de contabilidad

Alcance acotado a **dos funciones**:

1. **Procesar más rápido los reportes y la data de la clínica en base al flujo de caja actual.**
   Consolidar automáticamente los ingresos ya registrados (`payments`, `appointments`, `service_orders`) en vistas y reportes de flujo de caja, por período, sede, método de pago y tipo de servicio.

2. **Exportar documentos/tablas para simplificar el trabajo con el software externo de facturación/contabilidad y permitir un contraste básico.**
   Generar archivos (Excel/CSV/PDF) con la estructura que el sistema contable externo pueda consumir o con la que el contador pueda cuadrar cifras rápidamente.

> **Fuera de alcance (explícito):** emisión de comprobantes electrónicos (SUNAT), libros contables oficiales, asientos de partida doble, declaración de impuestos. Todo eso permanece en el sistema externo.

---

## 4. Borrador de requerimientos

### 4.1 Requerimientos funcionales

**RF-01 — Consolidación de ingresos (flujo de caja)**
El sistema consolidará los pagos registrados (`payments`) enriquecidos con su cita, paciente, servicio y sede, para producir un flujo de caja por período.

**RF-02 — Filtros de reporte**
Los reportes deberán filtrarse, como mínimo, por: rango de fechas (`payment_date`), sede (`location_id`), método de pago (`payment_method`), tipo/categoría de servicio (`appointment_category`), estado de confirmación (`confirmed`/`payment_confirmed`) y doctor.

**RF-03 — Vistas de resumen (dashboard básico)**
Totales por día/semana/mes, por método de pago, por sede y por servicio. Comparativo simple entre `total_amount` esperado y `amount_paid` real (saldos pendientes).

**RF-04 — Reporte de cuentas por cobrar (saldos)**
A partir de `service_orders` (`total_amount` vs `paid_amount`, `status`) y de citas con pago parcial, listar los saldos pendientes por paciente/orden.

**RF-05 — Exportación de tablas**
Exportar cualquier reporte a **Excel/CSV**. Incluir una **exportación "puente"** con columnas alineables al sistema externo (fecha, comprobante `billing_number`, tipo `billing_type`, monto, método, paciente/DNI, sede, servicio, código de transacción).

**RF-06 — Exportación de documentos**
Generar PDF de: resumen de caja por período y detalle de pagos, con identificación de la clínica y la sede.

**RF-07 — Reconciliación básica**
Marcar/identificar diferencias entre lo registrado en el módulo y lo esperado (ej. citas atendidas sin pago confirmado, pagos sin `billing_number`, sumas de `payments` que no cuadran con `total_amount`).

**RF-08 — Trazabilidad**
Todo reporte/exportación debe ser reproducible y auditable (apoyarse en `audit_logs`; registrar quién exportó qué y cuándo).

### 4.2 Requerimientos no funcionales (borrador)
- **RNF-01 — Solo lectura sobre datos operativos.** El módulo lee `payments`, `appointments`, `service_orders`, etc.; no modifica la operación clínica.
- **RNF-02 — Control de acceso por rol.** Solo Administración/Contabilidad accede a los reportes financieros.
- **RNF-03 — Moneda y formato local.** Soles (PEN), formato de fecha e importes peruano.
- **RNF-04 — Consistencia de catálogos.** Normalizar `payment_method` / `billing_type` (idealmente vía enum o tabla de catálogo) para que los reportes sean confiables.
- **RNF-05 — Multi-sede.** Todos los reportes deben poder segmentarse y consolidarse por sede.

### 4.3 Entregables sugeridos (exportaciones concretas para el contador)
1. **Libro de ingresos / registro de ventas simplificado** (una fila por pago/comprobante).
2. **Resumen de caja diario** por sede y método de pago.
3. **Reporte de cuentas por cobrar** (saldos abiertos por orden de servicio).
4. **Reporte de excepciones/reconciliación** (pagos sin comprobante, citas sin pago, descuadres).

---

## 5. Supuestos y preguntas abiertas (para validar antes de detallar)

- ¿Qué **formato exacto** consume el software contable externo (plantilla Excel, CSV con columnas específicas, TXT SUNAT-like)?
- ¿Se registran **egresos/gastos** en algún lugar, o el módulo se limita a ingresos?
- ¿Existe manejo de **caja física** (apertura/cierre de turno) que debamos reflejar?
- ¿Cómo se tratan hoy **anulaciones, reembolsos y notas de crédito**?
- ¿El **comprobante** (`billing_number`/`billing_type`) se emite antes o después de registrar el pago en este sistema?
- ¿Los **descuentos/modalidades** afectan el importe cobrado y cómo debe reflejarse en el reporte?

---

## 6. Próximos pasos propuestos

1. Validar este borrador y las preguntas de la sección 5 con la clínica y su contador.
2. Confirmar el/los **formatos de exportación** objetivo del sistema externo.
3. Definir el **catálogo normalizado** de métodos de pago y tipos de comprobante.
4. Priorizar los reportes de la sección 4.3 para un primer MVP.
