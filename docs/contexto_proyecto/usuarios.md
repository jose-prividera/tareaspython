# Usuarios del Sistema — Front-API / LaravelCiaf

## Credenciales de Demo

| Email | Contraseña | Rol |
|-------|-----------|-----|
| `admin@example.com` | `password` | Super Admin |
| `analista@example.com` | `password` | Programador |
| `user@example.com` | `password` | Operador |
| `maria.gonzalez@demo.com` | `password123` | Operador |

---

## Roles y Alcance

### 1. Super Admin

**Descripción:** Administrador técnico/IT con acceso total al sistema.

**Alcance:**
- Gestión completa de usuarios (crear, editar, eliminar, asignar roles)
- Gestión de teams (crear, activar, desactivar)
- Visibilidad global de todos los clientes, sin restricción de team
- Acceso al dashboard administrativo con métricas del sistema
- Todas las operaciones de todos los demás roles
- No puede eliminarse a sí mismo si es el último admin (protección en `UserController`)

**Rutas:** `/admin/*`

**Permisos Spatie:** Wildcard — todos los permisos del sistema.

---

### 2. Manager (Gerente)

**Descripción:** Rol gerencial con capacidad de administrar clientes y configurar adaptadores, sin poder gestionar usuarios.

**Alcance:**
- CRUD de clientes (incluyendo soft delete y restauración)
- Reasignar clientes entre usuarios
- Configurar adaptadores bancarios
- No puede crear ni modificar usuarios
- No puede gestionar teams

**Rutas:** `/admin/*` (comparte panel con Super Admin)

**Permisos Spatie:**
- `view clients`, `create clients`, `edit clients`, `delete clients`
- `restore clients`, `reassign clients`, `configure adapters`

---

### 3. Programador (Analista)

**Descripción:** Especialista técnico que recibe solicitudes de workflows de los Operadores, las activa y ejecuta los análisis financieros.

**Alcance:**
- Revisar y activar solicitudes de workflow enviadas por Operadores (`/programadores/solicitudes`)
- Cargar y ejecutar workflows (conciliación, arqueo, informes)
- Ver historial completo de ejecuciones
- Generar reportes PDF (conciliación, anulaciones, arqueo, informes)
- Acceder al dashboard de conciliación con métricas detalladas
- Gestionar credenciales bancarias (encriptadas)
- Visibilidad **global** de todos los clientes (sin scoping de team)
- No está asignado a ningún team

**Rutas:** `/programadores/*`

**Permisos Spatie:**
- `view clients`, `create clients`, `edit clients`, `reassign clients`
- `manage credentials`, `configure adapters`

**Componentes Livewire principales:**
- `WorkflowFileUploadWizard` — Wizard 4 pasos de carga
- `WorkflowHistoryTable` — Historial de ejecuciones
- `ConciliationDashboard`, `ConciliationDetail` — Análisis de conciliación
- `InformesPdfWizard`, `ArqueoWizard` — Generación de reportes
- `Programmer\WorkflowRequestInbox` — Bandeja de solicitudes entrantes
- `Programmer\WorkflowRequestReview` — Revisión y activación de solicitudes

---

### 4. Operador

**Descripción:** Usuario operativo perteneciente a un team. Gestiona las sucursales asignadas a su team, configura adaptadores bancarios y solicita workflows al Programador.

**Alcance:**
- **Scoping obligatorio por team:** solo ve clientes/sucursales asignados a su team
- Configurar adaptadores bancarios (Mercado Pago, Getnet, etc.) para sus sucursales
- Subir archivos Excel manualmente vía RPA
- Ver y completar tareas asignadas (TaskBoard)
- Crear y hacer seguimiento de solicitudes de workflow
- No puede ejecutar workflows directamente
- No puede ver clientes fuera de su team
- No puede generar reportes PDF

**Rutas:** `/operador/*`

**Middleware adicional:** `team.scope` (`EnsureTeamClientScope`) — filtra automáticamente todas las queries por `team_id`

**Permisos Spatie:**
- `view clients`, `create clients`, `edit clients`, `configure adapters`

**Componentes Livewire principales:**
- `Operator\ClientBranchList` — Listado de sucursales del team
- `Operator\AdapterConfigWizard` — Configuración de adaptadores
- `Operator\RpaManualUpload` — Subida manual de Excel
- `Operator\TaskBoard` — Tablero de tareas asignadas
- `Operator\WorkflowRequestWizard` — Crear solicitud de workflow
- `Operator\WorkflowRequestList`, `WorkflowRequestDetail` — Seguimiento de solicitudes

---

## Matriz de Permisos

| Permiso | Super Admin | Manager | Programador | Operador |
|---------|:-----------:|:-------:|:-----------:|:--------:|
| Gestionar usuarios | ✓ | | | |
| Gestionar teams | ✓ | | | |
| Ver clientes (global) | ✓ | ✓ | ✓ | |
| Ver clientes (solo team) | | | | ✓ |
| Crear clientes | ✓ | ✓ | ✓ | ✓ |
| Editar clientes | ✓ | ✓ | ✓ | ✓ |
| Eliminar/restaurar clientes | ✓ | ✓ | | |
| Reasignar clientes | ✓ | ✓ | ✓ | |
| Gestionar credenciales | ✓ | | ✓ | |
| Configurar adaptadores | ✓ | ✓ | ✓ | ✓ |
| Ejecutar workflows | ✓ | | ✓ | |
| Ver conciliación (análisis) | ✓ | | ✓ | |
| Generar PDFs | ✓ | | ✓ | |
| Crear solicitudes de workflow | | | | ✓ |
| Revisar/activar solicitudes | ✓ | | ✓ | |
| Ver tablero de tareas | | | | ✓ |
| Subir archivos RPA | | | | ✓ |

---

## Cómo se Conectan los Roles

```
┌─────────────────────────────────────────────────────────┐
│                      SUPER ADMIN                        │
│  • Crea usuarios y les asigna roles                     │
│  • Crea Teams y asigna Operadores + Clientes/Sucursales  │
└────────────────────┬────────────────────────────────────┘
                     │ crea y configura
          ┌──────────▼──────────┐
          │        TEAM         │
          │  (contiene N ops)   │
          │  (tiene N sucursal) │
          └──┬──────────────┬───┘
             │              │
    ┌─────────▼────┐   ┌────▼──────────────────────┐
    │  OPERADORES  │   │  CLIENTES / SUCURSALES     │
    │              │   │  (asignadas al team)       │
    └─────┬────────┘   └────────────────────────────┘
          │
          │ 1. Configura adaptadores bancarios para sucursales
          │ 2. Sube Excels vía RPA
          │ 3. Completa tareas asignadas
          │ 4. Crea Solicitud de Workflow
          │
          │           Solicitud (Pendiente)
          ▼─────────────────────────────────────────►
                                            ┌────────────────┐
                                            │  PROGRAMADOR   │
                                            │  (Analista)    │
                                            │                │
                                            │ • Revisa solic │
                                            │ • Activa workflow│
                                            │ • Ejecuta análisis│
                                            │ • Genera PDFs  │
                                            └────────────────┘
```

### Flujo de una Solicitud de Workflow

```
Operador                    Programador
   │                             │
   ├─ Crea solicitud ──────────► │
   │  (estado: Pendiente)        │
   │                             ├─ Revisa adjuntos
   │                             ├─ Hace dry run (opcional)
   │                             ├─ Completa checklist
   │◄── Solicitud EnRevision ────┤
   │                             │
   │◄── Solicitud Activa ────────┤ (workflow ejecutado)
   │                             │
   └─ Ve estado y resultados     └─ Genera PDFs / exporta
```

### Estados de una Solicitud de Workflow

```
Pendiente → EnRevision → Activo
                       ↘ Anulada
```

---

## Relaciones entre Modelos Clave

```
User ──────────── team_id ──────────► Team
 │                                     │
 │ (Operador)                          ├── operators: HasMany(User)
 │                                     └── clients:   BelongsToMany(Client)
 │                                              via pivot: team_client_branches
 │
 ├── clients: HasMany(Client)         Client
 │                                     ├── parent_id → Client (sede → sucursal)
 │                                     └── teams: BelongsToMany(Team)
 │
 └── assignedTasks: HasMany(Task)     Task
                                       ├── team_id → Team
                                       └── adapter → ClientCredential
                                                       └── branch → Client
                                                       └── apiService → ApiService (banco)

WorkflowRequest
 ├── user_id → User (Operador que creó)
 ├── analyst_id → User (Programador asignado)
 ├── team_id → Team
 ├── client_id → Client
 ├── branch_id → Client (sucursal específica)
 └── adapters: BelongsToMany(ClientCredential)
```

---

## Notas de Seguridad

- Las credenciales bancarias (`ClientCredential.credentials`) están **siempre encriptadas**. Nunca se exponen en logs ni en UI post-guardado.
- El RPA se autentica con tokens separados (`rpa.auth` middleware) — no usa credenciales de usuario.
- El middleware `team.scope` garantiza que un Operador **nunca pueda ver clientes de otro team**, sin importar la URL.
- Un Super Admin no puede eliminarse a sí mismo si es el último administrador del sistema.
