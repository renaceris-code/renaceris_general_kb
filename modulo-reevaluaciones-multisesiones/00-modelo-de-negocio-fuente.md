# Modelo de negocio — Fuente

Lectura del negocio extraída de la conversación con la clínica. Es el insumo original del módulo; los documentos siguientes lo traducen a diagnóstico técnico y esquema de sistema.

## 1. Consulta inicial

La paciente paga una consulta inicial, S/ 50 como ejemplo, y desde ese momento las consultas posteriores pueden realizarse sin volver a cobrar consulta durante un periodo de vigencia.

El beneficio no depende de que la paciente contrate o se realice el procedimiento. Puede pagar una consulta por rinoplastia, no operarse y meses después consultar por bótox sin volver a pagar consulta.

Queda pendiente definir con Ronald cómo se calcula la vigencia: año calendario de enero a diciembre, o 12 meses desde la primera consulta. Importa porque una consulta de noviembre o diciembre tendría muy poca vigencia bajo el modelo de año calendario.

## 2. Servicios de una sola sesión

Ejemplo: criolipólisis.

```text
Consulta -> Proforma -> Tratamiento
```

Si un mes después quiere otra sesión, se trata como una nueva atención con su propia proforma. No necesariamente vuelve a pagar consulta si sigue dentro de la vigencia.

## 3. Servicios de varias sesiones con una sola proforma

Caso principal: masajes.

```text
Consulta o cirugía -> una proforma por el paquete -> múltiples sesiones
```

La paciente asiste a sus sesiones y paga según corresponda. No necesita pasar de nuevo por evaluación médica ni generar una proforma por cada masaje. Con frecuencia estos masajes ya se dejan indicados y proformados como parte del proceso de una cirugía.

## 4. Servicios de varias sesiones y varias proformas

Casos: láser CO₂ y láser para eliminación de tatuajes.

```text
Consulta -> sesión 1 + proforma 1 -> sesión 2 + proforma 2 -> sesión 3 + proforma 3 ...
```

Cada sesión puede implicar revisar la evolución, aplicar nuevamente el tratamiento y emitir una proforma nueva. Por eso llamar a estas citas solo "reevaluación" es incorrecto: la categoría correcta es **reevaluación y tratamiento**, porque en la misma cita se evalúa, se trata y se cotiza.

## 5. Reevaluaciones puras

En cirugías, las citas posteriores normalmente son solo reevaluaciones.

```text
Cirugía -> reevaluación -> reevaluación -> reevaluación
```

Se revisa evolución, se dan indicaciones y termina la atención. En principio no hay tratamiento nuevo ni proforma asociada.

| Tipo de cita | Evalúa | Hace tratamiento | Genera proforma |
|---|---|---|---|
| Consulta | Sí | Puede depender del flujo | Sí |
| Reevaluación | Sí | No | Normalmente no |
| Reevaluación y tratamiento | Sí | Sí | Sí |
| Tratamiento o sesión | No necesariamente | Sí | Depende del servicio |

## 6. Una reevaluación puede terminar generando otro tratamiento

Una paciente puede venir a la reevaluación de una cirugía y aprovechar la visita para realizarse otro procedimiento.

```text
Rinoplastia -> reevaluación cuatro meses después -> decide colocarse bótox
```

La cita era una reevaluación, se añade un tratamiento nuevo y se genera una proforma de bótox.

## Modelo general

Lo importante es no modelar "número de sesiones igual a número de proformas", porque no siempre se cumple. La relación correcta es:

```text
Paciente -> Consulta -> Tratamiento o servicio -> Proforma(s) -> Sesión(es)
```

Cada servicio define su propio comportamiento:

| Servicio | Sesiones | Proformas | Cita posterior |
|---|---|---|---|
| Cirugía | 1 procedimiento más seguimientos | 1 principal | Reevaluación |
| Masajes | Varias | 1 | Sesión o tratamiento |
| Láser CO₂ | Varias | Varias | Reevaluación y tratamiento |
| Láser tatuajes | Varias | Varias | Reevaluación y tratamiento |
| Criolipólisis | Normalmente 1 | 1 por sesión realizada | Nueva sesión si corresponde |

La conclusión funcional es que hay tres dimensiones distintas que no deben mezclarse: el tipo de cita, la cantidad de sesiones y la generación de proformas. Una cita puede incluso empezar como reevaluación y terminar incorporando un tratamiento adicional con una proforma nueva.
