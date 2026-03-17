# 7. Calculadora KPI

## Que es

La Calculadora KPI es la herramienta de modelado financiero de Albedo. Calcula indicadores clave de rendimiento para escenarios de financiamiento solar a 25 anios.

## Tabs disponibles

| Tab | Descripcion | Acceso |
|-----|-------------|--------|
| Quote Wizard | Asistente paso a paso (ver [Cotizaciones](06-cotizaciones.md)) | Todos |
| Calculator | Analisis manual de escenarios | Todos |
| Admin Calculator | Opciones avanzadas con overrides de configuracion y pricing | Solo administradores |
| Configuration | Tablas de configuracion (tasas, comisiones, pricing por ubicacion) | Solo administradores |

## Datos de entrada

El formulario de la calculadora recibe:

**Estructura del prestamo:**
- **Costo del activo** — Costo de instalacion en moneda del contrato
- **Plazo del prestamo** — Duracion en meses
- **Tasa de interes anual** — APR (puede ingresarse manualmente o resolverse con goal-seek)
- **Enganche** — Porcentaje del precio retail
- **Periodo de gracia** — Meses sin pago de principal
- **Costos legales** — Calculados segun precio retail y moneda
- **Opcion de compra** — Porcentaje para compra al final del lease (default 1%)

**Seguros:**
- **Porcentaje de seguro** — Obtenido de la ubicacion o ingresado manualmente
- **Tasa de deflacion del seguro** — Reduccion anual del costo (default 0.96 = 4% menos por anio)
- **Override manual de seguro** — Toggle para sobreescribir el porcentaje automatico

**Mantenimiento:**
- **Porcentaje por visita** — Obtenido del proveedor+ubicacion o editable directamente
- **Tasa de inflacion de mantenimiento** — Aumento anual del costo
- **Prima de mantenimiento** — Markup sobre el costo calculado

**Parametros financieros:**
- **Comisiones** — Calculadas del tipo de comision, o editables
- **WACC** — Costo promedio ponderado de capital
- **Tasa libre de riesgo** — Para calculo de NPV
- **Margen objetivo** — Margen deseado sobre el proyecto
- **Nivel de riesgo** — Low / Medium / High

**Ubicacion y proveedor (solo lectura en modo estimacion):**
- Pais, departamento, municipio, region
- Proveedor (tipo de margen y porcentaje de margen mayorista)

## Modos de calculo

### Modo Referencia de Estimacion
Usa los datos de una estimacion existente como base. Los valores se cargan automaticamente desde el proveedor y la configuracion del proyecto. Dentro de este modo hay tres sub-modos:

- **Manual** — El usuario define la tasa de interes directamente
- **Goal-seek TIR** — El sistema resuelve la tasa de interes necesaria para alcanzar un TIR objetivo
- **Goal-seek Pago Mensual** — El sistema resuelve la tasa para alcanzar un pago mensual objetivo

### Modo Manual
Puedes ingresar todos los parametros libremente para hacer simulaciones sin necesidad de una estimacion previa.

## Escenarios de analisis

La calculadora evalua 4 escenarios simultaneamente:

1. **Leasing** — Solo arrendamiento base
2. **Leasing con Deuda** — Arrendamiento incluyendo servicio de deuda (nota: en modo estimacion, este es identico a Leasing)
3. **Con Servicios** — Incluye seguros y mantenimiento
4. **Total con IVA** — Monto total incluyendo impuestos

## Resultados (KPIs)

Para cada escenario se muestran:

| KPI | Descripcion |
|-----|-------------|
| Pago mensual | Cuota mensual del cliente |
| VPN (NPV) | Valor presente neto de los flujos |
| TIR (IRR) | Tasa interna de retorno |
| Periodo de recuperacion | Meses hasta recuperar la inversion (9999 si no se recupera) |
| Total pagado | Suma total de pagos del cliente |

## Tabla de amortizacion

Muestra el cronograma de pagos a 25 anios con:

- Capital e intereses por periodo
- Costos de seguro y mantenimiento (con inflacion/deflacion)
- Saldo pendiente
- Acumulados por anio
