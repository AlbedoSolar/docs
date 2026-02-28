# 6. Cotizaciones y Quote Wizard

## Quote Wizard

El Quote Wizard es el asistente principal para generar cotizaciones. Te guía en 4 pasos:

### Paso 1 — Cliente

- Selecciona un cliente existente de la lista
- O crea un cliente nuevo directamente aquí
- Al seleccionar, se avanza al paso 2

### Paso 2 — Proyecto

- Selecciona un proyecto existente del cliente
- O crea un proyecto nuevo
- Define: moneda, sector, ubicación

### Paso 3 — Estimación

- Selecciona una estimación existente del proyecto
- O crea una nueva con el desglose de costos:
  - Costo del sistema
  - Impuesto solar
  - Precio retail (con margen)
  - Costos legales
  - Comisiones
  - Valor de transferencia de activos

### Paso 4 — Cotización

- Configura los términos financieros:
  - Plazo del préstamo
  - Tasa de interés
  - Período de gracia
  - Seguros y mantenimiento
  - Porcentajes de markup y WACC
- Calcula los KPIs financieros
- Previsualiza el flujo de caja
- Guarda la cotización

## Estados de una cotización

Una cotización pasa por estos estados:

```
Pendiente → Aprobada → Firmada → Activa
                     → Rechazada
                     → Firmada y con addendum
                     → Cancelada
```

## Addendums

Las cotizaciones firmadas pueden recibir modificaciones (addendums). El sistema registra la fecha y los cambios realizados, y mantiene un historial completo.
