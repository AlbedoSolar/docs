# 6. Cotizaciones y Quote Wizard

## Quote Wizard

El Quote Wizard es el flujo principal para generar cotizaciones. Es un asistente de 4 pasos accesible desde `/loan-kpi-calculator`.

### Paso 1 — Cliente

- Seleccionar un cliente existente de la lista
- O crear un cliente nuevo directamente en el wizard
- Al seleccionar, se avanza al paso 2

### Paso 2 — Proyecto

- Seleccionar un proyecto existente del cliente
- O crear un proyecto nuevo
- Se definen: moneda, sector, ubicación

### Paso 3 — Estimación

- Seleccionar una estimación existente del proyecto
- O crear una nueva con el desglose de costos:
  - Costo del sistema
  - Impuesto solar
  - Precio retail (con margen)
  - Costos legales
  - Comisiones
  - Valor de transferencia de activos

### Paso 4 — Cotización

- Configurar los términos financieros:
  - Plazo del préstamo
  - Tasa de interés
  - Período de gracia
  - Seguros y mantenimiento
  - Porcentajes de markup y WACC
- Calcular KPIs financieros
- Previsualizar flujo de caja
- Guardar la cotización

## Navegación por pasos

Cada paso tiene su URL: `/loan-kpi-calculator/1`, `/loan-kpi-calculator/2`, etc. El estado del wizard se mantiene en Redux y persiste entre pasos.

## Estados de una cotización

```
pending → approved → signed → active
                  → rejected
                  → signed_and_addended
                  → cancelled
```

## Addendums

Las cotizaciones firmadas pueden recibir addendums (modificaciones). El sistema trackea:
- Fecha del addendum
- Cambios realizados
- Historial de estados

## Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/LoanKPICalculatorPage.tsx` |
| Wizard | `components/organisms/QuoteWizard.tsx` |
| Paso 1 | `components/organisms/QuoteWizardStep1Client.tsx` |
| Paso 2 | `components/organisms/QuoteWizardStep2Project.tsx` |
| Paso 3 | `components/organisms/QuoteWizardStep3Estimate.tsx` |
| Paso 4 | `components/organisms/QuoteWizardStep4Quote.tsx` |
| Info financiera | `components/organisms/QuoteFinancialInfo.tsx` |
| Preview flujo | `components/organisms/QuotePreviewCashFlow.tsx` |
| Slice Redux | `features/quotes/quotesSlice.ts` |
| Estimaciones | `features/estimates/estimatesSlice.ts` |

## Documentación técnica relacionada

Para detalles sobre las fórmulas de cálculo, flujos de datos e inconsistencias conocidas, ver la [documentación de generación de cotizaciones](../quote-generation/).
