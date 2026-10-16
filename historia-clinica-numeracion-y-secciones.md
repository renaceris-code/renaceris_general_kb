# Historia clínica — numeración, sede y secciones

**Estado:** vigente desde el 29/09/2026 · **Migraciones:** 045, 046 (numeración) y 047 (secciones)
**Backlog:** [backlog-nuevas-funcionalidades-2026-09-28-10-20.md](backlog-nuevas-funcionalidades-2026-09-28-10-20.md) §3

Esta es la referencia para no volver a confundir la sede de la historia clínica con la sede del paciente.

---

## 1. Reglas

1. **Una sola historia clínica (HC) por paciente.** Formato `SHCO-000123` (Huánuco), `SLIM-000123` (Lima), `SPUC-00123` (Pucallpa).
2. **La sede del prefijo solo indica dónde fue la primera cita elegible del paciente**, quirúrgica o no:
   - Primera cita elegible: la primera consulta, o la primera cita de un tratamiento o reevaluación de un tratamiento previo.
   - Se fija una vez y es **permanente**. Queda guardada en `patients.first_location_id`.
3. **El paciente no está afiliado a ninguna sede.** Se atiende en cualquier sede y conserva siempre la misma HC. Ningún permiso ni regla de agenda debe **bloquear** pacientes por la sede de su HC.
   - Sí se permite un **filtro de vista** en la lista de Pacientes, "Sede de 1.ª atención": la sede de la HC (`first_location_id`) o, si el paciente aún no tiene HC (solo procedimientos directos), la sede de su primera cita.
   - El filtro arranca en la sede del usuario (Huánuco si ve varias) y siempre ofrece "Todos". Es una comodidad, no una restricción de acceso.
4. **La HC se divide en dos secciones**, solo para organizarla:
   - `01` No quirúrgica: láser, INDIBA, BOTOX y el resto de servicios no quirúrgicos.
   - `02` Quirúrgica: servicios marcados como "Servicio quirúrgico".
5. **Código de sección:** la HC con sufijo, `SHCO-000123-02`. No es otra HC ni otra numeración.
6. **La sección de una cita la define su servicio** (`services.is_surgical`). Marcar o desmarcar un servicio reclasifica todas sus citas y no cambia ningún número.
7. **Una sección está abierta** si el paciente tiene al menos una cita no cancelada de ese tipo. Un paciente que solo se operó tiene únicamente la sección 02.
8. **Lo que no pertenece a una cita es común a toda la HC**: alergias, antecedentes, medicación y notas generales. Aparece en cualquier sección.

## 2. Ejemplos

| Caso | Resultado |
|---|---|
| Primera consulta de láser en Huánuco | HC `SHCO-000123`, sección 01 |
| Luego se opera en Lima | Misma HC `SHCO-000123`; se abre la sección 02: `SHCO-000123-02` |
| Primera cita: consulta quirúrgica en Pucallpa | HC `SPUC-00042`, solo sección 02 |
| Solo tiene una cirugía cancelada | Sin sección abierta |
| La clínica marca "Rinoplastia" como quirúrgica después | Sus citas pasan a la sección 02; ningún número cambia |

## 3. Dónde está en el código

| Pieza | Ubicación |
|---|---|
| Numeración y sede de la HC | `renaceris_backend/migrations/045_…`, `046_clinical_records_first_consultation.sql` |
| Marca quirúrgica y vista de secciones | `renaceris_backend/migrations/047_clinical_record_sections.sql` |
| Secciones en la API de pacientes | `renaceris_backend/src/patients/clinical-record-sections.ts` |
| Marca en Servicios | `renaceris_backend/src/services/services.service.ts` (`update`) |
| Utilidades y etiquetas del frontend | `renaceris_front/src/utils/clinicalRecordSections.ts`, `components/ClinicalRecordSectionBadges.tsx` |
| Pantallas | `Services.tsx`, `Patients.tsx`, `PatientProfile.tsx` (selector y PDF por sección), `NewAppointment.tsx` |

## 4. Decisiones descartadas

- **Dos HC por paciente** (`SHCO-01-00001` / `SHCO-02-00001`): se descartó el 29/09. Obligaba a renumerar de forma irreversible y contradecía la regla de HC única por paciente.
- **Guardar la sección en cada cita**: se descartó porque la sección derivada del servicio permite corregir marcas sin migraciones ni cargas manuales.
