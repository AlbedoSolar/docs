# 10. Administración

## Configuración de cotizaciones

**Ruta:** `/quote-config-admin`

Permite configurar los parámetros globales que afectan el cálculo de cotizaciones:

- Tasas por defecto (interés, seguro, mantenimiento)
- Márgenes y markup
- Parámetros de cálculo financiero
- Valores por defecto del formulario

El formulario `QuoteConfigForm` guarda estos valores en la tabla de configuración de Supabase.

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/QuoteConfigAdminPage.tsx` |
| Contenido | `components/organisms/QuoteConfigAdminContent.tsx` |
| Formulario | `components/organisms/QuoteConfigForm.tsx` |
| Slice Redux | `features/quote-config/quoteConfigSlice.ts` |

---

## Tipos de comisión

**Ruta:** `/commission-types`

Gestión de los diferentes tipos de comisión que se aplican en las cotizaciones:

- Porcentajes de comisión
- Condiciones de aplicación
- Cronogramas de pago de comisiones

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/CommissionTypesPage.tsx` |
| Formulario | `components/organisms/CommissionTypeForm.tsx` |
| Tabla | `components/organisms/CommissionTypesTable.tsx` |
| Slice Redux | `features/commission-types/commissionTypesSlice.ts` |

---

## Códigos de usuario

**Ruta:** `/user-codes`

Gestión de códigos promocionales asignados a usuarios:

- Creación y asignación de códigos
- Vista de detalle: `/user-codes/:user_code_id`

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/UserCodesPage.tsx` |
| Detalles | `pages/UserCodeDetailsPage.tsx` |
| Slice Redux | `features/user-codes/userCodesSlice.ts` |

---

## Actividad de usuarios

**Ruta:** `/user-activity` (acceso restringido)

Log de todas las acciones realizadas por los usuarios en la aplicación. Útil para auditoría y seguimiento.

- Registra automáticamente acciones de CRUD
- Muestra usuario, acción, entidad y timestamp
- Servicio: `activity-logging-service.ts`

---

## Log de cambios en base de datos

**Ruta:** `/database-change-log` (acceso restringido)

Registro de auditoría de todos los cambios en la base de datos:

- Cambios en registros con valores anteriores y nuevos
- Filtrado por tabla, usuario y fecha
- Modal de detalle para ver el cambio completo (`ChangeLogDetailModal`)

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/DatabaseChangeLogPage.tsx` |
| Slice Redux | `features/database-change-log/databaseChangeLogSlice.ts` |
