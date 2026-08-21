# 15. Anulaciones

Una **anulación** deja sin efecto un contrato **ya firmado**. El sistema lo maneja como una sola acción: no se editan a mano las cotizaciones, los contratos ni los flujos de caja.

## Cuándo es una anulación (y cuándo no)

| Situación | Qué usar |
|---|---|
| El cliente no firmó y ya no va a firmar | Estado de la cotización: **Rechazada** o **Cancelada** |
| Se firmó, pero cambian las condiciones y sigue habiendo contrato | Una **adenda** (ver [Adendas](14-adendas.md)) |
| Se firmó y el contrato queda sin efecto | **Anulación** |

Si el proyecto no tiene una cotización firmada, el sistema no te deja anularlo: te va a pedir que uses Rechazada o Cancelada.

## Quién puede anular

Solo **administradores**. El botón no aparece para los demás roles, y aunque alguien intente hacerlo por otra vía, la base de datos lo rechaza.

## La fecha de anulación es una decisión, no un trámite

Es lo más importante de todo el procedimiento.

La fecha que elijas parte el flujo de pagos en dos:

- Los pagos **hasta esa fecha, inclusive**, se conservan como historial real. Ya se cobraron, ya se facturaron y ya salieron en los reportes que se enviaron a los inversionistas.
- Los pagos **posteriores** quedan sin efecto.

Es la misma lógica que usa una adenda: lo que ya se cobró no se borra. Por eso la fecha normalmente es **el día en que el contrato dejó de tener efecto**, no el día en que estás registrando la anulación en el sistema.

La fecha no puede ser anterior a la firma del contrato. Si necesitas dejar sin efecto el contrato completo, incluyendo pagos ya cobrados, eso no es una anulación: escríbele a sistemas.

## Procedimiento

### 1. Abre la página del proyecto

Entra al proyecto que vas a anular.

### 2. Presiona «Anular proyecto»

Está arriba a la derecha, en la tarjeta de información del proyecto.

### 3. Llena el diálogo

- **Fecha de anulación** — la fecha desde la cual el contrato deja de tener efecto (ver arriba).
- **Motivo** — obligatorio. Queda guardado en el proyecto y se muestra a quien lo abra después. Escribe algo que le sirva a alguien que lea esto dentro de un año.
- Escribe **anular** para confirmar.

### 4. Verifica

Al terminar, el proyecto muestra una banda roja con la fecha y el motivo, y el estado cambia a **Anulado**.

## Qué pasa después, automáticamente

El proyecto sale de:

- Proyectos firmados y **Finanzas**
- **Operaciones** y el listado de mantenimientos
- El dashboard de riesgo financiero
- Los reportes y el loan tape
- El buscador de proyectos del asistente de cotizaciones
- La página pública de la oferta (el enlace que tenga el cliente deja de abrir)

Los pagos anteriores a la fecha de anulación **siguen** apareciendo en el historial de pagos, que es lo correcto: ese dinero entró.

## ⚠️ Paso manual: Zoho

**El sistema no borra nada en Zoho.** Los flujos que ya se subieron al módulo de Cash Flow siguen ahí.

Si ya se emitió alguna factura correspondiente a un pago **posterior** a la fecha de anulación, hay que emitir la **nota de crédito** correspondiente a mano.

Revísalo siempre después de anular.

## Si te equivocaste

En la banda roja hay un enlace **«Revertir anulación»**. Devuelve el proyecto y todos sus pagos a como estaban. También es solo para administradores.
