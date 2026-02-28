# 1. Visión General

## Qué es Quotes App

Quotes App es la aplicación interna de Albedo para crear cotizaciones de financiamiento de paneles solares. Permite gestionar clientes, proyectos, estimaciones y cotizaciones, así como ejecutar modelos financieros a 25 años.

## Tecnologías principales

| Tecnología | Versión | Uso |
|------------|---------|-----|
| React | 19 | Interfaz de usuario |
| TypeScript | 5.8 | Tipado estático |
| Vite | 7.1 | Build y dev server |
| Redux Toolkit | 2.8 | Estado global |
| React Router | 7.8 | Navegación y rutas |
| React Hook Form + Zod | 7.x / 4.x | Formularios y validación |
| Tailwind CSS | 4.1 | Estilos |
| Supabase | — | Autenticación y base de datos |

## Estructura del proyecto

```
frontend/quotes-app/src/
├── app/              # Store de Redux y hooks globales
├── assets/           # Logos e imágenes
├── components/
│   ├── atoms/        # Botones, inputs, loader, iconos
│   ├── molecules/    # Campos de formulario, links de nav, tabs
│   ├── organisms/    # Formularios, tablas, modales, Quote Wizard
│   └── layouts/      # MainLayout (sidebar + header) y AuthLayout
├── features/         # 21 slices de Redux (uno por entidad)
├── hooks/            # Hooks personalizados (traducción, roles, etc.)
├── pages/            # Componentes de página (uno por ruta)
├── routes/           # Router y guards de protección
├── services/         # Comunicación con backend (Supabase, Lambda)
├── types/            # Tipos compartidos
└── utils/            # Helpers, constantes, i18n
```

## Patrones de diseño

**Atomic Design** — Los componentes están organizados en átomos (botón, input), moléculas (campo de formulario, link de navegación) y organismos (formularios completos, tablas, wizard).

**Feature Slices** — Cada entidad del negocio (proyectos, clientes, cotizaciones, etc.) tiene su propio directorio bajo `features/` con:
- `interface.ts` — tipos TypeScript
- `Slice.ts` — estado, reducers
- `AsyncThunk.ts` — llamadas async al backend
- `Selectors.ts` — selectores de datos

**Service Factory** — Un servicio de datos abstracto (`data-service.interface.ts`) con implementación concreta en Supabase (`supabase-data-service.ts`), lo que permite cambiar de backend sin tocar componentes.

## Idiomas

La app soporta español e inglés. Las traducciones están en `utils/language/es.json` y `en.json`. Se usa el hook `useTranslation()` que lee el idioma actual del store de Redux.

## Despliegue

- **Build:** `npm run build` (TypeScript + Vite)
- **Dev:** `npm run dev`
- **Hosting:** Vercel
