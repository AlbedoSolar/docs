# 14. Adendas

Una adenda reemplaza la cotización firmada vigente de un proyecto por una nueva. El sistema la maneja como un flujo automatizado: no se editan a mano las cotizaciones ni los flujos de caja.

## Antes de empezar

Una adenda siempre parte de un proyecto que **ya tiene una cotización firmada y vigente**. Si el proyecto no está firmado, no es una adenda — es simplemente una cotización nueva.

## Procedimiento en el sistema

### 1. Duplica el estimado firmado

En la página del proyecto, duplica el estimado vigente y aplica los cambios que motivan la adenda (alcance, plazo, precio, condiciones).

### 2. Ajusta los costos ya pagados

Este es el paso que más se olvida y el que genera cobros dobles.

- **Costos legales.** El motor recalcula el costo legal en cada generación e **ignora** el valor guardado en el estimado. Si el cliente ya pagó los costos legales en el contrato original, tienes que ponerlos en cero explícitamente:
  **Opciones Avanzadas → Otros → «Costos Legales (override, sin IVA)» = 0**.
  Si no lo haces, la adenda vuelve a cobrar un monto ya pagado.
- **Enganche ya pagado.** Usa los campos de enganche normales: 0, o el crédito que corresponda.

### 3. Genera la oferta

Genera la oferta como en cualquier cotización.

### 4. Aprueba como ADENDA

En la matriz de variantes, presiona **Aprobar** sobre la variante elegida. Como el proyecto está firmado, aparece un diálogo de confirmación para aprobar **como adenda**.

Al confirmar, el sistema enlaza la nueva cotización con la firmada vigente y le asigna su número de adenda. Aprobar es solo intención: se puede revertir con **Desaprobar** sin consecuencias.

### 5. Genera el contrato

Al generar el contrato se abre el formulario previo. Dos campos importan:

- **Fecha planeada de firma** — ancla las fechas del flujo de pagos del contrato.
- **Primer mes de gracia (opcional)** — **déjalo vacío en una adenda**, salvo que quieras restatear el historial a propósito.

  Si lo llenas con la fecha de inicio del **contrato original**, el contrato queda con pagos en meses ya transcurridos. El sistema ahora muestra en qué mes caerá el pago 1 y bloquea la generación si esa fecha es anterior a la firma, hasta que confirmes que es intencional.

Cada generación crea un borrador nuevo. Si generas varias veces, firma el borrador correcto.

### 6. Firma

Al firmar, el sistema hace la sustitución automáticamente:

- La cotización y el estimado anteriores quedan marcados como sustituidos con la fecha de firma (la **costura**).
- Los pagos **anteriores o iguales** a la costura **siguen activos** — son historial real y así deben quedar para facturación, Zoho y reportes.
- Los pagos **posteriores** a la costura quedan anulados; desde ahí gobierna el flujo nuevo.

La regla que se mantiene: los flujos activos de toda la cadena forman **una sola línea de tiempo**, sin meses duplicados ni huecos en la costura.

## Cuándo cambia la facturación

**La facturación cambia al momento de la firma, no antes.**

Mientras la adenda está en negociación, el sistema sigue facturando con el contrato vigente, que es el único que el cliente ha firmado. No hay un estado intermedio que cambie los montos facturados antes de la firma.

Consecuencia práctica: si la negociación se alarga, se emiten facturas con el esquema anterior. Esas facturas son válidas hasta la firma; los ajustes que correspondan se manejan por nota de crédito o por el diseño del flujo nuevo, no adelantando la sustitución en el sistema.

## Errores frecuentes

| Error | Qué pasa | Cómo evitarlo |
|---|---|---|
| No poner el override legal en 0 | La adenda vuelve a cobrar los costos legales | Paso 2 |
| Llenar «Primer mes de gracia» con la fecha del contrato original | El contrato incluye meses ya transcurridos | Paso 5 — déjalo vacío |
| Aprobar sin el diálogo de adenda | La cotización no queda enlazada a la cadena | Aprueba desde la página del proyecto, no desde /offers |
| Generar varios borradores y firmar el equivocado | Se firma un flujo que no es el acordado | Verifica el borrador antes de firmar |

## Referencias

- Flujo automatizado disponible desde el 2026-07-20.
- Semántica de la costura y casos de restatement completo: `docs/quote-generation/6-business-decisions-log.md`.
