# Diseño UX/UI — US-0.3: Asignación de Clientes a Teams

> **Referencia de diseño:** [`docs/guia_base_ux_ui.md`](guia_base_ux_ui.md)
> **PRD:** [`docs/PRD_MAESTRO.md`](PRD_MAESTRO.md) — Bloque 0 (Roles v2: Admin → Team → Operador)
> **Sprint:** Sprint 0 | **Estado:** Diseño
> **Autor:** | **Fecha:** Marzo 2026

---

## 1. Alcance de la Pantalla / Flujo

**Vista(s) afectadas:**

| Vista | Archivo | Acción |
|-------|---------|--------|
| Panel de Clientes (Admin) | `resources/views/admin/clients/index.blade.php` | Nueva |
| Crear Cliente (Admin) | `resources/views/admin/clients/create.blade.php` | Nueva |
| Perfil de Cliente (Admin) | `resources/views/admin/clients/show.blade.php` | Nueva |
| Sidebar Admin | `resources/views/layouts/admin-navigation.blade.php` | Modificada — agregar link "Clientes" |

**Flujo de navegación:**
```
/admin/dashboard
└── /admin/clients                        ← panel consolidado (AC-0.3.7)
    ├── /admin/clients/create             ← crear cliente + sucursal inicial (AC-0.3.1)
    └── /admin/clients/{client}           ← perfil: sucursales + teams asignados
        ├── Modal: Agregar/Editar sucursal       (AC-0.3.2)
        ├── Modal: Asignar a team (team + sucursales) (AC-0.3.3, AC-0.3.4)
        └── Modal: Confirmar desvinculación (con alerta bloqueante) (AC-0.3.6)
```

**Componentes Livewire involucrados:**
- Clase PHP: `app/Livewire/Admin/ClientList.php` (nuevo)
- Vista: `resources/views/livewire/admin/client-list.blade.php` (nueva)
- Clase PHP: `app/Livewire/Admin/ClientProfile.php` (nuevo)
- Vista: `resources/views/livewire/admin/client-profile.blade.php` (nueva)

---

## 2. Estructura Visual

### 2.1a Layout y Zonas — Vista `/admin/clients` (AC-0.3.7)

- **Layout base:** `layouts/admin.blade.php`
- **Slot `$header`:** "Clientes" + breadcrumb: Admin → Clientes + botón [+ Nuevo Cliente] alineado a la derecha

```
┌──────────────────────────────────────────────────────────┐
│  Header: "Clientes"                  [+ Nuevo Cliente]   │
├──────────────────────────────────────────────────────────┤
│  Filtros: [Team ▼]  [Estado ▼]  [Buscar razón social…]  │
├──────────────────────────────────────────────────────────┤
│  Tabla:                                                   │
│  ┌──────────┬──────────┬──────────────┬────────┬───────┐ │
│  │ Cliente  │ CUIT     │ Teams        │ Sucs.  │ Adap. │ │
│  ├──────────┼──────────┼──────────────┼────────┼───────┤ │
│  │ ACME SA  │ 30-…     │ [Team A]     │ 3      │ 2     │ │
│  │          │          │ [Team B]     │        │ [Ver] │ │
│  └──────────┴──────────┴──────────────┴────────┴───────┘ │
│  [Paginación]                                             │
└──────────────────────────────────────────────────────────┘
```

### 2.1b Layout y Zonas — Vista `/admin/clients/{client}` (perfil)

- **Layout base:** `layouts/admin.blade.php`
- **Slot `$header`:** [Razón Social] + CUIT + badge estado + breadcrumb: Admin → Clientes → [Nombre]

```
┌──────────────────────────────────────────────────────────┐
│  "ACME S.A." | CUIT: 30-xxxx-x | [Activo] [Editar info] │
├──────────────────────────────────────────────────────────┤
│  [Tab: Sucursales ●]  [Tab: Teams asignados]             │
├──────────────────────────────────────────────────────────┤
│  Tab "Sucursales":                                        │
│  ┌───────────────────────────────────────────────────┐   │
│  │ Nombre sucursal │ Dirección │ Estado │ Teams │ ✏️  │   │
│  │ ─────────────────────────────────────────────── │   │
│  │ Sucursal Norte  │ Av. …     │ [Act.] │[TeamA]│ ✏️ │   │
│  └───────────────────────────────────────────────────┘   │
│  [+ Agregar sucursal]                                     │
├──────────────────────────────────────────────────────────┤
│  Tab "Teams asignados":                                   │
│  ┌───────────────────────────────────────────────────┐   │
│  │ Team     │ Sucursales visibles    │ [Gestionar][×]│   │
│  │ ─────────────────────────────────────────────── │   │
│  │ Team A   │ Sucursal Norte, Sur    │ [Gestionar][×]│   │
│  └───────────────────────────────────────────────────┘   │
│  [+ Asignar a team]                                       │
└──────────────────────────────────────────────────────────┘
```

### 2.2 Grid y Espaciado

| Elemento | Clases Tailwind | Referencia guía |
|----------|----------------|-----------------||
| Contenedor principal | `p-6` | Padding estándar de página |
| Panel de tabla / perfil | `bg-white/10 backdrop-blur-sm border border-white/20 rounded-2xl shadow-lg p-6` | Aurora Glass |
| Tabs container | `border-b border-white/20 flex gap-6 mb-6` | — |
| Tab activo | `border-b-2 border-brand-accent text-brand-accent pb-3 text-sm font-medium` | Acento marca |
| Tab inactivo | `text-light-text-muted hover:text-light-text pb-3 text-sm transition-colors` | Muted |
| Fila de tabla | `flex items-center justify-between py-3 border-b border-white/10` | — |
| Filtros | `flex flex-wrap gap-3 mb-6` | — |
| Header de página | `bg-white/80 backdrop-blur-sm rounded-2xl p-4 mb-6 flex items-center justify-between` | Patrón header existente |

---

## 3. Componentes a Usar

### Componentes existentes a reutilizar

| Componente | Uso en esta pantalla | Props relevantes |
|------------|---------------------|-----------------||
| `<x-card>` | Panel de tabla y secciones | `title="Clientes"` |
| `<x-primary-button>` | "Nuevo Cliente", "Asignar a team", "Guardar sucursal" | `wire:click`, `wire:loading.attr="disabled"` |
| `<x-secondary-button>` | "Cancelar" en modales | — |
| `<x-danger-button>` | "Desvincular" (cuando no hay bloqueantes) | `wire:click="executeDetach"` |
| `<x-modal>` | Agregar/editar sucursal | `name="branch-form"` `maxWidth="lg"` |
| `<x-modal>` | Asignar a team | `name="assign-team"` `maxWidth="lg"` |
| `<x-modal>` | Confirmar desvinculación | `name="confirm-detach"` `maxWidth="md"` |
| `<x-text-input>` | Buscar clientes, campos del formulario de sucursal | `wire:model.live.debounce.300ms` |
| `<x-input-label>` | Labels de todos los formularios | `for`, `value` |
| `<x-input-error>` | Errores de validación inline | `:messages="$errors->get('campo')"` |
| `<x-team-badge>` | Badges de teams en tabla y filas de sucursales | `:team="$team"` — creado en US-0.2 |

### Componentes nuevos requeridos

| Componente | Archivo propuesto | Justificación |
|------------|------------------|---------------|
| `<x-branch-row>` | `components/branch-row.blade.php` | Fila reutilizable: nombre + dirección + badge estado + teams badge + botón editar. Usada en tab Sucursales del perfil. |
| `<x-client-status-badge>` | `components/client-status-badge.blade.php` | Badge Activo/Inactivo para clientes y sucursales. Semántica diferente a `<x-team-badge>`. |

### Componentes Livewire

| Componente | Tipo | Archivo |
|------------|------|---------||
| `ClientList` | Nuevo | `app/Livewire/Admin/ClientList.php` |
| `ClientProfile` | Nuevo | `app/Livewire/Admin/ClientProfile.php` |

**Props públicas de `ClientList`:**
```
$search          — string, buscar razón social/CUIT
$filterTeam      — int|null, filtrar por team
$filterStatus    — string: 'all' | 'active' | 'inactive'
```

**Props públicas de `ClientProfile`:**
```
$client              — Client model (inyectado via mount)
$activeTab           — string: 'branches' | 'teams'
$searchBranches      — string, buscar sucursales en el tab
$editingBranchId     — int|null, ID de sucursal en edición (null = crear nueva)
$branchForm          — array [name, address, is_active]
$selectedTeamId      — int|null, team seleccionado en modal asignar
$selectedBranchIds   — array, IDs de sucursales a asignar al team
$pendingDetachTeamId — int|null, team en proceso de desvinculación
$detachBlockers      — array, lista de adaptadores/workflows bloqueantes
```

**Métodos de `ClientProfile`:**
```
saveBranch()             — crea o actualiza sucursal (AC-0.3.2); valida nombre único dentro del cliente
deactivateBranch($id)    — soft-desactiva sucursal; bloquea si tiene adaptadores activos
openAssignModal()        — abre modal assign-team con $selectedTeamId y $selectedBranchIds vacíos
assignToTeam()           — persiste pivot team_client_branches (AC-0.3.3, AC-0.3.4)
confirmDetach($teamId)   — chequea bloqueantes via Service; abre confirm-detach con datos del caso A o B
executeDetach()          — desvincula si no hay bloqueantes; Service elimina registros pivot
```

---

## 4. Tokens de Diseño Aplicados

### Colores

| Elemento | Token / Clase | Valor |
|----------|---------------|-------|
| Fondo de paneles | `bg-white/10 backdrop-blur-sm border border-white/20` | Aurora Glass |
| Texto razón social | `text-light-text font-medium text-sm` | `#f3f4f6` |
| Texto CUIT / dirección | `text-light-text-muted text-xs` | `#9ca3af` |
| Badge "Activo" (cliente/sucursal) | `bg-green-900/20 text-green-400 border border-green-400/30` | Semántico éxito |
| Badge "Inactivo" | `bg-gray-900/20 text-gray-400 border border-gray-400/30` | Neutral |
| Tab activo | `border-b-2 border-brand-accent text-brand-accent` | `#FE9192` |
| Banner de bloqueo (AC-0.3.6 caso B) | `bg-yellow-900/20 border border-yellow-400/30 text-yellow-300 rounded-lg p-3` | Warning |
| Ícono de bloqueo | `text-red-400` | Error semántico |
| Sucursal desactivada en tabla | `opacity-50` sobre toda la fila | Deshabilitado |
| Checkbox seleccionado en modal | `accent-brand-accent` | `#FE9192` |

### Tipografía

| Elemento | Clases |
|----------|--------|
| Título de panel | `font-headings text-lg font-semibold text-light-text` |
| Razón social en tabla | `text-sm font-medium text-light-text` |
| CUIT / dirección | `text-xs font-normal text-light-text-muted` |
| Label de input | `text-sm font-medium text-light-text` |
| Texto de advertencia en modal | `text-sm text-light-text-muted` |
| Nombre destacado en advertencia | `font-semibold text-brand-accent` |
| Contador de sucursales/teams | `text-xs font-medium text-light-text-muted` |

### Bordes y superficies

| Elemento | Clases |
|----------|--------|
| Paneles principales | `rounded-2xl shadow-lg border border-white/20` |
| Modal | `rounded-2xl shadow-2xl` |
| Fila hover en tabla | `hover:bg-white/5 transition-colors duration-150` |
| Input focus | `focus:ring-brand-accent/50 border-white/20` |

---

## 5. Interacciones y Estados

### Estados de UI requeridos

| Estado | Elemento | Implementación |
|--------|----------|----------------|
| Loading tabla | `ClientList` (durante filtrado/búsqueda) | `wire:loading.class="opacity-50"` en el `<tbody>` |
| Loading asignación | Botón "Asignar a team" | `wire:loading.attr="disabled"` + spinner `wire:loading` |
| Loading desvinculación | Botón "Desvincular" | `wire:loading.attr="disabled" wire:target="executeDetach"` |
| Sin clientes | Panel de tabla | Empty state: `fas fa-building` + "No hay clientes registrados." + [+ Nuevo Cliente] |
| Sin sucursales | Tab Sucursales | Empty state: `fas fa-map-marker-alt` + "Este cliente no tiene sucursales." + [+ Agregar sucursal] |
| Sin teams asignados | Tab Teams | Empty state: `fas fa-users` + "Este cliente no está asignado a ningún team." + [+ Asignar a team] |
| Todas las sucursales ya asignadas al team | Modal assign-team | Mensaje: "Todas las sucursales ya están asignadas a este team." sin checkboxes |
| Bloqueo en desvinculación (AC-0.3.6) | Modal confirm-detach | Banner warning + lista de bloqueantes; solo botón [Cerrar] |
| Desvinculación libre | Modal confirm-detach | Texto estándar + [Cancelar] [Desvincular] |
| Sucursal desactivada | Fila en tab Sucursales | `opacity-50` en toda la fila + badge "Inactivo" + sin acciones de team |

### Modal: Asignar a Team (AC-0.3.3 / AC-0.3.4)

**Contenido del modal:**
```
Título: "Asignar cliente a team"
Ícono: fas fa-plus-circle (text-brand-accent)

Sección 1 — Seleccionar team:
  Dropdown de teams disponibles (excluye teams donde este cliente ya está asignado con esas sucursales)

Sección 2 — Seleccionar sucursales:
  Checkboxes de sucursales activas del cliente
  (cada sucursal ya asignada al team seleccionado aparece pre-marcada y deshabilitada)

Footer: "Un mismo cliente puede asignarse a múltiples teams con distintas sucursales."

Botones: [Cancelar] (secondary) | [Asignar sucursales seleccionadas] (primary)
```

### Modal: Confirmar Desvinculación (AC-0.3.6)

**Caso A — Sin bloqueantes:**
```
Título: "Desvincular cliente de team"
Ícono: fas fa-unlink (text-yellow-400)
Cuerpo: "Vas a desvincular a [CLIENTE] del Team [NOMBRE].
         Los operadores de ese team dejarán de ver este cliente y sus sucursales."
Botones: [Cancelar] (secondary) | [Desvincular] (danger)
```

**Caso B — Con bloqueantes activos:**
```
Título: "No se puede desvincular"
Ícono: fas fa-exclamation-circle (text-red-400)
Cuerpo: "Existen adaptadores o workflows activos vinculados a este cliente en [Team NOMBRE]:
         • [Adaptador X — Sucursal Y]
         • [Workflow Z — activo]
         Desactivalos primero para poder desvincular."
Botones: [Cerrar] (secondary — único botón disponible)
```

### Flujo de feedback al usuario

```
Admin crea cliente → [Guardar]
  → Validación: razón social requerida, CUIT formato XX-XXXXXXXX-X, al menos 1 sucursal
  → Éxito: redirect a /admin/clients/{nuevo} + flash "Cliente creado correctamente."
  → Error: mensajes de validación inline en el formulario

Admin agrega sucursal → modal branch-form → [Guardar sucursal]
  → wire:loading en botón
  → Éxito: cierra modal + fila nueva en tabla + flash "Sucursal agregada."
  → Error: validación inline (nombre único dentro del cliente)

Admin asigna a team → modal assign-team → selecciona team + sucursales → [Asignar]
  → wire:loading en botón
  → Éxito: cierra modal + fila nueva en tab "Teams asignados" + flash "Cliente asignado al team."

Admin desvincula → click [×] en fila del team → confirmDetach($teamId)
  → Backend chequea bloqueantes via ClientTeamAssignmentService
  → Caso A (libre): $dispatch('open-modal', 'confirm-detach') mostrando datos caso A
  → Caso B (bloqueado): $dispatch('open-modal', 'confirm-detach') mostrando datos caso B
  → Si confirma (caso A): executeDetach() + flash "Cliente desvinculado del team." + fila desaparece
```

---

## 6. Responsive

| Elemento | Mobile (< md) | Tablet (md) | Desktop (lg+) |
|----------|--------------|-------------|---------------|
| Tabla de clientes | `overflow-x-auto` + columna "Adaptadores" oculta | Teams abreviados (count) | ✅ Badges completos |
| Columna "Teams asignados" | Solo contador "2 teams" | Badges en una línea | Badges completos |
| Tabs (Sucursales / Teams) | Full width, texto centrado | ✅ Normal | ✅ Normal |
| Modal asignar a team | `px-4 py-6` compacto, scroll interno | `sm:max-w-lg` | `max-w-lg` |
| Filtros de tabla | Stack vertical (full width) | Fila horizontal | Fila horizontal |
| Botones de acción por fila | Solo ícono con `aria-label` | Ícono + texto abreviado | Ícono + texto completo |
| Tab sucursales — columna Teams | Oculta en mobile | Visible | ✅ Normal |

---

## 7. Accesibilidad

| Requisito | Implementación |
|-----------|----------------|
| Labels en todos los inputs del formulario | `<x-input-label for="...">` + `id` correspondiente en cada input |
| Checkboxes de sucursales en modal | `<label>` wrapeando checkbox + nombre visible de la sucursal |
| Tabs con roles ARIA | `role="tablist"` en container; `role="tab"` + `aria-selected` en cada tab; `role="tabpanel"` + `aria-labelledby` en cada panel |
| Botón "×" desvincular | `aria-label="Desvincular de [Team nombre]"` + `aria-hidden="true"` en el ícono FA |
| Modal caso B (bloqueante) | `role="alert"` en el bloque de advertencia para lectura inmediata por screen readers |
| Contraste textos muted sobre Aurora Glass | `text-light-text-muted` (#9ca3af) sobre `bg-white/10` — ratio ~4.6:1 ✅ |
| Focus visible en interactivos | Mantener `focus:ring` de Tailwind en inputs, botones y checkboxes |
| Estado deshabilitado | `disabled:opacity-50 disabled:cursor-not-allowed` en botón "Asignar" sin sucursales seleccionadas |

---

## 8. Vistas / Archivos a Crear o Modificar

| Archivo | Acción | Notas |
|---------|--------|-------|
| `resources/views/admin/clients/index.blade.php` | Crear | Contiene `<livewire:admin.client-list />` + botón [+ Nuevo Cliente] en header |
| `resources/views/admin/clients/create.blade.php` | Crear | Formulario POST estándar (sin Livewire): razón social, CUIT, sucursal inicial (AC-0.3.1) |
| `resources/views/admin/clients/show.blade.php` | Crear | Contiene `<livewire:admin.client-profile :client="$client" />` |
| `app/Livewire/Admin/ClientList.php` | Crear | Búsqueda + filtros team/estado + paginación Livewire |
| `resources/views/livewire/admin/client-list.blade.php` | Crear | Vista reactiva de tabla con filtros inline |
| `app/Livewire/Admin/ClientProfile.php` | Crear | Tabs: sucursales + teams. Modales: branch-form, assign-team, confirm-detach |
| `resources/views/livewire/admin/client-profile.blade.php` | Crear | Vista reactiva con tabs Alpine.js + modales |
| `resources/views/components/branch-row.blade.php` | Crear | Fila: nombre + dirección + badge estado + teams + [✏️] |
| `resources/views/components/client-status-badge.blade.php` | Crear | Pill Activo/Inactivo para cliente y sucursal |
| `app/Models/Client.php` | Modificar | Agregar: `branches()` hasMany (parent_id), `teams()` belongsToMany via pivot, scope `forTeam($teamId)` |
| `app/Models/Team.php` | Modificar | Agregar relación `clients()` via `team_client_branches` |
| `database/migrations/` | Crear | `create_team_client_branches_table` — columnas: id, team_id (FK), client_id (FK branch), unique(team_id, client_id) |
| `app/Http/Controllers/Admin/ClientController.php` | Crear | Métodos: index, create, store, show — full-page loads; lógica en Livewire |
| `app/Services/ClientTeamAssignmentService.php` | Crear | `assign(Client $client, Team $team, array $branchIds)`, `detach(Client $client, Team $team): DetachResult` con validación de bloqueantes |
| `app/Http/Middleware/EnsureTeamClientScope.php` | Crear | Para rutas Operador: aplica scope `forTeam()` en queries de Client |
| `routes/web.php` | Modificar | Agregar rutas: `GET /admin/clients`, `GET|POST /admin/clients/create`, `GET /admin/clients/{client}` |
| `resources/views/layouts/admin-navigation.blade.php` | Modificar | Agregar link "Clientes" en sidebar admin (después de "Teams") |
| `bootstrap/app.php` (o `Http/Kernel.php`) | Modificar | Registrar `EnsureTeamClientScope` en el grupo de middleware del rol Operador |

---

## 9. Bugs / Inconsistencias del Sistema a Resolver en esta US

| Bug | Ubicación | Acción en esta US |
|-----|-----------|-------------------|
| Secondary button con `bg-white` sobre tema dark | `components/secondary-button.blade.php` | Verificar fix de US-0.2 aplicado — esta US usa el componente en 3 modales |
| `ConciliacionDataService` (deprecado con acento) | `app/Services/` | No aplica en esta US — posponer |

---

## 10. Notas y Decisiones de Diseño

- **Pivot a nivel de sucursal (`team_client_branches`):** Se elige el pivot en nivel de sucursal (branch) y no de cliente padre porque AC-0.3.4 requiere visibilidad per-sucursal por team. El cliente padre es "visible" para un team si al menos una de sus sucursales está en el pivot. Alternativa descartada: pivot a nivel de cliente con JSON de branch IDs — no permite constraints de FK ni queries eficientes.

- **Formulario de creación sin Livewire:** `create.blade.php` usa POST estándar Laravel porque la creación de cliente es un formulario simple de un solo disparo (razón social + CUIT + 1 sucursal inicial). No hay interactividad reactiva que justifique Livewire. Alternativa descartada: wizard multi-paso — overkill para 3-4 campos.

- **Tabs con Alpine.js en `ClientProfile`:** Los tabs (Sucursales / Teams asignados) se implementan con Alpine.js (`x-data`, `x-show`, `:class`) porque son UI-only: ambos conjuntos de datos se cargan en `mount()` del componente. No hay carga diferida por tab. Alternativa descartada: rutas separadas por tab con query param — introduce URL state innecesario para contenido estático dentro del mismo contexto.

- **`EnsureTeamClientScope` como Middleware (no Global Scope):** El filtrado de clientes para Operadores se implementa como middleware, no como Eloquent Global Scope, para no afectar las queries del Admin. El middleware registra el team_id en el contexto del request; los componentes Livewire del Operador aplican el scope `forTeam()` explícitamente. Alternativa descartada: Global Scope con `auth()->user()->isAdmin()` check — acopla lógica de autorización al modelo y dificulta el testing.

- **Modal de desvinculación bifurcado (AC-0.3.6):** El chequeo de bloqueantes ocurre en `confirmDetach()` (antes de abrir el modal), no en `executeDetach()`. Esto permite mostrar dos UIs distintas en el mismo modal `confirm-detach` — caso A con acción destructiva y caso B solo informativo — sin condicionales complejos en la vista. La validación en `executeDetach()` es un fallback de seguridad en el Service.

- **`Admin/ClientController` separado del `ClientController` compartido:** Para no romper las rutas existentes del Operador/Programador, el controlador admin vive en `app/Http/Controllers/Admin/ClientController.php`. Las rutas admin tienen prefijo `/admin/clients` y el middleware de rol Admin. El `ClientController.php` raíz permanece sin cambios.

- **Sin `wire:navigate`:** Consistente con el resto del proyecto — la navegación entre `/admin/clients` y `/admin/clients/{client}` es full-page load.

---

*Actualizar este documento si el diseño cambia durante la implementación.*
