# Diseño UX/UI — US-0.2: Asignación de Operadores a Teams

> **Referencia de diseño:** [`docs/guia_base_ux_ui.md`](guia_base_ux_ui.md)
> **PRD:** [`docs/PRD_MAESTRO.md`](PRD_MAESTRO.md) — Bloque 0 (Roles v2: Admin → Team → Operador)
> **Sprint:** Sprint 0 | **Estado:** Diseño
> **Autor:** | **Fecha:** Marzo 2026

---

## 1. Alcance de la Pantalla / Flujo

**Vista(s) afectadas:**

| Vista | Archivo | Acción |
|-------|---------|--------|
| Detalle de Team (Admin) | `resources/views/admin/teams/show.blade.php` | Nueva |
| Índice de Teams (Admin) | `resources/views/admin/teams/index.blade.php` | Nueva |
| Sidebar del Operador | `resources/views/layouts/navigation.blade.php` | Modificada — agregar nombre de team |

**Flujo de navegación:**
```
/admin/dashboard
└── /admin/teams                          ← listado de teams
    └── /admin/teams/{team}               ← detalle + gestión de operadores
        ├── Modal: Confirmar reasignación  ← si operador ya tiene team
        └── Modal: Confirmar remoción      ← antes de desvincular
```

**Componente Livewire involucrado:**
- Clase PHP: `app/Livewire/Admin/TeamOperatorAssignment.php` (nuevo)
- Vista: `resources/views/livewire/admin/team-operator-assignment.blade.php` (nueva)

---

## 2. Estructura Visual

### 2.1 Layout y Zonas — Vista `/admin/teams/{team}`

- **Layout base:** `layouts/admin.blade.php`
- **Slot `$header`:** Nombre del team + breadcrumb: Admin → Teams → [Nombre del team]

**Organización del contenido:**

```
┌─────────────────────────────────────────────────────┐
│  Header: "Team: [Nombre]"  [fecha creación] [badge operadores activos]
├──────────────────────────┬──────────────────────────┤
│  Panel izquierdo (60%)   │  Panel derecho (40%)      │
│  "Operadores asignados"  │  "Agregar operadores"     │
│  ─────────────────────   │  ─────────────────────    │
│  [search inline]         │  [search inline]          │
│  ┌─────────────────────┐ │  [checkbox] Operador 1    │
│  │ Avatar Nombre Email │ │  [checkbox] Operador 2    │
│  │              [×]    │ │  [checkbox] Operador 3    │
│  └─────────────────────┘ │  ...                      │
│  (tabla con scroll)      │  [Asignar seleccionados]  │
└──────────────────────────┴──────────────────────────┘
```

### 2.2 Grid y Espaciado

| Elemento | Clases Tailwind | Referencia guía |
|----------|----------------|-----------------|
| Contenedor de dos paneles | `grid grid-cols-1 lg:grid-cols-5 gap-6` | Adaptación del patrón de grid |
| Panel izquierdo (asignados) | `lg:col-span-3` | — |
| Panel derecho (disponibles) | `lg:col-span-2` | — |
| Cada panel | `bg-white/10 backdrop-blur-sm border border-white/20 rounded-2xl shadow-lg p-6` | Patrón Aurora Glass |
| Fila de operador asignado | `flex items-center justify-between py-3 border-b border-white/10` | — |
| Header de página | `bg-white/80 backdrop-blur-sm rounded-2xl p-4 mb-6` | Patrón header existente |

---

## 3. Componentes a Usar

### Componentes existentes a reutilizar

| Componente | Uso en esta pantalla | Props relevantes |
|------------|---------------------|-----------------|
| `<x-card>` | Panel "Operadores asignados" y panel "Agregar operadores" | `title="Operadores asignados"` |
| `<x-primary-button>` | "Asignar seleccionados" | `wire:click="assignSelected"` `wire:loading.attr="disabled"` |
| `<x-secondary-button>` | "Cancelar" en modales | — |
| `<x-danger-button>` | "Remover" por fila en tabla de asignados | `wire:click="confirmRemove({{ $operatorId }})"` |
| `<x-modal>` | Modal confirmación de reasignación | `name="confirm-reassign"` `maxWidth="md"` |
| `<x-modal>` | Modal confirmación de remoción | `name="confirm-remove"` `maxWidth="md"` |
| `<x-text-input>` | Buscador inline en cada panel | `wire:model.live.debounce.300ms="searchAssigned"` |
| `<x-input-label>` | Label del buscador | — |
| `<x-nav-link>` | Link "Teams" en sidebar admin (nuevo item) | `:href="route('admin.teams.index')"` |

### Componentes nuevos requeridos

| Componente | Archivo propuesto | Justificación |
|------------|------------------|---------------|
| `<x-operator-row>` | `components/operator-row.blade.php` | Fila reutilizable: avatar + nombre + email + badge de team actual. Usada en ambos paneles y potencialmente en otras vistas de gestión de equipo. |
| `<x-team-badge>` | `components/team-badge.blade.php` | Pill/badge con nombre de team. Reutilizable en sidebar operador, tabla de usuarios y listado de teams. |

### Componentes Livewire

| Componente | Tipo | Archivo |
|------------|------|---------|
| `TeamOperatorAssignment` | Nuevo | `app/Livewire/Admin/TeamOperatorAssignment.php` |

**Props públicas del componente:**
```
$team          — Team model (inyectado via mount)
$searchAssigned — string, wire:model para filtrar panel izquierdo
$searchAvailable — string, wire:model para filtrar panel derecho
$selectedOperators[] — IDs seleccionados para bulk assign
$pendingOperatorId  — ID del operador en proceso de reasignación/remoción
$pendingTargetTeam  — nombre del team origen (para mensaje en modal)
```

**Métodos del componente:**
```
assignSelected()     — asigna $selectedOperators al team; dispara confirm-reassign si alguno ya tiene team
confirmReassign()    — confirma reasignación individual tras advertencia
removeOperator($id)  — abre modal confirm-remove con $pendingOperatorId
confirmRemove()      — ejecuta la desvinculación
```

---

## 4. Tokens de Diseño Aplicados

### Colores

| Elemento | Token / Clase | Valor |
|----------|---------------|-------|
| Fondo de ambos paneles | `bg-white/10 backdrop-blur-sm border border-white/20` | Aurora Glass |
| Texto nombre operador | `text-light-text font-medium text-sm` | `#f3f4f6` |
| Texto email / subtexto | `text-light-text-muted text-xs` | `#9ca3af` |
| Badge "Sin team" | `bg-red-900/20 text-red-400 border border-red-400/30` | Semántico error |
| Badge "Team activo" | `bg-green-900/20 text-green-400 border border-green-400/30` | Semántico éxito |
| Badge nombre de team | `bg-brand-accent/20 text-brand-accent border border-brand-accent/30` | Acento marca |
| Checkbox seleccionado | `accent-brand-accent` | `#FE9192` |
| Divider entre filas | `border-white/10` | Aurora Glass sutil |
| Texto advertencia en modal | `text-yellow-400` | Warning (Tailwind estándar) |

### Tipografía

| Elemento | Clases |
|----------|--------|
| Título del panel | `font-headings text-lg font-semibold text-light-text` |
| Nombre del operador en fila | `text-sm font-medium text-light-text` |
| Email / subtexto | `text-xs font-normal text-light-text-muted` |
| Texto del modal de advertencia | `text-sm text-light-text-muted` |
| Nombre del team en advertencia | `font-semibold text-brand-accent` |
| Contador "N operadores" | `text-xs font-medium text-light-text-muted` |

### Bordes y superficies

| Elemento | Clases |
|----------|--------|
| Paneles principales | `rounded-2xl shadow-lg border border-white/20` |
| Modal | `rounded-2xl shadow-2xl` |
| Fila hover en lista disponibles | `hover:bg-white/5 transition-colors duration-150` |
| Avatar placeholder | `rounded-full bg-brand-dark border border-white/20 w-8 h-8` |

---

## 5. Interacciones y Estados

### Estados de UI requeridos

| Estado | Elemento | Implementación |
|--------|----------|----------------|
| Loading asignación | Botón "Asignar seleccionados" | `wire:loading.attr="disabled"` + spinner `wire:loading` inline |
| Loading remoción | Botón "Remover" de la fila | `wire:loading.attr="disabled" wire:target="confirmRemove"` |
| Sin operadores asignados | Panel izquierdo | Empty state: ícono `fas fa-users` + "Este team no tiene operadores asignados." + hint "Buscá operadores disponibles en el panel derecho." |
| Sin operadores disponibles | Panel derecho | Empty state: "Todos los operadores están asignados a un team." |
| Sin resultados de búsqueda | Ambos paneles | "Sin resultados para '[término]'." |
| Checkbox seleccionado | Fila en panel derecho | `bg-white/5 border-l-2 border-brand-accent` |
| Bulk sin selección | Botón "Asignar seleccionados" | `disabled` + tooltip "Seleccioná al menos un operador" |

### Modal: Confirmar Reasignación (AC-0.2.3)

**Contenido del modal:**
```
Título: "Reasignar operador"
Ícono: fas fa-exclamation-triangle (text-yellow-400)
Cuerpo: "Este operador ya pertenece al Team [NOMBRE]. ¿Deseas reasignarlo a este team?
         Perderá acceso inmediato al team anterior."
Botones: [Cancelar] (secondary) | [Confirmar reasignación] (primary)
```

### Modal: Confirmar Remoción (AC-0.2.5)

**Contenido del modal:**
```
Título: "Remover operador"
Ícono: fas fa-user-times (text-red-400)
Cuerpo: "Al remover a [NOMBRE] del team quedará sin asignar y no podrá
         operar hasta ser asignado a un nuevo team."
Botones: [Cancelar] (secondary) | [Remover] (danger)
```

### Sidebar del Operador — Banner de team (AC-0.2.6)

**Implementación:** Antes del primer ítem de navegación en `layouts/navigation.blade.php`:

- Si tiene team → Badge con nombre: `<x-team-badge :team="auth()->user()->team" />`
- Si no tiene team → Banner de advertencia:
  ```
  bg-yellow-900/20 border border-yellow-400/30 rounded-lg p-3 text-xs text-yellow-300
  "No estás asignado a ningún team. Contactá a tu administrador."
  ```

### Flujo de feedback al usuario

```
Admin selecciona operadores → click "Asignar seleccionados"
  → wire:loading activo en botón
  → Backend verifica si alguno ya tiene team
  → Caso A — Ninguno tiene team:
      Asignación directa → flash success "N operadores asignados al team."
      Panel izquierdo se actualiza con los nuevos operadores
  → Caso B — Alguno ya tiene team:
      $dispatch('open-modal', 'confirm-reassign')
      Modal muestra operadores con conflicto
      Confirmar → reasignación + log + flash success
      Cancelar → cierra modal, sin cambios
Admin click "Remover"
  → $dispatch('open-modal', 'confirm-remove')
  → Confirmar → desvinculación + log + fila desaparece del panel izquierdo
  → Cancelar → cierra modal, sin cambios
```

---

## 6. Responsive

| Elemento | Mobile (< md) | Tablet (md) | Desktop (lg+) |
|----------|--------------|-------------|---------------|
| Dos paneles | Stack vertical (full width) | Stack vertical | Side by side (60/40) |
| Tabla de asignados | `overflow-x-auto` + columnas reducidas (ocultar email) | Email visible | ✅ Normal |
| Botones de acción por fila | Solo ícono (`fas fa-times`) con tooltip | Ícono + texto abreviado | Ícono + "Remover" |
| Modal | `px-4 py-6` reducido | `sm:max-w-md` | `max-w-md` |
| Checkboxes bulk | Visibles, táctil-friendly (`h-5 w-5`) | ✅ Normal | ✅ Normal |

---

## 7. Accesibilidad

| Requisito | Implementación |
|-----------|----------------|
| Labels en buscadores | `<x-input-label for="search-assigned">` + `id` en el input |
| Checkboxes con label | `<label>` wrapeando checkbox + nombre del operador |
| Ícono "×" en remover | `aria-hidden="true"` + `aria-label="Remover [nombre]"` en el botón |
| Modal focus trap | Heredado de `<x-modal>` — ya implementado |
| Advertencia de reasignación | `role="alert"` en el contenido del modal |
| Badge de team en sidebar | Texto visible, no solo color |
| Contraste badges | Verificar `text-brand-accent` (#FE9192) sobre `bg-brand-accent/20` — ratio ~3.5:1 (texto grande OK) |

---

## 8. Vistas / Archivos a Crear o Modificar

| Archivo | Acción | Notas |
|---------|--------|-------|
| `resources/views/admin/teams/index.blade.php` | Crear | Listado de teams con cantidad de operadores por team |
| `resources/views/admin/teams/show.blade.php` | Crear | Contiene el componente Livewire `<livewire:admin.team-operator-assignment :team="$team" />` |
| `app/Livewire/Admin/TeamOperatorAssignment.php` | Crear | Lógica de asignación, búsqueda, bulk select, modales |
| `resources/views/livewire/admin/team-operator-assignment.blade.php` | Crear | Vista reactiva de dos paneles |
| `resources/views/components/operator-row.blade.php` | Crear | Avatar + nombre + email + badge team |
| `resources/views/components/team-badge.blade.php` | Crear | Pill reutilizable con nombre de team |
| `resources/views/layouts/navigation.blade.php` | Modificar | Banner de team del operador (antes del primer nav-item) |
| `resources/views/layouts/admin-navigation.blade.php` | Modificar | Agregar link "Teams" en sidebar admin |
| `routes/web.php` | Modificar | Agregar rutas `GET /admin/teams`, `GET /admin/teams/{team}` |
| `app/Models/User.php` | Modificar | Agregar `team_id` (FK nullable) + relación `belongsTo(Team::class)` |
| `database/migrations/` | Crear | `add_team_id_to_users_table` con FK nullable + `create_team_operator_logs_table` |
| `app/Models/Team.php` | Crear (o verificar si existe) | Relaciones: `hasMany(User::class)` (operadores) + `hasMany(Client::class)` |
| `app/Models/TeamOperatorLog.php` | Crear | Auditoría: `operator_id`, `from_team_id`, `to_team_id`, `admin_id`, `created_at` |
| `app/Services/TeamAssignmentService.php` | Crear | Lógica de asignación/reasignación/remoción + escritura del log |

---

## 9. Bugs / Inconsistencias del Sistema a Resolver en esta US

| Bug | Ubicación | Acción en esta US |
|-----|-----------|-------------------|
| Secondary button con `bg-white` sobre tema dark | `components/secondary-button.blade.php` | Corregir: cambiar a `bg-white/10 border border-white/20 text-light-text hover:bg-white/20` — los modales de esta US lo usan directamente |

---

## 10. Notas y Decisiones de Diseño

- **Dos paneles vs tabs:** Se eligió layout de dos paneles simultáneos (asignados + disponibles) en lugar de tabs para que el Admin pueda ver el estado actual y los disponibles en la misma vista, reduciendo clicks y contexto perdido al asignar.

- **Bulk assign primero, reasignación después:** El flujo bulk procesa primero los operadores sin conflicto, y luego presenta el modal de confirmación solo para los que tienen conflicto. Alternativa descartada: rechazar todo el bulk si hay un conflicto — demasiado disruptivo para lotes grandes.

- **Service layer para la asignación:** Toda la lógica de cambio de team + escritura del log va en `TeamAssignmentService`, no en el componente Livewire, para mantener las convenciones del proyecto y facilitar testing.

- **Sin `wire:navigate`:** Consistente con el resto del proyecto — la navegación a `/admin/teams/{team}` es un full-page load.

- **TeamOperatorLog vs usar un paquete de auditoría:** Se crea un modelo propio liviano en lugar de introducir `spatie/laravel-activitylog` para mantener el alcance acotado a esta entidad y evitar dependencias nuevas no acordadas.

- **`team_id` nullable en `users`:** Cumple AC-0.2.5 — el operador sin team tiene `team_id = null`. Se evita una tabla pivote porque la relación es 1 operador → 1 team (no M:M).

---

*Actualizar este documento si el diseño cambia durante la implementación.*
