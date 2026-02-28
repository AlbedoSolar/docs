# 8. Sistemas y Sitios

## Sitios

Los sitios representan las ubicaciones físicas donde se instalan los sistemas solares.

### Funcionalidades

- **Lista:** `/sites` — Tabla con todos los sitios
- **Detalle:** `/sites/:site_id` — Información completa del sitio
- **CRUD:** Crear, editar y eliminar sitios

### Relación con otras entidades

Un sitio se asocia a un proyecto y puede contener uno o más sistemas solares.

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/SitesPage.tsx` |
| Detalles | `pages/SiteDetailsPage.tsx` |
| Slice Redux | `features/sites/sitesSlice.ts` |

## Sistemas

Los sistemas representan las instalaciones solares concretas con sus especificaciones técnicas.

### Funcionalidades

- **Lista:** `/sistemas` — Tabla con todos los sistemas
- **Detalle:** `/sistemas/:system_id` — Especificaciones completas
- **CRUD:** Crear, editar y eliminar sistemas con formulario completo

### Datos del sistema

- Capacidad instalada (kW)
- Tipo y especificaciones del inversor
- Tipo y capacidad de batería
- Cantidad y tipo de paneles
- Credenciales de monitoreo
- Proyecto y sitio asociados

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/SistemasPage.tsx` |
| Detalles | `pages/SistemaDetailsPage.tsx` |
| Formulario | `components/organisms/SystemForm.tsx` |
| Tabla | `components/organisms/SystemsTable.tsx` |
| Slice Redux | `features/systems/systemsSlice.ts` |

> **Nota:** Esta funcionalidad fue agregada recientemente (febrero 2026) y está en desarrollo activo.
