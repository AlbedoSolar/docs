# 12. Roles y Permisos

## Resumen

La aplicación utiliza un sistema de roles para controlar qué secciones y funcionalidades están disponibles para cada usuario. Los roles se asignan a cada cuenta de usuario y determinan qué páginas ve en el menú lateral, qué acciones puede realizar, y qué datos puede modificar.

Un usuario **sin roles asignados** ve todas las secciones por defecto (como respaldo), pero no tiene acceso a funcionalidades restringidas como reportes o actividad de usuarios.

---

## Roles disponibles

| Rol | Descripción |
|-----|-------------|
| `sales` | Equipo comercial — gestión de clientes, proyectos en pipeline y cotizaciones |
| `finances` | Equipo financiero — seguimiento de proyectos firmados y flujos de caja |
| `operations` | Equipo de operaciones — gestión de proyectos activos, sistemas e instalaciones |
| `admin` | Administrador — acceso completo a todas las secciones y funcionalidades |
| `view-reports` | Acceso a la sección de reportes |
| `view-user-stats` | Acceso a actividad de usuarios y log de cambios en la base de datos |
| `quote-machine-admin` | Administración de la configuración del motor de cotizaciones |

Un usuario puede tener **múltiples roles** simultáneamente.

---

## Visibilidad del menú por rol

### Sales (`sales`)

Secciones visibles:
- **Ventas** — Clientes, Proyectos de Ventas, Calculadora de Cotizaciones
- **Gestión** — Sitios, Equipos, Afiliados, Proveedores, Tipos de Comisión, Códigos de Usuario

### Finances (`finances`)

Secciones visibles:
- **Finanzas** — Proyectos Firmados

### Operations (`operations`)

Secciones visibles:
- **Operaciones** — Proyectos de Operaciones, Sistemas
- **Gestión** — Sitios, Equipos, Afiliados, Proveedores, Tipos de Comisión, Códigos de Usuario

### Admin (`admin`)

Secciones visibles:
- **Todas las secciones** de Sales, Finances y Operations
- **Reportes** (equivalente a tener `view-reports`)

### Sin roles asignados

Si un usuario no tiene ningún rol asignado, ve **todas las secciones** como comportamiento por defecto. Sin embargo, no puede acceder a funcionalidades restringidas que requieren roles específicos.

---

## Permisos por funcionalidad

| Funcionalidad | Rol requerido |
|---------------|---------------|
| Ver y gestionar clientes | `sales` (o sin roles) |
| Ver pipeline de ventas | `sales` (o sin roles) |
| Crear cotizaciones (Quote Wizard) | `sales` (o sin roles) |
| Ver proyectos firmados y flujos de caja | `finances` (o sin roles) |
| Ver y editar proyectos de operaciones | `operations` (o sin roles) |
| Ver y gestionar sistemas | `operations` (o sin roles) |
| Ver reportes | `view-reports` o `admin` |
| Ver actividad de usuarios | `view-user-stats` |
| Ver log de cambios en base de datos | `view-user-stats` |
| Configurar motor de cotizaciones | `quote-machine-admin` |
| Gestión de entidades (sitios, equipos, proveedores, afiliados, comisiones, códigos) | `sales`, `operations` (o sin roles) |
| Crear/editar/eliminar proveedores | `admin` |

---

## Visibilidad de columnas en tablas

Algunas tablas ajustan las columnas visibles por defecto según el rol del usuario:

### Proyectos de Operaciones

| Rol | Columnas por defecto |
|-----|---------------------|
| `admin` / `operations` | Todas las columnas |
| `sales` / `finances` / sin roles | Subconjunto reducido (Ref, Cliente, País, Estado, fechas clave, seguro, proveedor eléctrico) |

El usuario puede personalizar las columnas visibles manualmente durante su sesión, independientemente de su rol.

---

## Protección de rutas

Las rutas protegidas redirigen al usuario a la página principal (`/projects`) si no tienen el rol requerido:

| Ruta | Rol requerido |
|------|---------------|
| `/reports` | `view-reports` o `admin` |
| `/user-activity` | `view-user-stats` |
| `/database-change-log` | `view-user-stats` |
| Todas las demás rutas | Solo autenticación (cualquier usuario) |
