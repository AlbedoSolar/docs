# Reunión ADA — Resumen Ejecutivo

**Fecha**: Miércoles 8 de Abril 2026, 8:00 AM
**Convocada por**: Nausica Fiorelli (ADA)
**Doc completo**: [Reunión ADA - Alineación de Reportes de Impacto](https://docs.google.com/document/d/1XuqfcmLhwy7-5XjrnmPQ2B0eymwn2jmf_fMPiPzaAcU/edit)

---

## ¿De qué trata la reunión?

ADA combinó dos conversaciones que tenían pendientes con nosotros:

1. **Cifras del reporte trimestral inconsistentes** — Nausica detectó tres problemas concretos comparando los reportes Q3 y Q4 que les hemos enviado.
2. **Metodología de clasificación rural** — Alex propuso un cambio de metodología (proxy administrativo → GPS-based con GHS-SMOD) y Smiti respondió.

## Los temas que vamos a discutir

### 1. Clasificación rural (el más estratégico)

- **Hoy** clasificamos como rural cualquier proyecto fuera de cabeceras municipales y departamentales. Ejemplo del problema: **Santa Cruz La Laguna (~3,000 hab)** queda como "no rural" por ser cabecera municipal.
- **Alex propuso** pasar a clasificación basada en GPS usando el dataset GHS-SMOD del European Commission (estándar UN, alineado con IRIS+, IFC, SDGs).
- **Smiti (ADA) respondió** endosando el enfoque, recomendando que se podría destacar como **USP** de Albedo, y sugiriendo que **antes de migrar** hagamos un assessment comparativo en una muestra del portafolio donde sí tengamos GPS.
- **Buen alineamiento**: nuestro propio documento técnico recomienda exactamente la misma estructura de dos capas que sugirió Smiti (GPS primario + administrativo como fallback).
- **Bloqueador real**: capturar GPS para proyectos nuevos (¿al cotizar? ¿al instalar? ¿requerirlo a socios solares?) y backfill de los 269 proyectos firmados existentes.

### 2. Cifras de CO2 cambiantes entre versiones

Nausica notó que las cifras de CO2 del reporte Q3 no coinciden con las del reporte Q4 para los mismos trimestres históricos. La causa real: **estamos limpiando los datos en la migración de Quickbase a Supabase y el método de cálculo cambió**. Necesitamos:
- Confirmar las cifras definitivas y mandárselas
- Acordar un proceso para destacar cambios retroactivos (color, comentario, etc.) o congelar cifras históricas
- Validar los factores de emisión que usamos (GT 0.630, SV 0.410, HN 0.520)

### 3. Inconsistencia de unidades en el conteo (línea 26 vs línea 32)

Nausica detectó que: *"el número total de clientes de la línea 26 (leasing activo + venta al contado) es **menor** que el número de clientes con leasing activo (línea 32)"*. **Probablemente no es un bug** — más bien las dos líneas se cuentan en unidades diferentes (una en clientes únicos, otra en proyectos), entonces el "total" parece menor cuando se compara contra un componente que cuenta proyectos. Necesitamos el archivo exacto del reporte para confirmar y, sobre todo, **alinear con ADA qué unidad usa cada fila** (cliente vs proyecto).

### 4. Categorización de género de clientes

Stephanie le dijo a Nausica que tenemos "144 clientes catalogados como male and female" y propuso agregar una fila **"otros"** para los demás (sociedades sin género). **Salvedad**: revisé la BD y no existe ningún campo de género en la tabla `clients` (ni en otra tabla). Los 144 probablemente vienen de una clasificación manual que Stephanie hace al armar el reporte. Hay que confirmar la fuente con Stephanie antes de proponer una solución técnica.

### 5. Métricas faltantes en el reporte trimestral

Hoy el reporte trimestral solo tiene conteos demográficos. Las métricas de kW/kWh/CO2/ahorros existen pero no por trimestre. Hay que decidir si ADA las necesita desglosadas.

### 6. (Interno) Repensar el nivel granular de cada categoría de impacto

Hoy las 6 categorías viven en `projects`, pero conceptualmente son propiedades de niveles distintos:

- **Cliente**: `women_led`, `youth_led`, `non_profit` (propiedades del liderazgo / estatus legal de la organización)
- **Sitio**: `educational_institution` (propiedad del lugar físico)
- **Municipio**: `rural_area`, `impoverished_area` (ya derivados ahí ✅)

**Por qué importa**: hoy hay 3 clientes con proyectos que tienen valores **distintos** de `women_led` (imposible si lo pensamos como liderazgo). El "9-client gap" del conteo de mujeres viene exactamente de esta inconsistencia. Y el conflicto "clientes vs proyectos" que ADA flagueó se resuelve por construcción si cada categoría se cuenta en su unidad natural.

**Implicaciones**: cambio de schema (clients y sites), frontend (mover formularios), dbt (rewrite de los marts), y formato del reporte. Es trabajo, pero elimina varios bugs estructurales y conecta directamente con la captura de GPS al sitio (la tabla `sites` ya tiene columnas lat/long).

**Plan**: discusión interna primero, después decidir si y cómo presentárselo a ADA. No es para meter al meeting con ADA esta vez.

---

## Foto del portafolio (datos al día de hoy)

| Métrica | Valor |
|---|---|
| Proyectos firmados | 269 |
| Clientes únicos | 184 |
| Proyectos financiados | 206 (77%) |
| Proyectos de Alto Impacto Social | 120 (45%) |

---

## Lo que ya hicimos antes de la reunión (no requiere discusión)

- **Migramos `impoverished_area`** a derivarse del municipio (regla del 20% más pobre, alineada con ADA). Aplica para Guatemala; HN/SV no tienen datos municipales todavía.
- **Extendimos `mart_impact_quarterly`** hasta Q2 2026, con nuevas filas para Non-Profit y "High Social Impact" (composito).
- **Limpiamos los campos manuales** de las 6 categorías de impacto (NULLs → "Not Sure", corregido un proyecto con valor `'Si '`).
- **Bloqueamos `rural_area` y `impoverished_area` en el formulario de operaciones** (son derivados ahora) y arreglamos un bug donde no se podían editar las otras 4 categorías de impacto.

---

## Decisiones que necesitamos cerrar en la reunión

1. Plan operacional para capturar GPS en proyectos nuevos (cotización, instalación, socios solares)
2. Plan de backfill para los 269 proyectos existentes
3. Proceso para manejar cambios retroactivos en el reporte (destacar en color vs versionar)
4. Validar factores de emisión CO2 y supuestos de vida útil/degradación
5. Aprobar la fila "otros" para los clientes que son sociedades
6. Alinear definiciones de unidades por fila del reporte (cliente vs proyecto) — esto resuelve la "inconsistencia" línea 26 vs 32

---

## Acciones técnicas inmediatas (independientes de la reunión)

- Conseguir el archivo exacto del reporte trimestral que ADA recibe
- **Confirmar con Stephanie de dónde sale la clasificación de género de clientes** (¿Excel manual? ¿QuickBase? ¿otra fuente?)
- Correr el assessment GPS vs administrativo en una muestra del portafolio (lo que pidió Smiti)
- Descargar el raster GHS-SMOD de Centro América (~50MB) y montar un script de lookup en Python con `rasterio`
