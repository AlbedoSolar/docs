# Plan de Desarrollo — Solarbase
## Plataforma Tecnológica de Albedo Solar

---

## Visión

Solarbase es la plataforma centralizada de Albedo para la gestión de arrendamientos solares — desde la cotización inicial hasta el monitoreo de cartera. Reemplaza múltiples herramientas fragmentadas (QuickBase, hojas de cálculo, procesos manuales) con un sistema integrado y escalable.

**Objetivo:** Un flujo completamente digital donde un cliente puede recibir una cotización personalizada en minutos, firmar un contrato, y tanto Albedo como el cliente pueden monitorear pagos, producción solar e impacto social en tiempo real.

---

## Módulos Principales

### 1. Cotización y Originación
Herramienta para el equipo de ventas que genera cotizaciones de arrendamiento solar con cálculos financieros automatizados.

- Asistente de cotización paso a paso (cliente → proyecto → estimado → cotización)
- Generación automática de opciones de financiamiento (diferentes plazos, enganches, tasas)
- Página de oferta compartible con el cliente vía WhatsApp/correo
- Cálculo automático de TIR, cuota mensual, flujos de caja
- Manejo multi-moneda (GTQ, USD, HNL) y multi-país (Guatemala, Honduras, El Salvador)

### 2. Gestión de Operaciones
Panel de control para el equipo de operaciones que administra proyectos activos desde la firma hasta la instalación y más allá.

- Vista de portafolio con filtros por estado, país, tipo de instalación
- Seguimiento de hitos: firma → pre-instalación → instalación → conexión a red → monitoreo
- Información de equipos, proveedores, fases del proyecto
- Integración con facturación para ver estado de pagos por proyecto

### 3. Reportes Financieros y de Cartera
Reportes automatizados para análisis de cartera, due diligence de inversionistas, y gestión de mora.

- **Loan Tape:** Estado actual de cada arrendamiento — saldos, morosidad, DPD, clasificación de riesgo
- **Payment Tape:** Historial detallado de pagos recibidos por proyecto
- **Snapshots Mensuales:** Evolución de la cartera mes a mes (análisis de vintage y cohortes)
- Integración con Zoho Books (Guatemala) y sistemas contables locales (Honduras, El Salvador)

### 4. Reportes de Impacto
Métricas de impacto social y ambiental para reportes a donantes e inversionistas.

- Indicadores por proyecto: liderazgo femenino, área rural, zona empobrecida, juventud, sector educativo, sin fines de lucro
- Ahorro solar proyectado y real por proyecto
- Evitación de CO₂
- Reportes agregados por país, departamento, periodo

### 5. Integraciones
Conexiones bidireccionales con los sistemas existentes de Albedo.

- **Zoho Books** — Sincronización de facturas y pagos (facturación ↔ cartera)
- **QuickBase** — Importación de proyectos firmados (en transición hacia retiro)
- **HubSpot** — CRM y gestión de leads
- **Asana** — Flujos de trabajo operativos
- **Google Drive** — Almacenamiento de documentos contractuales

---

## Línea de Tiempo

### Fase 1: Originación + Cartera (Abril – Junio 2026)

| Entregable | Descripción | Meta |
|---|---|---|
| Cotización para ventas | Lanzar herramienta de cotización al equipo de ventas principal | Mayo 2026 |
| Sincronización Zoho automatizada | Facturas y pagos fluyen automáticamente de Zoho a Solarbase | Junio 2026 |
| Reportes de cartera (Debita) | Loan tape, payment tape, y snapshots mensuales operativos para GT + HN | Completado |
| Reportes de impacto | Métricas de impacto social y ambiental por proyecto y agregadas | Mayo 2026 |

### Fase 2: Operaciones + Consolidación (Julio – Septiembre 2026)

| Entregable | Descripción | Meta |
|---|---|---|
| Panel de operaciones | Flujos de trabajo de instalación y post-instalación en Solarbase | Agosto 2026 |
| Comisiones y márgenes | Cálculo automático de comisiones por cotización | Agosto 2026 |
| Migración de QuickBase | Período paralelo, migración de datos, capacitación | Septiembre 2026 |
| Generación de PDFs | Cotizaciones y contratos descargables | Septiembre 2026 |

### Fase 3: Inteligencia + Escala (Octubre 2026 – Marzo 2027)

| Entregable | Descripción | Meta |
|---|---|---|
| Portal de clientes | Clientes ven su cronograma de pagos, producción solar, ahorro | Q4 2026 |
| Monitoreo predictivo | Detección temprana de riesgo de mora basada en patrones de pago | Q4 2026 |
| Extracción de datos de facturas de luz | IA extrae datos de fotos de facturas para cotización automática | Q1 2027 |
| Retiro de QuickBase | Descomisión completa | Q4 2026 |

---

## Cobertura Geográfica

| País | Cotización | Facturación | Reportes de Cartera | Estado |
|---|---|---|---|---|
| **Guatemala** | ✅ | Zoho Books | ✅ Operativo | Sistema principal |
| **Honduras** | ✅ | Sistema contable local | ✅ Operativo | Datos integrados |
| **El Salvador** | ✅ | Sistema contable local | 🔜 En proceso | 4 proyectos activos |

---

## Arquitectura Técnica (resumen)

- **Frontend:** Aplicación web React (accesible desde cualquier navegador)
- **Backend:** Supabase (PostgreSQL) como fuente única de verdad
- **Integraciones:** AWS Lambda para conexiones con Zoho, QuickBase, HubSpot
- **Reportes:** dbt para transformaciones de datos, vistas materializadas para dashboards
- **Infraestructura:** AWS + Supabase, despliegue automatizado via GitHub Actions
- **Seguridad:** Autenticación con controles por rol, row-level security en base de datos, secretos encriptados
