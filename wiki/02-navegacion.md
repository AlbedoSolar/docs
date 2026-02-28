# 2. Navegación

## Layout principal

La aplicación usa `MainLayout` como estructura base para todas las páginas protegidas. Consiste en:

- **Sidebar izquierdo** — Menú de navegación con secciones colapsables
- **Header superior** — Título de la página actual, barra de búsqueda de proyectos y toggle de idioma
- **Área de contenido** — Donde se renderiza la página activa

## Secciones del menú

### Ventas
| Ruta | Página | Descripción |
|------|--------|-------------|
| `/loan-kpi-calculator` | Quote Wizard / Calculadora | Crear cotizaciones y calcular KPIs |
| `/projects/active` o `/sales` | Proyectos Activos | Pipeline de ventas |
| `/clients` | Clientes | Lista y gestión de clientes |

### Finanzas
| Ruta | Página | Descripción |
|------|--------|-------------|
| `/projects/signed` o `/finances` | Proyectos Firmados | Proyectos completados y sus flujos de caja |

### Operaciones
| Ruta | Página | Descripción |
|------|--------|-------------|
| `/projects` | Todos los Proyectos | Lista completa con filtros |
| `/sistemas` | Sistemas | Gestión de sistemas solares |
| `/sites` | Sitios | Sitios de instalación |
| `/equipments` | Equipos | Catálogo de equipos y marcas |
| `/providers` | Proveedores | Gestión de proveedores |
| `/affiliates` | Afiliados | Gestión de afiliados |
| `/commission-types` | Tipos de Comisión | Configuración de comisiones |
| `/user-codes` | Códigos de Usuario | Códigos promocionales |

### Reportes
| Ruta | Página | Acceso |
|------|--------|--------|
| `/reports` | Reportes | Solo usuarios con rol `view-reports` o `admin` |

### Administración
| Ruta | Página | Descripción |
|------|--------|-------------|
| `/quote-config-admin` | Config de Cotizaciones | Parámetros globales de cálculo |
| `/user-activity` | Actividad de Usuarios | Log de acciones (acceso restringido) |
| `/database-change-log` | Log de Cambios en BD | Auditoría de cambios (acceso restringido) |

## Búsqueda de proyectos

El header incluye una barra de búsqueda (`ProjectSearchBar`) que permite buscar proyectos por referencia desde cualquier página de la app.

## Toggle de idioma

El componente `LanguageToggle` en el header permite cambiar entre español e inglés. El idioma seleccionado se guarda en el store de Redux y afecta todas las etiquetas de la interfaz.
