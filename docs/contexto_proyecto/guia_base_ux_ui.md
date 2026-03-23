# Guía Base UX/UI — Front-API / LaravelCiaf

> **Versión:** 1.1 — Marzo 2026
> **Stack:** PHP 8.2 + Laravel 12 + TALL Stack (Tailwind CSS 3, Alpine.js 3, Livewire 3.7) + MySQL 8
> **Documento vivo:** Actualizar tras cada sprint que modifique componentes visuales o flujos de navegación.

---

## Tabla de Contenidos

1. [Inventario de Vistas y Componentes](#1-inventario-de-vistas-y-componentes)
2. [Flujos de Navegación](#2-flujos-de-navegación)
3. [Inconsistencias y Bugs Activos](#3-inconsistencias-y-bugs-activos)
4. [Sistema de Diseño (Design Tokens)](#4-sistema-de-diseño-design-tokens)
5. [Librería de Componentes UI](#5-librería-de-componentes-ui)
6. [Principios de UX y Directrices de Interacción](#6-principios-de-ux-y-directrices-de-interacción)

---

## 1. Inventario de Vistas y Componentes

### Layouts

| Archivo | Propósito | Contexto |
|---------|-----------|----------|
| `layouts/app.blade.php` | Layout principal: sidebar + main content con fondo imagen | Todas las rutas autenticadas |
| `layouts/guest.blade.php` | Layout para pantallas de autenticación | `/login`, `/register`, etc. |
| `layouts/admin.blade.php` | Layout alternativo para admin | Rutas `/admin/*` |
| `layouts/navigation.blade.php` | Sidebar Programador (6 items + perfil) | — |
| `layouts/admin-navigation.blade.php` | Sidebar Admin/Manager | — |

### Vistas por Módulo

**Auth — Breeze:** `login`, `register`, `forgot-password`, `reset-password`, `verify-email`, `confirm-password`

**Dashboard:** `dashboard.blade.php` (contextual por rol) + 3 partials: `personal-stats`, `personal-pipeline`, `current-month`

**Programador (6):** `programmer/dashboard`, `api-dashboard`, `endpoints/create`, `integrations/create`, `integrations/{id}/configure`, `workflows/execute-request`

**Clientes (6):** `clients/index`, `create`, `edit`, `show`, `_form` (partial reutilizable), `transfer`

**Workflows (2):** `workflows/batch-show`, `workflows/test`

**Admin (13):** `admin/dashboard`, `api-dashboard`, `users/{index,create,edit}`, `api_services/{index,create,edit}`, `logs`, `maintenance`, `performance/index`, `stats`, `layout`

**Otras:** `profile/edit` + 3 partials, `users/{index,create,edit,show}`, `settings/index`, `errors/403`, `welcome`

**PDFs (14 en `resources/views/pdfs/`):**

| Archivo | Propósito |
|---------|-----------|
| `workflow-execution.blade.php` | Reporte de ejecución general |
| `workflow-conciliacion-pdf.blade.php` | Reporte conciliación (PDF principal) |
| `workflow-conciliacion-preview.blade.php` | Preview HTML en iframe |
| `conciliacion/main.blade.php` | Estructura principal del PDF |
| `conciliacion/enviar-sucursal.blade.php` | Reporte por sucursal |
| `conciliacion/enviar-egresos.blade.php` | Reporte egresos |
| `conciliacion/enviar-anulaciones.blade.php` | Reporte anulaciones |
| `conciliacion/enviar-no-conciliados.blade.php` | Reporte no conciliados |
| `conciliacion/preview.blade.php` | Preview en modal |
| `conciliacion/partials/styles.blade.php` | Estilos inline para PDF |
| `sucursal-report.blade.php` | Reporte sucursal standalone |
| `anulaciones-report.blade.php` | Reporte anulaciones standalone |
| `egresos-report.blade.php` | Reporte egresos standalone |
| `no-conciliados-report.blade.php` | Reporte no conciliados standalone |

**Vistas Livewire (en `resources/views/livewire/`):**

| Vista | Componente PHP | Propósito |
|-------|---------------|-----------|
| `workflow-file-upload-wizard.blade.php` | `WorkflowFileUploadWizard` | Wizard principal 4 pasos |
| `wizard-steps/step1-client-branch.blade.php` | (incluida en wizard) | Paso 1: Selección cliente/sucursal |
| `wizard-steps/step2-workflow.blade.php` | (incluida en wizard) | Paso 2: Tipo de workflow |
| `wizard-steps/step3-files.blade.php` | (incluida en wizard) | Paso 3: Carga de archivos |
| `wizard-steps/step4-confirm.blade.php` | (incluida en wizard) | Paso 4: Confirmación |
| `wizard-steps/progress-modal.blade.php` | (incluida en wizard) | Modal de progreso |
| `conciliation/dashboard.blade.php` | `ConciliationDashboard` | Listado de empresas con KPIs |
| `conciliation/detail.blade.php` | `ConciliationDetail` | Detalle con 10+ tabs |
| `conciliation/partials/search-bar.blade.php` | (parcial) | Barra de búsqueda |
| `conciliation/partials/sortable-header.blade.php` | (parcial) | Headers ordenables |
| `informes-pdf-wizard.blade.php` | `InformesPdfWizard` | Wizard selección/generación informes |
| `anulaciones-pdf-wizard.blade.php` | `AnulacionesPdfWizard` | Wizard anulaciones |
| `arqueo-wizard.blade.php` | `ArqueoWizard` | Wizard arqueo mensual |
| `workflow-history-table.blade.php` | `WorkflowHistoryTable` | Tabla paginada con filtros |
| `workflow-execution-panel.blade.php` | `WorkflowExecutionPanel` | Panel estado/ejecución |
| `buscador-clientes.blade.php` | `BuscadorClientes` | Buscador/filtro de clientes |

---

## 2. Flujos de Navegación

```
/ (welcome)
└── /login
    └── /dashboard  [rol-contextual]
        ├── [Programador] /programadores/dashboard
        │   ├── /programadores/clients
        │   │   ├── /clients/create
        │   │   ├── /clients/{id}/edit
        │   │   └── /clients/{id}/transfer
        │   │
        │   ├── /programadores/workflows/upload  [Livewire Wizard]
        │   │   └── Steps: 1.Cliente → 2.Workflow → 3.Archivos → 4.Confirm → ProgressModal
        │   │       └── [éxito] → /programadores/conciliacion/cliente/{client}
        │   │
        │   ├── /programadores/conciliacion
        │   │   ├── /conciliacion/dashboard  [ConciliationDashboard Livewire]
        │   │   │   └── Click empresa → /conciliacion/cliente/{client}
        │   │   └── /conciliacion/cliente/{client}  [ConciliationDetail Livewire]
        │   │       └── 10 tabs: sistema|arqueo|resumen|getnet|mp|caja|turnos|devoluciones|mp_negativos|no_conciliados
        │   │           └── Exports: /conciliacion/cliente/{client}/export/{type}
        │   │
        │   ├── /programadores/informes  [InformesPdfWizard Livewire]
        │   │   └── Selección cascada: Cliente → Sede → Fecha → Turno → Tipo → Preview iframe + Download
        │   │
        │   └── /programadores/arqueo  [ArqueoWizard Livewire]
        │       └── Selección cascada: Cliente → Sucursal → Año → Mes → ProgressModal → Resultado
        │
        └── [Admin] /admin/dashboard
            ├── /admin/users (CRUD)
            └── /admin/api-services (CRUD)

/profile  [Auth]
/settings  [Auth]
```

**Notas de navegación Livewire:**
- El wizard de workflows usa `$this->dispatch('...')` para emitir eventos, no `redirect()`
- `ConciliationDetail` y `ConciliationDashboard` persisten filtros en query string (`?activeTab=mp&search=...`)
- `WorkflowHistoryTable` persiste todos los filtros en URL via `#[Url]`
- No se usa `wire:navigate` — las navegaciones SPA-like son via Livewire full-page components

---

## 3. Inconsistencias y Bugs Activos

### Naming y convenciones

| Problema | Ubicación | Impacto |
|----------|-----------|---------|
| `ConciliacionDataService` (con ó) vs `ConciliationDataService` — servicio deprecado aún existe | `app/Services/` | ⚠️ Confusión en DI |
| Sidebar usa `@click.away` (deprecated) junto a `@click.outside` | `navigation.blade.php` línea 75 | ⚠️ Deprecado en Alpine v3 |
| `font-sans` configurado para 'Figtree' en tailwind.config pero `app.blade.php` carga Inter vía Google Fonts | Config vs HTML | ⚠️ Fuente efectiva no es la configurada |

### Tokens no definidos

| Clase usada | Archivo | Estado |
|-------------|---------|--------|
| `text-text-muted` | `components/kpi-card.blade.php` | ❌ No definida en tailwind.config.js |
| `text-text-main` | `components/kpi-card.blade.php` | ❌ No definida en tailwind.config.js |

### Bugs activos

| Bug | Ubicación | Descripción |
|-----|-----------|-------------|
| Font Awesome cargado **dos veces** | `layouts/app.blade.php` líneas 21-27 | Carga duplicada de 6.4.2 CDN |
| Hamburger mobile sin estado compartido | `layouts/app.blade.php` líneas 57-62 | `x-data="{ open: false }"` está en el `<button>`, no en el wrapper — el panel mobile nunca se muestra |
| Historial Workflows comentado en sidebar | `layouts/navigation.blade.php` líneas 36-40 | Ruta existe pero no está accesible desde la nav |

### Inconsistencias visuales temáticas

| Problema | Componente | Descripción |
|----------|-----------|-------------|
| KPI card con `bg-white` en app dark | `components/kpi-card.blade.php` | Rompe el tema oscuro Aurora Glass |
| Secondary button con `bg-white text-gray-700` | `components/secondary-button.blade.php` | Claro sobre fondo oscuro: contraste incorrecto |
| Focus ring en input usa `indigo-500` | `components/text-input.blade.php` | Debería usar `brand-accent` para coherencia |
| Primary button sin estado `disabled` | `components/primary-button.blade.php` | Falta `disabled:opacity-50 disabled:cursor-not-allowed` |

---

## 4. Sistema de Diseño (Design Tokens)

### 4.1 Paleta de Colores

#### Colores de Marca

| Token | Valor HEX | Clase Tailwind | Uso |
|-------|-----------|----------------|-----|
| `brand-dark` | `#0C263B` | `bg-brand-dark` | Sidebar, navbar, fondos principales oscuros |
| `brand-accent` | `#FE9192` | `text-brand-accent` | Links activos en sidebar, elementos destacados, avatar |
| `btn-start` | `#F7838F` | `from-btn-start` | Inicio del gradiente del botón primario |
| `btn-end` | `#FCB5AA` | `to-btn-end` | Fin del gradiente del botón primario |
| `primary` | `#0C263B` | (alias de brand-dark) | Compatibilidad legacy |
| `secondary` | `#FE9192` | (alias de brand-accent) | Compatibilidad legacy |

#### Colores Aurora (Tema Visual)

| Token | Valor HEX | Uso |
|-------|-----------|-----|
| `aurora-cyan` | `#06b6d4` | Decorativo — Cyan 500 |
| `aurora-pink` | `#ec4899` | Decorativo — Pink 500 |
| `aurora-purple` | `#a855f7` | Decorativo — Purple 500 |
| `light-text` | `#f3f4f6` | Texto claro sobre fondos oscuros |
| `light-text-muted` | `#9ca3af` | Texto secundario/muted oscuro |
| `dark-void` | `#0f172a` | Overlay de modales (Slate 900) |

#### Colores Semánticos (sin tokens custom — clases Tailwind estándar)

| Propósito | Clase usada | Valor |
|-----------|------------|-------|
| Error / Danger | `text-red-400`, `bg-red-900/10`, `border-red-300` | `#f87171` |
| Éxito / Success | `text-green-400`, `text-green-500` | `#4ade80` |
| Info | `text-blue-400` | `#60a5fa` |
| Warning | No detectado como patrón consistente | — |

#### Superficies

| Superficie | Clase usada | Descripción |
|-----------|-------------|-------------|
| Sidebar | `bg-brand-dark` (`#0C263B`) | Fijo |
| Contenido principal | Imagen de fondo `img/marketing-bg8.png` | Aurora Glass sobre imagen |
| Card / Panel | `bg-white/10 backdrop-blur-sm border border-white/20` | **Patrón Aurora Glass** |
| Modal backdrop | `bg-dark-void/70 backdrop-blur-md` | Glass oscuro |
| Modal content | `bg-gray-900/70 backdrop-blur-xl border border-white/10` | Glass oscuro |
| Dropdown | `bg-white/10 backdrop-blur-xl border border-white/20` | Aurora Glass |
| Header de página | `bg-white/80 backdrop-blur-sm` | Glass claro |

> **Patrón Aurora Glass:** `bg-{color}/{opacity}` + `backdrop-blur-{size}` + `border border-white/{opacity}`. Es el lenguaje visual dominante del proyecto.

---

### 4.2 Tipografía

#### Familias de fuente

| Variable | Familia | Uso |
|----------|---------|-----|
| `font-sans` | `Figtree` (config) / `Inter` (HTML efectivo) | Body, texto general |
| `font-headings` | `Poppins` | Títulos, encabezados (`font-headings`) |

> ⚠️ **Inconsistencia:** tailwind.config define `font-sans: 'Figtree'` pero se carga Inter. La fuente efectiva es Inter.

#### Escala tipográfica

| Tamaño | px | Contexto |
|--------|----|----------|
| `text-xs` | 12px | Labels, captions, badges |
| `text-sm` | 14px | Navegación, botones, table cells, errores |
| `text-base` | 16px | Cuerpo de texto general |
| `text-xl` | 20px | Títulos de cards, headers de página |
| `text-2xl` | 24px | KPI values, títulos de sección |
| `text-3xl` | 30px | KPI values grandes |

#### Pesos

| Peso | Clase | Uso |
|------|-------|-----|
| Regular (400) | `font-normal` | Texto de tabla, descripciones |
| Medium (500) | `font-medium` | Texto de navegación, labels |
| Semibold (600) | `font-semibold` | Headers, nombres en sidebar |
| Bold (700) | `font-bold` | Botones, KPI values, títulos de cards |

---

### 4.3 Espaciado y Layout

| Contexto | Clase | Descripción |
|----------|-------|-------------|
| Sidebar | `w-64` fijo (256px) | No redimensionable |
| Contenido principal | `flex-1` | Ocupa espacio restante |
| Padding de páginas | `p-6` | En el `<main>` del layout |
| Ancho máximo de header | `max-w-7xl mx-auto` | Solo en el slot `$header` |
| Cards conciliación dashboard | `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3` | 3 columnas en lg |
| KPIs globales | `grid grid-cols-2 md:grid-cols-4` | 4 columnas en desktop |
| Gap entre cards | `gap-4`, `gap-6`, `space-y-4` | — |
| Padding interno de card | `p-6` | — |
| Padding sidebar nav items | `px-4 py-3` | — |

**Breakpoints en uso:**

| Nombre | Valor | Uso detectado |
|--------|-------|---------------|
| `sm` | 640px | Modales: `sm:max-w-*` |
| `md` | 768px | Sidebar: `md:block`; hamburger: `md:hidden`; grids |
| `lg` | 1024px | Grids de 3 columnas |

---

### 4.4 Decoración

#### Border Radius

| Uso | Clase |
|-----|-------|
| Cards principales (Aurora Glass) | `rounded-2xl` |
| Botón primario | `rounded-full` |
| Botón secundario | `rounded-lg` |
| Inputs | `rounded-md` |
| Nav links en sidebar | `rounded-lg` |
| Modales | `rounded-2xl` |
| Dropdown panel | `rounded-2xl` |

#### Sombras

| Uso | Clase |
|-----|-------|
| Cards | `shadow-lg` |
| Botón primario | `shadow-[0_4px_14px_0_rgba(247,131,143,0.39)]` (glow rosa) |
| Modales | `shadow-2xl` |
| Nav link activo | `shadow-[0_0_15px_rgba(254,145,146,0.1)]` (glow sutil) |

#### Bordes y Glass

| Patrón | Clases | Uso |
|--------|--------|-----|
| Aurora Glass estándar | `border border-white/20` | Cards, dropdowns |
| Aurora Glass sutil | `border border-white/10` | Sidebar border, separadores en modal |

#### Iconografía

**Librería:** Font Awesome 6.4.2 (CDN) — ⚠️ cargado dos veces en `app.blade.php`.

**Convención en sidebar:** `<i class="fas fa-{nombre} w-6 text-center mr-3"></i>` dentro de nav-links.

**SVG Inline:** usado con `fill="currentColor"`. Tamaños: `h-5 w-5` (error icon), `h-6 w-6` (close button), `h-8 w-8` (decorativos en formularios).

---

## 5. Librería de Componentes UI

### 5.1 Componentes Livewire (`app/Livewire/`)

#### `WorkflowFileUploadWizard`
- **Vista:** `livewire/workflow-file-upload-wizard.blade.php` + sub-vistas en `wizard-steps/`
- **Props públicas:** `$currentStep`, `$selectedClientId`, `$selectedBranchId`, `$selectedWorkflowTypeId`, `$uploadedFiles[]`, `$fileMatches[]`, `$validationErrors[]`, `$arqueoFechaInicio/Fin`, `$isProcessing`, `$showProgressModal`, `$currentProgress`, `$progressPercentage`, `$failedStep`
- **Eventos emitidos:** `workflow-progress` (message, percentage), `workflow-error` (step, message)
- **Variantes:** detecta automáticamente `conciliacion`, `conciliacion_arqueo` o `arqueo`
- **Dependencias:** `FileValidationService`, `WorkflowExecutionService`, `WorkflowJsonGeneratorService`

#### `ConciliationDashboard`
- **Vista:** `livewire/conciliation/dashboard.blade.php`
- **Props públicas:** `$search` (string, persistida en URL)
- **Datos calculados en render:** KPIs globales, array `$clientsWithConciliation` con métricas por empresa

#### `ConciliationDetail`
- **Vista:** `livewire/conciliation/detail.blade.php`
- **Props públicas:** `$clientId`, `$client`, `$execution`, `$fechaInicio/Fin`, `$activeTab`, `$search`, `$filterTurno`, `$filterEstado`, `$filterMetodoPago`, `$filterTipo`, `$filterFecha`, `$perPage`, `$platformLeft/Right`, `$sortField`, `$sortDirection`
- **Query string:** todos los filtros + `activeTab` persistidos en URL
- **Tabs disponibles:** `sistema`, `arqueo`, `resumen`, `getnet`, `mp`, `caja`, `turnos`, `devoluciones`, `mp_negativos`, `no_conciliados`

#### `WorkflowHistoryTable`
- **Vista:** `livewire/workflow-history-table.blade.php`
- **Props públicas:** `$searchClient`, `$searchStatus`, `$dateFrom/To`, `$perPage` (default 10)
- **Métodos:** `downloadPdf($executionId)`, `clearFilters()`
- **Query string:** todos los filtros persistidos
- **⚠️ Link en sidebar comentado** — la ruta existe pero no es accesible desde la nav

#### `BuscadorClientes`
- **Vista:** `livewire/buscador-clientes.blade.php`
- **Props públicas:** `$search`, `$status` (active/inactive), `$perPage` (default 10)

#### `ArqueoWizard`
- **Vista:** `livewire/arqueo-wizard.blade.php`
- **Props públicas:** `$selectedClientId`, `$selectedBranchId`, `$selectedYear/Month`, `$isProcessing`, `$showProgressModal`, `$currentProgress`, `$progressPercentage`, `$failedStep`, `$availableMonths[]`
- **Lógica cascada:** `updatedSelectedClientId/BranchId()` → `loadAvailableMonths()`

#### `InformesPdfWizard`
- **Vista:** `livewire/informes-pdf-wizard.blade.php`
- **Props públicas:** `$selectedClientId`, `$selectedSedeId`, `$selectedDate`, `$selectedTurno`, `$selectedReport`, `$availableSedes/Dates/Turnos[]`, `$dataCount`, `$previewUrl`
- **Lógica cascada:** Cliente → Sede → Fecha → Turno → Preview automático

#### `WorkflowExecutionPanel`
- **Vista:** `livewire/workflow-execution-panel.blade.php`
- **Props públicas:** `$batch` (WorkflowFileBatch), `$isExecuting`, `$execution`, `$errorMessage`
- **Eventos emitidos:** `execution-completed`

---

### 5.2 Componentes Blade Anónimos (`resources/views/components/`)

#### `<x-card>`
- **Props:** `title` (string|null) — **Slots:** `$slot`, `$header`, `$footer`
- **Clases base:** `bg-white/10 backdrop-blur-sm border border-white/20 rounded-2xl shadow-lg text-white`

#### `<x-modal name="" maxWidth="">`
- **Props:** `name` (requerido), `show` (bool, default false), `maxWidth` (sm|md|lg|xl|2xl|3xl|4xl, default 2xl)
- **Features:** Focus trap, keyboard nav (Tab/Esc), overlay glass, animaciones, body scroll lock
- **Apertura:** `$dispatch('open-modal', 'nombre-modal')` desde cualquier componente

#### `<x-primary-button>`
- **Clases:** `bg-gradient-to-r from-btn-start to-btn-end hover:to-btn-start text-white rounded-full font-bold text-sm uppercase tracking-widest shadow-[0_4px_14px_0_rgba(247,131,143,0.39)] transition-all hover:scale-105`
- **⚠️ Sin estado disabled** — falta `disabled:opacity-50 disabled:cursor-not-allowed disabled:transform-none`

#### `<x-secondary-button>`
- **Clases:** `bg-white border border-gray-300 rounded-lg text-gray-700 hover:bg-pink-50 hover:text-pink-500 hover:border-pink-300`
- **⚠️ Problema:** Fondo blanco sobre tema dark — inconsistente con Aurora Glass

#### `<x-text-input>`
- **Props:** `disabled` (bool, default false)
- **Clases:** `border-gray-300/50 bg-white/60 backdrop-blur-md text-gray-900 focus:border-indigo-500 focus:ring-indigo-500 rounded-md`
- **⚠️ Focus ring usa `indigo-500`** en lugar de `brand-accent`

#### `<x-dropdown align="" width="" contentClasses="">`
- **Props:** `align` (left|top|right, default right), `width` (48|56|64, default 48)
- **Slots:** `$trigger`, `$content`
- **Panel:** `bg-white/10 backdrop-blur-xl border border-white/20 rounded-2xl text-white`
- **Animación:** scale + opacity (200ms enter / 75ms leave)

#### `<x-kpi-card title="" value="" icon="">`
- **Clases base:** `bg-white border border-gray-200 rounded-lg shadow-sm p-6`
- **❌ Bug:** Usa `text-text-muted` y `text-text-main` (no definidos en tailwind.config)
- **⚠️ Inconsistencia:** `bg-white` sobre fondo dark rompe Aurora Glass

#### `<x-nav-link :href="" :active="">`
- **Estado activo:** `font-bold bg-white/10 text-brand-accent shadow-[0_0_15px_rgba(254,145,146,0.1)]`
- **Estado inactivo:** `font-medium text-gray-400 hover:text-white hover:bg-white/5`

#### `<x-input-error :messages="">`
- **Clases:** `text-sm text-red-400 flex items-start space-x-2` + ícono SVG de alerta

#### Otros componentes Blade
- `<x-input-label>`, `<x-danger-button>`, `<x-auth-session-status>`, `<x-application-logo>`, `<x-smart-date>`, `<x-lead-reminders>`, `<x-responsive-nav-link>`

---

### 5.3 Patrones Alpine.js

| Patrón | Descripción | Usado en |
|--------|-------------|----------|
| Dropdown | `x-data="{ open: false }"` en wrapper + `@click.outside="open = false"` + `x-show` con `x-transition` scale+opacity | `components/dropdown.blade.php`, `navigation.blade.php` |
| Modal con Focus Trap | `x-data` con métodos `focusables()`, `nextFocusable()`, `prevFocusable()` + listeners `keydown.escape`, `keydown.tab`, `keydown.shift.tab` | `components/modal.blade.php` |
| Toggle Mobile (buggy) | `x-data="{ open: false }"` en el `<button>` — el estado no es accesible fuera del botón | `layouts/app.blade.php` líneas 57-62 |

**Componentes faltantes / no implementados:**
- ❌ Toast/Notification system
- ❌ Skeleton loaders
- ❌ Confirm dialog para acciones destructivas
- ❌ Empty states estandarizados
- ❌ Breadcrumbs
- ❌ Badge/Pill component reutilizable
- ❌ Spinner/Loading component standalone

**Candidatos a unificar:**
- Profile dropdown en `navigation.blade.php` replica `<x-dropdown>` manualmente — usar el componente
- `WorkflowFileUploadWizard`, `ArqueoWizard` comparten lógica idéntica de progress modal — candidato a `<x-progress-modal>`
- `AnulacionesPdfWizard` e `InformesPdfWizard` son casi idénticos

---

## 6. Principios de UX y Directrices de Interacción

### 6.1 Feedback al Usuario

| Mecanismo | Estado | Notas |
|-----------|--------|-------|
| Flash messages (session) | ⚠️ Sin área de renderizado consistente en `app.blade.php` | `auth-session-status` solo cubre auth |
| `wire:loading` en botones | ⚠️ Parcial — solo en wizard de workflows | No hay spinner centralizado |
| Estados vacíos | ❌ No hay componente estándar | Cada vista los maneja por separado o no los implementa |
| Validación server-side | ✅ `@error('campo')` + `<x-input-error>` funcional | — |

### 6.2 Accesibilidad (a11y)

| Área | Estado | Nivel WCAG |
|------|---------|-----------|
| Focus trap en modales | ✅ `components/modal.blade.php` | AA |
| Keyboard nav (Esc, Tab en modal) | ✅ | AA |
| Labels en formularios | ✅ via `<x-input-label>` | AA |
| `aria-current="page"` en nav links | ❌ No implementado | A |
| `sr-only` en íconos Font Awesome | ❌ No detectado | AA |
| Skip links | ❌ No implementados | AA |
| Contraste de color | ⚠️ Revisar pares críticos | AA |
| Navegación por teclado en dropdowns | ⚠️ Sin flechas Arriba/Abajo | AA |

**Pares de contraste a verificar (WCAG AA: mínimo 4.5:1 texto normal, 3:1 texto grande):**

| Texto | Fondo | Riesgo |
|-------|-------|--------|
| `text-gray-400` (#9ca3af) | `bg-brand-dark` (#0C263B) | ⚠️ Probablemente bajo |
| `text-brand-accent` (#FE9192) | `bg-white/10` sobre imagen | ⚠️ Depende del fondo imagen |
| `text-light-text-muted` (#9ca3af) | `bg-gray-900/70` modal | ⚠️ Verificar |

### 6.3 Responsive

| Elemento | Mobile | Tablet (md) | Desktop (lg+) |
|----------|--------|------------|---------------|
| Sidebar | ❌ Oculto, hamburger buggy | ✅ Visible | ✅ Visible |
| Grids de cards | 1 columna | 2 columnas | 3 columnas |
| Tablas de datos | ⚠️ Sin `overflow-x-auto` en todas las vistas | Parcial | ✅ |
| Modales | `px-4` reducido | `sm:max-w-*` aplica | ✅ |

**Estrategia:** Mobile-first en breakpoints (`md:hidden`, `md:block`) pero el contenido no está optimizado para uso real en mobile.

### 6.4 Microinteracciones

| Interacción | Implementación | Estado |
|-------------|---------------|--------|
| Hover en nav links | `hover:bg-white/5 hover:text-white transition-all duration-200` | ✅ |
| Hover en botón primario | `hover:scale-105 hover:to-btn-start transition-all ease-in-out duration-300` | ✅ |
| Hover en secondary button | `hover:bg-pink-50 hover:text-pink-500 hover:border-pink-300 hover:shadow-md` | ✅ |
| Animaciones de dropdown | `x-transition` scale + opacity (200ms / 75ms) | ✅ |
| Animaciones de modal | `x-transition` scale + opacity + translate (300ms / 200ms) | ✅ |
| `x-cloak` | ❌ No detectado — puede causar flash de contenido Alpine | ⚠️ |
| `wire:loading` visual | Parcial — botones en wizard | ⚠️ |
| `wire:navigate` | ❌ No en uso — navegaciones con full-page reload | — |

---

*Fin del documento — Actualizar con cada sprint que modifique flujos o componentes visuales.*
