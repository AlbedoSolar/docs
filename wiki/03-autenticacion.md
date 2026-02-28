# 3. Autenticación

## Flujo de login

1. El usuario accede a `/login` y ve el formulario de inicio de sesión
2. Ingresa email y contraseña
3. La app llama a `getToken()` que autentica contra Supabase
4. Si es exitoso, se guarda el token JWT y el objeto de usuario en el store de Redux
5. Supabase persiste la sesión en el almacenamiento local del navegador
6. El usuario es redirigido a la página principal

## Restauración de sesión

Cuando la app se carga, el componente `AuthInitializer` ejecuta `restoreSession()`:

- Intenta recuperar la sesión guardada en Supabase
- Si existe una sesión válida, restaura el estado de autenticación sin pedir login
- Si no hay sesión o expiró, redirige a `/login`

Las rutas protegidas muestran un loader mientras se completa este proceso.

## Niveles de protección

La app usa tres tipos de guards para las rutas:

| Guard | Archivo | Acceso |
|-------|---------|--------|
| `ProtectedRoute` | `routes/ProtectedRoute.tsx` | Cualquier usuario autenticado |
| `RoleProtectedRoute` | `routes/RoleProtectedRoute.tsx` | Usuarios con roles específicos (ej. `admin`, `view-reports`) |
| `UserProtectedRoute` | `routes/UserProtectedRoute.tsx` | Usuarios específicos por nombre |

## Estado de autenticación

El slice `auth` en Redux mantiene:

- `token` — JWT de acceso
- `user` — Objeto con email y array de roles
- `isAuthenticated` — Bandera booleana
- `sessionRestored` — Si ya se intentó restaurar la sesión
- `loading` — Operación en progreso
- `error` — Último mensaje de error

## Cerrar sesión

El botón de logout en el sidebar llama a `signOut()`, que limpia la sesión en Supabase y resetea el estado de Redux.
