
## Daily 29/09 — Historia clínica y Punto de venta

**Ayer / últimos días:**

- Se definió que cada paciente mantiene una sola historia clínica, dividida en dos secciones: 01 No quirúrgica y 02 Quirúrgica (por ejemplo, SHCO-000123-02). No se cambió ningún número existente.
- En Servicios ahora se puede marcar un servicio como quirúrgico. Cada cita va a la sección que corresponde a su servicio, y si un servicio se marca mal, al corregirlo sus citas pasan solas a la sección correcta.
- En Pacientes se ve la etiqueta "Quirúrgica" o "No quirúrgica" y se puede filtrar por sección. En el perfil se elige ver la historia completa o solo una sección, y el PDF se exporta por sección. Alergias y antecedentes se muestran en ambas secciones.
- Quedó escrito en el sistema y en la documentación que la sede de la historia clínica solo indica dónde fue la primera cita del paciente y no cambia. El paciente no pertenece a ninguna sede: se atiende en cualquiera y conserva su historia clínica.

**Hoy me enfoco en:**

- Publicar el cambio de historia clínica: aplicar el ajuste en la base de datos real y luego subir el sistema.
- Empezar el Punto de venta para recepción: preparar dónde se guardarán las ventas de productos, el cierre de caja de cada día y el catálogo de productos (nombre, categoría y precio, sin control de stock).
- Registrar la venta de productos por sede, de modo que cada recepcionista solo vea y registre las ventas de su sede.

**Dependencias:**

- La clínica debe marcar sus servicios quirúrgicos en cada sede. Mientras no lo haga, todos los pacientes aparecerán solo con la sección 01.
- **Lista de productos** que vende recepción, con su categoría y precio. Sin ella, el catálogo arranca vacío.
- **Yape y transferencias al cerrar caja:** falta definir si esos cobros pasan a la caja general o al banco.
- **Validación de transferencias entre cajas:** falta definir quién la hace (quien recibe, el contador o el admin). Esto también aplica al registro de ingresos, que el Punto de venta necesita para enviar el cierre validado a la caja general (CAJA HUANUCO, CAJA LIMA o CAJA PUCALLPA).
- **Captura del voucher:** falta definir si el contador puede verla, porque puede mostrar el nombre del paciente.

## Daily 21/09 — Módulo Contabilidad

**Ayer / últimos días:**

- Cargué las 10 cuentas bancarias y las 9 cajas chicas que entregó Ronald y las asocié a su sede cuando la relación era clara. Publicidad, Ronald, Balsa y Eventos quedaron sin sede hasta confirmarlo. También corregí el resumen, que repetía los mismos ingresos en todas las cajas. [REN-29](https://linear.app/sprinta/issue/REN-29)
- Completé el registro de gastos con número de operación, medio de pago (Efectivo, Transferencia o Yape), caja o banco de origen, sede o proyecto de destino y un detalle de hasta 50 caracteres. Si hay un error, el formulario conserva lo que ya se escribió. [REN-28](https://linear.app/sprinta/issue/REN-28)
- Cada administrador de sede ahora solo ve y opera las cajas de su sede, aunque intente acceder de otra forma. Habilité al superusuario (Jonathan), que revisa todas las sedes y valida o rechaza los gastos al día siguiente. Nadie puede validar un gasto que registró él mismo. [REN-27](https://linear.app/sprinta/issue/REN-27) · [REN-30](https://linear.app/sprinta/issue/REN-30)
- Ya se puede adjuntar la foto o el PDF de la factura o boleta, y el revisor la ve dentro de la revisión. El sistema bloquea un comprobante ya registrado, salvo que se escriba un motivo, que queda guardado. [REN-31](https://linear.app/sprinta/issue/REN-31)
- Agregué el registro de proveedores: al escribir el RUC se completan solos el nombre y la dirección, y si la consulta falla se pueden ingresar a mano. Los pagos al personal ahora se asignan a un usuario registrado del sistema, en lugar de escribir el nombre a mano. [REN-32](https://linear.app/sprinta/issue/REN-32) · [REN-33](https://linear.app/sprinta/issue/REN-33)

**Esta semana me enfoco en:**

- Aplicar en la base de datos real los cambios de estructura que requieren estas mejoras y publicar el código, que por ahora solo está en local. [REN-29](https://linear.app/sprinta/issue/REN-29)
- Dar a Jonathan el permiso de superusuario y verificar que cada administrador vea solo su sede. [REN-27](https://linear.app/sprinta/issue/REN-27)
- Probar el flujo completo con datos reales: registrar un gasto, adjuntar el comprobante, validarlo o rechazarlo y corregirlo. [REN-30](https://linear.app/sprinta/issue/REN-30)
- Confirmar con una consulta real que el servicio de RUC devuelve bien el nombre y la dirección. [REN-32](https://linear.app/sprinta/issue/REN-32)
- Vincular los pagos al personal anteriores que figuran como "responsable sin vincular". [REN-33](https://linear.app/sprinta/issue/REN-33)

**Dependencias:** acceso para aplicar los cambios en la base de datos de Renaceris (hoy no la tengo conectada), confirmar a qué sede pertenecen Publicidad, Ronald, Balsa y Eventos, y un RUC real para probar la consulta automática.