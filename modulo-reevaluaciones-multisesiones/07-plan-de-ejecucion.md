# Plan de ejecución final — Reevaluación y Sesiones

Plan de trabajo sobre el modelo cerrado en [06-modelo-de-negocio-final.md](06-modelo-de-negocio-final.md). Sustituye a [04-plan-de-implementacion.md](04-plan-de-implementacion.md) donde difieran: aquel se escribió antes de auditar la base y antes de decidir la reutilización de la cita en el caso del mismo día.

Tres decisiones de alcance que definen todo lo que sigue:

1. **`treatment_group_id` está 100% en NULL.** Lo que era una migración de datos es una limpieza.
2. **En el caso del mismo día se reutiliza la cita de reevaluación.** No se crea una segunda cita `treatment`.
3. **Nada histórico se convierte.** El módulo rige para lo que se cree a partir del despliegue. Las órdenes y citas anteriores siguen funcionando exactamente como hoy, sin episodio y sin contadores de sesión. Si alguna vez hay que convertir un caso viejo, se hace por BD con un script puntual, nunca desde la UI.

## 1. Orden de ejecución

| Fase | Entrega | Desbloquea | Riesgo |
|---|---|---|---|
| **0. Catálogo y limpieza** ✅ | El servicio declara su modalidad, editable para todos los servicios | Todo lo demás | Aplicada en producción |
| **1. Episodios y paquetes** | Masajes: una orden, N sesiones, con control de entrega | Punto 3 del negocio | Medio. Toca `service_orders` |
| **2. Reevaluación como tipo de cita** | Controles post-quirúrgicos sin consulta falsa | Punto 5 | Medio. Enum nuevo, reportes |
| **3. Cotizar en reevaluación y tratar el mismo día** | Láser CO₂, tatuajes, bótox tras rinoplastia | Puntos 4 y 6 | Alto. Levanta el bloqueo central |
| **4. Vigencia de consulta** | Punto 1 | Nada de estas dos features | Aplazable |

La fase 0 va primero porque es la única que se despliega sin que nadie note un cambio de comportamiento, y deja a supervisión configurando el catálogo mientras se construye el resto. La fase 4 no pertenece a estas dos features y no debe retenerlas.

## 2. Regla de convivencia con lo existente

Es la regla que gobierna cada pantalla y cada consulta del módulo:

> Una orden sin `episode_id` es una orden legada. Se muestra y se cobra como hasta hoy, sin contador de sesiones, sin vencimiento y sin agrupación por episodio.

| Qué | Decisión |
|---|---|
| Órdenes y citas anteriores al despliegue | `episode_id` y `session_number` quedan en `NULL`. **Sin backfill.** |
| `sessions_total` en órdenes viejas | Se queda en su default `1`, que describe la realidad: vendieron una ejecución |
| Convertir un caso viejo en paquete | Script SQL puntual, ejecutado por un responsable técnico, con `audit_logs`. No hay UI para esto y no debe haberla |
| Catálogo de servicios legacy | **Sí se edita desde la UI.** Cambiar el modelo de un servicio afecta solo a las órdenes que se creen después; las emitidas conservan su comportamiento |
| Reportes | La separación de reevaluaciones aplica desde la fecha de despliegue, y el reporte lo dice en una nota al pie |

Cada componente que lea sesiones debe tolerar el nulo y no renderizar el bloque, en lugar de mostrar "sesión 1 de 1" en historiales de años anteriores.

## 3. Marcador de feature nueva

Las dos features introducen elementos que nadie sabe que existen. El precedente es caro: `treatment_group_id` murió en NULL porque su única entrada era un botón solo-ícono `UserPlus` sin label ([NewAppointment.tsx:918](../../renaceris_front/src/pages/NewAppointment.tsx#L918)). Un elemento que no se anuncia no se usa.

### Componente

`NewBadge` en `renaceris_ds`, exportado desde `src/index.ts` junto al resto:

```tsx
<NewBadge feature="reevaluation" />   // → píldora "NUEVO"
```

- Píldora de 9px, mayúsculas, borde `--primary` sobre `--primary-light`.
- **No** se implementa con `StatusPill`: sus tonos (amber, blue, green, red, slate) ya significan estado de cita, y un "NUEVO" en verde se leería como "completada". El marcador necesita una identidad visual propia que nunca compita con el estado.
- `ChoiceCard` recibe una prop nueva `badge?: ReactNode`, para que la tarjeta de "Reevaluación" lo lleve sin bifurcar el componente.
- Con `title` para que al pasar el cursor explique la feature en una línea. Es lo que convierte el marcador en descubrimiento y no en decoración.

### Registro con fecha de retiro

`renaceris_front/src/config/newFeatures.ts`:

```ts
export const NEW_FEATURES = {
  reevaluation:  { until: '2026-12-31', hint: 'Nuevo: cita de seguimiento, puede cotizar sin ser consulta' },
  serviceModality: { until: '2026-12-31', hint: 'Nuevo: el servicio define si se vende como sesión única, paquete o serie' },
  sameDayLink:   { until: '2026-12-31', hint: 'Nuevo: el tratamiento del mismo día usa esta misma cita' },
  sessionCounter:{ until: '2026-12-31', hint: 'Nuevo: control de sesiones vendidas y entregadas' },
} as const;
```

`NewBadge` devuelve `null` pasada la fecha. Un solo archivo retira todos los marcadores; sin esto, en seis meses la mitad de la UI dice "NUEVO" y el marcador deja de significar algo.

### Dónde aparece, exactamente

| Pantalla | Elemento marcado | Feature |
|---|---|---|
| `Services.tsx` | Encabezado del bloque "Modalidad del Servicio" | `serviceModality` |
| `NewAppointment.tsx` | La cuarta `ChoiceCard` de tipo de cita, "Reevaluación" | `reevaluation` |
| `NewAppointment.tsx` | Encabezado del bloque de sesiones del paquete | `sessionCounter` |
| `Appointments.tsx` | Chip del filtro por tipo "Reevaluación" | `reevaluation` |
| `OrderManagement.tsx` | Junto al checkbox de tratamiento el mismo día | `sameDayLink` |
| `ProformaPanel.tsx` | Aviso "Estás cotizando desde una reevaluación" | `reevaluation` |
| `PatientProfile.tsx` | Encabezado del agrupador por episodio | `sessionCounter` |
| `Reports.tsx` | Filas nuevas de reevaluaciones y sesiones entregadas | `reevaluation` |

**Una vez por pantalla, nunca por fila.** Marcar cada cita de la agenda convierte el marcador en ruido y vuelve la agenda ilegible.

## 4. Fase 0 — Catálogo y limpieza

### Migraciones

`024_drop_treatment_group_id.sql`

- `ALTER TABLE appointments DROP COLUMN treatment_group_id`, ídem en `proformas`.
- Reescribir `create_service_order_from_proforma` (`007:202`) y la función de `018:100` eliminando la rama `OR (v_proforma.treatment_group_id ...)`. La resolución del servicio queda solo por `appointment_id`.
- Verificación previa en un `DO $$`: abortar si existe alguna fila con el campo no nulo, siguiendo el patrón de `021`. Si alguien usó el botón entre la auditoría y el despliegue, la migración falla en vez de perder datos.

`025_session_model_enum.sql` — el enum solo, aislado por la restricción de `ALTER TYPE`.

`026_service_session_model.sql` — las ocho columnas de catálogo.

`027_rename_price_and_service_modality.sql` — desambigua las dos "modalidades" del sistema y fija los nombres definitivos:

| Antes | Ahora |
|---|---|
| `service_modality_enum` (nivel de precio) | `price_modality_enum`, en `service_orders.price_modality` |
| `session_model_enum` | `service_modality_enum`, en `services.service_modality` |
| `package_single_quote`, `series_quote_per_session` | `package`, `per_session` |
| `allowed_session_models` | **Eliminada.** Un servicio tiene una sola modalidad |

Estado final de las columnas de catálogo:

| Columna en `services` | Tipo | Default |
|---|---|---|
| `service_modality` | `service_modality_enum` | `single` |
| `default_sessions` | int | `1` |
| `package_validity_days` | int null | `null` |
| `min_days_between_sessions` | int null | `null` |
| `followup_schedule` | `int[]` null | `null` |
| `reevaluation_pricing` | text | `free` |
| `reevaluation_duration_minutes` | int null | `null` |

**El backfill no adivina.** Todo servicio existente arranca en `single`, que es exactamente su comportamiento actual, así que el despliegue no cambia nada para nadie. Masajes, láser CO₂ y láser de tatuajes se reconfiguran a mano desde la UI, y esa UI es la entrega de esta fase. No se infiere el modelo desde el nombre del servicio ni desde el histórico de citas: un backfill heurístico produciría paquetes fantasma sobre órdenes reales.

### Backend

- `update_service_with_prices` recibe los campos nuevos. Es un RPC de firma fija ([services.service.ts:175](../../renaceris_backend/src/services/services.service.ts#L175)): hay que versionar la firma, no basta con agregar columnas.
- `CreateServiceDto` y `UpdateServiceDto`: `default_sessions >= 2` si la modalidad es `package`, y solo un paquete puede vender más de una sesión por orden. La base tiene los mismos CHECK; validar en el backend devuelve el mensaje en castellano.
- `per_session` no se ofrece a servicios de naturaleza procedimiento directo: cotizar cada sesión es justamente lo que ese flujo evita.
- Cambiar la modalidad de un servicio **no** toca órdenes ni episodios existentes. Es configuración prospectiva, y el campo congelado en el episodio es lo que lo hace cierto.

### UI — `Services.tsx` · roles `admin` y `supervisor`

Bloque nuevo "Modelo de sesiones", con `<NewBadge feature="sessionModel" />` en el encabezado, dentro del formulario de servicio ya existente:

| Control | Tipo | Comportamiento |
|---|---|---|
| Modalidad | Tres tarjetas de selección única | Filtradas por la naturaleza del servicio: procedimiento directo solo admite `single` y `package` |
| Sesiones que vende la orden | `Input` numérico | Visible solo en paquete. Mínimo 2 |
| Vigencia del paquete (días) | `Input` numérico, opcional | Visible solo en paquete |
| Días mínimos entre sesiones | `Input` numérico, opcional | Visible en paquete o serie. Genera advertencia al agendar, no bloqueo |
| Esquema de controles | Chips editables `[7, 30, 90]` | Deshabilitado hasta la fase 2 |
| ¿Se cobra la reevaluación? | `Toggle` | Deshabilitado hasta la fase 2 |
| Duración de la reevaluación | `Input` numérico | Deshabilitado hasta la fase 2 |

Aviso permanente al pie del bloque: *"Los cambios aplican a las órdenes que se creen desde ahora. Las órdenes ya emitidas conservan su comportamiento."* Sin esa línea, supervisión va a creer que reconfigurar masajes arregla retroactivamente los paquetes vendidos, que es justo lo que no ocurre.

Los campos de la fase 2 se muestran deshabilitados con la nota "disponible al habilitar reevaluaciones", en lugar de ocultarse: anuncian lo que viene sin permitir configurar algo que todavía no tiene efecto.

### Orden de despliegue

Las migraciones van **antes** que el código. `findAll` de servicios pide las columnas nuevas por nombre, así que un backend nuevo contra una base sin migrar falla al listar el catálogo. En sentido inverso no hay problema: la base migrada funciona con el código anterior, que simplemente ignora las columnas.

La eliminación de `treatment_group_id` es segura en cualquier orden, porque el código nuevo ya no la pide en ningún `select`.

### Criterios de aceptación

1. Ningún servicio cambia de comportamiento tras el despliegue.
2. Supervisión convierte masajes en `package` con 10 sesiones y guarda.
3. `treatment_group_id` no existe en el esquema y los RPC siguen creando órdenes.
4. Cambiar un servicio a procedimiento directo con la serie seleccionada corrige la modalidad a `single` en vez de rechazar el guardado.

## 5. Fase 1 — Episodios y paquetes

### Migraciones

`028_treatment_episodes.sql`: tabla `treatment_episodes` con `patient_id`, `service_id`, `location_id`, `status` (`active`, `completed`, `abandoned`, `cancelled`), `origin_appointment_id`, `origin` (`system`, `legacy`, `external`), `service_modality` congelada, `opened_at`, `closed_at`.

`029_service_orders_sessions.sql`: `episode_id` **nullable**, `sessions_total` (default 1), `sessions_delivered` (default 0), `valid_until`, `included_in_order_id` en `service_orders`; `episode_id` **nullable** y `session_number` en `appointments`. Y un tercer valor en el CHECK de `source`: `direct_sale`, para el paquete de procedimiento directo que se vende sin proforma.

**Sin backfill.** `episode_id` nulo es un estado legítimo y permanente para todo lo anterior al despliegue. Ninguna consulta del módulo puede asumir que existe.

### Backend

- Trigger que recalcula `sessions_delivered` desde las citas `completed` de la orden, con `pg_advisory_xact_lock` sobre el `order_id`, siguiendo el patrón del trigger de límite de proformas de `020`.
- Al agendar contra una orden: rechazar si `sessions_delivered >= sessions_total`, salvo rol `admin` con motivo, que amplía `sessions_total` y queda en `audit_logs`.
- Cancelación y no-show **no** consumen sesión (decisión 1.1 por defecto).
- Agendamiento en bloque: crea las N citas en una transacción. El índice `(patient_id, service_id, datetime)` de `021` no estorba porque las fechas difieren; si el usuario repite fecha y hora, el error debe traducirse a un mensaje legible y no al error crudo de Postgres.
- Órdenes sin episodio: todos los endpoints nuevos devuelven `null` en los campos de sesión en vez de calcular.

### UI

**`OrderManagement.tsx` · roles `recepcionista`, `admin`, `ventas`** — al generar la orden desde una proforma de un servicio con paquete:

- Campo "Sesiones vendidas", prellenado con `default_sessions` del servicio y editable dentro de los límites del catálogo.
- "Vence el" calculado desde `package_validity_days`, editable por `admin`.
- En el listado de órdenes activas, columna nueva "Sesiones" con `3 / 10` y barra de avance. Órdenes legadas muestran un guion, no `1 / 1`.

**`NewAppointment.tsx` · roles `recepcionista`, `admin`** — el bloque "Sesiones del Tratamiento" se rehace por completo:

- Deja de basarse en el `additionalDates` suelto de hoy y pasa a leer `sessions_total` de la orden seleccionada.
- El disparador deja de ser el ícono `UserPlus` sin label y pasa a ser un botón con texto: **"+ Agendar sesión"**, con el contador "Quedan 7 de 10" al lado y `<NewBadge feature="sessionCounter" />` en el encabezado del bloque.
- Con el paquete agotado, el botón queda deshabilitado con el motivo visible; `admin` ve además "Ampliar paquete", que pide motivo obligatorio.
- Con `min_days_between_sessions` configurado, una fecha demasiado próxima muestra advertencia ámbar y permite continuar. La decisión clínica es del médico (decisión 2.8).

**`PatientProfile.tsx` · todos los roles con acceso al paciente** — historial agrupado por episodio, con `<NewBadge feature="sessionCounter" />` en el encabezado del agrupador, y un bloque final **"Antes del módulo de episodios"** que contiene, sin agrupar, las órdenes y citas legadas. Ese bloque es la traducción visible de la regla de convivencia: nadie tiene que preguntarse por qué su historial viejo se ve distinto.

### Criterios de aceptación

1. Una orden de masajes de 10 sesiones agenda 10 citas y muestra el avance.
2. La sesión 11 se bloquea; `admin` puede ampliar dejando rastro en auditoría.
3. Cancelar la sesión 4 no altera `sessions_delivered`.
4. El saldo de la orden no cambia por entregar sesiones, y entregar no exige saldo cero.
5. Una orden anterior al despliegue no muestra contador de sesiones en ninguna pantalla.

## 6. Fase 2 — Reevaluación como tipo de cita

### Migraciones

`030_appointment_type_reevaluation.sql`: solo `ALTER TYPE appointment_type_enum ADD VALUE 'reevaluation'`. Sola en su migración, porque Postgres no permite usar el valor nuevo en la misma transacción que lo agrega.

`031_reevaluation_support.sql`: precios referenciales de reevaluación en `service_prices` (la tabla ya está tipada por el enum, no hace falta estructura nueva).

### Backend

- `assertVirtualModalityAllowed` y el cálculo de duración de slot deben contemplar el tipo nuevo; hoy asumen consulta cuando el tipo no es tratamiento.
- Precio: `free` por defecto según el servicio; `charged` toma el referencial de `service_prices`.
- La cita de reevaluación exige `episode_id`. Si la paciente viene de otra clínica o de antes del sistema, se abre un episodio `external` o `legacy` en el mismo acto de agendar — eso reemplaza la práctica de crear una orden `legacy_manual` de percha.

### UI

**`NewAppointment.tsx` · roles `recepcionista`, `admin`** — paso 3, bloque "Tipo de Cita":

- Cuarta `ChoiceCard`, junto a Consulta, Tratamiento y Proc. Directo:

  ```
  🔁  REEVALUACIÓN            [NUEVO]
      Seguimiento de un tratamiento
  ```

  El grid pasa de tres a cuatro tarjetas y sigue en `grid-cols-2`, así que no hay reflow: queda 2×2.
- Al elegirla aparece **"Tratamiento a evaluar"**, un `Select` con tres grupos:

  | Grupo | Qué lista | Qué pasa al elegir |
  |---|---|---|
  | Tratamientos en curso | Episodios activos, con servicio, fecha de apertura y sesiones entregadas | Se usa ese episodio |
  | Tratamientos anteriores | Órdenes de la paciente sin episodio | **Se adopta la orden**: se crea su episodio `legacy` y se le vinculan las citas que ya colgaban de ella |
  | Sin tratamiento previo en el sistema | — | Abre un episodio `external` y avisa que la atención se cobra como consulta (decisión 2.3) |

  El grupo del medio es lo que hace usable la feature desde el día uno. Sin backfill no existe ningún episodio al desplegar, así que sin él toda reevaluación caería en la tercera opción. La adopción ocurre solo cuando una persona señala esa orden concreta, así que sigue sin haber conversión masiva de datos.
- El monto se fija en 0 y se deshabilita cuando el servicio tiene `reevaluation_pricing = free`, con la leyenda "Reevaluación incluida en el seguimiento". Con `charged`, propone el referencial.
- La duración del slot sale de `reevaluation_duration_minutes`.

**`Appointments.tsx` y `Calendar.tsx` · agenda** — el tipo entra en el filtro con `<NewBadge feature="reevaluation" />` en el chip. Se reutiliza el verde que la categoría `reevaluation` ya usa hoy ([Calendar.tsx:315](../../renaceris_front/src/components/Calendar.tsx#L315)) para que la agenda no cambie de idioma visual. `category` pasa a autocompletarse desde el tipo y deja de decidir nada.

**`AppointmentDetailModal.tsx`** — muestra el episodio evaluado, la etiqueta derivada y, si hay orden vinculada, "sesión k de N".

**`PatientProfile.tsx`** — el botón que hoy preagenda con `prefillCategory: 'reevaluation'` ([:1252](../../renaceris_front/src/pages/PatientProfile.tsx#L1252)) pasa a preagendar con el tipo y el episodio ya seleccionados. Es el atajo que hace que la feature se use.

**`Reports.tsx` · roles `admin`, `admin_sede`, `supervisor`, `contador`** — filas separadas: Consultas, Reevaluaciones, Reevaluaciones con tratamiento, Sesiones entregadas. Nota al pie con la fecha de corte del despliegue.

### Criterios de aceptación

1. Un control post-operatorio se agenda sin crear consulta ni orden ficticia.
2. No suma al conteo de consultas en reportes.
3. Fuera de la ventana de seguimiento del servicio, propone el precio de reevaluación.
4. Una paciente sin episodios puede agendar reevaluación abriendo un episodio `external`.

## 7. Fase 3 — Cotizar en reevaluación y tratar el mismo día

Desbloquea láser, tatuajes y el bótox tras rinoplastia. Es la fase de riesgo alto.

### Cambios de regla

Levantar el filtro de "solo consulta" en los tres puntos donde está escrito:

| Lugar | Cambio |
|---|---|
| [proformas.service.ts:75](../../renaceris_backend/src/proformas/proformas.service.ts#L75) | Aceptar `consultation` y `reevaluation` |
| [proformas.service.ts:182](../../renaceris_backend/src/proformas/proformas.service.ts#L182) | Ídem |
| [service-orders.service.ts:36](../../renaceris_backend/src/service-orders/service-orders.service.ts#L36) | Ídem, y el mensaje deja de decir "de consulta" |

El límite de tres proformas por cita y la unicidad por servicio se heredan sin tocar nada: el trigger de `020` opera sobre `appointment_id` y es agnóstico al tipo.

### La reutilización de la cita

**Endpoint nuevo:** `PATCH /appointments/:id/service-order`, roles `recepcionista` y `admin`. Hoy `service_order_id` solo se puede fijar al crear la cita y se fuerza a `null` cuando el tipo no es tratamiento ([appointments.service.ts:215](../../renaceris_backend/src/appointments/appointments.service.ts#L215)). Es el único trabajo estructural de la fase.

Validaciones: misma paciente, orden no cancelada, cita no cancelada, asignación de `session_number`. **No** se exige que el servicio de la orden coincida con el de la cita — la igualdad que hoy impone [:184](../../renaceris_backend/src/appointments/appointments.service.ts#L184) rompería el caso bótox-tras-rinoplastia, donde el servicio evaluado y el ejecutado son distintos por diseño. La cita conserva el servicio que evaluó; la sesión entregada se cuenta desde la orden.

**Alineación de `isTreatment`:** en creación se define solo por el tipo ([:161](../../renaceris_backend/src/appointments/appointments.service.ts#L161)), mientras que la capa de dinero ya usa `type === 'treatment' || !!service_order_id` ([:1584](../../renaceris_backend/src/appointments/appointments.service.ts#L1584), [:1630](../../renaceris_backend/src/appointments/appointments.service.ts#L1630)). Se adopta la segunda definición en creación y se deja de anular `service_order_id`. La guarda "las citas de tratamiento deben tener orden" se conserva solo para `type === 'treatment'`.

### UI

**`ProformaPanel.tsx` · roles `medico`, `admin`** — el panel se monta ahora también en citas de reevaluación:

- Aviso superior con `<NewBadge feature="reevaluation" />`: *"Estás cotizando desde una reevaluación. La cotización puede ser de otro servicio."*
- El selector de servicio a cotizar deja de estar atado al servicio de la cita. Se mantiene la restricción de sede.
- El contador "1 de 3 tratamientos cotizados" se muestra explícito, porque en reevaluación el límite sorprende más que en consulta.

**`OrderManagement.tsx` · roles `recepcionista`, `admin`, `ventas`** — es la pantalla donde se decide todo:

- En la lista de proformas pendientes, cada fila indica su origen: `Consulta` o `Reevaluación · hoy`. Sin eso, recepción no sabe si el checkbox va a aparecer.
- En el modal de generar orden, cuando la proforma nació en una cita `reevaluation` del mismo día no cancelada:

  ```
  ☑ El tratamiento se realiza hoy, en esta misma atención   [NUEVO]
     Se usa la cita de reevaluación en curso. No se crea una cita nueva.
  ```

  Marcado por defecto. En cualquier otro caso el checkbox no se renderiza y el flujo es el estándar.
- Marcado: al confirmar, se genera la orden y se llama al endpoint de vinculación. Confirmación visible: *"Orden generada y vinculada a la cita de las 10:30."*
- Desmarcado: se genera la orden y se ofrece "Agendar la sesión", que abre `NewAppointment` con paciente, orden y tipo `treatment` prellenados.

**`DoctorPanel.tsx` · roles `medico`, `especialista_estetico`** — el médico no hace nada distinto, y eso es el objetivo:

- La tarjeta de la cita en curso cambia su etiqueta a "Reevaluación y tratamiento" cuando recepción vincula la orden, con actualización en vivo. Sin cerrar el procedimiento, sin notificación nueva, sin segunda tarjeta.
- El banner de "procedimiento en curso sin concluir" ([DoctorPanel.tsx:566](../../renaceris_front/src/pages/DoctorPanel.tsx#L566)) sigue mostrando una sola entrada. Si apareciera duplicada, la vinculación se implementó mal.
- Al concluir, la sesión se cuenta como entregada en la orden.

### Por qué no se crea una segunda cita

| Guarda | A quién aplica |
|---|---|
| Trigger de conflicto de médico (`013`) | Solo cuando el `doctor_id` tiene rol `medico` |
| Un procedimiento activo a la vez ([:616](../../renaceris_backend/src/appointments/appointments.service.ts#L616)) | Solo cuando quien inicia tiene rol `medico` |
| Índice anti-duplicados (`021`) | **Todos**, incluidos especialistas estéticos |
| Notificación al médico en cada creación ([:258](../../renaceris_backend/src/appointments/appointments.service.ts#L258)) | **Todos** |

Las dos primeras no cubren a `especialista_estetico`; las dos últimas sí, y bastan. Importa al escribir las pruebas: el caso de láser ejecutado por especialista estético no falla por las mismas razones que el de un médico, así que necesita su propio test.

### Criterios de aceptación

1. Un médico cotiza bótox dentro de una reevaluación de rinoplastia; se genera la orden y la sesión se ejecuta en la misma cita.
2. La agenda no muestra una cita extra, el médico recibe una sola notificación y concluye una sola vez.
3. La sesión de láser cotizada en reevaluación cuenta como entregada al concluir el procedimiento.
4. Desmarcando el checkbox, se agenda la cita `treatment` y ahí sí se notifica.
5. La cita de reevaluación con orden aparece como "Reevaluación y tratamiento" y no suma a consultas.

## 8. Cómo se ve el cambio en cada rol

Los ocho roles del sistema (`010`, `013`, `016`), con la pantalla donde lo viven:

| Rol | Dónde | Qué cambia |
|---|---|---|
| **admin** | Todas | Configura catálogo, decide mismo día, amplía paquetes agotados con motivo auditado, edita vencimientos |
| **supervisor** | `Services.tsx` | **Dueño del catálogo.** Define modelos admitidos y por defecto, incluidos los servicios legacy. Protagonista de la fase 0. No toca proformas ni órdenes |
| **recepcionista** | `OrderManagement.tsx`, `NewAppointment.tsx` | Decide mismo día u otro día al generar la orden; agenda sesiones contra la orden; ve "quedan k de N" y el bloqueo por paquete agotado |
| **medico** | `DoctorPanel.tsx`, `ProformaPanel.tsx` | Cotiza dentro de una reevaluación, incluso de otro servicio; en el caso mismo día concluye una sola vez, sin cita extra |
| **especialista_estetico** | `DoctorPanel.tsx` | Ejecuta y concluye sesiones igual que hoy, pero **no puede crear proformas** ([proformas.controller.ts:42](../../renaceris_backend/src/proformas/proformas.controller.ts#L42)). Si es quien atiende láser, no podrá cotizar la sesión siguiente. Ver §10 |
| **ventas** | `OrderManagement.tsx` | La cotización personalizada queda disponible también en proformas nacidas de reevaluación |
| **contador** | `Accounting.tsx`, `Reports.tsx` | Indicador nuevo de sesiones cobradas y no entregadas, presentado **fuera** del balance por ser obligación de servicio, no asiento |
| **admin_sede** | Todas, acotado | Igual que admin dentro de su sede. Una sesión ejecutada en otra sede se registra con la sede real de atención (decisión 1.5) |

## 9. Pruebas mínimas

| Fase | Prueba que no puede faltar |
|---|---|
| 0 | Un servicio sin tocar se comporta idéntico antes y después del despliegue |
| 0 | Quitar un modelo de sesión en uso por un episodio activo es rechazado con el conteo |
| 1 | Concurrencia: dos recepcionistas agendando la última sesión del paquete, solo una pasa |
| 1 | Cancelar una sesión no altera `sessions_delivered` |
| 1 | Una orden con `episode_id` nulo no rompe ninguna pantalla ni muestra contadores |
| 2 | Una reevaluación no aparece en el conteo de consultas del reporte |
| 2 | Paciente sin episodios agenda reevaluación abriendo uno `external` |
| 3 | Proforma creada desde reevaluación genera orden correctamente |
| 3 | Vinculación mismo día: una fila de cita, una notificación, un ciclo de procedimiento |
| 3 | Orden de un servicio distinto al de la cita se vincula sin error (bótox tras rinoplastia) |
| 3 | El mismo caso ejecutado por `especialista_estetico`, que no pasa por las guardas de rol `medico` |
| UI | `NewBadge` desaparece de toda la aplicación al vencer la fecha del registro |

## 10. Riesgos y decisiones a cerrar en el camino

| # | Asunto | Cuándo bloquea | Recomendación |
|---|---|---|---|
| 1 | ¿`especialista_estetico` puede cotizar en una reevaluación? | Antes de la fase 3, solo para láser | Confirmar quién ejecuta láser. Si es el especialista, ampliar el permiso de creación de proforma **solo** para el tipo `reevaluation`; si es el médico, no tocar nada |
| 2 | Ventana de seguimiento post-quirúrgico | Fase 2, solo para el precio | 6 meses por tipo de cirugía. Sin respuesta, toda reevaluación es gratuita y no bloquea |
| 3 | No-show y vencimiento consumen sesión | Fase 1 | No consumen, con registro |
| 4 | Comprobante de un paquete en SIBI | Fase 1, lado contable | Documentar la práctica actual antes de tocar el puente |
| 5 | El enum `appointment_type_enum` lo usa `service_prices` | Fase 2 | Migración aislada para el `ADD VALUE`; nada más en esa transacción |
| 6 | Convivencia visual de lo legado | Fases 1 y 2 | El bloque "Antes del módulo de episodios" y la nota al pie del reporte son parte del alcance, no un extra |
| 7 | Fecha de retiro de los marcadores | Al desplegar | Fijar `until` en el registro al desplegar la última fase, no antes |
