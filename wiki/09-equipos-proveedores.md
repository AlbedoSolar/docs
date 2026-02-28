# 9. Equipos y Proveedores

## Equipos

### Vista general

La página de equipos (`/equipments`) gestiona el catálogo de equipos solares y sus marcas.

### Datos del equipo

- Nombre y marca
- Potencia (watts)
- Años de garantía
- Tasa de pérdida de producción
- Especificaciones técnicas adicionales

### Marcas de equipos

Los equipos se organizan por marcas. La gestión de marcas se hace desde la misma página con el formulario `EquipmentBrandForm`.

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/EquipmentsPage.tsx` |
| Formulario equipo | `components/organisms/EquipmentForm.tsx` |
| Formulario marca | `components/organisms/EquipmentBrandForm.tsx` |
| Tabla | `components/organisms/EquipmentsTable.tsx` |
| Slice Redux | `features/equipments/equipmentsSlice.ts` |

---

## Proveedores

### Vista general

La página de proveedores (`/providers`) gestiona las empresas que suministran equipos e instalaciones.

### Datos del proveedor

- Información de la empresa
- Equipos que suministra
- Comisiones y márgenes
- Cronogramas de pago

### Vista de detalle

La ruta `/providers/:provider_id` muestra la información completa del proveedor, incluyendo sus equipos asociados y condiciones comerciales.

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/ProvidersPage.tsx` |
| Detalles | `pages/ProviderDetailsPage.tsx` |
| Slice Redux | `features/providers/providersSlice.ts` |

---

## Afiliados

### Vista general

La página de afiliados (`/affiliates`) gestiona los socios comerciales de Albedo.

### Archivos clave

| Archivo | Ubicación |
|---------|-----------|
| Página | `pages/AffiliatesPage.tsx` |
| Formulario | `components/organisms/AffiliateForm.tsx` |
| Tabla | `components/organisms/AffiliatesTable.tsx` |
| Slice Redux | `features/affiliates/affiliatesSlice.ts` |
