# Auditoría de paginación de pacientes y agenda

## Causa y corrección

La API de Supabase limita el tamaño de cada respuesta. Una consulta sin rangos
puede devolver un subconjunto sin error. Agenda mostraba `N/A` porque buscaba el
paciente exclusivamente en ese listado incompleto, aun cuando la cita incluía
su relación con `patients`.

Se corrigieron los caminos relacionados:

- Listado de pacientes: lectura por páginas con orden estable y opción de omitir
  la consulta de últimas citas en los selectores de agenda.
- Nombres y datos de pacientes en agenda y calendarios: recuperación de la
  relación incluida en la cita, además del directorio local.
- Pacientes del médico: paginación de citas antes de deduplicar pacientes.
- Búsqueda avanzada: paginación tanto con filtros de citas como sin ellos.
- Expediente clínico: eliminación del corte de 200 citas que afectaba historial,
  conteos y PDF. Se conserva el fallback para esquemas sin `service_order_id`.
- Órdenes por paciente: paginación de órdenes y citas asociadas; los filtros
  `IN` se agrupan en lotes de 100 órdenes para limitar el tamaño de la URL.
- Reportes: paginación de las 12 consultas usadas por métricas, detalle de citas,
  ejecución, caja y exportación SIBI para evitar totales parciales.

Las consultas ordenan por un identificador único como desempate. El helper
reconstruye cada consulta, avanza por las filas efectivamente devueltas y termina
al recibir una página vacía; así tolera un `max_rows` del servidor menor que el
tamaño solicitado. Los errores de páginas posteriores y el límite de seguridad
de 50 páginas provocan un error visible, no una respuesta parcial.

## Índices y rendimiento

Las migraciones existentes definen un índice parcial para `patients(dni)` y
índices únicos para DNI e historia clínica normalizados con `btrim`. También
definen `appointments(patient_id, service_id, datetime)` parcial para citas no
canceladas y los índices de `service_order_id` y pagos por cita/orden.

No se añadieron índices: el conector SQL de Supabase no tiene permisos para este
proyecto y no permitió verificar los índices realmente instalados ni ejecutar
`EXPLAIN (ANALYZE, BUFFERS)`. El acceso REST sí permitió verificar el listado
corregido contra la BD: se recuperaron los 1.065 pacientes presentes, incluida
Kenia, sin consultar el historial de citas. Los siguientes son candidatos a
evaluar con planes reales:

- `patients(created_at DESC, id ASC)` para el directorio y sus páginas.
- `appointments(patient_id, datetime DESC, id ASC)` para historial y última cita.
- `appointments(doctor_id, datetime DESC, id ASC)` para pacientes de un médico.
- `appointments(datetime, id)` y `payments(created_at, id)` según volumen y
  selectividad de los rangos usados en agenda/reportes.

El índice existente `(patient_id, service_id, datetime)` no sustituye directamente
al de historial: `service_id` se interpone entre paciente y fecha; además el
predicado parcial excluye citas canceladas que sí aparecen en el expediente.
La búsqueda `ILIKE '%texto%'` tampoco se acelera normalmente con un B-tree; si
se demuestra que es costosa, evaluar índices trigramas en las columnas usadas,
con umbral mínimo de búsqueda y medición de su costo de escritura.

Para conjuntos grandes, la evolución recomendada es un buscador remoto de
pacientes, paginación en la API y agregados SQL para reportes. El último historial
por paciente debería resolverse en SQL en vez de transferir todas las citas.
La paginación por offset también puede variar ante escrituras concurrentes;
para exportaciones que exijan una instantánea exacta, usar una consulta SQL
transaccional o un corte consistente con paginación por cursor.

## Verificación y alcance pendiente

Se añadieron regresiones con más de 1.000 registros para pacientes de médicos,
búsqueda, expediente, órdenes, métricas y detalle de reportes; también se prueban
errores de páginas posteriores y el fallback de esquema del expediente.

Se detectaron consultas sin rangos en catálogos de servicios, sedes y usuarios,
así como otros módulos de formularios/contabilidad. No se modificaron en este
cambio: deben auditarse según su cardinalidad y contrato de respuesta. Tampoco
se cambió el esquema remoto ni se confirmó el plan de ejecución de producción.
