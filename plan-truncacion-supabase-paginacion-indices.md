# Truncación silenciosa en Supabase: pacientes, ventana del calendario e índices

## Contexto

Arreglamos el calendario que dejaba de pintar citas desde el 12/09: `AppointmentsService.findAll` traía 1000 de 1060 filas porque **PostgREST corta todo `select` en 1000 filas** (`db-max-rows`) sin devolver error. Ya está paginado con `.range()` + desempate por `id`.

Ese fix destapó una clase de bug, no un caso aislado. Al auditar el resto encontré **un bug activo hoy en producción** más grave que el original, y confirmé que el crecimiento de la tabla (~350 citas/mes, hoy 1060 filas / 1.4 MB por request) vuelve inevitable acotar la ventana.

Datos medidos contra la BD real (no estimaciones):

| Medición | Valor |
|---|---|
| `appointments` en BD | 1060 (rango jul-2026 → oct-2026) |
| `patients` en BD | 618 |
| Payload de `GET /appointments` | **1.4 MB**, 38 columnas + 6 joins |
| Payload de una ventana de 1 mes | 565 KB (419 filas) |
| Citas por paciente | máx 28, mediana 1 |

### El bug activo: `GET /patients` está roto ahora mismo

[patients.service.ts:51-57](renaceris_backend/src/patients/patients.service.ts#L51-L57) arma el mapa de "última cita" con un `.in('patient_id', patientIds)` donde `patientIds` son **los 618 pacientes**. Eso genera una URL de **22.973 caracteres**; Node aborta con `UND_ERR_HEADERS_OVERFLOW`. Verificado: con 380 ids → HTTP 200; con 400 ids → falla. Ya pasamos ese umbral.

Y el error **no se chequea** — `const { data: appointments } = await …` descarta el `error`, así que `appointments` queda `undefined`, el `Map` vacío y **todos** los pacientes salen con `last_appointment: null` en [Patients.tsx:91](renaceris_front/src/pages/Patients.tsx#L91). Falla silenciosa y total.

### Resultado buscado

1. `GET /patients` vuelve a devolver la última cita real de cada paciente.
2. `GET /appointments` acepta una ventana de fechas y deja de crecer sin techo, **sin cambiar ningún comportamiento de UI**.
3. Los índices que sirven a estas consultas quedan versionados en `migrations/`, no solo en el proyecto remoto.

---

## Alcance

### 1. Helper de paginación compartido — `renaceris_backend/src/utils/supabase-pagination.ts` (nuevo)

Extraer lo ya escrito en `findAll` a un helper reutilizable, junto a `date.utils.ts`:

```ts
export async function fetchAllPages<T>(
  buildQuery: () => PostgrestFilterBuilder<any, any, T[]>,
  pageSize = 1000,
  maxPages = 50,
): Promise<T[]>
```

Recibe un **builder**, no una query: el builder de `supabase-js` se consume al hacer `await`, así que cada página necesita una query nueva. Corta cuando una página vuelve incompleta o vacía; lanza si `error`.

Refactorizar [appointments.service.ts:298](renaceris_backend/src/appointments/appointments.service.ts#L298) `findAll` para usarlo (mismo comportamiento, menos código inline).

### 2. Fix de `PatientsService.findAll` — [patients.service.ts:26-74](renaceris_backend/src/patients/patients.service.ts#L26-L74)

Dos cambios:

- **Quitar el `.in('patient_id', patientIds)`.** No filtra nada: `patientIds` es la lista completa de pacientes, así que toda cita del sistema la satisface. Es un no-op que cuesta 23 KB de URL y hace fallar la request. Se reemplaza por `fetchAllPages` sobre `appointments` con el mismo `select` y `.order('datetime', { ascending: false })`. La lógica del `Map` (primera aparición = más reciente) queda igual.
- **Chequear el `error`.** Hoy se descarta. Que falle ruidoso, no que devuelva 618 pacientes sin historial.

Agregar además `.order('id')` como desempate en el `select` de `patients` ([:44](renaceris_backend/src/patients/patients.service.ts#L44), ordena por `created_at DESC` sin tiebreaker) — misma inestabilidad que corregimos en citas, relevante cuando `patients` pase de 1000 y necesite paginarse.

> Nota de deuda: aun sin el `.in()`, esto sigue trayendo la tabla `appointments` entera en cada carga de `/patients` para quedarse con una fila por paciente. El fix correcto a futuro es un `DISTINCT ON (patient_id)` en la base. Queda anotado, no en este cambio.

### 3. Ventana de fechas en `GET /appointments` — cambio compatible hacia atrás

**Backend** — [appointments.controller.ts:60-75](renaceris_backend/src/appointments/appointments.controller.ts#L60-L75) y `findAll`:

Agregar `startDate` / `endDate` opcionales (`YYYY-MM-DD`). Reutilizar el nombre y el formato que **ya existe** en `GET /appointments/paid/list` ([:103-134](renaceris_backend/src/appointments/appointments.controller.ts#L103-L134)) en vez de inventar `from`/`to`. Sin parámetros el endpoint se comporta exactamente como hoy.

Convención horaria — crítica: el front trata los timestamps como **hora de pared local** ([Calendar.tsx:183-187](renaceris_front/src/components/Calendar.tsx#L183-L187) borra el offset). El filtro debe construirse igual que `findSameDayPatientServiceAppointments` ([appointments.service.ts:361](renaceris_backend/src/appointments/appointments.service.ts#L361)), que ancla en `-05:00`, o los días del borde se pierden. Propagar también a `findMyAppointments` ([:339](renaceris_backend/src/appointments/appointments.service.ts#L339)).

**Frontend** — [api.ts:195-208](renaceris_front/src/lib/api.ts#L195-L208): agregar `startDate`/`endDate` a `getAll` y `getMyAppointments` (una línea cada uno, ya serializan params).

[Appointments.tsx](renaceris_front/src/pages/Appointments.tsx) — `loadData` ([:263-274](renaceris_front/src/pages/Appointments.tsx#L263-L274)) pide una **ventana amplia por defecto: −3 meses / +12 meses**. Ninguna vista cambia de comportamiento porque los datos actuales (jul→oct 2026) caben enteros dentro de esa ventana.

Una salvedad obligatoria: el filtro **Desde/Hasta** ([:1038-1040](renaceris_front/src/pages/Appointments.tsx#L1038-L1040), UI en :1219-1240) hoy alcanza cualquier fecha porque el array no tiene límite. Si el usuario elige un `Desde` anterior a la ventana, `loadData` debe **ensanchar** el fetch a `min(filterDateFrom, hoy−3m)` / `max(filterDateTo, hoy+12m)` y `filterDateFrom`/`filterDateTo` entran como dependencias del efecto. Sin esto la pestaña Lista se degrada en silencio — exactamente el bug que estamos arreglando.

Espejar los mismos parámetros en el refetch post-creación ([:615](renaceris_front/src/pages/Appointments.tsx#L615)), que hoy además **pierde el filtro `doctorId`** de los médicos. Ese olvido es preexistente; corregirlo acá evita repetirlo con la ventana.

**No tocar** [DoctorPanel.tsx:289](renaceris_front/src/pages/DoctorPanel.tsx#L289): su banner de "procedimiento sin concluir" ([:566](renaceris_front/src/pages/DoctorPanel.tsx#L566)) es un escaneo all-time a propósito — un procedimiento colgado de días pasados es justo lo que debe encontrar. Se queda sin ventana.

### 4. Índices — `renaceris_backend/migrations/023_indexes_for_paginated_queries.sql` (nuevo)

Los índices versionados hoy son todos `patient_id`- o `doctor_id`-leading (migraciones 007, 013, 017, 018, 021). **Ninguno sirve** a `WHERE location_id = ? AND datetime BETWEEN ? AND ? ORDER BY datetime, id`.

Advertencia importante: el esquema base (`CREATE TABLE` de `patients`, `appointments`, `services`, `locations`, `users`, `proformas`, `clinical_notes`, `audit_logs`) **no está en el repo** — vive solo en el proyecto Supabase remoto. Tampoco puedo leer `pg_indexes` desde acá: no hay RPC de SQL y PostgREST tiene `EXPLAIN` deshabilitado (probado, devuelve `PGRST107`). Por eso el primer paso de la migración es un bloque de auditoría comentado que se corre en el SQL editor antes de aplicar nada:

```sql
-- Correr primero y pegar el resultado:
SELECT tablename, indexname, indexdef FROM pg_indexes
WHERE schemaname = 'public'
  AND tablename IN ('appointments','patients','payments','service_orders','clinical_notes','proformas')
ORDER BY tablename, indexname;
```

Índices propuestos, todos `IF NOT EXISTS` (idempotentes, seguros aunque ya existan):

```sql
-- findAll: ventana de fechas + orden estable, con y sin filtro de sede
CREATE INDEX IF NOT EXISTS idx_appointments_datetime_id
  ON public.appointments (datetime, id);
CREATE INDEX IF NOT EXISTS idx_appointments_location_datetime
  ON public.appointments (location_id, datetime);

-- getFullProfile / findByPatient: .eq(patient_id) ORDER BY datetime DESC
CREATE INDEX IF NOT EXISTS idx_appointments_patient_datetime
  ON public.appointments (patient_id, datetime DESC);

-- findAll de pacientes: orden con desempate
CREATE INDEX IF NOT EXISTS idx_patients_created_at_id
  ON public.patients (created_at DESC, id);
```

**Expectativa honesta:** con 1060 filas Postgres va a seguir haciendo seq scan — a este tamaño leer la tabla entera es más barato que el índice, y estos índices **no van a mejorar los tiempos hoy**. Los 800-1670 ms medidos son payload y red, no falta de índice. Son seguro para cuando la tabla llegue a decenas de miles, y el de `(patient_id, datetime DESC)` es el único que puede notarse antes, porque sirve directo al Cronograma sin ordenar.

Los índices de `payments.created_at`, `service_orders.status`, `clinical_notes.patient_id` y `audit_logs.(entity_type, entity_id)` quedan **fuera**: pertenecen al ticket de reportes (ver abajo).

---

## Respuesta a las dos preguntas del requerimiento

### 1. ¿Conviene optimizar el "Cronograma de Continuidad"?

**No necesita el fix de paginación, y ya está razonablemente optimizado.** No lo alimenta `findByPatient` sino `PatientsService.getFullProfile` ([patients.service.ts:245-329](renaceris_backend/src/patients/patients.service.ts#L245-L329)), que ya hace lo correcto: `select` de columnas acotadas, las dos queries en `Promise.all`, y `.limit(50)`. Con máximo 28 citas por paciente en la BD, el tope de 1000 filas ni se acerca.

Pero tiene el **mismo patrón de falla, en chico**: ese `.limit(50)` ([:297](renaceris_backend/src/patients/patients.service.ts#L297) y la ruta de fallback en [:315](renaceris_backend/src/patients/patients.service.ts#L315)) recorta en silencio, sin "ver más" ni aviso. Un paciente de tratamiento largo con 51 citas pierde su historial más antiguo y nadie se entera — y [PatientProfile.tsx](renaceris_front/src/pages/PatientProfile.tsx) hace derivaciones **all-time** sobre ese array: total de citas ([:215](renaceris_front/src/pages/PatientProfile.tsx#L215), [:921](renaceris_front/src/pages/PatientProfile.tsx#L921)), completadas ([:897](renaceris_front/src/pages/PatientProfile.tsx#L897)) y el **PDF del expediente** ([:245-261](renaceris_front/src/pages/PatientProfile.tsx#L245-L261)). O sea que hoy el tope de 50 no solo esconde filas: produce **conteos y un expediente PDF incorrectos** para cualquier paciente que lo supere.

Acción en este cambio, mínima y suficiente: **subir el tope a 200** y dejar comentado por qué existe. 200 cubre con holgura el máximo real (28) y el PDF deja de mentir. Paginar de verdad el Cronograma no se justifica con estos volúmenes.

Aparte: [api.appointments.getByPatient](renaceris_front/src/lib/api.ts#L295) y su ruta `GET /appointments/by-patient/:id` **no tienen ningún llamador en el front**. Código muerto; lo señalo, no lo borro en este cambio.

### 2. ¿Los índices ayudan a este fix?

Sí, pero **no como aceleración hoy** — ver la advertencia de la sección 4. Lo que sí resuelven los índices es un problema que la paginación *introduce*: `.range(from, to)` con offsets profundos degrada de forma cuadrática contra un seq scan + sort, porque cada página vuelve a ordenar la tabla para descartar las filas anteriores. Con `(datetime, id)` indexado, el orden ya viene dado y el offset se resuelve por el índice. Con 2 páginas no se nota; con 10 sí.

Lo verdaderamente importante que sale del punto 2 no es qué índices faltan, sino que **el esquema base no está versionado**. Las definiciones de las tablas centrales solo existen en el proyecto remoto, la numeración de `migrations/` está duplicada (hay dos `012_`, dos `013_`, dos `015_`, dos `016_`, dos `017_`, dos `018_`) y no hay runner ni CI: se pegan a mano en el SQL editor. Vale un ticket propio para hacer `pg_dump --schema-only` y versionarlo.

---

## Fuera de alcance (documentado, sin tocar código)

Auditoría completa de `.select()` sin `.limit()`/`.range()`. **Estos suman totales en JS sobre queries que pueden truncarse — el resultado no son filas faltantes, son montos mal calculados.** Hoy se salvan porque `resolveRange` ([reports.service.ts:42-52](renaceris_backend/src/reports/reports.service.ts#L42-L52)) usa "solo hoy" por defecto; se rompen cuando alguien elige un trimestre o un año:

| Sitio | Riesgo |
|---|---|
| [reports.service.ts:374](renaceris_backend/src/reports/reports.service.ts#L374), [:398](renaceris_backend/src/reports/reports.service.ts#L398), [:416](renaceris_backend/src/reports/reports.service.ts#L416), [:425](renaceris_backend/src/reports/reports.service.ts#L425) | Ingresos de caja y saldos pendientes |
| [reports.service.ts:268](renaceris_backend/src/reports/reports.service.ts#L268) | Export de citas — un CSV truncado parece completo |
| [reports.service.ts:521](renaceris_backend/src/reports/reports.service.ts#L521) | Export SIBI/SUNAT — consecuencia tributaria |
| [accounting.service.ts:879](renaceris_backend/src/accounting/accounting.service.ts#L879), [:889](renaceris_backend/src/accounting/accounting.service.ts#L889), [:466](renaceris_backend/src/accounting/accounting.service.ts#L466), [:498](renaceris_backend/src/accounting/accounting.service.ts#L498), [:628](renaceris_backend/src/accounting/accounting.service.ts#L628), [:818](renaceris_backend/src/accounting/accounting.service.ts#L818) | Flujo de caja y gastos |
| [service-orders.service.ts:286](renaceris_backend/src/service-orders/service-orders.service.ts#L286), [:191](renaceris_backend/src/service-orders/service-orders.service.ts#L191) | Órdenes activas; proformas ya aceptadas reaparecen como pendientes |
| [patients.service.ts:211](renaceris_backend/src/patients/patients.service.ts#L211) | `getMyPatients` — mismo fan-out, un médico con +1000 citas pierde pacientes |
| [patients.service.ts:347](renaceris_backend/src/patients/patients.service.ts#L347) | `advancedSearch` con join `!inner` sin límite (la rama sin filtros sí tiene `.limit(200)`) |

Una vez que exista `fetchAllPages`, aplicarlo acá es mecánico — pero cada sitio necesita verificar que los montos no se muevan, y eso merece su propio ticket.

---

## Verificación

**1. Fix de pacientes (el bug activo).** Antes del cambio, reproducir el fallo:
```bash
cd renaceris_backend && node -e "…"   # .in() con los 618 ids -> UND_ERR_HEADERS_OVERFLOW
```
Después: `GET /patients` con el backend levantado debe devolver 618 pacientes y **≥ 1 con `last_appointment` no nulo**. Contraste contra la BD: 588 pacientes distintos tienen al menos una cita, así que ese debe ser el conteo de `last_appointment` no nulos. En la UI, [Patients.tsx](renaceris_front/src/pages/Patients.tsx) debe mostrar la columna de última cita poblada.

**2. Paginación de citas.** `GET /appointments` sin parámetros → **1060 filas**, ids únicos 1060, última `2026-10-07`. Las 10 citas del 12/09 presentes y `377d5fad-60d6-404d-8925-678c7d6d68b0` incluida. (Ya verificado para el fix aplicado; re-verificar tras el refactor a `fetchAllPages`.)

**3. Ventana de fechas.** `GET /appointments?startDate=2026-09-01&endDate=2026-09-30` → **419 filas**, ninguna fuera de septiembre. Verificar los bordes con la convención `-05:00`: una cita del 01/09 temprano y otra del 30/09 tarde deben entrar. Sin parámetros → 1060 (compatibilidad).

**4. Regresión de UI** — el punto que más riesgo tiene. En `/appointments`:
- Calendario: navegar Día → Semana → Mes y adelantar más allá del 12/09; las citas siguen apareciendo.
- Lista: la tabla muestra el mismo total que antes del cambio.
- **Desde/Hasta con una fecha anterior a hoy−3 meses**: confirmar que el fetch se ensancha y los resultados aparecen. Este es el caso que rompe si el ensanchado no se implementa.
- Crear una cita y confirmar que el refetch la trae y que un médico conserva su filtro.
- `/doctor-panel`: el banner de procedimiento sin concluir sigue detectando un procedimiento de un día pasado.
- `/patients/:id`: el Cronograma sigue listando y el PDF exporta el total correcto.

**5. Índices.** Correr primero la query de auditoría de `pg_indexes` y comparar contra lo propuesto — puede que alguno ya exista creado desde el dashboard. Aplicar y re-verificar el punto 3; no esperar cambio de tiempos a este volumen.

**6.** `npx tsc --noEmit` en backend y `npm run build` en front.
