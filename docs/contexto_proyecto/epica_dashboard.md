# Épica Dashboard: Gestión de Proyectos y Automatizaciones

**Versión:** 1.0 — 2026-03-23
**Estado:** Especificación
**Módulo asociado:** M-9 (Dashboard Programador + Métricas) — extensión
**Épica:** E-DASH

---

## 1. Descripción y Objetivo

### 1.1 Contexto

La plataforma Front-API/LaravelCiaf gestiona workflows de conciliación para múltiples clientes. A medida que escala la cantidad de procesos activos y empresas atendidas, el equipo de Programadores necesita una herramienta centralizada para hacer seguimiento del avance de implementación de cada proceso por empresa, sin tener que consultar ticket a ticket ni depender de planillas externas.

Esta épica define el **Dashboard de Gestión de Proyectos y Automatizaciones**: un panel interno orientado al equipo técnico (Programadores y Admin) que centraliza el estado de implementación de procesos por empresa, expone métricas de avance, y ofrece integración bidireccional con Google Sheets para colaboración asíncrona.

### 1.2 Objetivo

Proveer al equipo de Programadores una vista de gestión consolidada donde puedan:

1. Ver de un vistazo el estado global de todas las implementaciones activas.
2. Organizar el trabajo por **Proceso** (qué se está construyendo) y por **Empresa** (para quién).
3. Actualizar el avance y observaciones de cada implementación sin abandonar la plataforma.
4. Sincronizar datos con una hoja de cálculo Google Sheets para reportes y trabajo colaborativo offline.
5. Capturar snapshots visuales del panel para incluir en reportes o comunicaciones.

### 1.3 Usuarios objetivo

| Rol | Uso principal |
|-----|--------------|
| **Programador** | Consulta y actualización de avance diario |
| **Super Admin** | Supervisión global, gestión de entidades |
| **Manager** | Consulta de estado (solo lectura) |

---

## 2. Modelo de Datos

### 2.1 Entidades principales

#### `Proceso`

Representa un tipo de automatización o servicio que la empresa ofrece a sus clientes.

| Campo | Tipo | Restricciones |
|-------|------|---------------|
| `id` | UUID / string | PK, único |
| `name` | string(120) | obligatorio, único |
| `desc` | text | opcional |
| `color` | string(7) | hex color, default `#003F6E` |
| `is_active` | boolean | default true |
| `created_at` | timestamp | auto |
| `updated_at` | timestamp | auto |

**Valores de color disponibles:** Azul corporativo (`#003F6E`), Azul claro (`#0072c6`), Verde (`#16a34a`), Ámbar (`#b45309`), Naranja (`#c2410c`), Rojo (`#dc2626`), Violeta (`#7c3aed`), Cian (`#0891b2`).

#### `Empresa`

Representa una empresa cliente, grupo de empresas, o módulo funcional para el que se implementa uno o más procesos.

| Campo | Tipo | Restricciones |
|-------|------|---------------|
| `id` | UUID / string | PK, único |
| `name` | string(120) | obligatorio, único |
| `notes` | text | opcional |
| `is_active` | boolean | default true |
| `created_at` | timestamp | auto |
| `updated_at` | timestamp | auto |

> **Nota:** `Empresa` en este contexto es una entidad de agrupación interna del dashboard. No necesariamente coincide 1:1 con el modelo `Cliente` de M-2; puede representar grupos, rubros, proyectos internos o módulos funcionales.

#### `Implementacion`

Registro del estado de implementación de un `Proceso` para una `Empresa` específica.

| Campo | Tipo | Restricciones |
|-------|------|---------------|
| `id` | UUID / string | PK, único |
| `proc_id` | FK → Proceso | obligatorio |
| `emp_id` | FK → Empresa | obligatorio |
| `avance` | decimal(3,2) | 0.00 a 1.00, default 0 |
| `obs` | text | observaciones de estado, opcional |
| `created_at` | timestamp | auto |
| `updated_at` | timestamp | auto |

**Constraint:** par (`proc_id`, `emp_id`) único — no puede existir más de una implementación del mismo proceso para la misma empresa.

### 2.2 Relaciones

```
Proceso (1) ─────────────────────── (N) Implementacion
Empresa (1) ─────────────────────── (N) Implementacion
```

### 2.3 Cálculos derivados

- **Avance del proceso:** promedio de `avance` de todas las `Implementacion` asociadas al proceso.
- **Avance de la empresa:** promedio de `avance` de todas las `Implementacion` asociadas a la empresa.
- **Estado badge de implementación:**
  - `avance == 1.0` → **Completa** (verde)
  - `avance >= 0.75` → **Avanzada** (azul)
  - `avance >= 0.25` → **En curso** (ámbar)
  - `avance > 0` → **Iniciada** (naranja)
  - `avance == 0` → **Pendiente** (gris)

---

## 3. Vistas Principales

### 3.1 Panel Operativo (`/dashboard/panel`)

**Propósito:** Vista de aterrizaje. Muestra el resumen global del equipo y agrupa las implementaciones por estado para identificar rápidamente qué necesita atención.

**Elementos de la vista:**

- **Fila de estadísticas** (KPI cards):
  - Total implementaciones
  - Implementaciones completas (avance = 100%)
  - Implementaciones en curso (0% < avance < 100%)
  - Implementaciones pendientes (avance = 0%)

- **Secciones agrupadas por estado:** las implementaciones se agrupan en secciones: "Completas", "Avanzadas", "En curso", "Iniciadas", "Pendientes". Cada sección muestra una tabla con columnas: Proceso, Empresa, Observaciones, Estado (badge), Avance (barra de progreso con porcentaje), Acciones.

- **Acciones por fila:** botón de editar implementación (abre modal).

- **Filtros:** no aplican en esta vista; el agrupamiento por estado es el mecanismo de navegación.

**Componente Livewire:** `DashboardPanelView`

---

### 3.2 Vista Procesos (`/dashboard/procesos`)

**Propósito:** Muestra el avance de implementación agrupado por proceso. Permite gestionar los procesos como entidades.

**Elementos de la vista:**

- **Fila de estadísticas:**
  - Total de procesos activos
  - Procesos al 100% completados
  - Procesos en curso
  - Total de implementaciones registradas

- **Grilla de tarjetas de proceso** (layout auto-fill, ~320px mínimo por tarjeta):
  - Cada tarjeta muestra:
    - Nombre del proceso con color identificador
    - Cantidad de implementaciones
    - Barra de progreso promedio con porcentaje
    - Lista de implementaciones del proceso, cada una con: dot de color por estado, nombre de empresa, porcentaje de avance
    - Botón "+" para agregar nueva implementación al proceso
    - Botones de editar y eliminar proceso

**Acciones disponibles en la vista:**
- Crear nuevo proceso (modal)
- Editar proceso existente (modal)
- Eliminar proceso (con confirmación; bloqueado si tiene implementaciones activas)

**Componente Livewire:** `DashboardProcesosView`

---

### 3.3 Vista Empresas (`/dashboard/empresas`)

**Propósito:** Muestra todas las empresas y su estado de implementación global. Permite gestionar empresas.

**Elementos de la vista:**

- **Buscador** en tiempo real por nombre de empresa.

- **Tabla de empresas** con columnas:
  - Empresa / Módulo (nombre)
  - Implementaciones (cantidad)
  - Avance promedio (barra + porcentaje)
  - Estado (badge calculado)
  - Acciones (editar, eliminar)

- **Pie de tabla:** contador "Mostrando X de Y empresas".

**Acciones disponibles:**
- Crear nueva empresa (modal)
- Editar empresa existente (modal)
- Eliminar empresa (con confirmación; bloqueado si tiene implementaciones activas)

**Componente Livewire:** `DashboardEmpresasView`

---

### 3.4 Lista Completa (`/dashboard/lista`)

**Propósito:** Tabla maestra con todas las implementaciones. Optimizada para búsqueda, filtrado y acceso rápido a cualquier registro.

**Elementos de la vista:**

- **Barra de filtros:**
  - Filtros rápidos por estado: "Todas", "Completas", "En curso", "Pendientes"
  - Select de proceso
  - Input de búsqueda libre (busca en nombre de proceso, empresa y observaciones)

- **Tabla de implementaciones** con columnas:
  - Proceso
  - Empresa
  - Observaciones (truncado a 2 líneas con tooltip para el texto completo)
  - Estado (badge)
  - Avance (barra de progreso + porcentaje)
  - Acciones (editar, eliminar)

- **Pie de tabla:** contador de registros visibles vs totales.

**Componente Livewire:** `DashboardListaView`

---

## 4. Features Complementarias

### 4.1 Slide-over de Detalle de Implementación

Al hacer clic sobre una implementación desde cualquier vista, se abre un panel lateral deslizante (slide-over) desde el lado derecho de la pantalla con el detalle completo.

**Comportamiento:**
- Se abre sin navegar a otra página (transición CSS `translateX`).
- Un overlay semitransparente cubre el contenido principal; clic en overlay cierra el panel.
- Botón "✕" en el header del panel para cerrar.

**Contenido del slide-over:**
- Header: nombre de la implementación, proceso al que pertenece.
- Sección "Información general": Proceso, Empresa, Avance (con barra grande), Estado (badge).
- Sección "Observaciones": texto completo de observaciones en un área de lectura destacada.
- Sección "Metadatos": fecha de creación, fecha de última actualización.
- Botón "Editar" que abre el modal de edición sin cerrar el panel.

**Componente:** `DetailSlideover` (Alpine.js con Livewire)

---

### 4.2 CRUD mediante Modales

Todas las operaciones de creación y edición se realizan en modales centrados, sin navegación de página. Los modales tienen animación de entrada (`scale + translateY`), fondo con `backdrop-filter: blur`.

#### Modal: Nueva / Editar Implementación

Campos:
- **Proceso** (select, obligatorio; carga lista de procesos activos)
- **Empresa / Módulo** (select con opción "+ Nueva empresa"; al seleccionar esta opción aparece campo de texto para nombre)
- **Observaciones / Estado** (textarea, opcional)
- **Avance** (input range 0–100, step 5; muestra valor en tiempo real como "XX%")

Validaciones:
- Proceso y empresa obligatorios.
- Combinación proceso + empresa única (no duplicados).
- Avance entre 0 y 100.

#### Modal: Nuevo / Editar Proceso

Campos:
- **Nombre del proceso** (text, obligatorio, único)
- **Descripción** (textarea, opcional)
- **Color identificador** (select con 8 opciones de color predefinidas)

#### Modal: Nueva / Editar Empresa

Campos:
- **Nombre de la empresa / módulo** (text, obligatorio, único)
- **Notas** (textarea, opcional)

---

### 4.3 Exportación e Importación XLSX

**Exportar a Excel:**
- Botón "Exportar xlsx" disponible en la barra de acciones global.
- Genera un archivo `.xlsx` con tres hojas: "Implementaciones", "Procesos", "Empresas".
- El nombre del archivo incluye la fecha: `proyectos_automatizaciones_YYYY-MM-DD.xlsx`.
- Los valores de avance se exportan como porcentaje (0–100) con sufijo `%`.

**Importar desde Excel:**
- Botón "Importar xlsx" abre un selector de archivo (acepta `.xlsx`, `.xls`).
- Al cargar el archivo, el sistema parsea las tres hojas y actualiza el estado local.
- Si hay registros con el mismo ID ya existentes, se actualizan (upsert). Si son nuevos, se insertan.
- Se muestra un toast con el resultado: "Importado: X implementaciones, Y procesos, Z empresas".
- En caso de error de formato, se muestra toast de error describiendo el problema.

**Componente / librería:** `maatwebsite/excel` (Laravel Excel) para el backend; o generación en cliente con librería JS para la versión frontend standalone.

---

### 4.4 Captura de Pantalla (Screenshot JPG)

- Botón "📸 Capturar panel" disponible en el topbar cuando se está en cualquier vista del dashboard.
- Al hacer clic, se usa `html2canvas` para generar una captura PNG/JPG del contenido visible actual.
- La descarga se dispara automáticamente con nombre: `dashboard_panel_YYYY-MM-DD.png`.
- Mientras se procesa la captura, el botón se deshabilita y muestra "Procesando…".
- No se capturan elementos superpuestos (modales abiertos, toasts).

**Nota de implementación:** `html2canvas` se ejecuta en el cliente (Alpine.js). El botón invoca `window.html2canvas(element).then(canvas => canvas.toBlob(...))`.

---

### 4.5 Integración Google Sheets (Sincronización Bidireccional)

Permite sincronizar el estado del dashboard con una Google Sheet externa, habilitando colaboración y reportes fuera de la plataforma.

#### Configuración de la integración

- Menú de configuración accesible desde la sidebar (sección "Google Sheets").
- Modal de configuración con:
  - **URL del Apps Script Web App** (input URL).
  - **Auto-sync cada N minutos** (número, 0 = desactivado, máximo 60).
  - Botón "Probar conexión": realiza una llamada `GET ?action=getAll` a la URL y muestra el resultado (ok / error).
  - Botón "Guardar".
- La URL y la configuración de auto-sync se persisten en `localStorage` / base de datos del usuario.

#### Banner de estado de conexión

Visible en la parte superior de todas las vistas del dashboard:
- **Sin conectar:** fondo neutro, texto "Google Sheets no configurado".
- **Conectado (no probado):** fondo azul claro.
- **Conectado y sincronizado:** fondo verde, fecha y hora de última sincronización.
- **Error:** fondo rojo, descripción del error.

#### Indicador en sidebar

Punto de color junto al ítem de navegación:
- Verde parpadeante: sincronizando.
- Verde fijo: última sync exitosa (muestra timestamp).
- Rojo: último intento fallido.
- Gris: sin configurar.

#### Importar desde Sheets

- Botón "↓ Importar de Sheets" en sidebar.
- El sistema llama `GET ?action=getAll` a la URL configurada.
- Si la respuesta tiene `status: ok`, sobreescribe los datos locales con los recibidos.
- Muestra toast con conteo de registros importados.

#### Exportar a Sheets

- Botón "↑ Exportar a Sheets" en sidebar.
- El sistema realiza `POST` con `{ action: 'syncAll', impls, processes, empresas }` a la URL configurada.
- El Apps Script del lado de Sheets recibe los datos y actualiza la hoja.
- Muestra toast de confirmación o error.

#### Auto-sync

- Si `autosync > 0`, el sistema ejecuta automáticamente "Exportar a Sheets" cada N minutos mientras la pestaña está abierta.
- El timer se resetea al cambiar la configuración.

#### Protocolo del Apps Script (referencia)

El Apps Script del lado de Google debe implementar los siguientes endpoints:

| Método | Parámetro | Acción |
|--------|-----------|--------|
| GET | `?action=getAll` | Retorna `{ status: 'ok', impls: [...], processes: [...], empresas: [...] }` |
| POST | body `{ action: 'syncAll', ... }` | Reemplaza todos los datos en la hoja. Retorna `{ status: 'ok' }` |

---

## 5. Componentes UI Necesarios

### 5.1 Componentes Livewire

| Componente | Responsabilidad |
|-----------|----------------|
| `DashboardLayout` | Layout base del dashboard: sidebar, topbar, área de contenido |
| `DashboardPanelView` | Vista Panel Operativo con KPIs y tabla agrupada por estado |
| `DashboardProcesosView` | Vista Procesos con grilla de tarjetas |
| `DashboardEmpresasView` | Vista Empresas con tabla y búsqueda |
| `DashboardListaView` | Vista Lista Completa con filtros |
| `ImplModal` | Modal CRUD de Implementación (crear / editar) |
| `ProcesoModal` | Modal CRUD de Proceso (crear / editar) |
| `EmpresaModal` | Modal CRUD de Empresa (crear / editar) |
| `DetailSlideover` | Panel lateral de detalle de implementación |
| `SheetsConfigModal` | Modal de configuración de Google Sheets |
| `SyncStatusBar` | Banner de estado de sincronización |

### 5.2 Componentes Alpine.js

| Componente | Uso |
|-----------|-----|
| `x-data="slideoverState()"` | Manejo de apertura/cierre del slide-over |
| `x-data="sheetsSync()"` | Estado de sincronización con Google Sheets y auto-sync timer |
| `x-data="screenshotHandler()"` | Lógica de captura con html2canvas |

### 5.3 Design Tokens

El dashboard usa el sistema de tokens definido en `guia_base_ux_ui.md`. Colores primarios:

| Token | Valor | Uso |
|-------|-------|-----|
| `--primary` | `#003F6E` | Encabezados, sidebar, elementos de marca |
| `--accent` | `#0072c6` | Links, botones secundarios |
| `--green` | `#16a34a` | Estado "Completa", toasts de éxito |
| `--yellow` | `#b45309` | Estado "En curso", alertas |
| `--orange` | `#c2410c` | Estado "Iniciada" |
| `--red` | `#dc2626` | Estado "Error", botones destructivos |
| `--muted` | `#5a7a96` | Labels, texto secundario |

### 5.4 Badges de Estado

| Estado | Clase CSS | Condición |
|--------|-----------|-----------|
| Completa | `badge-done` | `avance == 1.0` |
| Avanzada | `badge-adv` | `avance >= 0.75` |
| En curso | `badge-curso` | `avance >= 0.25` |
| Iniciada | `badge-init` | `avance > 0` |
| Pendiente | `badge-pend` | `avance == 0` |

---

## 6. Historias de Usuario con Criterios de Aceptación

---

### US-DASH-1: Panel Operativo

**Como** Programador, **quiero** ver un panel centralizado con el estado de todas las implementaciones agrupadas por estado de avance, **para** identificar rápidamente qué proyectos necesitan atención.

- **AC-DASH-1.1:** El panel muestra 4 KPI cards: Total, Completas, En curso y Pendientes, actualizadas en tiempo real sin recargar la página.
- **AC-DASH-1.2:** Las implementaciones se muestran agrupadas en secciones por estado (Completas, Avanzadas, En curso, Iniciadas, Pendientes). Cada sección tiene un título con separador visual.
- **AC-DASH-1.3:** Cada fila de la tabla muestra: nombre del proceso, nombre de la empresa, observaciones (truncadas a 2 líneas), badge de estado y barra de progreso con porcentaje.
- **AC-DASH-1.4:** Las secciones vacías no se renderizan (si no hay implementaciones "Avanzadas", esa sección no aparece).
- **AC-DASH-1.5:** Al hacer clic sobre cualquier fila, se abre el slide-over de detalle sin navegar.
- **AC-DASH-1.6:** El ícono de proceso activo en el sidebar muestra un badge con el conteo de implementaciones en curso.

**DoD:**
- [ ] Componente `DashboardPanelView` implementado con renderizado reactivo via Livewire.
- [ ] KPIs calculados via `computed properties` en el componente.
- [ ] Agrupamiento por estado calculado en el backend, no en el frontend.
- [ ] Tests: panel con 0 impls muestra estado vacío; con impls en todos los estados, todas las secciones se renderizan.

---

### US-DASH-2: Gestión de Procesos

**Como** Programador, **quiero** crear, editar y eliminar procesos desde la vista de Procesos, **para** mantener actualizado el catálogo de servicios que se están implementando.

- **AC-DASH-2.1:** El Programador puede crear un proceso completando nombre (obligatorio), descripción (opcional) y color identificador.
- **AC-DASH-2.2:** El sistema no permite dos procesos con el mismo nombre (case-insensitive). Muestra: "Ya existe un proceso con ese nombre".
- **AC-DASH-2.3:** Cada proceso se muestra como una tarjeta con barra de progreso promedio calculada sobre sus implementaciones.
- **AC-DASH-2.4:** El botón de eliminar proceso está deshabilitado si tiene implementaciones activas. Tooltip: "Eliminá primero las implementaciones asociadas".
- **AC-DASH-2.5:** Al editar, los cambios de nombre y color se reflejan de inmediato en todas las vistas sin recargar.
- **AC-DASH-2.6:** La tarjeta de proceso muestra un botón "+" para agregar una nueva implementación con el proceso pre-seleccionado.

**DoD:**
- [ ] Modelo `Proceso` con validación de unicidad en backend.
- [ ] Endpoint de eliminación con check de implementaciones dependientes.
- [ ] Grilla de tarjetas responsive (auto-fill, mínimo 320px).
- [ ] Tests: crear proceso duplicado → error; eliminar con impls → bloqueo; eliminar sin impls → éxito.

---

### US-DASH-3: Gestión de Empresas

**Como** Programador, **quiero** gestionar el catálogo de empresas/módulos desde la vista de Empresas, **para** tener control sobre qué entidades están siendo atendidas y su avance general.

- **AC-DASH-3.1:** El Programador puede crear una empresa completando nombre (obligatorio) y notas (opcional).
- **AC-DASH-3.2:** El buscador filtra en tiempo real (debounce 250ms) sin recargar la página.
- **AC-DASH-3.3:** La tabla muestra para cada empresa: nombre, cantidad de implementaciones, barra de progreso promedio y badge de estado.
- **AC-DASH-3.4:** El botón de eliminar empresa está deshabilitado si tiene implementaciones activas.
- **AC-DASH-3.5:** Al crear una empresa desde el modal de implementación (opción "+ Nueva empresa"), la empresa se persiste y queda seleccionada en el select.

**DoD:**
- [ ] Buscador con debounce implementado en Alpine.js o Livewire `wire:model.lazy`.
- [ ] Validación de unicidad de nombre en backend.
- [ ] Tests: crear empresa duplicada → error; eliminar con impls → bloqueo.

---

### US-DASH-4: Lista Completa con Filtros

**Como** Programador, **quiero** acceder a una tabla con todas las implementaciones y poder filtrarlas por estado, proceso y texto libre, **para** encontrar rápidamente cualquier registro específico.

- **AC-DASH-4.1:** Los filtros de estado rápido ("Todas", "Completas", "En curso", "Pendientes") son mutuamente excluyentes y se activan visualmente al seleccionarlos.
- **AC-DASH-4.2:** El select de proceso lista todos los procesos activos más la opción "Todos los procesos".
- **AC-DASH-4.3:** La búsqueda de texto libre funciona sobre nombre de proceso, nombre de empresa y contenido de observaciones.
- **AC-DASH-4.4:** Los filtros son acumulativos: un usuario puede filtrar por "En curso" + proceso "Conciliación y Arqueo" + buscar "efectivo" simultáneamente.
- **AC-DASH-4.5:** El pie de tabla muestra "Mostrando X de Y implementaciones".
- **AC-DASH-4.6:** Al eliminar una implementación desde la lista, se muestra modal de confirmación antes de proceder.

**DoD:**
- [ ] Filtrado implementado en el backend (query con `WHERE`), no con JS sobre el DOM.
- [ ] Tests: combinación de filtros devuelve solo los registros que cumplen todas las condiciones.

---

### US-DASH-5: Detalle Slide-over

**Como** Programador, **quiero** ver el detalle completo de una implementación en un panel lateral sin abandonar la vista actual, **para** consultar la información sin perder el contexto del listado.

- **AC-DASH-5.1:** El slide-over se abre con animación deslizante desde la derecha (transición 280ms). El contenido principal queda visible pero con overlay semitransparente.
- **AC-DASH-5.2:** El panel muestra: nombre del proceso, empresa, estado badge, barra de avance, observaciones completas (sin truncar), fecha de creación y última actualización.
- **AC-DASH-5.3:** El botón "Editar" en el slide-over abre el modal de edición. Al guardar cambios, el slide-over se actualiza con los nuevos datos sin cerrarse.
- **AC-DASH-5.4:** Clic en el overlay, tecla `Escape` o el botón "✕" cierran el panel.
- **AC-DASH-5.5:** Solo puede haber un slide-over abierto a la vez. Si el usuario abre el detalle de otra implementación estando el panel abierto, el contenido se reemplaza.

**DoD:**
- [ ] Animación CSS `transform: translateX(100%) → translateX(0)` sin JS de posicionamiento.
- [ ] Listener de tecla `Escape` registrado al abrir y eliminado al cerrar.
- [ ] Tests de comportamiento: apertura, actualización en lugar, cierre por overlay y por tecla.

---

### US-DASH-6: Exportación e Importación XLSX

**Como** Programador, **quiero** exportar todos los datos del dashboard a un archivo Excel e importar datos desde un Excel, **para** poder trabajar offline, compartir el estado con el equipo o restaurar datos.

- **AC-DASH-6.1:** El archivo exportado tiene tres hojas: "Implementaciones" (con columnas: ID, Proceso, Empresa, Avance %, Observaciones, Creado, Actualizado), "Procesos" (ID, Nombre, Descripción, Color) y "Empresas" (ID, Nombre, Notas).
- **AC-DASH-6.2:** El nombre del archivo exportado sigue el patrón `proyectos_automatizaciones_YYYY-MM-DD.xlsx`.
- **AC-DASH-6.3:** Al importar, si un registro con el mismo ID ya existe, se actualiza (upsert). Si el ID no existe, se inserta como nuevo.
- **AC-DASH-6.4:** Si el archivo importado tiene un formato de columnas incorrecto (columna obligatoria faltante), el sistema muestra error descriptivo y no realiza ningún cambio.
- **AC-DASH-6.5:** Tras una importación exitosa, todas las vistas se actualizan sin recargar la página.
- **AC-DASH-6.6:** El botón de importar solo acepta `.xlsx` y `.xls`. Otros formatos muestran: "Formato no soportado. Usá .xlsx o .xls".

**DoD:**
- [ ] Exportación usando `maatwebsite/excel` o librería equivalente.
- [ ] Importación con validación de estructura de columnas antes de persistir.
- [ ] Tests: exportar → reimportar → datos idénticos; importar con columna faltante → error sin cambios.

---

### US-DASH-7: Captura de Pantalla

**Como** Programador, **quiero** capturar el contenido visible del dashboard como imagen PNG, **para** incluirlo en reportes o compartirlo por mensajería sin salir de la plataforma.

- **AC-DASH-7.1:** El botón "📸 Capturar panel" está visible en el topbar en todas las vistas del dashboard.
- **AC-DASH-7.2:** Al hacer clic, el sistema genera una captura del área de contenido principal (excluyendo modales abiertos y toasts).
- **AC-DASH-7.3:** La imagen se descarga automáticamente con nombre `dashboard_[vista]_YYYY-MM-DD.png`.
- **AC-DASH-7.4:** Durante el procesamiento de la captura, el botón muestra "Procesando…" y queda deshabilitado.
- **AC-DASH-7.5:** Si la captura falla (ej. error de CORS en recursos externos), se muestra un toast de error descriptivo.

**DoD:**
- [ ] Integración de `html2canvas` como dependencia frontend.
- [ ] El nombre del archivo incluye el nombre de la vista activa y la fecha actual.
- [ ] Tests manuales en Chrome, Firefox y Edge.

---

### US-DASH-8: Integración Google Sheets

**Como** Programador, **quiero** sincronizar el estado del dashboard con una Google Sheet, **para** que el equipo pueda consultar y actualizar datos desde la hoja de cálculo sin acceso a la plataforma.

- **AC-DASH-8.1:** El modal de configuración incluye instrucciones paso a paso para conectar la Google Sheet (configurar Apps Script, obtener URL).
- **AC-DASH-8.2:** El botón "Probar conexión" verifica que la URL responda correctamente y muestra el conteo de registros encontrados en Sheets o el error específico.
- **AC-DASH-8.3:** "Importar de Sheets" trae todos los datos de la hoja y sobreescribe el estado local. Muestra toast con conteo de registros importados.
- **AC-DASH-8.4:** "Exportar a Sheets" envía todos los datos locales a la hoja. Si la operación falla por timeout o error de red, se reintenta una vez; si sigue fallando, se muestra toast de error.
- **AC-DASH-8.5:** Si `autosync > 0`, el auto-sync se ejecuta automáticamente cada N minutos mientras la sesión esté activa. El indicador en sidebar muestra el último timestamp de sync exitoso.
- **AC-DASH-8.6:** El banner de estado en la parte superior de las vistas del dashboard refleja el estado de conexión en tiempo real: "Sin conectar", "Conectado", "Sincronizando…", "Error: [descripción]".
- **AC-DASH-8.7:** Al configurar la URL, el sistema valida que sea una URL válida de `script.google.com` antes de guardar.

**DoD:**
- [ ] Configuración de Sheets persistida en BD (tabla `user_settings` o equivalente) vinculada al usuario autenticado.
- [ ] Llamadas a Sheets realizadas via backend (controller Laravel) para evitar problemas de CORS; el frontend solo muestra el estado.
- [ ] Auto-sync implementado como intervalo en Alpine.js que llama a un endpoint interno.
- [ ] Tests: configurar URL inválida → error de validación; probar conexión con URL válida de mock → ok; importar → datos actualizados.

---

## 7. Mapeo con Módulos Existentes

| Módulo | Relación con esta épica |
|--------|------------------------|
| **M-1 (Auth & RBAC)** | Las rutas del dashboard requieren rol Programador o superior. El acceso de solo lectura para Manager se configura en Spatie. |
| **M-9 (Dashboard Programador)** | Esta épica extiende M-9 agregando el seguimiento de implementaciones. Las vistas pueden integrarse en el sidebar existente del Programador. |
| **M-10 (Admin Panel)** | El Super Admin tiene acceso completo. La gestión de Procesos y Empresas puede exponerse también desde el panel de Admin. |
| **M-2 (Clientes)** | Las `Empresa` del dashboard son independientes del modelo `Cliente` de M-2, pero pueden vincularse en una iteración futura si se necesita trazabilidad entre clientes reales e implementaciones. |

---

## 8. Datos Iniciales (Seed)

Para facilitar el arranque del feature en producción y el testing, se incluye un seeder con los siguientes datos de referencia:

### Procesos

| # | Nombre | Descripción | Color |
|---|--------|-------------|-------|
| 1 | Conciliación y Arqueo | Conciliación contable y arqueo de caja | `#003F6E` |
| 2 | Dashboard Cheques | Visualización y seguimiento de cheques | `#0072c6` |
| 3 | Costos | Análisis y seguimiento de costos | `#b45309` |
| 4 | Rubreo | Proceso de rubreo y validación | `#c2410c` |
| 5 | Automatización Ventas | Automatización de reportes de ventas | `#7c3aed` |
| 6 | Digitalización de Facturas | Lectura OCR y digitalización | `#dc2626` |
| 7 | Análisis de Precios | Monitoreo y análisis de precios | `#16a34a` |
| 8 | Conciliación Financiera | Banco vs ingresos | `#0891b2` |
| 9 | Económico y Financiero | Dashboard económico-financiero | `#7c3aed` |

### Empresas

| # | Nombre |
|---|--------|
| 1 | Parador |
| 2 | Playita |
| 3 | Chimbas |
| 4 | Invernadas |
| 5 | Invernadas Paula |
| 6 | Invernadas Bonafide |
| 7 | Rocky Bay |
| 8 | Don Américo |
| 9 | Ancestral |
| 10 | Del Parque |
| 11 | Algar |
| 12 | Ventas Semanales |
| 13 | Lectura OCR |
| 14 | Precios |
| 15 | Banco vs Ingresos |
| 16 | Dashboard |

### Implementaciones iniciales (muestra)

| Proceso | Empresa | Avance | Observaciones |
|---------|---------|--------|---------------|
| Conciliación y Arqueo | Parador | 100% | — |
| Conciliación y Arqueo | Playita | 100% | — |
| Conciliación y Arqueo | Chimbas | 100% | — |
| Conciliación y Arqueo | Invernadas | 100% | — |
| Conciliación y Arqueo | Don Américo | 85% | Pruebas de conciliación ok, falta agregar efectivo |
| Conciliación y Arqueo | Ancestral | 75% | — |
| Dashboard Cheques | Del Parque | 100% | En funcionamiento |
| Dashboard Cheques | Don Américo | 100% | En funcionamiento |
| Dashboard Cheques | Algar | 100% | En funcionamiento |
| Costos | Don Américo | 95% | — |
| Rubreo | Parador | 80% | Código ok, falta implementación final |
| Automatización Ventas | Ventas Semanales | 75% | — |
| Digitalización de Facturas | Lectura OCR | 50% | Código ok, falta automatización |
| Análisis de Precios | Precios | 40% | Código ok, falta automatización de base de datos |
| Conciliación Financiera | Banco vs Ingresos | 0% | Pendiente de inicio |
| Económico y Financiero | Dashboard | 0% | Pendiente de inicio |

**Archivo de seeder:** `database/seeders/DashboardProyectosSeeder.php`

**Comando:** `php artisan db:seed --class=DashboardProyectosSeeder`

---

## 9. Definition of Done (épica completa)

- [ ] Los 8 modelos / tablas de BD migrados con sus constraints e índices.
- [ ] CRUD completo (create, read, update, delete) para Proceso, Empresa e Implementación funcionando en producción.
- [ ] Las 4 vistas (Panel, Procesos, Empresas, Lista) renderizan correctamente con datos reales.
- [ ] Slide-over de detalle funcional en las 4 vistas.
- [ ] Exportación XLSX generando archivo con las 3 hojas correctamente estructuradas.
- [ ] Importación XLSX con validación de estructura y upsert.
- [ ] Captura PNG descargable desde cualquier vista.
- [ ] Integración Google Sheets: configuración, probar conexión, importar y exportar funcionando.
- [ ] Auto-sync con intervalo configurable.
- [ ] RBAC: Programador y Admin con acceso completo; Manager con solo lectura; Operador sin acceso.
- [ ] Cobertura de tests ≥ 80% en servicios y componentes críticos.
- [ ] Seeder con datos iniciales ejecutable sin errores en entorno limpio.
- [ ] Revisión de código aprobada por al menos 1 desarrollador.
- [ ] Documentación UX/UI creada siguiendo la plantilla `ux_ui_design_epica_o_us_0_1.md` para las historias con wireframes complejos (US-DASH-1, US-DASH-5).
