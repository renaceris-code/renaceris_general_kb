# Modelo funcional propuesto — Reevaluaciones y multisesiones

Esquema objetivo del módulo. Parte del diagnóstico en [01-diagnostico-sistema-actual.md](01-diagnostico-sistema-actual.md) y se implementa según [04-plan-de-implementacion.md](04-plan-de-implementacion.md).

## 1. Principio rector: tres ejes que no deben mezclarse

| Eje | Pregunta que responde | Entidad que lo representa |
|---|---|---|
| **Clínico** | ¿Qué pasa en esta visita? ¿Se evalúa, se ejecuta, o ambas? | `appointments.appointment_type` |
| **Operativo** | ¿Cuántas veces tiene que venir la paciente y cuántas ya vino? | Sesiones: citas de ejecución vinculadas a una orden, dentro de un episodio |
| **Comercial** | ¿Cuánto se cotizó, cuánto se debe y cuánto se pagó? | `proformas` → `service_orders` → `payments` |

El error a evitar es asumir que una sesión equivale a una proforma, o que una cita equivale a un cobro. Los tres contadores son independientes y cada servicio los combina distinto.

## 2. Invariantes del modelo

1. Una **cita** es un acto en la agenda. Solo tiene precio propio si es consulta, procedimiento directo o reevaluación cobrada.
2. Una **proforma** cotiza un tratamiento para un paciente y nace en una cita evaluadora: consulta **o reevaluación**.
3. Una **orden** es el compromiso económico. Es la única fuente del total, el pagado y el saldo de un tratamiento.
4. Una **sesión** es una cita de ejecución vinculada a una orden. La orden sabe cuántas vendió; las citas dicen cuántas se entregaron.
5. Un **episodio** agrupa las órdenes y las citas de una misma línea de tratamiento a lo largo del tiempo.
6. Ninguna regla de negocio se apoya en `appointment_category`, que queda como etiqueta visual de agenda.

## 3. Entidades: qué se agrega y qué se conserva

### 3.1 Tipo de cita ampliado

Se agrega un cuarto valor al enum `appointment_type_enum`:

| Valor | Significado |
|---|---|
| `consultation` | Evaluación inicial cobrada. Puede cotizar. |
| `reevaluation` | **Nuevo.** Evaluación de seguimiento de un episodio existente. Sin costo por defecto. Puede cotizar. |
| `treatment` | Ejecución de una sesión contra una orden. Sin precio propio. |
| `direct_procedure` | Atención de catálogo con precio propio, sin proforma ni orden. |

**"Reevaluación y tratamiento" no es un quinto tipo.** Es una cita `reevaluation` que además terminó con una orden y una sesión ejecutada en la misma visita. Se modela como estado derivado, no como tipo, porque la visita empieza siendo una reevaluación y solo a mitad de la atención se sabe si habrá tratamiento. Un tipo fijo obligaría a mutar el tipo de la cita durante la atención, con todo lo que eso arrastra en precio, permisos y reportes.

En agenda y reportes se muestra la etiqueta derivada:

```text
appointment_type = 'reevaluation' y existe orden creada en esa cita  ->  "Reevaluación y tratamiento"
appointment_type = 'reevaluation' y no hay orden                     ->  "Reevaluación"
```

### 3.2 Episodio de tratamiento

Entidad nueva `treatment_episodes`. Agrupa lo que hoy queda suelto:

| Campo | Uso |
|---|---|
| `patient_id`, `service_id`, `location_id` | Identidad del episodio |
| `status` | `active`, `completed`, `abandoned`, `cancelled` |
| `origin_appointment_id` | Consulta que lo originó |
| `origin` | `system`, `legacy` (tratamiento previo al sistema), `external` (operada en otra clínica) |
| `opened_at`, `closed_at` | Ciclo de vida |

Un episodio agrupa:

- **Cirugía**: la orden de la cirugía + todas sus reevaluaciones.
- **Masajes**: una orden de paquete + sus N sesiones.
- **Láser CO₂ / tatuajes**: N órdenes, una por sesión, más sus citas.
- **Criolipólisis**: una orden por sesión realizada, agrupadas si la paciente repite.

`treatment_group_id`, hoy casi sin uso en `appointments` y `proformas`, se sustituye por `episode_id` y se migra su contenido.

### 3.3 Sesiones dentro de la orden

No se crea una tabla de sesiones. La sesión física ya existe: es la cita de tratamiento. Se agrega a `service_orders`:

| Campo | Uso |
|---|---|
| `episode_id` | Episodio al que pertenece |
| `sessions_total` | Cuántas sesiones vendió esta orden. `1` para cirugía, criolipólisis y cada sesión de láser; `N` para un paquete de masajes |
| `sessions_delivered` | Contador materializado, recalculado desde las citas completadas |
| `valid_until` | Vencimiento del paquete, si aplica |
| `billing_mode` | `upfront`, `per_session`, `installments` |
| `included_in_order_id` | Cuando el paquete viene incluido en otra venta, típicamente masajes dentro de una cirugía |

`sessions_delivered` se deriva de las citas del episodio con esa orden y estado `completed`. Se materializa por rendimiento y por control de concurrencia, con la misma técnica de bloqueo que ya usa el trigger de límite de proformas.

Y a `appointments`:

| Campo | Uso |
|---|---|
| `episode_id` | Episodio de la visita, incluidas las reevaluaciones |
| `session_number` | Índice de la sesión dentro de la orden. Permite mostrar "sesión 3 de 10" |

### 3.4 Configuración por servicio

El comportamiento no se decide en el código sino en el catálogo. Se agrega a `services`:

| Campo | Valores | Efecto |
|---|---|---|
| `session_model` | `single`, `package_single_quote`, `series_quote_per_session`, `direct` | Determina si la orden vende una o N sesiones y si cada sesión necesita proforma nueva |
| `default_sessions` | entero | Sesiones sugeridas al crear la orden |
| `package_validity_days` | entero o nulo | Vigencia del paquete desde la primera sesión |
| `min_days_between_sessions` | entero o nulo | Intervalo clínico mínimo, por ejemplo 30 días entre sesiones de láser CO₂ |
| `followup_model` | `none`, `reevaluation_series` | Si al completar el tratamiento se proponen controles |
| `followup_schedule` | lista de días, por ejemplo `[7, 30, 90]` | Controles sugeridos |
| `reevaluation_pricing` | `free`, `charged` | Si la reevaluación de este servicio se cobra |
| `reevaluation_duration_minutes` | entero | Duración del slot de reevaluación, hoy inexistente |

## 4. Matriz de tipos de cita

| Tipo de cita | Evalúa | Ejecuta tratamiento | Puede emitir proforma | Exige orden previa | Precio propio | Cuenta como consulta |
|---|---|---|---|---|---|---|
| Consulta | Sí | Solo vía orden creada en la misma visita | Sí, hasta 3 | No | Sí | Sí |
| Reevaluación | Sí | No | Sí, opcional | No, requiere episodio | No, 0 por defecto | No |
| Reevaluación y tratamiento (derivado) | Sí | Sí | Sí | No, la crea en la visita | No | No |
| Sesión de tratamiento | No necesariamente | Sí | No | Sí | No, el dinero está en la orden | No |
| Procedimiento directo | No | Sí | No | No | Sí | No |

## 5. Comportamiento por servicio

| Servicio | `session_model` | Sesiones | Proformas | Cita posterior típica | Notas |
|---|---|---|---|---|---|
| Cirugía | `single` + `followup_model = reevaluation_series` | 1 procedimiento | 1 principal | Reevaluación | Los masajes post-operatorios pueden proformarse en la misma consulta como segundo tratamiento |
| Masajes | `package_single_quote` | Varias | 1 | Sesión | Admite `included_in_order_id` cuando vienen dentro de la cirugía |
| Láser CO₂ | `series_quote_per_session` | Varias | Varias | Reevaluación y tratamiento | `min_days_between_sessions` activo |
| Láser tatuajes | `series_quote_per_session` | Varias | Varias | Reevaluación y tratamiento | Precio por sesión variable según respuesta |
| Criolipólisis | `single` repetible | Normalmente 1 | 1 por sesión realizada | Nueva sesión si corresponde | Cada sesión es una orden dentro del mismo episodio |

## 6. Flujos objetivo

### 6.1 Punto 3 — Paquete de sesiones con una sola proforma (masajes)

```text
Consulta o cirugía
  -> proforma (servicio: masajes, N sesiones)
  -> orden: total del paquete, sessions_total = N, valid_until
  -> sesión 1 ... sesión N   (citas de tratamiento contra la misma orden)
       cada sesión puede registrar un pago parcial contra la orden
  -> orden completada cuando sessions_delivered = sessions_total y saldo = 0
```

Reglas nuevas:

- Al agendar una sesión, el sistema muestra "sesión k de N" y bloquea si ya no quedan sesiones disponibles, salvo que un administrador amplíe el paquete.
- Una cita cancelada o no-show **no** consume sesión, con la excepción configurable de no-show con penalidad.
- Vencido `valid_until`, las sesiones no consumidas quedan bloqueadas y requieren reactivación explícita con motivo.
- Agendamiento en bloque: una acción crea las N citas del paquete con la periodicidad indicada.

### 6.2 Punto 4 — Serie con proforma por sesión (láser CO₂, tatuajes)

```text
Consulta -> proforma 1 -> orden 1 -> sesión 1
Reevaluación (cita nueva, mismo episodio)
  -> se evalúa la evolución
  -> proforma 2 -> orden 2 -> la sesión 2 se ejecuta en esa misma cita
Reevaluación -> proforma 3 -> orden 3 -> sesión 3
...
```

Reglas nuevas:

- Una cita de reevaluación puede emitir proforma. Es el cambio que desbloquea este caso.
- La cita de reevaluación puede además quedar vinculada como sesión de la orden que acaba de crear, sin necesidad de agendar una segunda cita el mismo día.
- El precio de cada proforma es independiente: la sesión 3 puede costar distinto que la 1.
- El episodio muestra la serie completa: sesiones ejecutadas, órdenes emitidas y saldo consolidado.
- Si la paciente no vuelve, el episodio se marca `abandoned` y sus órdenes pendientes salen del reporte de cuentas por cobrar activas.

### 6.3 Punto 5 — Reevaluaciones puras (cirugía)

```text
Cirugía (orden + sesión ejecutada)
  -> al completar, el sistema propone controles según followup_schedule
  -> reevaluación 7 días   : evalúa, indica, cierra
  -> reevaluación 30 días  : evalúa, indica, cierra
  -> reevaluación 90 días  : evalúa, indica, cierra
```

Reglas nuevas:

- La reevaluación no exige orden ni proforma y no genera cobro por defecto.
- Requiere episodio de origen. Si la paciente fue operada en otra clínica o antes del sistema, se abre un episodio con `origin = external` o `legacy`, lo que reemplaza el uso actual de órdenes `legacy_manual` como percha.
- La reevaluación no cuenta como consulta en reportes ni consume vigencia de consulta.
- Slot con duración propia, distinta de la consulta.

### 6.4 Punto 6 — La reevaluación que incorpora otro tratamiento

```text
Rinoplastia -> reevaluación a los 4 meses
                 -> se evalúa la cirugía (episodio A)
                 -> se cotiza bótox: proforma -> orden (episodio B, nuevo)
                 -> el bótox se aplica en la misma visita o se agenda
```

Reglas nuevas:

- Una cita puede evaluar un episodio y originar otro. El vínculo de evaluación es `episode_id`; el vínculo de ejecución es `service_order_id`. Son campos distintos y pueden apuntar a episodios distintos.
- El límite de tres tratamientos cotizados por cita se mantiene también para reevaluaciones.
- Se conserva la restricción vigente: el tratamiento cotizado debe pertenecer a la sede de la cita.

### 6.5 Punto 2 — Servicio de sesión única repetible (criolipólisis)

Cada sesión es una orden propia dentro del mismo episodio. La segunda sesión se registra desde una reevaluación o desde una consulta según corresponda, pero ya no obliga a inventar una consulta solo para poder cotizar.

### 6.6 Punto 1 — Consulta inicial y vigencia

Entidad nueva a nivel de paciente, `patient_consultation_credits`:

| Campo | Uso |
|---|---|
| `patient_id` | Titular del beneficio |
| `origin_appointment_id` | Consulta pagada que lo genera |
| `granted_at`, `valid_until` | Vigencia calculada según la política elegida |
| `scope` | `global` o `location`, según decisión pendiente |
| `status` | `active`, `expired`, `revoked` |

Al agendar una consulta, el sistema evalúa si el paciente tiene crédito vigente y propone el precio en cero, dejando registrado el motivo de exoneración. Las decisiones abiertas de este punto están en [05-decisiones-pendientes.md](05-decisiones-pendientes.md); es el punto de menor prioridad y depende de definiciones de Ronald.

## 7. Reglas de dinero

| Situación | Dónde se registra el cobro |
|---|---|
| Consulta | En la cita, como hoy |
| Reevaluación gratuita | No genera cobro; queda registrada como atención sin ingreso |
| Reevaluación cobrada | En la cita, con precio propio del servicio |
| Sesión de un paquete | En la orden del paquete |
| Sesión de una serie con proforma propia | En la orden de esa sesión |
| Procedimiento directo | En la cita |

Se mantiene el criterio contable vigente: el ingreso se reconoce por pago confirmado en `payments`. Lo que se agrega es un indicador operativo nuevo, **sesiones cobradas y no entregadas**, calculado como la suma sobre órdenes activas de `(sessions_total - sessions_delivered) x valor unitario`. Es una obligación de servicio, no un asiento contable, y debe presentarse separado del balance.

## 8. Reglas de reportes

| Métrica | Definición nueva |
|---|---|
| Consultas | Citas `consultation`, cobradas o exoneradas |
| Reevaluaciones | Citas `reevaluation` sin orden creada en la visita |
| Reevaluaciones con tratamiento | Citas `reevaluation` con orden creada en la visita |
| Sesiones entregadas | Citas `treatment` en estado `completed` |
| Procedimientos directos | Sin cambios |
| Atenciones totales | Suma de las anteriores |

Ninguna reevaluación debe sumar al conteo de consultas. Este es el ajuste que evita que el indicador de consultas se infle cuando se libere el flujo de reevaluación con proforma.

## 9. Lo que este modelo deliberadamente no hace

- **No crea una tabla de sesiones separada de las citas.** Una sesión sin cita no existe operativamente; los casos de walk-in se registran como cita retroactiva.
- **No mueve el dinero a la cita de tratamiento.** El total sigue viviendo en la orden, lo que preserva la lógica de saldos y cuentas por cobrar ya construida.
- **No convierte "reevaluación y tratamiento" en un tipo de cita.** Es un estado derivado.
- **No usa `appointment_category` para decidir nada.** Solo colorea la agenda.
- **No asume que el número de sesiones iguala al número de proformas.** Esa equivalencia es justamente lo que el modelo separa.
