# 11. Integraciones

## Supabase

Supabase es el backend principal de la aplicación. Se usa para:

### Autenticación
- Login con email/contraseña
- Persistencia de sesión en el navegador
- JWT para llamadas protegidas

### Base de datos
- Todas las operaciones CRUD pasan por `supabase-data-service.ts`
- Row-Level Security (RLS) para control de acceso a nivel de fila
- Soporte para suscripciones en tiempo real

### Configuración
Las credenciales de Supabase se configuran en variables de entorno:
- `VITE_SUPABASE_URL`
- `VITE_SUPABASE_ANON_KEY`

---

## API Lambda (Python)

Un servicio Python en AWS Lambda maneja los cálculos financieros complejos:

| Endpoint | Servicio | Descripción |
|----------|---------|-------------|
| `/calculate/estimates` | `estimate-calculations-service.ts` | Cálculo completo de KPIs y amortización |
| `/calculate/retail-price` | `estimate-calculations-service.ts` | Cálculo de precio retail |

### Comunicación
- Las llamadas usan `apiFetchProtected()` de `utils/api.ts`
- Inyecta automáticamente el Bearer token desde el store de Redux
- URL base configurada en `VITE_MAIN_API_BASE_URL`

---

## Zoho Books

Integración con Zoho Books para sincronizar proyectos y flujos de caja.

### Autenticación OAuth
1. El usuario hace clic en "Login Zoho" en el sidebar
2. Se redirige a Zoho para autorización
3. Callback en `/oauth/callback` recibe el código
4. Se intercambia por tokens de acceso/refresh

### Funcionalidades
- Crear proyectos en Zoho Books
- Crear registros de flujo de caja
- Sincronizar estados de proyecto

### Estado en la UI
Indicadores visuales en el sidebar muestran si el usuario está conectado a Zoho (`ZohoProjectStatus` components).

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Callback | `pages/OAuthCallbackPage.tsx` |
| Botón login | `components/molecules/LoginZohoButton.tsx` |
| Slice Redux | `features/zoho/zohoSlice.ts` |

---

## Servicios adicionales

| Servicio | Archivo | Descripción |
|----------|---------|-------------|
| Tasas de cotización | `quote-rates-service.ts` | Cálculo de tasas de interés |
| Moneda | `currency-service.ts` | Conversión de tipos de cambio |
| Addendums | `addendum-service.ts` | Gestión de modificaciones a cotizaciones |
| Timeline | `pipeline-timeline-service.ts` | Timeline del workflow de proyectos |
| Actividad | `activity-logging-service.ts` | Registro de acciones de usuario |
