# Reunión ADA - Alineación de Reportes de Impacto

**Fecha**: Miércoles 8 de Abril 2026, 8:00 AM
**Convocada por**: Nausica Fiorelli (ADA)
**Asunto**: Albedo - Seguimiento monitoreo 2026

## Participantes
- **ADA**: Nausica Fiorelli, Smiti Sahoo
- **Albedo**: Alex Macfarlan, Fernanda Díaz Ospina, Stephanie Girón, Jake Leonardis
- **Valuing Impact**: Jessica Hammer, Cristina Renedo (probablemente)

---

## TL;DR — lo que tienes que saber antes de entrar

**La reunión cubre dos conversaciones que ADA inició por separado** y que decidieron combinar en este espacio:

1. **Cifras de impacto inconsistentes** (Nausica): el reporte trimestral que les enviamos tiene problemas concretos — las cifras de CO2 cambian entre versiones, hay una inconsistencia entre dos líneas del conteo (probablemente porque mezclan unidades cliente vs proyecto), y la categorización de género está incompleta.

2. **Metodología de clasificación rural** (Alex): Alex propuso pasar de un proxy administrativo a clasificación basada en GPS usando GHS-SMOD. Smiti respondió endosando el enfoque y pidiendo un assessment comparativo antes de migrar.

**Posiciones de cada lado en este momento**:

- **ADA está cómoda con el proxy administrativo** (es lo que reciben de la mayoría de sus inversiones), pero **endosó el enfoque GPS de Albedo** como mejor práctica y sugirió que se podría destacar como USP de Albedo.
- **Albedo ya tiene un documento técnico** (`rural_classification_northern_triangle.pages` en `external-resources/`) que recomienda exactamente el mismo enfoque de dos capas que sugiere Smiti: GPS + GHS-SMOD como primario, y proxy administrativo como interim.
- **El bloqueador real para hacer el cambio** no es la metodología — es operacional: capturar GPS en proyectos nuevos y backfill para los 269 firmados existentes.

**Las cinco cosas concretas a resolver en la reunión**:

1. ¿Empezamos a capturar GPS al cotizar y/o al instalar? ¿Cómo lidiamos con socios solares que no proveen coordenadas?
2. ¿Cómo proponemos manejar los cambios retroactivos en el reporte trimestral (para que ADA no tenga que estar adivinando cuál versión es la correcta)?
3. Inconsistencia de unidades línea 26 vs 32 (probablemente no es bug, es una falta de definición clara cliente-vs-proyecto) — necesitamos ver el reporte exacto y alinear definiciones con ADA
4. ¿ADA acepta agregar una fila "otros" para los 40 clientes que son sociedades sin género asignado?
5. ¿Necesitan kW/kWh/CO2 desglosados por trimestre? Hoy solo damos los conteos demográficos por trimestre

**Lo que ya hicimos antes de la reunión** (no requiere discusión, solo informar):
- Migramos `impoverished_area` a derivarse del municipio (regla del 20% más pobre, aplicada en Guatemala; HN y SV se reportan como Not Sure por falta de datos)
- Extendimos el reporte trimestral hasta Q2 2026 + agregamos categorías Non-Profit y "High Social Impact"
- Limpiamos el campo manual `rural_area` (un proyecto con valor `'Si '` con espacio) y convertimos NULLs a Not Sure
- Bloqueamos rural_area e impoverished_area en el formulario de operaciones (son derivados ahora)
- Arreglamos un bug en el formulario de operaciones donde no se podían editar 4 categorías de impacto

**Discusión interna pendiente (Tema 7)**: estamos proponiendo mover las categorías de impacto a sus niveles correctos — `women_led` y `youth_led` al cliente, `educational_institution` al sitio, `non_profit` al cliente. Esto resuelve por construcción varios bugs ("9-client gap", clientes con valores inconsistentes, conflicto cliente-vs-proyecto). Vale la pena hablarlo internamente antes del meeting con ADA.

**Lectura previa recomendada (5 min)**: Tema 1 (Rural) y Tema 2 (CO2). Tema 7 es discusión interna importante. Los demás son técnicos o ya resueltos.

---

## Contexto
Esta reunión surge de dos hilos de correo paralelos con ADA:

1. **"Cifras por revisar emisiones CO2"** (17-20 mar): Nausica detectó tres problemas concretos comparando los reportes Q3 y Q4: (a) las cifras de CO2 no coinciden entre versiones, (b) hay una inconsistencia entre dos líneas del conteo de clientes (línea 26 < línea 32) que probablemente viene de mezclar unidades cliente y proyecto, y (c) solo 144 clientes están clasificados por género. Alex copió a Jake y acordó traer la conversación a esta reunión.

2. **"Albedo Solar - Impact Data Discussion"** (23 mar): Alex inició una conversación con ADA y Valuing Impact proponiendo una nueva metodología para clasificar proyectos rurales. Su propuesta concreta: **clasificación basada en GPS usando GHS-SMOD**. Adjuntó el documento `rural_classification_northern_triangle.docx`. Smiti respondió el 1 de abril; aún estamos esperando el resto de la discusión.

El reporte que ADA revisa tiene su equivalente en Albedo en el modelo dbt `mart_impact_quarterly`.

---

## Foto general del portafolio (al día de hoy)

| Métrica | Valor |
|---|---|
| Proyectos firmados | **269** |
| Clientes únicos | **184** |
| Proyectos financiados | 206 (77%) |
| Proyectos de Alto Impacto Social | 120 (45%) |

**Factores de emisión por país (toneladas CO2 / MWh)**:
- Guatemala: 0.630
- El Salvador: 0.410
- Honduras: 0.520

---

## Tema 1: Clasificación Rural — la propuesta GPS de Alex

### Lo que ya propuso Alex (correo del 23 mar)

Alex presentó tres opciones a ADA y Valuing Impact, con una preferencia clara:

1. **Proxy administrativo (actual)**: proyectos fuera de cabeceras municipales y departamentales son rurales. Es lo que viene en Section 2.3 del documento adjunto (estándar de Honduras). Alex reconoció que **subestima la exposición rural**. Ejemplo concreto: **Santa Cruz La Laguna (~3,000 habitantes)** queda clasificado como no-rural por ser cabecera municipal, aunque obviamente es una comunidad pequeña y remota.

2. **Basado en población (en consideración)**: tiene problemas prácticos — los umbrales varían por país, los datos a nivel localidad son difíciles de obtener y no son confiables, y los proyectos en el borde de centros poblados son ambiguos.

3. **Basado en GPS (dirección propuesta)** ⭐: cada proyecto tendría coordenadas GPS, mapeadas contra un dataset estandarizado como **GHS-SMOD** (Global Human Settlement - Settlement Model). Da una clasificación rural/urbano consistente y comparable internacionalmente.

### Lo que respondió Smiti / ADA (1 abr)

Resumen de la posición de ADA:

- **ADA típicamente usa el proxy administrativo** (lo mismo que tenemos hoy). Reconocen que puede no ser totalmente preciso, como demuestra el ejemplo de Santa Cruz La Laguna.
- **ADA no exige GPS** porque no quiere aumentar la carga de reporte de sus inversiones, que generalmente son etapa temprana y no tienen el ancho de banda para metodologías más avanzadas. La mayoría de los inversionistas se basan en lo que el investee provee.
- **GPS-based no es práctica común todavía**, ni siquiera para los DFIs grandes. Pero entre las opciones disponibles, **GPS es claramente mejor que la basada en población**.
- **Smiti felicita el enfoque proactivo de Albedo** y sugiere que podríamos **destacarlo como USP** (Unique Selling Proposition) frente a otros: aporta comparabilidad cross-country (crítico para portafolios multi-país) y permite validar las claims de alcance.
- **Sugerencia concreta de Smiti**: que el field staff capture las coordenadas durante visitas de campo, compartiendo ubicación por Google Maps o WhatsApp. Reconoce que no todos los proyectos tienen visita de campo previa al desembolso, y que cuando el proyecto viene de un socio, no tendríamos los datos.
- **Recomendación clave de Smiti** ⭐: **antes de cambiar de metodología, hacer un assessment del portafolio actual (o un sample donde sí tengamos GPS) comparando el resultado GPS-based vs el proxy administrativo**. Si la diferencia es grande, vale la pena el cambio; si es chica, quizás no.
- ADA está abierta a seguir discutiendo.

### Lo que dice el documento técnico de Albedo

El documento `rural_classification_northern_triangle.pages` (en `external-resources/`, March 2026) hace un análisis riguroso del problema y aterriza varias cosas importantes:

**Las definiciones nacionales son fundamentalmente distintas entre los tres países**:
- **Guatemala (INE)**: cabeceras municipales y departamentales = urbano. Algunas variantes del censo agregan requisitos adicionales (>2,000 habitantes + electricidad + plomería). Censo 2018: 51.5% rural.
- **El Salvador (DIGESTYC)**: la más prescriptiva — necesita 500+ viviendas contiguas, alumbrado público, escuela primaria, transporte regular, calles pavimentadas, y teléfono público para ser urbano.
- **Honduras (INE)**: la más simple — cabeceras municipales = urbano, aldeas y caseríos = rural. Esto significa que un cabecera municipal de 500 habitantes en un departamento remoto cuenta como urbano.

Por eso una comunidad de 1,500 personas sin caminos pavimentados podría clasificarse de manera diferente en cada país. **Para reportes cross-country como el de ADA, las definiciones nacionales son insuficientes.**

**El documento recomienda exactamente un enfoque de dos capas** (alineado con lo que sugirió Smiti):

1. **Primaria — Opción A: GPS + GHS-SMOD** ⭐
   - Capturar coordenadas GPS para cada sitio de proyecto
   - Mapear contra el dataset GHS-SMOD del European Commission Joint Research Centre
   - GHS-SMOD usa la metodología "Degree of Urbanisation" endosada por la UN Statistical Commission en 2020
   - Cobertura completa de los tres países con la misma metodología, en grilla de 1km × 1km
   - Datos abiertos y gratis (CC BY 4.0), raster de Centro América ~50MB
   - Reconocido por IRIS+, IFC, OECD, World Bank, FAO, UN-Habitat
   - **Threshold sugerido**: GHS-SMOD clases 11, 12, 13 = rural; clase 21+ = urbano

2. **Interim fallback — Opción D + C: jerarquía administrativa con suplemento poblacional**
   - Para proyectos donde aún no tengamos GPS: clasificar por nivel administrativo (cabecera = urbano, aldea/caserío = rural)
   - Suplementar con datos de población donde estén disponibles
   - **Marcar para revisión cualquier cabecera con menos de 5,000 habitantes** (porque probablemente debería ser rural)

**Pasos de implementación que recomienda el documento** (ya en orden):

1. Agregar campo de coordenadas GPS al formulario de intake de proyectos (puede ser una foto del celular con metadata de ubicación)
2. Descargar el raster GHS-SMOD para Centro América
3. Decidir el threshold (recomendación: clases 11–13 = rural)
4. Backfill los proyectos existentes con el método interim (administrativo + población)
5. Redactar una nota metodológica corta (1–2 páginas) para incluir en reportes a inversionistas
6. Spot-check de una muestra de sitios contra GHS-SMOD para validar contra realidad de campo

**Alineación con frameworks internacionales**:
- **IRIS+** (GIIN): tiene la métrica PI1190 (Client Individuals: Rural) que pide documentar las asunciones metodológicas. Adoptar GHS-SMOD satisface ese requerimiento y hace los datos comparables con otros usuarios de IRIS+.
- **OPIM** (IFC): los nueve principios requieren metodologías documentadas y verificables — GHS-SMOD aplica.
- **SDGs**: el indicador 9.1.1 (Rural Access Index) y la disagregación geográfica de los SDGs específicamente recomiendan la metodología Degree of Urbanisation que implementa GHS-SMOD.

### Implicaciones operacionales que mencionó Alex

- **Capturar GPS al momento de cotizar**, para poder aplicar cualquier descuento de impacto rural desde el inicio
- Si el GPS no está disponible al cotizar: excluir el proyecto del descuento, y capturar el GPS más tarde durante la instalación para fines de reporte
- Los socios solares actualmente **no siempre nos dan datos GPS**; tendríamos que empezar a requerirlo
- Tener GPS estándar en todos los proyectos también traería beneficios más allá de la clasificación

### Implicaciones comerciales

Hay un **descuento de impacto rural** atado a esta clasificación, no es solo un tema de reporte. Cambiar la metodología afecta el pricing aguas arriba.

### Lo que necesitamos resolver con ADA en la reunión

- **Validación del enfoque**: ADA ya endosó GPS-based como el mejor de los tres en su correo. No necesitamos una "aprobación" — la decisión es nuestra. Pero sí podemos confirmar que estamos alineados.
- **El assessment que recomendó Smiti**: ¿cuántos proyectos del portafolio actual tienen GPS? ¿Podemos correr la comparación GPS-based vs administrativo en una muestra? Esto es la pre-condición que pone Smiti antes de cambiar la metodología.
- **Plan operacional para capturar GPS**: ¿al cotizar?, ¿al instalar?, ¿requerirlo a socios solares? ¿Field staff con WhatsApp como sugirió Smiti?
- **Mientras tanto, qué metodología usamos como puente**: el campo `municipalities.territorial_distinction` que tenemos hoy es exactamente el proxy administrativo. ADA ya dijo que es lo que normalmente reciben. Podemos usarlo como base mientras desarrollamos lo de GPS, pero hay que decidir si vale la pena migrar el código intermedio o esperar al cambio definitivo.
- **Qué hacemos con los 269 proyectos firmados existentes** que no tienen coordenadas GPS. ¿Backfill via field staff? ¿Estimación desde la dirección del cliente?

---

## Tema 2: Cifras de CO2 que cambiaron entre los reportes Q3 y Q4

### Lo que detectó Nausica (correo del 17 mar)

> "El equipo monitoreo FIT vio que comparando el informe Q3 y el Q4 que recibimos, los valores de emisiones CO2 capturadas son distintas. Me podrías confirmar la cifra correcta al cierre de cada trimestre (q1, q2, q3, q4). En general, si cambian algunas cifras en los trimestres anteriores al del informe, por favor destacarlo en otro color para que lo veamos y agregar comentario de explicación."

Es decir: ADA recibe nuestro reporte cada trimestre. Cuando comparan el reporte Q3 (que tiene Q1, Q2, Q3) contra el reporte Q4 (que tiene Q1, Q2, Q3, Q4), las cifras de CO2 para los **mismos trimestres históricos** son diferentes. Esto los obliga a pedir confirmación de cuál es la versión correcta.

### Lo que respondió Stephanie

> "Te comento que las cifras presentadas son las correctas, tenemos un nuevo sistema que me genera el equipo de ops y esas son las cifras finales. Por el cambio de posición seguramente hubo algún inconveniente con la data."

### Lo que pasó realmente (lo que sabemos hoy)

Las cifras cambiaron porque **estamos limpiando los datos históricos en la migración de Quickbase a Supabase** y porque el cálculo de CO2 evitado se hace ahora de forma diferente:

1. Producción total del sistema durante su vida útil = `kW instalados × multiplicador de producción mensual × meses dentro de la garantía × (1 - degradación anual)`
2. CO2 evitado = `producción total (MWh) × factor de emisión del país (tCO2/MWh)`
3. Factores de red usados: GT 0.630, SV 0.410, HN 0.520

### Lo que necesitamos resolver con ADA

- ¿Cuáles son las cifras definitivas por trimestre? Las generamos del modelo nuevo y se las mandamos
- ¿Cómo proponemos manejar **cambios retroactivos**? Las opciones son:
  - Marcarlos en otro color en el reporte (lo que pidió Nausica) y agregar un comentario explicando el cambio
  - Versionar los reportes y dejar las cifras históricas congeladas (no recalcularlas)
  - Re-enviar reportes anteriores con las cifras corregidas
- ¿Validan los factores de emisión que usamos? ¿Hay una fuente oficial (IPCC, factores regionales) que ellos prefieren?
- ¿Qué período de vida útil esperan usar para la proyección? Hoy usamos los años de garantía del equipo
- ¿Aplican degradación del panel? ¿Con qué tasa? Nosotros usamos los valores específicos del equipo
- ¿Cómo manejan la inflación energética? Hoy asumimos 3% anual

---

## Tema 3: Inconsistencia de unidades en el conteo (línea 26 vs línea 32)

### Lo que detectó Nausica (correo del 17 mar)

> "Une pregunta más, nos parece que haya un error en el número de clientes porqué el número total de clientes de la línea 26 (leasing activo + venta al contado - al cierre de cada trimestre) es **menor** que el número de clientes con leasing activo (línea 32). ¿Podrías verificar?"

### El diagnóstico real

A primera vista parece matemáticamente imposible (un total siendo menor que uno de sus componentes), pero **probablemente no es un bug**. Es más probable que las dos líneas se cuenten en **unidades diferentes**:

- **Línea 26**: cuenta `distinct client_id` (clientes únicos) con leasing activo + venta al contado
- **Línea 32**: cuenta `count(*)` (proyectos) con leasing activo

Si un cliente tiene varios proyectos de leasing, sale una vez en la línea 26 (cliente) pero varias veces en la línea 32 (proyectos). Por eso el "total" puede parecer menor.

Es exactamente el mismo síntoma que vimos en Tema 6 (Total Clientes 184 ≠ Woman Led + Non-Woman Led 175): números que parecen contradictorios pero son legítimamente lo que son si entiendes la unidad. **Y se conecta directamente con el Tema 7** — si las categorías estuvieran en sus niveles correctos, este tipo de inconsistencia desaparecería por construcción.

### Lo que necesitamos resolver

- **No es "arreglar el bug"** — es **alinear las definiciones**
- Conseguir el archivo exacto del reporte que ADA recibe para ver qué unidad usa realmente cada fila
- Documentar explícitamente la unidad de cada fila (cliente vs proyecto vs sitio)
- Acordar con ADA cuál unidad quieren para cada bloque del reporte
- Hacer que la aritmética cuadre dentro de cada bloque que use la misma unidad
- A largo plazo: Tema 7 elimina este problema desde el diseño

---

## Tema 4: Categorización de género de clientes

### Lo que respondió Stephanie (correo del 18 mar)

> "No es que exista diferencia, es que en nuestro sistema solo tenemos **144 clientes catalogados como male and female**, el resto probablemente es una sociedad con una junta directiva y no se categorizó. Lo que sugeriría es agregar una fila de 'otros' y ahí clasificar el resto, ¿qué piensas?"

### El contexto (con una salvedad importante)

ADA tiene una sección del reporte que muestra los clientes por género. Stephanie dice que hay 144 categorizados.

**Salvedad**: revisé la base de datos y **no existe ningún campo de género ni en la tabla `clients` ni en ninguna otra tabla**. Tampoco hay tabla de contactos o representantes legales. Eso significa que los "144 catalogados" probablemente vienen de:

1. **Una clasificación manual que Stephanie hace al armar el reporte** (en Excel, mirando los nombres legales) — la opción más probable
2. **Un campo en QuickBase que aún no se migró** a Supabase
3. **Otra hoja de cálculo intermedia** que vive solo en el proceso de armado del reporte

**Hay que confirmar con Stephanie de dónde sale esa clasificación antes de la reunión**, porque define mucho qué proponemos.

### Posibles caminos según de dónde venga

- **Si es manual en Excel**: la opción correcta es agregar un campo a la tabla `clients` (algo como `client_type`: individual / sociedad, y opcionalmente `gender`: M / F si es individual). Y eventualmente migrar la clasificación que Stephanie ya tiene.
- **Si está en QuickBase**: priorizar la migración de ese campo a Supabase.
- **Si está en otro lado**: investigar y consolidar.

### Lo que necesitamos resolver

- Confirmar con Stephanie de dónde viene el "144" — necesario antes de cualquier decisión técnica
- ¿ADA acepta la propuesta de Stephanie de agregar una fila "otros" / "personas jurídicas" para los clientes que son sociedades?
- Independiente del reporte: ¿queremos un campo formal en la BD para esto, o nos quedamos con la clasificación manual?

Ojo: esto es **distinto** del campo `women_led` a nivel proyecto. `women_led` indica si el proyecto es liderado por mujeres (puede aplicar incluso si el cliente legal es una sociedad). El género del cliente es a nivel de la persona.

---

## Tema 5: Métricas de energía y CO2 que faltan en el reporte trimestral

`mart_impact_quarterly` actualmente solo tiene conteos de clientes/proyectos por categoría de impacto. Las métricas más importantes para ADA — kW instalados, kWh producidos, toneladas CO2 evitadas, ahorros — viven en `mart_impact_overall_summary` pero **no están desglosadas por trimestre**.

Esto es probablemente parte del por qué Nausica está viendo cifras inconsistentes: si recibe un reporte trimestral con métricas de CO2, pero esas métricas se calculan al vuelo desde el resumen general, cualquier limpieza de datos afecta los trimestres pasados también.

### Lo que necesitamos resolver

- ¿ADA necesita el desglose trimestral oficialmente? Si sí, hay que extender `mart_impact_quarterly` con esas métricas
- ¿O mantenemos las cifras de CO2/kWh solo a nivel total del portafolio?

---

## Tema 6: Calidad de datos en las categorías manuales de impacto

Las categorías que aún dependen de captura manual del equipo (NO derivadas del municipio):

| Campo | Not Sure | % de 269 |
|---|---|---|
| women_led | 14 | 5.2% |
| youth_led | 14 | 5.2% |
| educational_institution | 2 | 0.7% |
| non_profit | 1 | 0.4% |

`rural_area` y `impoverished_area` ya no aparecen aquí porque son (o serán) campos derivados del municipio. Su completitud depende de la calidad de los datos a nivel municipio, no de la captura manual.

### Una inconsistencia que detectamos nosotros

Es la misma clase de problema que la línea 26/32 que flagueó Nausica — un mismatch de unidades cliente/proyecto. Las filas de "clientes liderados por mujeres" tampoco cuadran con el total:

| Fila | Conteo |
|---|---|
| Total Clientes | 184 |
| Clientes Liderados por Mujeres | 42 |
| Clientes No Liderados por Mujeres | 133 |
| **Suma** | **175** |
| **Faltante** | **9 clientes (4.9%)** |

Es la misma clase de bug: el campo `women_led` está a nivel **proyecto**, no cliente, y los 9 clientes faltantes tienen todos sus proyectos marcados como "Not Sure". El Tema 7 propone una solución estructural para esto.

---

## Tema 7: Repensar el nivel granular de cada categoría de impacto

(Discusión interna. Llevárnoslo internamente primero, después decidir si y cómo presentárselo a ADA.)

### El problema de fondo

Hoy las 6 categorías de impacto viven todas en `projects`, pero conceptualmente son propiedades de **niveles distintos**:

- `women_led` y `youth_led` son propiedades del **liderazgo de la organización** — no del proyecto. El liderazgo de un cliente no cambia entre proyectos.
- `educational_institution` es una propiedad del **sitio físico** — un colegio es un colegio independientemente del proyecto que se instale ahí.
- `non_profit` es una propiedad del **estatus legal del cliente** (probablemente cliente, aunque es defendible a nivel sitio también).
- `rural_area` y `impoverished_area` son propiedades del **municipio** (ya derivados ahí).

Tenerlos todos a nivel proyecto produce varios problemas que ya hemos visto:

- **Inconsistencia interna**: encontramos 3 clientes en la base de datos con proyectos que tienen valores **distintos** de `women_led`. Eso es imposible si pensamos en women_led como propiedad del liderazgo.
- **El "9-client gap"** del Tema 6 viene exactamente de aquí: como `women_led` está a nivel proyecto, agregar a nivel cliente es ambiguo.
- **El conflicto "clientes vs proyectos"** que ADA flagueó: la unidad correcta depende del tipo de propiedad, pero hoy todo es proyecto.

### La reorganización propuesta

| Categoría | Nivel propuesto | Razón |
|---|---|---|
| `women_led` | **Cliente** | Liderazgo de la organización |
| `youth_led` | **Cliente** | Mismo razonamiento |
| `non_profit` | **Cliente** (defendible: sitio) | Estatus legal del cliente |
| `educational_institution` | **Sitio** | Uso/naturaleza del lugar |
| `rural_area` | Municipio (✅ ya derivado) | Geografía |
| `impoverished_area` | Municipio (✅ ya derivado) | Geografía |

### Datos del estado actual

- **269 proyectos firmados** → **184 clientes únicos** → **198 sitios únicos**
- **8 proyectos firmados sin sitio asignado** (gap a llenar)
- **85 proyectos comparten cliente** (clientes con más de un proyecto)
- **71 proyectos comparten sitio** (sitios con más de un proyecto — extensiones, addendums)

**Importante**: la tabla `sites` ya tiene columnas `latitude` y `longitude`. Ese es exactamente el lugar correcto para capturar las coordenadas GPS del Tema 1 — al sitio, no al proyecto. Los proyectos heredan el GPS del sitio.

### Lo que esto resuelve

1. **El bug del 9-client gap** desaparece — al estar en el cliente, contar "clientes liderados por mujeres" es trivial y no ambiguo
2. **El conflicto "clientes vs proyectos"** se resuelve por construcción: cada categoría se cuenta en su unidad natural
3. **Los 3 clientes con valores inconsistentes** entre proyectos dejan de ser posibles por diseño
4. **Captura de GPS** se vuelve más natural — agregas las coordenadas al sitio (que ya tiene las columnas) y se hereda en todos los proyectos
5. **Onboarding más simple**: cuando un cliente regresa con un segundo proyecto, no hay que volver a clasificar el liderazgo — ya está en el cliente

### El costo del cambio

- **Database**: agregar columnas a `clients` y `sites`, backfill desde proyectos, eventualmente deprecar las columnas en `projects`
- **Frontend**: mover los campos del formulario de proyecto al formulario de cliente y al de sitio (Quote Wizard, ActiveProjectDetails, EditOperationsProjectForm)
- **dbt**: reescribir los impact marts para hacer JOIN con clientes/sitios en vez de leer las columnas del proyecto
- **Reportes**: actualizar el formato del reporte que ADA recibe para que cada categoría cuente en su unidad natural

### Plan sugerido

1. Discutirlo internamente primero (Alex, equipo de Ops, equipo técnico)
2. Si hay consenso, planificar el cambio en una fase separada después del meeting
3. Mencionarlo en el meeting con ADA solo como dirección estratégica, sin entrar al detalle técnico — la conversación con ADA debe enfocarse en los temas que ellos flaguearon, no en este rediseño

---

## Cambios ya realizados antes de la reunión

### En `mart_impact_quarterly`
- Trimestres agregados: **Q4 2025, Q1 2026, Q2 2026** (antes solo iba hasta Q3 2025)
- Nueva categoría **Non-Profit** (Total y New)
- Indicador compuesto **High Social Impact** (proyectos con cualquier categoría de impacto = Yes)

### En todos los marts de impacto
- **`impoverished_area` ahora se deriva de `municipalities.in_poorest_20_percent`** en lugar del campo manual de Quickbase. Aplica solo a Guatemala (HN/SV no tienen datos). Documentado en `dbt/main/CALCULATIONS.md`.

### Limpieza de datos en Supabase
- Convertidos todos los `NULL` a `'Not Sure'` en las 6 categorías de impacto (163 actualizaciones)
- Corregido un proyecto donde `rural_area = 'Si '` (con espacio) ahora es `'Yes'`

### En la UI (operaciones — formulario de edición de proyecto)
- Agregados los 4 campos editables de impacto que faltaban: women_led, youth_led, non_profit, educational_institution
- Agregada sección **"Indicadores Derivados"** mostrando rural_area e impoverished_area como solo lectura, con explicación de que vienen del municipio

---

## Decisiones requeridas en la reunión

| # | Decisión | Origen |
|---|---|---|
| 1 | Confirmar plan: hacer el assessment GPS vs administrativo en una muestra antes de migrar la metodología (lo que pidió Smiti) | Hilo "Impact Data Discussion" |
| 2 | Definir plan operacional para capturar GPS: ¿al cotizar?, ¿al instalar?, ¿field staff via WhatsApp como sugirió Smiti?, ¿requerirlo a socios solares? | Hilo "Impact Data Discussion" |
| 3 | Acordar cómo manejar el backfill de GPS para los 269 proyectos firmados existentes | Hilo "Impact Data Discussion" |
| 4 | Validar factores de emisión de CO2 (¿IPCC, factor regional, los que usamos?) | Hilo "CO2" |
| 5 | Acordar proceso para destacar cambios retroactivos en el reporte (color + comentario, versionado, etc.) | Hilo "CO2" |
| 6 | Alinear con ADA las definiciones de unidades (cliente vs proyecto) por fila del reporte — esto resuelve la "inconsistencia" línea 26 vs 32 | Hilo "CO2" |
| 7 | Aprobar la propuesta de fila "otros" para clientes que son sociedades sin género | Hilo "CO2" |
| 8 | Definir si necesitamos desglose trimestral de kW/kWh/CO2/ahorros en el reporte | Análisis interno |
| 9 | Definir cómo manejar proyectos con datos pendientes (gap entre Total y Yes+No) | Análisis interno |
| 10 | Conseguir datos de pobreza municipal para Honduras y El Salvador | Análisis interno |
| 11 | **Discusión interna**: aprobar la reorganización del modelo de datos del Tema 7 (women_led/youth_led al cliente, educational_institution al sitio, non_profit al cliente) | Análisis interno |

---

## Acciones técnicas inmediatas (independientes de la reunión)

- **Correr el assessment que recomendó Smiti**: identificar cuántos proyectos tienen GPS hoy, mapearlos contra GHS-SMOD, y comparar la clasificación rural resultante contra el proxy administrativo. Si hay diferencia material, justifica el cambio.
- Conseguir el archivo exacto del reporte trimestral que ADA recibe para entender qué unidad usa cada fila (clientes vs proyectos)
- **Confirmar con Stephanie de dónde sale la clasificación de género de clientes** (¿Excel manual? ¿QuickBase? ¿otra fuente?). No existe campo en la BD.
- Confirmar si los 14 "Not Sure" en `women_led` son realmente desconocidos o si se pueden clasificar
- Descargar el raster GHS-SMOD de Centro América (~50MB) y montar un script de lookup en Python con `rasterio`
- **Usar fecha de instalación real para el cálculo de "production to date"** en lugar de la fecha del proyecto (que es la fecha de firma). Hoy `mart_impact_projects_summary` asume que el proyecto empezó a producir el día que se firmó, lo cual sobreestima la producción acumulada y los ahorros acumulados en proyectos que tardaron en instalar. Datos disponibles: `actual_installation_start_date` está poblado en 242/269 proyectos firmados (90%), `actual_installation_end_date` solo en 21/269 (8%). El promedio observado entre firma e instalación es de **~62 días (2 meses)**. Propuesta de cadena de fallback: `COALESCE(actual_installation_end_date, actual_installation_start_date + 1 month, project_date + 2 months)`. Cubre el 90% con datos reales.
