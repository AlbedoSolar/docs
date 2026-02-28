# 5. Proyectos

## Vista general

Los proyectos son la entidad central del sistema. Cada proyecto representa una oportunidad de financiamiento solar vinculada a un cliente.

## Vistas disponibles

| Vista | Descripción |
|-------|-------------|
| Todos los Proyectos | Lista completa con filtros y acciones CRUD |
| Proyectos Activos | Pipeline de ventas — proyectos en progreso organizados por estado |
| Proyectos Firmados | Proyectos con contrato firmado, con sus flujos de caja |
| Detalle del Proyecto | Información completa de un proyecto individual |

## Crear un proyecto

1. Haz clic en **Crear Proyecto**
2. Completa los datos: moneda, sector, ubicación, etc.
3. Haz clic en **Guardar**

También puedes crear proyectos directamente desde el **Paso 2** del Quote Wizard.

## Flujo de estados

Un proyecto pasa por los siguientes estados a lo largo de su vida:

```
Borrador
  → Estimación solicitada
    → Estimación creada
      → Cotización generada
        → Cotización enviada
          → Due diligence en proceso
            → Due diligence completado
              → Aprobado / Rechazado
                → Firmado
                  → Activo
```

## Página de detalle

Al hacer clic en un proyecto, puedes ver:

- Información general del proyecto
- Detalles del contrato
- Flujo de caja mensual
- Historial de estados
- Acciones disponibles según el estado actual

## Búsqueda

Usa la barra de búsqueda en la parte superior de la app para encontrar un proyecto por su referencia desde cualquier página.

## Relación con otras entidades

- Cada proyecto pertenece a un **cliente**
- Un proyecto tiene una o más **estimaciones** de costos
- Cada estimación puede generar una **cotización**
- Los **sistemas solares** se asocian a un proyecto a través de un **sitio**
- Los proyectos firmados generan cronogramas de **pagos mensuales**
