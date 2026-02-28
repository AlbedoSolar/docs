# 7. Calculadora KPI

## Qué es

La Calculadora KPI es la herramienta de modelado financiero de Albedo. Calcula indicadores clave de rendimiento para escenarios de financiamiento solar a 25 años.

## Acceso

La página `/loan-kpi-calculator` tiene tres tabs:

| Tab | Descripción | Acceso |
|-----|-------------|--------|
| Quote Wizard | Asistente paso a paso (ver [Cotizaciones](06-cotizaciones.md)) | Todos |
| Calculadora | Análisis manual de escenarios | Todos |
| Admin Calculator | Opciones avanzadas de configuración | Solo admin |

## Datos de entrada

El formulario de la calculadora (`LoanKPILoanInformation`) recibe:

- **Plazo del préstamo** — Duración en meses
- **Tasa de interés** — APR anual
- **Período de gracia** — Meses sin pago de principal
- **Seguro** — Costo anual o cálculo automático
- **Mantenimiento** — Costo anual o cálculo automático
- **Costos de activos** — Valor de los equipos
- **Porcentaje de markup** — Margen sobre costo
- **WACC** — Costo promedio ponderado de capital

## Escenarios de análisis

La calculadora evalúa 4 escenarios simultáneamente:

1. **Leasing** — Solo arrendamiento base
2. **Leasing con Deuda** — Arrendamiento incluyendo servicio de deuda
3. **Con Servicios** — Incluye seguros y mantenimiento
4. **Total con IVA** — Monto total incluyendo impuestos

## Resultados (KPIs)

Para cada escenario se calculan:

| KPI | Descripción |
|-----|-------------|
| Pago mensual | Cuota mensual del cliente |
| VPN (NPV) | Valor presente neto de los flujos |
| TIR (IRR) | Tasa interna de retorno |
| Período de recuperación | Meses hasta recuperar la inversión |
| Total pagado | Suma total de pagos del cliente |

## Tabla de amortización

El componente `AmortizationSummary` muestra el cronograma de pagos a 25 años:

- Capital e intereses por período
- Costos de seguro y mantenimiento con inflación
- Saldo pendiente
- Acumulados por año

## Modos de cálculo

### Modo Referencia de Estimación
Usa los datos de una estimación existente como base. Los valores se pre-cargan automáticamente desde el proveedor y la configuración del proyecto.

### Modo Manual
El usuario ingresa todos los parámetros libremente para hacer simulaciones sin necesidad de una estimación previa.

## Backend de cálculo

Los cálculos pesados se ejecutan en un servicio Python Lambda:
- Endpoint: `/calculate/estimates`
- Endpoint: `/calculate/retail-price`
- La comunicación se hace a través de `estimate-calculations-service.ts`

## Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/LoanKPICalculatorPage.tsx` |
| Formulario | `components/molecules/LoanKPICalculatorForm.tsx` |
| Tabla KPI | `components/organisms/KPITable.tsx` |
| Amortización | `components/organisms/AmortizationSummary.tsx` |
| Admin Form | `components/organisms/AdminKPICalculatorForm.tsx` |
| Slice Redux | `features/kpi-calculator/kpiCalculatorSlice.ts` |
| Servicio | `services/estimate-calculations-service.ts` |
