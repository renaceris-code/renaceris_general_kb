# Propuesta del Nuevo Módulo de Contabilidad
### Una explicación clara para Renaceris

> Este documento resume, en lenguaje sencillo, qué queremos construir y por qué.
> Es un primer borrador para conversarlo juntos, no una versión final.

---

## En una frase

Queremos agregarle a su sistema una **sección de reportes de dinero**: un lugar donde, con un par de clics, usted vea cuánto entró, por qué concepto y en qué sede, y donde pueda **descargar esa información lista** para pasársela a su contador o a su sistema de facturación.

**Importante:** esto **no reemplaza** su sistema de facturación ni el trabajo de su contador. Es una ayuda que le ahorra tiempo y le da orden.

---

## El problema que resolvemos

Hoy, cada cita, cada pago y cada tratamiento ya quedan registrados en el sistema cuando la recepcionista agenda o cuando el doctor atiende. Toda esa información **ya existe**, pero está "suelta": para saber cuánto se cobró esta semana en tal sede, o cuánto le deben todavía los pacientes con tratamientos en cuotas, alguien tiene que sacar cuentas a mano.

> *Ejemplo:* hoy, si quiere saber "¿cuánto entró en efectivo el mes pasado en la sede de Lima?", probablemente alguien tiene que revisar registros uno por uno. Con el módulo, es un filtro y un botón.

---

## Qué hará el módulo (en dos partes)

### Parte 1 — Ver el dinero de forma rápida y ordenada

Un tablero donde usted pueda ver los ingresos ya registrados, filtrando por lo que le interese:

- **Por fecha** (hoy, esta semana, este mes, un rango cualquiera).
- **Por sede.**
- **Por forma de pago** (efectivo, tarjeta, transferencia, etc.).
- **Por tipo de servicio** (consulta, tratamiento, láser, etc.).
- **Por doctor.**

Y con resúmenes automáticos, por ejemplo:

> *"En junio, la sede Miraflores facturó S/ X: 60% en tarjeta y 40% en efectivo, y quedan S/ Y pendientes de cobro en tratamientos por cuotas."*

También le mostrará **lo que falta cobrar**: pacientes que iniciaron un tratamiento y aún deben saldo.

### Parte 2 — Descargar la información lista para usarla

Con un botón, el sistema genera **archivos (Excel o PDF)** que usted puede:

- Enviar a su **contador** para que cuadre las cifras sin tener que pedirle a nadie que las arme.
- Usar como **puente** con su sistema de facturación, para contrastar de forma básica que todo coincide.

> *Ejemplo de descargas útiles:*
> - Un **resumen de caja del día** por sede y forma de pago.
> - Un **listado de todos los pagos del mes**, con paciente, monto, comprobante y sede.
> - Un **reporte de lo que falta cobrar**.
> - Un **reporte de "alertas"**: por ejemplo, atenciones que se realizaron pero cuyo pago no quedó confirmado.

---

## Qué NO hace este módulo (para que quede claro)

- **No emite boletas ni facturas electrónicas** (eso sigue en su sistema actual y en SUNAT).
- **No reemplaza a su contador ni sus libros contables.**
- **No maneja impuestos ni declaraciones.**

Piénselo como un **asistente que ordena y entrega la información**, no como un reemplazo de lo que ya funciona.

---

## Un par de cosas que notamos y conviene conversar

Al revisar cómo está guardada hoy la información, encontramos algunos puntos que valdría la pena afinar para que los reportes salgan 100% confiables:

- **Las formas de pago se escriben libremente.** Si a veces se anota "Efectivo", otras "efectivo" y otras "EFEC", el reporte puede contarlas por separado. Convendría estandarizar una lista fija de opciones.
- **Hoy solo se registran los ingresos, no los gastos.** Si usted quisiera ver un flujo de caja completo (lo que entra y lo que sale), habría que definir cómo registrar los egresos. Si por ahora solo le interesan los ingresos, también está bien.
- **No hay un registro claro de anulaciones o devoluciones.** Si a un paciente se le devuelve un pago, hoy no queda del todo reflejado. Vale la pena decidir cómo tratarlo.

Ninguno de estos puntos impide arrancar; solo queremos acordarlos para que los números salgan bien.

---

## Preguntas para usted (para afinar la propuesta)

1. ¿En qué **formato exacto** le gusta recibir la información a su contador o a su sistema de facturación? (Si nos pasa una plantilla, la replicamos.)
2. ¿Le interesa por ahora **solo los ingresos**, o también quiere registrar **gastos**?
3. ¿Manejan **apertura y cierre de caja** por turno que debamos reflejar?
4. ¿Cómo tratan hoy las **devoluciones o anulaciones** de pagos?

---

## Cómo seguimos

1. Conversamos este documento y sus respuestas a las 4 preguntas de arriba.
2. Definimos juntos **qué reportes quiere ver primero**.
3. Armamos una primera versión sencilla (un MVP) con esos reportes y sus descargas, para que la use de inmediato y la vayamos mejorando.

> La idea es empezar simple, que le sea útil desde el día uno, y crecer desde ahí.
