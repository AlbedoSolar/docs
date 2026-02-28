# 7. Calculadora KPI

## Qué es

La Calculadora KPI es la herramienta de modelado financiero de Albedo. Calcula indicadores clave de rendimiento para escenarios de financiamiento solar a 25 años.

## Tabs disponibles

| Tab | Descripción | Acceso |
|-----|-------------|--------|
| Quote Wizard | Asistente paso a paso (ver [Cotizaciones](06-cotizaciones.md)) | Todos |
| Calculadora | Análisis manual de escenarios | Todos |
| Admin Calculator | Opciones avanzadas de configuración | Solo administradores |

## Datos de entrada

El formulario de la calculadora recibe:

- **Plazo del préstamo** — Duración en meses
- **Tasa de interés** — APR anual
- **Período de gracia** — Meses sin pago de principal
- **Seguro** — Costo anual (manual o automático)
- **Mantenimiento** — Costo anual (manual o automático)
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

Para cada escenario se muestran:

| KPI | Descripción |
|-----|-------------|
| Pago mensual | Cuota mensual del cliente |
| VPN (NPV) | Valor presente neto de los flujos |
| TIR (IRR) | Tasa interna de retorno |
| Período de recuperación | Meses hasta recuperar la inversión |
| Total pagado | Suma total de pagos del cliente |

## Tabla de amortización

Muestra el cronograma de pagos a 25 años con:

- Capital e intereses por período
- Costos de seguro y mantenimiento (con inflación)
- Saldo pendiente
- Acumulados por año

## Modos de cálculo

### Modo Referencia de Estimación
Usa los datos de una estimación existente como base. Los valores se cargan automáticamente desde el proveedor y la configuración del proyecto.

### Modo Manual
Puedes ingresar todos los parámetros libremente para hacer simulaciones sin necesidad de una estimación previa.
