# 5. Proyectos

## Vista general

Los proyectos son la entidad central del sistema. Representan una oportunidad de financiamiento solar vinculada a un cliente, con estimaciones, cotizaciones y flujos de caja.

## Vistas de proyectos

| Ruta | Vista | Descripción |
|------|-------|-------------|
| `/projects` | Todos los proyectos | Lista completa con filtros y CRUD |
| `/projects/active` | Proyectos activos | Pipeline de ventas |
| `/projects/signed` | Proyectos firmados | Proyectos con contrato firmado |
| `/projects/:project_reference` | Detalle | Información completa del proyecto |

## Flujo de estados (Workflow)

Un proyecto pasa por los siguientes estados:

```
draft
  → estimate_requested
    → estimate_created
      → quote_generated
        → quote_sent
          → due_diligence_in_process
            → due_diligence_completed
              → due_diligence_approved / due_diligence_rejected
                → signed
                  → active
```

## Página de detalles

La página de detalles del proyecto (`ProjectDetailsPage`) muestra:

- Información general del proyecto
- Detalles del contrato (`ContractDetails`)
- Flujo de caja mensual (`ActiveMonthlyCashFlow`)
- Historial de estados
- Acciones disponibles según el estado actual

## Proyectos activos (Pipeline)

La vista de proyectos activos (`ActiveProjectsPage`) funciona como un pipeline de ventas:

- Muestra proyectos en progreso organizados por estado
- Permite ver detalles expandidos de cada proyecto (`ActiveProjectDetails`)
- Incluye acciones rápidas según el estado

## CRUD de proyectos

- **Crear:** Formulario `ProjectForm` con datos del proyecto, moneda, sector, etc.
- **Editar:** Mismo formulario precargado con datos existentes
- **Eliminar:** Modal de confirmación (`DeleteProjectConfirmationModal`)
- **Buscar:** Barra de búsqueda global en el header

## Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Lista | `pages/ProjectsPage.tsx` |
| Activos | `pages/ActiveProjectsPage.tsx` |
| Firmados | `pages/SignedProjectsPage.tsx` |
| Detalles | `pages/ProjectDetailsPage.tsx` |
| Formulario | `components/organisms/ProjectForm.tsx` |
| Tabla | `components/organisms/ProjectsTable.tsx` |
| Búsqueda | `components/organisms/ProjectSearchBar.tsx` |
| Slice Redux | `features/projects/projectsSlice.ts` |

## Relación con otras entidades

- **Cliente** — Cada proyecto pertenece a un cliente
- **Estimaciones** — Un proyecto tiene una o más estimaciones de costos
- **Cotizaciones** — Cada estimación puede generar una cotización
- **Sistemas** — Los sistemas solares se asocian a un proyecto vía un sitio
- **Flujos de caja** — Los proyectos firmados generan cronogramas de pagos mensuales
