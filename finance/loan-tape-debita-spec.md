# Reportes de Cartera — Debita

## 4 Reportes

### 1. Loan Tape (`mart_financial_loan_tape`)

Una fila por arrendamiento. Estado actual de cada préstamo.

| Columna | Fuente |
|---------|--------|
| ID Préstamo | Solarbase: `quote_reference` |
| ID Deudor | Solarbase: `client_reference` |
| Nombre Deudor | Solarbase: `clients.legal_name` |
| Fecha de Originación | Solarbase: `contract_signing_date` |
| Fecha de Vencimiento | Solarbase: última fecha en cash flows |
| Monto Desembolsado (USD) | Solarbase: retail price − enganche |
| Saldo Vigente (USD) | Zoho: suma de balance de facturas no pagadas |
| Tasa de Interés Anual (%) | Solarbase: `annual_interest_rate` |
| Plazo Original (meses) | Solarbase: conteo de meses de pago |
| Plazo Remanente (meses) | Solarbase: meses con fecha futura |
| Cuota Mensual (USD) | Solarbase: mediana del pago mensual |
| DPD | Zoho: días desde la factura overdue más antigua |
| Cuotas en Mora | Zoho: conteo de facturas overdue |
| Estado del Crédito | Derivado de DPD: Vigente / Mora Temprana / En Mora / Default / Finalizado |
| Bucket de Mora | Derivado: Current / 1-30 / 31-60 / 61-90 / 90+ |
| Seasoning (meses) | Derivado: meses desde originación |
| Reestructurado | Solarbase: si tiene adenda |
| País | Solarbase |
| kW Instalados | Solarbase: capacidad del sistema |
| Moneda Original | Solarbase: GTQ / USD / HNL |
| Total Pagado (USD) | Zoho: suma de pagos recibidos |

### 2. Payment Tape (`mart_financial_payment_tape`)

Una fila por pago recibido.

| Columna | Fuente |
|---------|--------|
| ID Préstamo | Zoho → `invoice_project_map` → Solarbase |
| Fecha Esperada | Solarbase: fecha de vencimiento del cash flow |
| Fecha Real de Pago | Zoho: `lease_payments.payment_date` |
| Monto Pagado (USD) | Zoho: `amount_usd` |
| Monto Aplicado | Zoho: `amount_applied` |
| DPD al Momento del Pago | Derivado: fecha real − fecha esperada |
| Modo de Pago | Zoho: transferencia / efectivo |
| # Factura | Zoho: `zoho_invoice_number` |

### 3. Snapshots Mensuales (`mart_financial_monthly_snapshot`)

Misma estructura que loan tape, pero una fila por arrendamiento por mes (últimos 24 meses). Todos los campos calculados al cierre del mes.

### 4. Seguros y Garantías (`mart_financial_insurance`)

Una fila por arrendamiento asegurado.

| Columna | Fuente |
|---------|--------|
| ID Préstamo | Solarbase |
| Aseguradora | Solarbase: `insurers.name` |
| # Póliza | Solarbase |
| # Certificado | Solarbase |
| Estado de Cobertura | Solarbase |
| Vigencia Inicio/Fin | **No disponible** |
| Beneficiario | **No disponible** |
| Monto Seguro Total (USD) | Solarbase: suma de pagos de seguro en cash flows |

---

## Fuentes de Datos

**Solarbase (Supabase):** Datos de contrato, equipo, seguros, cash flows programados.

**Zoho Books:** Pagos reales recibidos. Importados a Supabase en dos tablas:
- `lease_payments` — cada pago con fecha, monto, factura vinculada
- `invoice_project_map` — mapeo de factura Zoho → proyecto Solarbase

## Vinculación Zoho → Solarbase

Los pagos de Zoho se vinculan a proyectos de Solarbase a través de las facturas:

```
Pago → Invoice Number → Factura → Project Name → Proyecto Solarbase
```

El mapeo requiere trabajo manual/semi-automático para ~200 proyectos distintos.
