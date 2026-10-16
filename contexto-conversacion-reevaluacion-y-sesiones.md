# Contexto de conversación: Reevaluación y Sesiones

Fuente: [Resumir modelo de consultas](chatgpt-conversation://6a922cfe-87cc-83e9-ab0d-ca461ea592c7)

> **Estado: preliminar / sujeto a validación con el negocio.**
>
> Este documento resume el análisis de la conversación y sirve como contexto de trabajo. No representa decisiones finales de producto, negocio, contabilidad ni implementación.

## 1. Reevaluación

### Situación identificada

Actualmente existen tres tipos principales de cita: `CONSULTA`, `TRATAMIENTO` y `PROCEDIMIENTO DIRECTO`. `REEVALUACIÓN` funciona principalmente como una categoría o indicador interno, no como un tipo de cita con reglas propias.

Esto genera problemas en casos donde el paciente vuelve después de una atención inicial:

- En una cirugía, la visita posterior normalmente es una reevaluación pura: se revisa la evolución, se dan indicaciones y no necesariamente se genera una nueva proforma.
- En láser CO₂ o láser para eliminación de tatuajes, la visita puede incluir reevaluación, una nueva cotización y la ejecución de otra sesión.
- Una reevaluación de una cirugía también puede terminar originando otro servicio, por ejemplo, bótox, que debe cotizarse como un tratamiento independiente.

### Alternativas analizadas

1. Mantener el tipo `TRATAMIENTO` y usar la categoría `REEVALUACIÓN` para habilitar una nueva proforma.
2. Crear `REEVALUACIÓN` como un nuevo tipo de cita, con capacidad de generar opcionalmente una proforma y, si corresponde, un tratamiento.

### Orientación preliminar

La segunda alternativa parece representar mejor la realidad operativa, porque al agendar la cita todavía no se sabe si habrá tratamiento. La visita puede terminar en cualquiera de estos resultados:

```text
REEVALUACIÓN
├── Solo evaluación
├── Nueva proforma, sin tratamiento todavía
├── Nueva proforma + tratamiento en la misma visita
└── Nueva proforma + tratamiento en una cita futura
```

En este enfoque:

- `REEVALUACIÓN` representa la evaluación de seguimiento.
- `PROFORMA` representa la cotización.
- `ORDEN DE SERVICIO` representa el compromiso económico autorizado por recepción.
- `TRATAMIENTO` representa la ejecución de algo que ya cuenta con una orden.
- “Reevaluación y tratamiento” sería un resultado o estado derivado, no necesariamente un tipo de cita adicional.

El servicio evaluado en la cita no debería obligar a que la nueva proforma sea del mismo servicio. Por ejemplo, una reevaluación de rinoplastia podría originar una proforma de bótox.

### Puntos todavía abiertos

- Si todas las reevaluaciones pueden generar proformas o solo ciertos servicios.
- Si el tratamiento puede ejecutarse en la misma cita después de que recepción genere la orden.
- Qué ocurre cuando el paciente cotiza pero decide tratarse otro día.
- Si una misma reevaluación puede generar más de una proforma.
- Reglas para reevaluaciones quirúrgicas gratuitas, cobradas o fuera de una ventana de seguimiento.

## 2. Sesiones y proformas

### Situación identificada

La conversación distingue dos modelos que no deben confundirse:

### A. Varias sesiones con una sola proforma

Es el caso mencionado de **masajes**. Se vende un paquete completo, se genera una sola orden y las sesiones posteriores se aplican contra esa orden.

```text
Consulta o cirugía
  → proforma del paquete
  → una orden de servicio
  → sesiones 1, 2, 3... contra la misma orden
```

Los pagos pueden hacerse parcialmente sobre la orden del paquete. La cantidad de sesiones entregadas debe controlarse por separado de los pagos.

### B. Varias sesiones con una proforma por sesión

Los casos identificados son **láser CO₂** y **láser para eliminación de tatuajes**. Cada visita puede tener un precio propio y generar una proforma y una orden independientes.

```text
Sesión 1 → proforma 1 → orden 1 → tratamiento
Sesión 2 → proforma 2 → orden 2 → tratamiento
Sesión 3 → proforma 3 → orden 3 → tratamiento
```

El precio por sesión podría mantenerse fijo o variar según la decisión médica y la evolución del paciente.

### Caso relacionado: criolipólisis

Se mencionó como un servicio que normalmente corresponde a una sesión por atención. Si la paciente desea una nueva sesión posteriormente, se generaría una nueva proforma y una nueva orden, sin asumir que todas las sesiones pertenecen a un paquete único.

### Orientación preliminar de modelado

La modalidad no debería guardarse únicamente en cada cita. Una separación propuesta es:

```text
SERVICIO
  → qué modalidades admite

TRATAMIENTO / EPISODIO DEL PACIENTE
  → qué modalidad se eligió para este caso

CITA
  → ejecuta una sesión o una reevaluación

PROFORMA / ORDEN
  → define el cobro de un paquete o de una sesión concreta
```

El servicio podría configurarse para admitir, por ejemplo, paquete o cobro por sesión. Durante la consulta, el médico elegiría la modalidad aplicable al paciente y, si corresponde, un precio por sesión fijo o variable. Esa decisión debería persistir en el tratamiento o episodio que agrupa las sesiones, no repetirse manualmente en cada cita.

### Reglas que conviene preservar

- Una sesión no equivale necesariamente a una proforma.
- Una cita no equivale necesariamente a un cobro.
- En un paquete, varias sesiones pueden consumir una sola orden.
- En una serie por sesión, cada sesión puede tener su propia orden.
- El total, pagado y saldo deben continuar viviendo en la orden de servicio.
- Las sesiones deben poder agruparse para consultar el historial completo del tratamiento.

### Puntos todavía abiertos

- Si la primera sesión por sesión se cotiza en la consulta o recién en una reevaluación posterior.
- Si el precio por sesión es fijo, variable o una combinación de precio sugerido y ajuste médico.
- Si un servicio puede ofrecer tanto paquete como pago por sesión.
- Cómo se controla el número de sesiones vendidas, entregadas, pendientes y vencidas.
- Qué ocurre si se cambia de modalidad a mitad del tratamiento.
- Si se permite pago parcial antes de ejecutar una sesión.

## Síntesis provisional

La separación conceptual más consistente sería:

> **El servicio define las modalidades posibles; el tratamiento del paciente conserva la modalidad elegida; la reevaluación decide la siguiente acción; la proforma cotiza; la orden autoriza el cobro; y la cita de tratamiento ejecuta una sesión.**

Esta síntesis es una propuesta de análisis, no una decisión aprobada. Debe validarse con los responsables del negocio antes de convertirla en reglas definitivas, migraciones o cambios de interfaz.
