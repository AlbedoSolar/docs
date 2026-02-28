# 4. Clientes

## Vista general

La página de clientes (`/clients`) permite gestionar la base de clientes de Albedo. Cada cliente puede estar asociado a uno o más proyectos.

## Funcionalidades

### Lista de clientes
- Tabla con todos los clientes registrados
- Columnas con información básica del cliente
- Acciones de editar y eliminar por fila

### Crear cliente
- Formulario con validación via Zod
- Campos requeridos según la interfaz del cliente
- Al guardar, se crea el registro en Supabase y se actualiza la tabla

### Editar cliente
- Mismos campos que el formulario de creación
- Se carga la información existente al abrir

### Ver detalles
- Ruta: `/clients/:client_reference`
- Muestra toda la información del cliente
- Lista de proyectos asociados

## Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/ClientsPage.tsx` |
| Detalles | `pages/ClientDetailsPage.tsx` |
| Formulario | `components/organisms/ClientForm.tsx` |
| Tabla | `components/organisms/ClientsTable.tsx` |
| Slice Redux | `features/clients/clientsSlice.ts` |
| Async Thunks | `features/clients/clientsAsyncThunk.ts` |
| Interfaz | `features/clients/clients.interface.ts` |

## Relación con Quote Wizard

En el Paso 1 del Quote Wizard, el usuario puede seleccionar un cliente existente o crear uno nuevo directamente desde el asistente.
