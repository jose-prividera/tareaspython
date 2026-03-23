# Diseño UX/UI — [ÉPICA E-X / US X.Y: Nombre]

> **Referencia de diseño:** [`docs/guia_base_ux_ui.md`](guia_base_ux_ui.md)
> **PRD:** [`docs/PRD_MAESTRO.md`](PRD_MAESTRO.md) — Sección [X]
> **Sprint:** [Sprint N] | **Estado:** Diseño / En desarrollo / Implementado
> **Autor:** | **Fecha:**

---

## 1. Alcance de la Pantalla / Flujo

> Describir qué vista(s) o flujo cubre esta US. Referenciar el árbol de navegación de la guía.

**Vista(s) afectadas:**

| Vista | Archivo | Acción |
|-------|---------|--------|
| [Nombre de la vista] | `resources/views/[ruta].blade.php` | Nueva / Modificada / Sin cambios |

**Flujo de navegación:**
```
[Describir el flujo: de dónde viene el usuario → qué ve → hacia dónde va]
Ej: /programadores/clientes → [nueva vista] → /programadores/dashboard
```

**Componente Livewire involucrado (si aplica):**
- Clase PHP: `app/Livewire/[NombreComponente].php`
- Vista: `resources/views/livewire/[nombre].blade.php`

---

## 2. Estructura Visual

### 2.1 Layout y Zonas

> Describir la estructura general de la pantalla. Indicar qué layout usa y cómo se organiza el contenido.

- **Layout base:** `layouts/app.blade.php` (autenticado) / `layouts/guest.blade.php` (público)
- **Slot `$header`:** [Título de la página + breadcrumb si aplica]
- **Organización del contenido:**
  - Zona superior: [KPIs / filtros / acciones]
  - Zona central: [Tabla / formulario / wizard / cards]
  - Zona inferior: [Paginación / botones de acción]

### 2.2 Grid y Espaciado

| Elemento | Clases Tailwind | Referencia guía |
|----------|----------------|-----------------|
| Contenedor principal | `p-6` | Padding estándar de página |
| Grid de cards | `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6` | Patrón conciliación dashboard |
| [Otro elemento] | [clases] | [sección guía] |

---

## 3. Componentes a Usar

> Listar cada componente reutilizable del sistema. No crear nuevos si ya existe uno equivalente.

### Componentes existentes a reutilizar

| Componente | Uso en esta pantalla | Props relevantes |
|------------|---------------------|-----------------|
| `<x-card>` | Panel principal de [sección] | `title="[Título]"` |
| `<x-primary-button>` | Acción principal: [nombre acción] | — |
| `<x-secondary-button>` | Acción secundaria: [nombre] | — |
| `<x-text-input>` | Campo [nombre campo] | `name`, `wire:model` |
| `<x-input-label>` | Label de [campo] | `for`, `value` |
| `<x-input-error>` | Error de validación en [campo] | `:messages="$errors->get('campo')"` |
| `<x-modal>` | Modal de [nombre] | `name="[nombre]"` `maxWidth="[lg/xl/2xl]"` |
| `<x-dropdown>` | Dropdown de [nombre] | `align="right"` |
| `<x-nav-link>` | Link en sidebar (si se agrega) | `:active`, `:href` |

### Componentes nuevos requeridos

> Solo si no existe equivalente en la guía. Justificar brevemente.

| Componente | Archivo propuesto | Justificación |
|------------|------------------|---------------|
| [nombre] | `components/[nombre].blade.php` | [razón] |

### Componentes Livewire

| Componente | Tipo | Archivo |
|------------|------|---------|
| [NombreComponente] | Nuevo / Modificación de [existente] | `app/Livewire/[ruta].php` |

---

## 4. Tokens de Diseño Aplicados

> Referenciar exactamente los tokens de `guia_base_ux_ui.md` sección 4. No inventar clases nuevas.

### Colores

| Elemento | Token / Clase | Valor |
|----------|---------------|-------|
| Fondo de cards | `bg-white/10 backdrop-blur-sm border border-white/20` | Aurora Glass (patrón estándar) |
| Texto principal | `text-light-text` | `#f3f4f6` |
| Texto secundario | `text-light-text-muted` | `#9ca3af` |
| Acento / activo | `text-brand-accent` | `#FE9192` |
| Error | `text-red-400` | `#f87171` |
| Éxito | `text-green-400` | `#4ade80` |
| [Elemento específico] | [clase] | [valor] |

### Tipografía

| Elemento | Clases |
|----------|--------|
| Título de página / sección | `font-headings text-xl font-semibold text-light-text` |
| KPI value | `text-2xl font-bold text-light-text` |
| Texto de tabla | `text-sm font-normal text-light-text-muted` |
| Label de input | `text-sm font-medium text-light-text` |
| [Elemento específico] | [clases] |

### Bordes y superficies

| Elemento | Clases |
|----------|--------|
| Card / panel | `rounded-2xl shadow-lg border border-white/20` |
| Input | `rounded-md border-gray-300/50` |
| Modal | `rounded-2xl shadow-2xl` |
| [Elemento específico] | [clases] |

---

## 5. Interacciones y Estados

### Estados de UI requeridos

| Estado | Elemento | Implementación |
|--------|----------|----------------|
| Loading / procesando | [Botón / sección] | `wire:loading.attr="disabled"` en botón + spinner `wire:loading` |
| Vacío (empty state) | [Tabla / lista] | Mensaje: "[Texto descriptivo]" + acción sugerida: "[acción]" |
| Error de validación | [Campo(s)] | `<x-input-error>` + borde rojo en input |
| Éxito | [Acción] | Flash message `session('success')` |
| Deshabilitado | [Botón] | `disabled:opacity-50 disabled:cursor-not-allowed` |

### Microinteracciones

| Interacción | Implementación | Prioridad |
|-------------|---------------|-----------|
| Hover en [elemento] | [clases hover + transition] | Alta / Media / Baja |
| Abrir [modal/dropdown] | `$dispatch('open-modal', '[nombre]')` / `@click="open = !open"` | — |
| [Otra interacción] | [implementación] | — |

### Flujo de feedback al usuario

```
Usuario hace [acción]
  → [Estado de loading] (wire:loading visible)
  → [Procesamiento server-side]
  → Éxito: [comportamiento: redirect / flash / dispatch event]
  → Error: [comportamiento: mensajes de validación / alert]
```

---

## 6. Responsive

| Elemento | Mobile (< md) | Tablet (md) | Desktop (lg+) |
|----------|--------------|-------------|---------------|
| [Layout principal] | [comportamiento] | [comportamiento] | [comportamiento] |
| [Tabla de datos] | `overflow-x-auto` + scroll horizontal | [comportamiento] | ✅ Normal |
| [Grid de cards] | 1 columna | 2 columnas | [N] columnas |
| [Sidebar / nav] | Oculto (hamburger) | Visible | Visible |

---

## 7. Accesibilidad

| Requisito | Implementación |
|-----------|----------------|
| Labels en todos los inputs | `<x-input-label for="[id]">` + `id` en el input |
| Textos alternativos en íconos FA | `<i class="fas fa-..." aria-hidden="true"></i>` + `<span class="sr-only">[descripción]</span>` |
| `aria-current="page"` en nav activo | En `<x-nav-link>` cuando `:active="true"` |
| Contraste mínimo 4.5:1 | Verificar pares: [texto] sobre [fondo] |
| Focus visible en interactivos | Mantener `focus:ring` en inputs y botones |
| [Requisito específico] | [implementación] |

---

## 8. Vistas / Archivos a Crear o Modificar

> Lista definitiva de archivos a tocar. Completar antes de implementar.

| Archivo | Acción | Notas |
|---------|--------|-------|
| `resources/views/[ruta].blade.php` | Crear / Modificar | [descripción del cambio] |
| `app/Livewire/[Clase].php` | Crear / Modificar | [props / métodos nuevos] |
| `resources/views/livewire/[ruta].blade.php` | Crear / Modificar | — |
| `resources/views/components/[nombre].blade.php` | Crear | [solo si es nuevo componente justificado] |
| `routes/web.php` | Modificar | Agregar ruta `[GET/POST] [/ruta]` |
| `layouts/navigation.blade.php` | Modificar | Agregar link "[Nombre]" al sidebar |

---

## 9. Bugs / Inconsistencias del Sistema a Resolver en esta US

> Referenciar los bugs activos de `guia_base_ux_ui.md` sección 3 si esta US los toca.

| Bug | Ubicación | Acción en esta US |
|-----|-----------|-------------------|
| [Nombre del bug] | [archivo] | Corregir / No aplica / Posponer |

---

## 10. Notas y Decisiones de Diseño

> Registrar decisiones tomadas durante el diseño que no son obvias, para contexto futuro.

- **[Decisión 1]:** [Razón por la que se eligió este enfoque sobre otras alternativas]
- **[Decisión 2]:** [...]

---

*Actualizar este documento si el diseño cambia durante la implementación.*
