# Plan de implementación — Reevaluaciones y multisesiones

Secuencia técnica para llevar el sistema actual al modelo de [06-modelo-de-negocio-final.md](06-modelo-de-negocio-final.md). El orden respeta la prioridad del negocio: primero los puntos 3 en adelante, y la vigencia de consulta al final por depender de definiciones abiertas.

> **Estado a la fecha.** La fase 0 —catálogo de modalidad y limpieza— ya está en producción, en tres tandas de migraciones (`024` a `027`). Este documento describe lo que falta. El detalle operativo por rol y pantalla vive en [07-plan-de-ejecucion.md](07-plan-de-ejecucion.md), que es el plan vigente; aquí queda la secuencia técnica.

## 0. Lo que ya está aplicado

| Migración | Qué dejó en producción |
|---|---|
| `024` | `treatment_group_id` eliminado de `appointments` y `proformas`, y la rama muerta del RPC de órdenes |
| `025` y `026` | Enum de modalidad y ocho columnas de catálogo en `services` |
| `027` | `price_modality_enum` y `service_modality_enum` desambiguados, valores `single` / `package` / `per_session`, y `allowed_session_models` eliminada |

Vocabulario que rige de aquí en adelante:

- **`service_orders.price_modality`** — a qué precio se cerró la orden. Antes se llamaba `modality`.
- **`services.service_modality`** — cuántas veces viene la paciente y cuántas veces se cotiza. Una sola por servicio, congelada en el episodio al abrirse. Antes se llamaba `session_model`.

## 1. Orden de ejecución

| Fase | Entrega | Puntos del modelo | Depende de |
|---|---|---|---|
| 1 | Episodios y paquetes multisesión | 3 | — |
| 2 | Reevaluación como tipo de cita | 5 | Fase 1 |
| 3 | Cotización desde una reevaluación | 4 y 6 | Fase 2 |
| 4 | Configuración por servicio y automatismos | 2, 3, 4, 5 | Fases 1 a 3 |
| 5 | Vigencia de consulta | 1 | Definiciones de Ronald |

Las fases 1 a 3 son el núcleo del módulo. La fase 4 convierte reglas fijas en configuración de catálogo. La fase 5 es independiente y puede reordenarse.

## 2. Fase 1 — Episodios y paquetes multisesión

**Objetivo.** Que un paquete de masajes se venda una vez y se consuma en N sesiones controladas.

### Migración

```sql
-- treatment_episodes
id                    uuid primary key default gen_random_uuid()
service_modality      service_modality_enum not null   -- congelada al abrir
patient_id            uuid not null references patients(id) on delete cascade
service_id            uuid not null references services(id) on delete restrict
location_id           uuid not null references locations(id)
origin_appointment_id uuid null references appointments(id) on delete set null
origin                text not null default 'system'
                        check (origin in ('system', 'legacy', 'external'))
status                text not null default 'active'
                        check (status in ('active', 'completed', 'abandoned', 'cancelled'))
opened_at             timestamptz not null default now()
closed_at             timestamptz null
created_by            uuid null references users(id)

-- service_orders
episode_id            uuid null references treatment_episodes(id) on delete set null
sessions_total        integer not null default 1 check (sessions_total >= 1)
sessions_delivered    integer not null default 0 check (sessions_delivered >= 0)
valid_until           date null
included_in_order_id  uuid null references service_orders(id) on delete set null

-- service_orders.source admite un tercer origen: el paquete de procedimiento
-- directo, que se vende sin proforma. El CHECK vigente exige proforma_id
-- cuando source = 'proforma', así que necesita su propia rama.
source                check (source in ('proforma', 'legacy_manual', 'direct_sale'))

-- appointments
episode_id            uuid null references treatment_episodes(id) on delete set null
session_number        integer null check (session_number >= 1)
```

Índices mínimos: `treatment_episodes(patient_id, status)`, `service_orders(episode_id)`, `appointments(episode_id)`, `appointments(service_order_id, session_number)`.

Restricción de integridad: `sessions_delivered <= sessions_total`, salvo ampliación explícita registrada.

Retrocompatibilidad, y es la restricción que más importa de la fase: **sin backfill**. `episode_id` queda nulo en todo lo anterior al despliegue y ese es un estado permanente y legítimo.

De ahí se desprende una regla que hay que escribir en el código, no dar por supuesta: **el bloqueo por sesiones agotadas solo aplica a órdenes con episodio**. Nada impuso nunca "una ejecución por orden" —`appointments.service_order_id` es N:1—, así que hoy ya hay órdenes con varias citas colgando, probablemente masajes. Validar `sessions_delivered >= sessions_total` contra órdenes legadas, que quedan en `sessions_total = 1`, rompería ese flujo en la segunda sesión.

### Backend

- Trigger o función que recalcule `sessions_delivered` al completar, cancelar o reabrir una cita de tratamiento, con bloqueo de fila sobre la orden para evitar sobreventa concurrente. Mismo patrón que `enforce_appointment_proforma_limit` en la migración `020`.
- Validación al crear una cita de tratamiento en `appointments.service.ts`: si la orden tiene episodio y ya entregó todas sus sesiones, rechazar con mensaje explícito. Las órdenes sin episodio no se validan.
- Venta de paquete de procedimiento directo: crear la orden con `source = 'direct_sale'` y sin `proforma_id`, con total igual al precio de catálogo por las sesiones vendidas. No pasa por médico ni por proforma, que es justamente lo que distingue a ese flujo.
- Asignación automática de `session_number` y de `episode_id` heredado de la orden.
- Endpoint de agendamiento en bloque que cree las N citas del paquete en una sola transacción lógica, validando disponibilidad de cada slot y devolviendo el detalle de las que no pudieron crearse.
- Endpoints de episodio: detalle consolidado con órdenes, sesiones, saldo y estado.

### Frontend

- En la selección de orden al agendar tratamiento, mostrar "sesión k de N", sesiones restantes y vencimiento.
- Bloquear la creación cuando no quedan sesiones, con acción alternativa de ampliar el paquete para roles autorizados.
- Vista de episodio en el perfil del paciente, con la línea de tiempo de sesiones y órdenes.
- Acción de agendar el paquete completo con periodicidad sugerida.

### Criterios de aceptación

- Un paquete de 10 sesiones permite agendar exactamente 10 citas activas y bloquea la 11.
- Cancelar una sesión libera el cupo y no altera el saldo de la orden.
- Los pagos siguen acumulándose en la orden y el saldo se calcula igual que hoy.
- Una orden anterior al despliegue, sin episodio, no muestra contador ni se bloquea al agendar la segunda cita.
- Un paquete de procedimiento directo genera su orden `direct_sale` y descuenta sesiones igual que un paquete de tratamiento.

## 3. Fase 2 — Reevaluación como tipo de cita

**Objetivo.** Poder agendar un control post-operatorio sin inventar una consulta ni una orden.

### Migración

```sql
ALTER TYPE public.appointment_type_enum ADD VALUE IF NOT EXISTS 'reevaluation';
```

Sola en su migración: Postgres no permite usar un valor de enum en la misma transacción en la que se agrega. Antes de aplicarla hay que revisar cada comparación con `'consultation'` y cada agrupación por `appointment_type`, porque un valor nuevo cambia el resultado de consultas ya escritas.

El precio de la reevaluación cobrada no necesita estructura nueva: `service_prices` ya está tipada por `appointment_type_enum`, así que el valor nuevo habilita su fila.

**Adopción de órdenes legadas, no reclasificación masiva.** Como no hay backfill, el día del despliegue no existe ningún episodio, así que toda reevaluación caería en "sin tratamiento previo" y la feature arrancaría con fricción tonta.

La solución es que el selector de tratamiento a evaluar liste **los episodios de la paciente más sus órdenes sin episodio**. Elegir una orden legada le crea el episodio en ese momento, con `origin = 'legacy'`, y la orden queda adoptada.

Es adopción a demanda: no se toca ninguna fila hasta que una persona señala esa orden concreta para atenderla. Preserva la regla de no hacer backfill —que existe para no inventar paquetes fantasma— y a la vez hace que lo que recepción ve en pantalla sea lo que espera ver: los tratamientos que la paciente ya tiene.

Las citas históricas con `category = 'reevaluation'` **no se reclasifican**. Conservan su tipo original; la separación en reportes aplica desde la fecha de despliegue y el reporte lo dice.

### Backend

- La reevaluación no exige orden ni total **para agendarse**. Requiere `episode_id`. Si durante la atención se genera una orden, esa orden se vincula a la misma cita; son dos cosas distintas y ninguna es requisito de la otra.
- Endpoint de adopción: dado un `service_order_id` sin episodio, crear el episodio `legacy` y vincular la orden y sus citas ya existentes. Idempotente: si la orden ya tiene episodio, devuelve el que hay.
- Precio cero por defecto. Si el servicio la cobra, se resuelve por `service_prices` con el nuevo tipo.
- `reports.service.ts`: separar reevaluaciones del conteo de consultas.
- Resumen contable: la reevaluación gratuita es una atención sin ingreso y no debe romper ningún total.
- Validación de modalidad virtual para el tipo nuevo.

### Frontend

- Nuevo tipo en el selector de tipo de cita, junto a consulta, tratamiento y procedimiento directo.
- Al abrir una reevaluación desde el perfil del paciente, preseleccionar el episodio de origen.
- El selector "Tratamiento a evaluar" muestra **dos grupos**: episodios activos y órdenes sin episodio, que se adoptan al elegirlas. El tercer grupo previsto —"Sin tratamiento previo en el sistema", que abría un episodio `external`— **queda suspendido**: ver el parche más abajo.
- Retirar el uso de `category = 'reevaluation'` como marca funcional en `NewAppointment.tsx` y `Appointments.tsx`; `category` queda solo como color.
- **"Gestionar tratamiento previo" se conserva.** La orden `legacy_manual` no era solo una percha: registra lo que esta misma clínica hizo antes del sistema, con su precio pendiente de regularizar si hace falta. Lo que cambia es el desenlace: al registrarla, la agenda salta al tipo `reevaluation` con esa orden ya elegida como tratamiento a evaluar, en vez de armar una cita `treatment` con `category = 'reevaluation'` a mano.

### Parche vigente — el episodio `external` no se ofrece en la interfaz

Decisión temporal y **solo de interfaz**, cerrada en [06-modelo-de-negocio-final.md](06-modelo-de-negocio-final.md#parche-vigente-el-episodio-external-no-se-ofrece-en-la-interfaz). La paciente operada en otra clínica pasa primero por consulta y de ahí sale su orden oficial, así que para cuando vuelve a control ya tiene historial: el atajo no era la puerta por la que el caso llega, y competía con "Gestionar tratamiento previo".

| Capa | Estado |
|---|---|
| `treatment_episodes.origin` | Sin cambios. Conserva `system`, `legacy` y `external`. Ninguna migración |
| `POST /treatment-episodes` | Sin cambios. Sigue aceptando `origin = 'external'` |
| `buildEpisodeOptions` | Devuelve dos grupos. El parche está anotado ahí mismo |
| Pantallas de agenda | Conservan la rama `external`, inalcanzable mientras el grupo no vuelva |

Revertir es devolver el grupo a `buildEpisodeOptions` y quitar el test que lo vigila. Ninguna otra capa asume su ausencia.

**Quién registra un tratamiento previo, y hasta dónde llega.** El médico entra al mismo modal que recepción, y a propósito: es quien sabe qué se le hizo a la paciente antes del sistema. Para él el flujo termina en ese registro, porque un médico no crea citas, solo las recibe — `POST /appointments` y la ruta `/appointments/new` están acotados a `admin` y `recepcionista`. Recepción agenda después la reevaluación, y el selector ya encuentra la orden lista para adoptar. No es una limitación de esta pantalla: es el reparto de roles del sistema, y el modal lo dice en vez de mandar al médico a un formulario que no le abre.

### Criterios de aceptación

- Una reevaluación se agenda **sin necesitar una orden previa**, sin proforma y sin cobro. Que después reciba una orden por un tratamiento del día no contradice esto: lo que no se exige es tenerla de antemano.
- No aparece en el conteo de consultas de ningún reporte.
- Elegir una orden legada en el selector crea su episodio y agrupa las citas que ya colgaban de esa orden.
- Una paciente sin ninguna orden llega a la reevaluación por una de dos vías: su consulta ya le dejó una orden, o la clínica registra lo que le hizo antes del sistema con "Gestionar tratamiento previo" y la reevaluación adopta esa orden. Abrir un episodio `external` a mano queda fuera de la interfaz mientras el parche siga vigente.

## 4. Fase 3 — Cotización desde una reevaluación

**Objetivo.** Desbloquear láser CO₂, tatuajes y la reevaluación que incorpora otro tratamiento.

### La guarda de la cita de tratamiento

Antes de tocar las proformas hay que corregir una regla más profunda. `appointments.service.ts:168` exige orden a toda cita de tratamiento, y en una serie `per_session` la orden de la sesión 2 nace **durante** la sesión 2. La guarda pasa a ser:

```text
cita de ejecución  →  exige orden  O  episodio en modalidad per_session
```

Acotada por la modalidad congelada en el episodio: en `single` y `package` la orden sigue siendo obligatoria al agendar. Sin esto, cada sesión de láser tendría que agendarse como reevaluación, lo que describe mal la visita y cuenta sesiones como controles en los reportes.

### Cambios de regla

La restricción "solo consultas pueden cotizar" vive en varios lugares y se cambia en todos a la vez:

| Lugar | Cambio |
|---|---|
| `renaceris_backend/src/proformas/proformas.service.ts:183` | Aceptar `consultation`, `reevaluation` y `treatment` |
| `renaceris_backend/src/proformas/proformas.service.ts:75` | `getForAppointment` devuelve proformas para los tres tipos |
| `renaceris_backend/src/appointments/appointments.service.ts:168` | Exigir orden **o** episodio `per_session` |
| RPC `create_service_order_from_proforma` | Aceptar los tres tipos de cita de origen |
| `renaceris_backend/src/service-orders/service-orders.service.ts:37` | Misma validación en el servicio |
| `renaceris_front/src/components/AppointmentDetailModal.tsx:281` | `canUseProforma` incluye reevaluación y tratamiento sin orden |

Se conservan sin cambios: la exigencia de procedimiento iniciado antes de cotizar, el límite de tres tratamientos por cita, la unicidad de proforma por tratamiento y la restricción de sede del tratamiento cotizado.

### Backend

- Permitir que una cita de reevaluación quede vinculada como sesión de la orden creada en esa misma visita, poblando `service_order_id` y `session_number` sin cambiar el tipo de cita.
- Regla nueva: cuando una reevaluación genera una orden, la etiqueta derivada de la cita pasa a "Reevaluación y tratamiento" en agenda y reportes.
- Cuando el tratamiento cotizado pertenece a otra línea de tratamiento, se abre un episodio nuevo. La cita mantiene el episodio que evalúa y la orden pertenece al episodio nuevo.

### Frontend

- Panel de proforma disponible en reevaluaciones, con el mismo comportamiento que en consulta.
- Al crear la orden desde una reevaluación, ofrecer ejecutar la sesión en la misma cita o agendarla.
- Vista de serie: sesiones y proformas de un episodio en una sola línea de tiempo.

### Criterios de aceptación

- La sesión 2 de láser CO₂ se agenda **como cita de tratamiento sin orden**, se cotiza en la visita y se cobra, con precio distinto al de la sesión 1.
- Una cita de tratamiento de un episodio `package` o `single` sigue exigiendo orden al agendarse.
- Una reevaluación de rinoplastia puede cotizar bótox y generar su orden sin registrar una consulta nueva.
- Ninguna de esas citas suma al conteo de consultas.

## 5. Fase 4 — Configuración por servicio y automatismos

**Objetivo.** Que el comportamiento deje de estar en el código y pase al catálogo.

Los campos ya existen en `services` desde la fase 0: `service_modality`, `default_sessions`, `package_validity_days`, `min_days_between_sessions`, `followup_schedule`, `reevaluation_pricing`, `reevaluation_duration_minutes`. Lo que falta es que el resto del sistema los consuma.

Efectos:

- La sección "Modalidad del Servicio" de `Services.tsx` ya permite editarlos, incluidos los servicios legados. La modalidad `per_session` no se ofrece a procedimientos directos, porque cotizar cada sesión es lo que ese flujo evita.
- Al crear una orden, `sessions_total` y `valid_until` se precargan desde el servicio.
- Al completar una cirugía, el sistema propone la serie de controles.
- Al agendar una sesión antes del intervalo mínimo, se muestra una advertencia.

## 6. Fase 5 — Vigencia de consulta

Depende de las definiciones de [05-decisiones-pendientes.md](05-decisiones-pendientes.md). Alcance técnico previsto:

```sql
-- patient_consultation_credits
id                    uuid primary key default gen_random_uuid()
patient_id            uuid not null references patients(id) on delete cascade
origin_appointment_id uuid not null references appointments(id) on delete cascade
granted_at            timestamptz not null default now()
valid_until           date not null
scope                 text not null check (scope in ('global', 'location'))
location_id           uuid null references locations(id)
status                text not null default 'active'
                        check (status in ('active', 'expired', 'revoked'))
```

Más un motivo de exoneración en la cita de consulta, con valores como `vigencia`, `campaña`, `referido`, `convenio` y `cortesía`, y un indicador de consultas exoneradas en reportes.

## 7. Definiciones técnicas no negociables

Se mantienen las mismas que rigen el módulo contable:

- El frontend no consulta Supabase directamente; toda lectura y escritura pasa por NestJS con `AuthGuard` y `RolesGuard`.
- Cada cambio de esquema es una migración versionada; `supabase-schema.ts` se regenera después.
- Las tablas nuevas nacen con RLS habilitado y sin permisos para `anon` ni `authenticated`.
- La zona horaria de negocio es `America/Lima`.
- Ningún cálculo financiero se hace en React.
- Ninguna regla de negocio se apoya en `appointment_category`.

## 8. Pruebas mínimas por fase

- **Fase 1**: sobreventa de sesiones, concurrencia de dos agendamientos simultáneos, cancelación que libera cupo, orden legada intacta.
- **Fase 2**: reevaluación agendada sin orden previa, adopción de una orden legada desde el selector, registro de tratamiento previo que desemboca en esa adopción, exclusión del conteo de consultas. El episodio `external` a mano queda fuera de la interfaz por el parche vigente.
- **Fase 3**: proforma desde reevaluación, orden creada desde reevaluación, sesión ejecutada en la misma cita, límite de tres tratamientos, restricción de sede.
- **Fase 4**: precarga desde catálogo, advertencia de intervalo mínimo, propuesta de controles.
- **Fase 5**: crédito vigente y vencido, exoneración con motivo, reporte de consultas exoneradas.
- **Transversal**: pruebas de roles para cada endpoint nuevo y verificación de que los reportes y el resumen contable no cambian sus totales históricos.

## 9. Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Ampliar el enum de tipo de cita rompe consultas existentes | Alto | Inventariar todas las comparaciones con `appointment_type` antes de migrar |
| Relajar la regla de proforma en un solo lugar | Alto | Los cinco puntos de la fase 3 se cambian juntos y se prueban de extremo a extremo |
| Contador de sesiones desincronizado | Medio | Recálculo derivado desde citas más bloqueo de fila; job de verificación periódica |
| Órdenes `legacy_manual` usadas como percha | Medio | No se reclasifican: el flujo sigue vivo porque registra tratamientos reales de esta clínica. La adopción a demanda las incorpora al modelo una por una, cuando alguien las señala |
| Reportes con un tipo de cita nuevo no contemplado | Medio | Actualizar reportes y contabilidad en la misma entrega que la fase 2 |
| Definiciones abiertas del punto 1 | Bajo | La fase 5 está aislada y no bloquea el resto |
| El límite de sesiones aplicado a órdenes legadas | Alto | El bloqueo solo corre si la orden tiene episodio; probar explícitamente una orden vieja con dos citas |
| `especialista_estetico` no puede crear proformas | Medio | Solo condiciona la fase 3 y solo para láser. Confirmar con el negocio quién ejecuta esas sesiones |
