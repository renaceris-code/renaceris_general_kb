# Modelo de negocio final — Reevaluación y Sesiones

Cierre de las dos features a partir de [contexto-conversacion-reevaluacion-y-sesiones.md](../contexto-conversacion-reevaluacion-y-sesiones.md), validado contra el esquema y el código en producción. A diferencia de los documentos 00 a 05, este no propone alternativas: fija la regla. Lo único que queda abierto son las tres preguntas de la sección 7, y ninguna bloquea el inicio de la implementación.

## 0. Qué resiste la validación contra el código

| Afirmación del análisis | Veredicto | Evidencia |
|---|---|---|
| La proforma solo puede nacer de una consulta | **Confirmado.** Es el bloqueo real | `proformas.service.ts:75` y `:182`; `service-orders.service.ts:36` ("Solo se puede generar una orden desde una proforma de consulta") |
| `REEVALUACIÓN` no es un tipo de cita | **Confirmado.** `appointment_type_enum` solo tiene `consultation`, `treatment`, `direct_procedure` | `supabase-schema.ts:1985`. `reevaluation` existe únicamente en `appointment_category` (`:1982`), que es color de agenda |
| El paquete de varias sesiones ya es casi representable | **Confirmado a medias.** `appointments.service_order_id` es N:1, así que varias citas ya pueden colgar de una orden | Falta que la orden sepa qué vendió: `service_orders` no tiene ni sesiones vendidas ni entregadas |
| El episodio hay que crearlo | **Confirmado.** `treatment_group_id` pretende ese rol, pero es un UUID libre, sin tabla, sin FK y sin estado | Pegamento en tres capas distintas: RPC `007:202`, `service-orders.service.ts:230-265`, `PatientProfile.tsx:496` |
| `treatment_group_id` tiene datos que rescatar | **Falso. Código vivo, dato muerto.** Auditoría de BD: la columna es 100% `NULL` | Su único escritor es `NewAppointment.tsx:398`, detrás de un botón solo-ícono `UserPlus` sin label ([:918](../../renaceris_front/src/pages/NewAppointment.tsx#L918)) que nadie usa. `proformas.treatment_group_id` lo hereda de la cita (`proformas.service.ts:179`), así que arrastra el mismo NULL |
| Una reevaluación podría emitir varias proformas | **Ya resuelto por el esquema.** El límite de 3 por cita es agnóstico al tipo, y el índice único obliga a que sean servicios distintos | `020_multi_treatment_proformas.sql`: trigger sobre `appointment_id` más `uq_proformas_appointment_treatment` |
| Cobrar una reevaluación exigiría estructura nueva | **Falso.** `service_prices` ya está tipada por `appointment_type_enum` | Agregar `reevaluation` al enum habilita su precio referencial sin tabla nueva |
| Agendar N sesiones seguidas chocaría con el anti-duplicados | **Falso.** El índice es `(patient_id, service_id, datetime)` | `021_prevent_duplicate_active_appointments.sql`: fechas distintas no colisionan |

### Las dos "modalidades", ya desambiguadas en el esquema

El análisis original chocaba con un nombre ocupado: `service_modality_enum` (`Sin modalidad`, `Modalidad 1`, `Modalidad 2`, `Cotización personalizada`) **no** describe si el tratamiento es por paquete o por sesión, sino el **nivel de precio** elegido de la proforma, y cada valor apunta a una columna concreta (`price_no_modality`, `price_modality_1`, `price_modality_2`).

La migración `027` lo resolvió con un renombre cruzado, aplicado en producción:

| Antes | Ahora | Qué responde |
|---|---|---|
| `service_modality_enum` | **`price_modality_enum`** | ¿A qué precio se cerró? Vive en `service_orders.price_modality` |
| `session_model_enum` | **`service_modality_enum`** | ¿Cuántas veces viene y cuántas veces se cotiza? Vive en `services.service_modality` |

El segundo nombre le corresponde a ese eje porque abarca tanto tratamientos como procedimientos directos, no solo "sesiones de tratamiento". Los valores también se renombraron: `single`, `package` y `per_session`, sin el sufijo `_quote`, porque un paquete de procedimientos directos no pasa por proforma.

Queda un tercer sentido de "modalidad" en el código, sin relación con estos dos: presencial contra virtual (`supports_virtual_consultation`, `assertVirtualModalityAllowed`). No se toca, pero conviene saber que existe.

## 1. Los cuatro ejes del modelo

| Eje | Pregunta | Entidad | Estado |
|---|---|---|---|
| Clínico | ¿Qué pasa en esta visita? | `appointments.appointment_type` | Se amplía con `reevaluation` |
| Operativo | ¿Cuántas veces viene y cuántas ya vino? | `treatment_episodes` más las citas contra la orden | Se crea |
| Comercial | ¿Cuánto se debe y cuánto se pagó? | `proformas` → `service_orders` → `payments` | Se conserva íntegro |
| Tarifario | ¿A qué precio se cerró? | `service_orders.price_modality` | Se conserva íntegro; solo cambió de nombre |

Un episodio agrupa; una orden cobra; una cita ejecuta o evalúa; una modalidad tarifa. Ningún eje se deduce de otro.

## 2. Feature A — Reevaluación

### Definición final

> Una reevaluación es una visita de seguimiento dentro de un episodio existente. Evalúa siempre, cotiza si el médico lo decide, y ejecuta solo si en esa misma visita recepción generó la orden.

`reevaluation` se agrega como cuarto valor de `appointment_type_enum`. **"Reevaluación y tratamiento" no es un quinto tipo**: es la etiqueta derivada de una cita `reevaluation` que terminó con orden creada en la visita. La razón es operativa, no estética: al agendar todavía no se sabe si habrá tratamiento, y un tipo fijo obligaría a mutar el tipo de la cita a mitad de atención, arrastrando precio, permisos y reportes.

```text
appointment_type = 'reevaluation' + orden creada en esa cita   →  "Reevaluación y tratamiento"
appointment_type = 'reevaluation' + sin orden                  →  "Reevaluación"
```

### Los cuatro desenlaces, todos con el mismo tipo de cita

```text
REEVALUACIÓN
├── Solo evaluación                           → cita cerrada, sin proforma
├── Proforma sin tratamiento todavía          → orden pendiente, sin sesión
├── Proforma + tratamiento en la misma visita → la cita se vincula como sesión de esa orden
└── Proforma + tratamiento otro día           → se agenda una cita 'treatment' contra la orden
```

### El caso del mismo día: se reutiliza la cita, no se crea una segunda

Es el 95% de los casos y define la implementación completa.

| Caso | Filas de cita | Cómo |
|---|---|---|
| Tratamiento el mismo día | **1** | La misma cita `reevaluation` recibe `service_order_id`. Etiqueta derivada: "Reevaluación y tratamiento" |
| Tratamiento otro día | **2** | Cita `treatment` nueva contra la orden. Flujo estándar, ya funciona hoy |

Crear una segunda cita el mismo día no es una preferencia de diseño: el sistema lo impide en tres puntos.

- **Trigger de conflicto de médico** (`013`): mismo `doctor_id` y mismo `datetime` no cancelado lanza excepción. La segunda cita ni se inserta a la misma hora.
- **Anti-duplicados** (`021`): `(patient_id, service_id, datetime)`. En láser, donde el tratamiento es el mismo servicio evaluado, choca también.
- **Un procedimiento activo por médico** ([appointments.service.ts:616](../../renaceris_backend/src/appointments/appointments.service.ts#L616)): el médico tendría que concluir la reevaluación para poder iniciar el tratamiento. Dos ciclos abrir y concluir para una atención que nunca se interrumpió, más una notificación extra por cada creación de cita ([:258](../../renaceris_backend/src/appointments/appointments.service.ts#L258)).

El código ya avala la reutilización: la capa de dinero define tratamiento como `appointment_type === 'treatment' || !!service_order_id` en [:1584](../../renaceris_backend/src/appointments/appointments.service.ts#L1584) y [:1630](../../renaceris_backend/src/appointments/appointments.service.ts#L1630). Vincular la orden a la cita de reevaluación hace que resumen de pagos y registro de abonos funcionen sin tocarlos. La creación es lo único desalineado: mira solo el tipo en [:161](../../renaceris_backend/src/appointments/appointments.service.ts#L161).

**Quién decide y cuándo.** No lo decide el médico ni el sistema: lo decide recepción en el momento de generar la orden, que es cuando el negocio ya sabe la respuesta. Un solo control en el modal de orden desde proforma, marcado por defecto: *"El tratamiento se realiza hoy, en esta misma atención"*. Solo aparece si la proforma nació en una cita `reevaluation` del mismo día no cancelada.

### Reglas cerradas

| Punto abierto en la conversación | Regla final | Por qué |
|---|---|---|
| ¿Todas las reevaluaciones pueden cotizar, o solo ciertos servicios? | **Todas.** Sin lista blanca por servicio | El permiso ya está acotado a médico y administrador, y el límite de 3 proformas por cita aplica igual. Una restricción por servicio sería configuración que no evita ningún error real |
| ¿El tratamiento puede ejecutarse en la misma cita tras generar la orden? | **Sí, y sin agendar una segunda cita.** La propia cita de reevaluación se vincula a la orden vía `service_order_id` | Es exactamente el caso de láser CO₂ y tatuajes. Obligar a una segunda cita duplicaría la agenda del día |
| ¿Y si cotiza pero se trata otro día? | La orden queda viva y sin sesión entregada; se agenda una cita `treatment` contra ella | No requiere ninguna regla nueva: es el flujo que ya existe |
| ¿Una misma reevaluación puede generar más de una proforma? | **Sí, hasta 3, una por servicio distinto** | Ya lo garantizan el trigger de límite y el índice único de `020`. Se hereda sin escribir código |
| Reevaluación gratuita, cobrada o fuera de ventana | **Gratuita dentro de la ventana de seguimiento del servicio; cobrada fuera de ella**, con precio propio en `service_prices` | La ventana por defecto es 6 meses, configurable por servicio. Confirmar con Ronald (§7) |
| ¿Y la paciente operada en otra clínica? | Se abre un episodio con `origin = 'external'` y se cobra como consulta. **Suspendido en la interfaz**: ver abajo | Ahí no hay nada que la clínica haya vendido, así que no hace falta orden: el episodio basta |
| ¿Y el tratamiento que hizo esta misma clínica antes del sistema? | Se registra con **"Gestionar tratamiento previo"** como orden `legacy_manual`, y la reevaluación la adopta en un episodio `origin = 'legacy'` | No es lo mismo que el caso externo: aquí sí hubo una venta de esta clínica, puede tener saldo y su precio puede quedar pendiente de regularizar. `external` **no** lo reemplaza |

### Parche vigente: el episodio `external` no se ofrece en la interfaz

**Decisión temporal, y solo de interfaz.** El selector "Tratamiento a evaluar" traía un tercer grupo, "Sin tratamiento previo en el sistema", que abría el episodio `external` de la fila anterior. Se retira de la pantalla.

El motivo es que el caso no llega por ahí. La paciente operada en otra clínica **pasa primero por consulta**, y de esa consulta sale su orden oficial; para cuando vuelve a control ya tiene historial y aparece en los dos grupos que quedan. Mientras el atajo estuvo disponible ofrecía una segunda forma de abrir el mismo episodio, esta vez sin la consulta que lo justifica, y competía visualmente con "Gestionar tratamiento previo", que es lo que de verdad resuelve el caso de la clínica propia. Dos puertas para lo que el negocio hace por una sola es como recepción termina eligiendo la equivocada.

Qué **no** cambia:

- `treatment_episodes.origin` conserva sus tres valores; no hay migración.
- `POST /treatment-episodes` sigue aceptando `origin = 'external'`, con sus permisos actuales.
- Las pantallas conservan su rama `external`, inalcanzable mientras el grupo no vuelva a la lista.

Revertirlo es devolver ese grupo en `buildEpisodeOptions` ([reevaluation.ts](../../renaceris_front/src/utils/reevaluation.ts)), donde el parche está anotado, y quitar el test que lo vigila. Ninguna otra capa lo asume ausente.

Cuándo reabrirlo: si aparece un caso real de paciente que llega directo a control sin haber pasado por consulta —una derivación de otro médico, por ejemplo—, el atajo vuelve tal cual, porque el modelo nunca dejó de contemplarlo.

### El servicio evaluado no ata al servicio cotizado

Una reevaluación de rinoplastia puede originar una proforma de bótox. El vínculo de evaluación es `episode_id`; el de ejecución es `service_order_id`. Son campos distintos y pueden apuntar a episodios distintos: la visita evalúa el episodio A y abre el episodio B. Se conserva la restricción vigente de que el tratamiento cotizado pertenezca a la sede de la cita.

### Lo que la feature A deliberadamente no hace

- No usa `appointment_category` para decidir nada. Sigue siendo color de agenda, y se autocompleta en `reevaluation` cuando el tipo lo es, para que agenda y lógica no diverjan.
- No cobra reevaluaciones por defecto. Una reevaluación gratuita es una atención con ingreso cero, no una atención invisible.
- No cuenta como consulta en reportes ni consume vigencia de consulta.

## 3. Feature B — Sesiones

### Definición final

> El servicio declara **una** modalidad; el episodio la congela para esa paciente; la orden dice cuántas sesiones vendió; las citas dicen cuántas se entregaron.

| Modalidad | `service_modality` | Órdenes | Sesiones | Casos |
|---|---|---|---|---|
| Sesión única repetible | `single` | 1 orden por sesión realizada | 1 cita | Cirugía, criolipólisis |
| Paquete | `package` | 1 orden con N sesiones | N citas contra esa orden | Masajes, paquetes de procedimiento directo |
| Serie por sesión | `per_session` | N órdenes de 1 sesión | 1 cita por orden | Láser CO₂, láser de tatuajes |

Criolipólisis no es un cuarto caso: es `single` repetido dentro del mismo episodio. Es la diferencia entre "volvió a hacerse una sesión" y "compró un paquete", y el episodio es lo que permite verlas juntas sin fingir que son un paquete.

**Una modalidad por servicio, no una lista.** El diseño inicial permitía que un servicio admitiera varias y que la elección se hiciera en cada venta. El negocio confirmó que no existe ese caso: masajes es siempre paquete, cirugía siempre única, láser siempre serie. La columna `allowed_session_models` se eliminó en la `027` sin haberse poblado nunca con otra cosa que su default.

Que la modalidad quede congelada en el episodio **sigue siendo necesario**, pero por otro motivo del original: no protege contra que un servicio tenga varias, sino contra que **el catálogo cambie con el tiempo**. Si supervisión pasa masajes de `package` a `per_session` el año que viene, los episodios abiertos antes tienen que seguir comportándose como paquete. El campo congelado es el mecanismo que hace verdadera la regla de que los cambios de catálogo son prospectivos.

### Procedimientos directos: la orden nace sin proforma

Un procedimiento directo no pasa hoy por proforma ni por orden. El precio vive en la cita, sale de `service_prices` con `appointment_type = 'direct_procedure'`, y lo atiende un especialista estético sin que ningún médico tenga que cotizar. Eso funciona bien y no se toca para el caso de sesión única.

El problema aparece con el paquete: **todo el conteo de sesiones vive en la orden**, así que sin orden no hay dónde contar. Y las puertas existentes están cerradas: `service_orders.source` solo admite `'proforma'` o `'legacy_manual'`, y `createLegacy` exige que el servicio tenga precio de tipo `treatment`, que un procedimiento directo no tiene.

La regla final es que **un paquete de procedimiento directo genera una orden sin proforma**, con un tercer valor de `source`:

```text
Cita de procedimiento directo con servicio en modalidad 'package'
  → orden con source = 'direct_sale', sin proforma_id
  → total = precio del catálogo × sesiones vendidas
  → sesiones 1..N como citas contra esa orden
```

Se descartó la alternativa de mover los contadores al episodio: contradice "el dinero vive en la orden" y obligaría a reescribir la parte del modelo que ya funciona.

Consecuencias:

- La modalidad `per_session` **no aplica** a procedimientos directos, porque cotizar cada sesión es justamente lo que ese flujo evita. El catálogo solo ofrece `single` y `package` cuando el servicio es de naturaleza procedimiento directo.
- El especialista estético no necesita permisos nuevos: la orden la genera recepción al vender el paquete, igual que hoy cobra la cita.
- Un procedimiento directo de sesión única sigue **sin** orden. La orden aparece solo cuando hay más de una sesión que controlar.

### La serie por sesión: la cita de tratamiento también cotiza

Este es el flujo principal de láser, y obliga a corregir una regla que el modelo daba por sentada.

La regla vieja era *"una cita de tratamiento exige una orden"*, escrita cuando todo tratamiento venía de una orden emitida antes. En `per_session` eso es falso por diseño: **la orden de la sesión 2 nace durante la sesión 2**, porque su precio depende de cómo respondió la piel a la sesión 1. Exigir la orden por adelantado obligaba a agendar cada sesión como reevaluación, que describe mal lo que pasa —la paciente viene a tratarse, no a que la evalúen— y ensucia los reportes contando sesiones como controles.

**Regla final:**

> Una cita de ejecución exige **orden o episodio**, no orden siempre.
>
> - **Con orden** se ejecuta contra ella. Es el caso del paquete y de la sesión única: lo vendido ya existe.
> - **Sin orden, con episodio en modalidad `per_session`**, la orden se crea durante la visita y se vincula a esa misma cita.

El relajamiento está acotado por la modalidad congelada en el episodio, no es una apertura general: en `single` y `package` la orden sigue siendo obligatoria al agendar, porque ahí sí existe antes.

El recorrido completo de un láser queda así:

```text
CONSULTA            → proforma sesión 1 → orden 1 → sesión 1 (misma visita o agendada)
CITA DE TRATAMIENTO → evalúa, proforma sesión 2 → orden 2 → ejecuta   (sesión 2)
CITA DE TRATAMIENTO → evalúa, proforma sesión 3 → orden 3 → ejecuta   (sesión 3)
```

Cada visita repite el mismo ciclo: el médico evalúa y cotiza, recepción genera la orden con "el tratamiento se realiza hoy" y la vincula a la cita en curso, y la sesión se ejecuta. Es el mismo mecanismo del caso del mismo día de la feature A, aplicado a una cita de tratamiento en vez de a una de reevaluación.

**Cuándo sigue usándose reevaluación en un servicio `per_session`:** cuando la visita es de control y no de sesión. La paciente que vuelve un mes después de terminar su serie para que le revisen el resultado va como reevaluación; la que viene por su sesión 3 va como tratamiento. Lo decide el propósito de la visita al agendar, no la modalidad del servicio.

### El episodio

`treatment_episodes` formaliza lo que hoy hace `treatment_group_id` sin poder hacerlo bien: `patient_id`, `service_id`, `location_id`, `status` (`active`, `completed`, `abandoned`, `cancelled`), `origin_appointment_id`, `origin` (`system`, `legacy`, `external`), `service_modality` congelado, `opened_at`, `closed_at`.

`treatment_group_id` **no se migra: se elimina.** La columna está 100% en `NULL`, así que no hay nada que trasvasar, y las ramas del RPC `007:202` y `018:100` que resuelven el servicio a través del grupo son código muerto que nunca se ejecutó. Se borran en lugar de portarse, lo que evita el riesgo de arrastrar una convención ambigua al modelo nuevo.

### El conteo de sesiones vive en la orden

Cuatro contadores, ninguno derivable de los otros:

| Contador | Origen |
|---|---|
| Vendidas | `service_orders.sessions_total` |
| Entregadas | `service_orders.sessions_delivered`, materializado desde las citas `completed` |
| Pendientes | Vendidas menos entregadas |
| Vencidas | Pendientes con `valid_until` cumplido |

`appointments.session_number` permite mostrar "sesión 3 de 10". La materialización usa el mismo bloqueo `FOR UPDATE` que ya emplea el trigger de límite de proformas, por concurrencia.

**No se crea una tabla de sesiones.** La sesión física es la cita. Una sesión sin cita no existe operativamente, y los walk-in se registran como cita retroactiva.

**El límite solo aplica a órdenes con episodio.** Nada en la base impuso nunca "una ejecución por orden": `appointments.service_order_id` es N:1, así que hoy ya se pueden colgar varias citas de la misma orden, y es probable que masajes venga funcionando así. Si el bloqueo por `sessions_delivered >= sessions_total` se aplicara a las órdenes anteriores al despliegue, que quedan en `sessions_total = 1`, rompería ese flujo al agendar la segunda sesión. Las órdenes sin `episode_id` no se validan: se comportan como hasta hoy.

### Reglas cerradas

| Punto abierto en la conversación | Regla final | Por qué |
|---|---|---|
| ¿Dónde se cotiza cada sesión de una serie? | **Sesión 1 en la consulta; de la 2 en adelante, en la propia cita de tratamiento** | La cita de ejecución de un episodio `per_session` puede crearse sin orden y cotizarla en la visita. Reevaluación queda para las visitas que de verdad son de control |
| ¿Una cita de tratamiento puede existir sin orden? | **Solo si su episodio es `per_session` y esa sesión aún no se cotizó** | En `single` y `package` lo vendido existe antes de agendar, así que ahí la orden sigue siendo obligatoria |
| ¿Precio por sesión fijo o variable? | **Sugerido y editable.** El referencial sale de `service_prices`; el médico puede ajustarlo en cada proforma | Las proformas ya tienen precio libre. Fijarlo obligaría a recotizar fuera del sistema cuando la evolución cambia |
| ¿Un servicio puede ofrecer paquete y pago por sesión? | **No. Una modalidad por servicio.** El episodio congela la vigente al abrirse | El negocio confirmó que ningún servicio se vende de dos formas. Una lista de admitidos era configuración que ningún caso real usaba |
| ¿Los procedimientos directos pueden venderse en paquete? | **Sí, con una orden `direct_sale` sin proforma** | Sin orden no hay dónde contar sesiones. Es el único camino que no rompe el invariante de que el dinero vive en la orden |
| ¿Se puede cambiar de modelo a mitad de tratamiento? | **No se muta la orden.** Se cierra por lo entregado y se abre otra en el mismo episodio | Mutar una orden con pagos aplicados rompe la trazabilidad de saldos |
| ¿Se permite pago parcial antes de ejecutar una sesión? | **Sí, y ya funciona.** Los pagos van contra la orden | Entregar una sesión no exige saldo cero: advierte, no bloquea. El cobro y la entrega son contadores independientes |
| ¿Un no-show consume sesión? | **No consume**, y queda registrado | Recomendación por defecto; sujeta a §7 |

### Lo que la feature B deliberadamente no hace

- No mueve el dinero a la cita. El total, el pagado y el saldo siguen viviendo en la orden.
- No asume que sesiones y proformas son el mismo número. Esa equivalencia es justo lo que el modelo separa.
- No reutiliza la palabra "modalidad" para el modelo de sesiones.

## 4. Cómo se combinan las dos features

| Servicio | `service_modality` | Cita posterior típica | Qué desbloquea |
|---|---|---|---|
| Cirugía | `single` con seguimiento | Reevaluación pura | Feature A: hoy obliga a crear una consulta falsa o una orden de percha |
| Masajes | `package` | Sesión contra la orden | Feature B: hoy no hay control de cuántas se vendieron ni cuántas quedan |
| Láser CO₂ y tatuajes | `per_session` | Reevaluación y tratamiento | Ambas: sin cotizar desde reevaluación, la serie no existe |
| Criolipólisis | `single` repetible | Nueva sesión | Feature B, solo para agrupar el historial |
| Paquete de procedimiento directo | `package` | Sesión contra la orden | Feature B, con orden `direct_sale` sin proforma |
| Bótox tras rinoplastia | `single` | — | Feature A: la reevaluación del episodio A abre el episodio B |

Las dos features se pueden construir por separado. Lo que láser y tatuajes necesitan **no es el tipo `reevaluation`**, sino que una visita que no es consulta pueda cotizar: con la regla de "orden o episodio", sus sesiones 2 en adelante son citas de tratamiento normales. La feature A sigue siendo indispensable para los controles post-quirúrgicos y para la reevaluación que origina otro tratamiento, que son casos distintos.

## 5. Cambios de esquema mínimos

Lo que ya está aplicado en producción:

| Migración | Qué hizo |
|---|---|
| `024` | Eliminó `treatment_group_id` de `appointments` y `proformas`, y la rama muerta del RPC |
| `025` y `026` | Crearon el enum de modalidad y las ocho columnas de catálogo en `services` |
| `027` | Renombró `price_modality_enum` y `service_modality_enum`, renombró los valores a `single` / `package` / `per_session`, y eliminó `allowed_session_models` |

Lo que falta:

| Cambio | Objeto |
|---|---|
| `reevaluation` como cuarto valor | `appointment_type_enum`. Habilita de paso su precio en `service_prices` |
| Tabla nueva | `treatment_episodes` |
| Columnas nuevas | `service_orders`: `episode_id`, `sessions_total`, `sessions_delivered`, `valid_until`, `included_in_order_id` |
| Columnas nuevas | `appointments`: `episode_id`, `session_number` |
| Valor nuevo | `service_orders.source`: `direct_sale`, para el paquete de procedimiento directo, con su rama propia en el CHECK que hoy exige `proforma_id` |
| Endpoint nuevo | `PATCH /appointments/:id/service-order`: hoy `service_order_id` solo se puede fijar al crear la cita ([appointments.service.ts:215](../../renaceris_backend/src/appointments/appointments.service.ts#L215)) |
| Levantar el bloqueo | Aceptar `reevaluation` **y `treatment`** en `proformas.service.ts:75` y `:182`, y en `create_service_order_from_proforma` (`service-orders.service.ts:36`) |
| Relajar la guarda | `appointments.service.ts:168` deja de exigir orden cuando el episodio es `per_session`; pasa a exigir episodio |

Nota técnica: `ALTER TYPE ... ADD VALUE` no puede usarse en la misma transacción que lo consume, así que el enum va en una migración propia, previa a las que lo referencian.

## 6. Dinero y reportes

| Situación | Dónde se cobra |
|---|---|
| Consulta | En la cita |
| Reevaluación dentro de ventana | No genera cobro; atención con ingreso cero |
| Reevaluación fuera de ventana | En la cita, con precio propio del servicio |
| Sesión de un paquete | En la orden del paquete |
| Sesión de una serie | En la orden de esa sesión |
| Procedimiento directo | En la cita |

El criterio contable no cambia: el ingreso se reconoce por pago confirmado en `payments`. Se agrega un indicador operativo, **sesiones cobradas y no entregadas**, que es una obligación de servicio y debe presentarse **fuera** del balance.

En reportes, ninguna reevaluación suma al conteo de consultas. Sin este ajuste, liberar la cotización desde reevaluación infla el indicador de consultas justo cuando empiece a usarse.

## 7. Lo único que sigue bloqueado

Todo lo demás queda decidido arriba. Estas tres necesitan al negocio porque cambian el dinero, no la estructura, y por lo tanto pueden responderse después sin rehacer migraciones:

1. **¿Cuánto dura el seguimiento incluido de una cirugía y qué pasa después?** Define si la reevaluación es siempre gratuita. Por defecto: 6 meses por tipo de cirugía; fuera de ventana se cobra.
2. **¿El no-show y el vencimiento consumen sesiones de un paquete?** Define el control de entrega. Por defecto: no consumen, con registro.
3. **¿La vigencia de consulta es de 12 meses desde la consulta pagada?** No pertenece a ninguna de estas dos features y no debería retenerlas.
