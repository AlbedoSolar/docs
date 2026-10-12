# 13. Mantenimientos

## Vista general

La sección de **Mantenimientos** (menú **Operaciones › Mantenimientos**) permite al equipo de operaciones consultar y **editar** el calendario de mantenimientos de cada proyecto financiado. (La **creación** de nuevos mantenimientos está temporalmente deshabilitada — ver la sección "Crear un mantenimiento".)

Cada proyecto financiado tiene un cronograma de mantenimientos —un evento por año de financiamiento— generado automáticamente a partir de la fecha del primer pago del leasing. Desde esta pantalla se da seguimiento a esos eventos: cuándo estaban programados, cuándo se realizaron, cuánto cuestan y si los cubre Albedo.

## Tarjetas de resumen

En la parte superior se muestran cuatro indicadores. **Cuentan los mantenimientos que coinciden con los filtros activos** (no los proyectos).

| Tarjeta | Significado |
|---------|-------------|
| Total | Número total de mantenimientos |
| Completados | Mantenimientos con fecha real registrada |
| Vencidos | Sin realizar y cuya fecha programada ya pasó |
| Pendientes | Programados a futuro, aún sin realizar |

> **Tip:** las tarjetas son **botones**. Haz clic en una (p. ej. *Vencidos*) para filtrar la tabla por ese estado; vuelve a *Total* para quitar el filtro de estado.

## Buscar y filtrar

Cada filtro tiene su etiqueta y un ícono **ⓘ** que explica qué hace al pasar el cursor.

| Filtro | Función |
|--------|---------|
| **Buscar** | Filtra por proyecto o cliente |
| **Año programado (calendario)** | Año del calendario en que está programado el mantenimiento (según la Fecha Programada). Las opciones se generan a partir de los datos. **No** es el N° de año de contrato de la tabla interna |
| **Estado del mantenimiento** | **Pendiente** (a futuro), **Vencido** (atrasado) o **Completado** |
| **Cobertura — ¿quién paga?** | Todas / Solo paga Albedo / Solo no paga Albedo |
| **Financiamiento del proyecto** | Estado del **leasing** del proyecto: Solo vigentes (por defecto), Solo finalizados o Vigentes + finalizados. **No** se refiere al estado del mantenimiento |

Debajo de los filtros, una línea muestra los **filtros activos** y un enlace **"Limpiar filtros"** para restablecerlos.

> Si no encuentras un proyecto, revisa que el filtro **"Financiamiento del proyecto"** no esté en "Solo vigentes" ocultándolo.

## Estructura de la tabla

La tabla muestra **una fila por proyecto**, con su referencia, cliente, país, socio instalador, estado de financiamiento, número de mantenimientos, costo total y próximo mantenimiento.

**Haz clic en la fila del proyecto para desplegarla.** Se abre una tabla interna con un renglón por cada mantenimiento, donde se editan.

> Al filtrar, bajo cada proyecto se muestran **solo los mantenimientos que coinciden** con los filtros; por eso el **# Mant.** y el **Costo Total** reflejan ese subconjunto, no necesariamente el total del proyecto.

## Editar un mantenimiento

> Los cambios se **guardan automáticamente**: no hay botón de "Editar" ni de "Guardar". Al modificar un campo, el sistema lo guarda solo y muestra un mensaje de confirmación.

1. Despliega el proyecto.
2. Modifica el campo necesario en la fila del año:

| Campo | Cómo se edita | Cuándo se guarda |
|-------|---------------|------------------|
| Año | Escribe el número | Al salir del campo (clic afuera o Tab) |
| Fecha Programada | Selecciona la fecha | Al elegir la fecha |
| Fecha Real | Selecciona la fecha en que se realizó | Al elegir la fecha |
| Cobertura (Paga Albedo / No paga Albedo) | Elige en la lista desplegable | Al elegir la opción |

3. Espera el mensaje verde de confirmación.

**Reglas a tener en cuenta:**

- Al registrar una **Fecha Real**, el estado pasa automáticamente a **Completado**. Si la borras, vuelve a **Pendiente**.
- La **Fecha Programada** es obligatoria.
- No puede haber dos mantenimientos con el **mismo año** en un proyecto. Mientras escribes el **Año**, si el valor ya existe en el proyecto, el campo se marca en rojo con el aviso *"El año N ya existe en este proyecto."*
- Si un cambio **no se guarda** (por ejemplo, un año duplicado o falta de permisos), el campo **vuelve automáticamente** a su valor anterior y se muestra un mensaje de error. Nada queda a medias.

## Crear un mantenimiento

> ⚠️ **Temporalmente deshabilitado.** La creación de nuevos mantenimientos está
> en pausa mientras el equipo confirma si esta función es necesaria. El botón
> **"+ Agregar mantenimiento"** sigue visible, pero al intentar guardar **aparece
> un error y el registro no se crea**. Por ahora la pantalla sirve para **editar**
> mantenimientos existentes. Si necesitas crear uno, contacta al equipo de sistemas.

Cuando se habilite, el flujo será:

1. Despliega el proyecto.
2. Debajo de la lista de años, haz clic en **"+ Agregar mantenimiento"**.
3. Completa el formulario:
   - **Año** — obligatorio (viene sugerido el siguiente; se puede cambiar)
   - **Fecha Programada** — obligatoria
   - **Costo** — opcional
   - **Cobertura** — Paga Albedo / No paga Albedo
4. Haz clic en **Guardar** (o **Cancelar** para descartar).

## Las tres fechas

| Fecha | Qué es | Quién la pone |
|-------|--------|---------------|
| **Fecha objetivo** | Cuándo corresponde el mantenimiento según el contrato (12 meses por año desde el ancla del proyecto) | El sistema |
| **Fecha de visita** | Fecha tentativa en que se hará la visita; se puede cambiar cada vez que se reprograme | Operaciones |
| **Fecha real** | Día en que se realizó el mantenimiento; al llenarla el estado pasa a Completado | Operaciones |

Usa **Fecha de visita** para anotar una visita coordinada. No uses Fecha real para eso: una fecha real a futuro cuenta como completado y el sistema la rechaza en la revisión nocturna.

Las tarjetas de arriba muestran un contador por estado. **Programación vencida** es la lista de seguimiento: o la visita se hizo y falta registrar la fecha real, o no se pudo hacer y hay que poner una nueva fecha de visita.

## Estados y cobertura

| Concepto | Valores |
|----------|---------|
| Estado | Se calcula con las tres fechas, nunca se elige a mano: **Pendiente** (no vence todavía, sin visita) · **Vencido** (ya venció, sin visita) · **Pendiente programado** (no vence todavía, con visita a futuro) · **Vencido programado** (ya venció, con visita a futuro) · **Programación vencida** (la visita ya pasó y no se marcó como realizada) · **Completado** (con fecha real) |
| Cobertura | **Paga Albedo** · **No paga Albedo** — indica quién asume el costo del mantenimiento frente al socio instalador |
| Financiamiento | **Vigente** · **Finalizado** — según la fecha de fin del financiamiento del proyecto |

## Exportar

El botón **Exportar CSV** descarga los mantenimientos **según los filtros y la búsqueda activos**, con una fila por mantenimiento (para abrir en Excel).

## Permisos

| Acción | Rol requerido |
|--------|---------------|
| Ver mantenimientos | Cualquier usuario autenticado |
| Editar mantenimientos existentes | `operations`, `operations-admin` o `admin` |
| Crear mantenimientos | Temporalmente deshabilitado (pendiente de confirmación del equipo) |

Si solo ves texto en lugar de campos editables, es porque tu usuario no tiene un rol con permiso de edición. Contacta al equipo de sistemas. Ver también [Roles y Permisos](12-roles-y-permisos.md).

## Relación con otras entidades

- Cada mantenimiento pertenece a un **proyecto** financiado (los proyectos de contado no tienen mantenimientos).
- El cronograma se crea **cuando Operaciones registra la fecha de Cambio Medidor** en la página del proyecto (ver abajo). El primer mantenimiento queda 12 meses después de esa fecha; luego uno por cada año del contrato.
- El costo mostrado proviene del flujo de caja de la cotización firmada (es informativo, no afecta cálculos financieros). Los años sin cargo de mantenimiento en el flujo son años cubiertos por el socio.

## Cómo se crea el cronograma

Desde el 2026-10-12 el cronograma de un proyecto nuevo se crea desde el panel de hitos de la página del proyecto, al registrar **Cambio Medidor**:

1. Al presionar **Registrar** en Cambio Medidor, el sistema muestra una ventana con lo que va a crear: cuántos mantenimientos, la fecha del primero (12 meses después) y del último, y qué años paga Albedo con su costo sin IVA.
2. Al confirmar, se registra el hito y se crean los mantenimientos. Hasta ese momento el proyecto **no aparece** en el módulo de mantenimientos.
3. Si después se **modifica** la fecha de Cambio Medidor, los mantenimientos pendientes se reprograman con la nueva fecha; la ventana de modificación lo indica antes de guardar. Si el proyecto ya tiene algún mantenimiento completado, las fechas no se mueven.
4. **Anular** el hito conserva los mantenimientos.

Los proyectos que ya tenían cronograma antes de esa fecha conservan sus fechas (anclaje en la entrega al cliente o el primer pago).

Si al confirmar aparece el mensaje "el calendario de mantenimientos no se creó", el hito sí quedó registrado; avisa a sistemas para crear el cronograma.

## Preguntas frecuentes

**¿Hay que guardar al final?**
No. Cada cambio se guarda por separado en el momento; solo confirma que aparezca el mensaje verde.

**No veo los campos para editar, solo texto.**
Verifica que (1) hayas desplegado el proyecto y (2) tengas un rol con permiso de edición.

**¿Puedo borrar un mantenimiento?**
No desde la pantalla. Para eliminar registros, contacta al equipo de sistemas.

**Me aparece "No tienes permiso para editar este mantenimiento."**
Tu usuario no tiene los permisos necesarios. Comunícate con sistemas.

**¿Por qué hay dos "años"?**
El filtro **"Año programado (calendario)"** es el año del calendario (2025, 2026…). La columna **"Año (N° contrato)"** dentro del proyecto es el número de año del contrato (1, 2, 3…). Son cosas distintas.

**Filtré por un estado y un proyecto ahora muestra menos mantenimientos. ¿Es normal?**
Sí. Al filtrar, bajo cada proyecto se muestran solo los mantenimientos que coinciden, y el **# Mant.** y **Costo Total** reflejan ese subconjunto. Usa **"Limpiar filtros"** para ver todo de nuevo.
