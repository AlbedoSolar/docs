# 6. Cotizaciones y Quote Wizard

## Quote Wizard

El Quote Wizard es el asistente principal para generar cotizaciones. Te guia en 4 pasos, con navegacion libre entre pasos ya visitados.

### Paso 1 — Cliente

- Selecciona un cliente existente de la lista
- O crea un cliente nuevo directamente aqui
- Al seleccionar, se avanza al paso 2

### Paso 2 — Proyecto

- Selecciona un proyecto existente del cliente
- O crea un proyecto nuevo
- Define: nombre, sitio, sector, ubicacion (pais, departamento, municipio)
- Indicadores de impacto social (checkboxes): liderado por mujeres, sin fines de lucro, area rural, liderado por jovenes, institucion educativa

### Paso 3 — Estimacion

- Selecciona una estimacion existente del proyecto
- O crea una nueva con:
  - **Fases del proyecto** — cada fase tiene tipo (Instalacion, Estructura, Solo Mantenimiento), proveedor, costo y moneda del contrato
  - **Equipo de ventas** — vendedor principal, vendedor secundario (opcional), director comercial (opcional)
  - **Afiliados** — aliados y embajadores (opcionales)
  - **Tipo de comision** — seleccionado automaticamente o manualmente segun el equipo de ventas
  - **Equipos** — paneles, inversores, baterias (informativo, no afecta calculos financieros)
  - **Produccion solar mensual** (kWh)
  - **Precio de electricidad** por kWh y su moneda
  - **Moneda del proyecto** (lo que paga el cliente)
  - **Fecha de inicio del proyecto**
  - Otros campos de energia (consumo baseline, factura actual, numero de medidores)

### Paso 4 — Cotizacion

- Configura los terminos financieros:
  - Costo del activo, plazo del prestamo, tasa de interes anual
  - Enganche (porcentaje), periodo de gracia, costos legales, opcion de compra
  - Seguro (porcentaje, tasa de deflacion, override manual opcional)
  - Mantenimiento (porcentaje por visita, tasa de inflacion, prima)
  - Comisiones, WACC, tasa libre de riesgo, margen objetivo, nivel de riesgo
- **Modos goal-seek**: ademas del modo manual, puede resolver automaticamente la tasa de interes para alcanzar un TIR o pago mensual objetivo
- Calcula los KPIs financieros
- Previsualiza el flujo de caja
- Guarda la cotizacion

## Navegacion del wizard

- El paso actual se muestra en verde
- Los pasos ya visitados se muestran en azul oscuro y son clickeables para volver
- Los pasos futuros ya alcanzados tambien son clickeables
- Los pasos nunca visitados aparecen en gris y no son clickeables

## Estados de una cotizacion

Una cotizacion pasa por estos estados:

```
Pendiente -> Aprobada -> Firmada
                      -> Rechazada
                      -> Firmada y con addendum
                      -> Cancelada
```

## Addendums

Las cotizaciones firmadas pueden recibir modificaciones (addendums). El sistema registra la fecha y los cambios realizados, y mantiene un historial completo.
