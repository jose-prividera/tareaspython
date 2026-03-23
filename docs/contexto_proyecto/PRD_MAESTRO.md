# PRD Maestro Incremental: Front-API / LaravelCiaf
**Versión:** 2.0 — 2026-03-18
**Estado:** Borrador
**Equipo:** 2 desarrolladores
**Rama activa:** `ciaf_v1.0.0`
**Producción:** VPS Hostinger + MySQL 8

---

## 1. Visión y Contexto

### 1.1 Estado actual de la plataforma

Front-API es un SaaS B2B operativo en producción que automatiza la conciliación contable y financiera para clientes del sector comercial/retail argentino. El sistema permite a **Programadores** configurar y ejecutar workflows de procesamiento de datos (actualmente vía Excel → Python → resultados estructurados), mientras los **Operadores** consultan los resultados por cliente. La plataforma está deployada en Hostinger VPS con MySQL, tiene usuarios reales activos, y su núcleo de workflows (Conciliación y Arqueo) funciona end-to-end. El vector de crecimiento inmediato es reemplazar la carga manual de Excel por integraciones directas con APIs bancarias (Mercado Pago, Getnet, AFIP, Ualá, Naranja X) y estandarizar ese framework para sumar nuevas entidades sin fricción.

### 1.2 Usuarios y workflows existentes

| Rol | Workflow principal | Frecuencia | Módulos que usa |
|-----|--------------------|------------|-----------------|
| **Super Admin** | Gestión de usuarios y configuración global | Esporádica | M-1, M-10 |
| **Manager** | Supervisión de clientes y usuarios | Semanal | M-2, M-10 |
| **Programador / Operador** | Cargar archivos → ejecutar conciliación → revisar resultado → descargar PDF | Diaria | M-5, M-6, M-7, M-8, M-9 |
| **Analista** | Recibe solicitudes de workflow; integra metadata; testea y activa | Según demanda | M-3, M-5 |

### 1.3 Objetivos estratégicos

1. **Modelo de Teams con Adaptadores bancarios:** Organizar a los operadores en equipos, automatizar la recolección de datos con adapters (API directa o RPA) y formalizar el ciclo Operador → Analista para habilitar nuevos workflows sin fricción.
2. **Eliminar la carga manual de Excel** conectando las APIs bancarias reales (MP, Getnet, AFIP, Ualá, Naranja X). Métrica: 0 archivos Excel cargados manualmente para clientes con API conectada.
3. **Framework de integraciones bancarias extensible**: agregar un nuevo banco debe costar < 1 sprint sin modificar código core. Métrica: tiempo de onboarding de nueva integración.
4. **Cobertura de tests ≥ 80%** en servicios críticos (FileValidation, Execution, ConciliationData). Métrica: `php artisan test --coverage`.
5. **Proceso de deploy reproducible a Hostinger**: cualquier devops puede hacer el deploy siguiendo `docs/DEPLOY_HOSTINGER.md` sin conocimiento del codebase.

### 1.4 Restricciones inamovibles

- **Base de datos:** MySQL 8 en Hostinger. No migrar a PostgreSQL en este horizonte.
- **Autenticación:** El módulo M-1 (Breeze + Spatie RBAC) no se toca. Los roles y su matriz de permisos son inamovibles.
- **Python server:** La API Python de conciliación es externa. El sistema debe funcionar en modo mock si el server Python no responde.
- **Hosting:** VPS Hostinger. Sin Docker, sin Kubernetes, sin pipelines automatizados aún.
- **Stack frontend:** TALL Stack (Tailwind 4 + Alpine 3 + Livewire 3.7). No introducir Vue/React.

---

## 2. Mapa de Módulos Existentes

| ID | Módulo | Problema que resuelve | Estado | Deuda técnica / Dolor | Dependencias |
|----|--------|-----------------------|--------|-----------------------|--------------|
| **M-1** | Auth & RBAC (Breeze + Spatie) | Acceso seguro por rol | ✅ Estable prod | Ninguna conocida | — |
| **M-2** | Gestión de Clientes | CRUD sedes/sucursales, jerarquía `parent_id` | ✅ Estable prod | — | M-1 |
| **M-3** | Catálogo de APIs e Integraciones | Configurar credenciales por cliente (MP, AFIP, Getnet…) | ⚠️ Infraestructura lista, sin llamadas reales | API real no activada; no hay adapter pattern | M-2 |
| **M-4** | Reglas de Negocio ETL (Pyodide/Monaco) | Editor Python en browser para testing de reglas | ✅ Estable prod | Ejecución solo en browser, no en servidor | — |
| **M-5** | Sistema de Workflows – Core | Tipos, batches, archivos, executions | ✅ Estable prod | Dos servicios de conciliación duplicados | M-2 |
| **M-6** | Workflow Conciliación | Conciliación bancaria end-to-end (8 entidades) | ✅ En prod, activo | `ConciliacionDataService` vs `ConciliationDataService` — duplicación | M-5 |
| **M-7** | Workflow Arqueo | Arqueo de caja sobre datos de conciliación existentes | ✅ En prod, nuevo | Poca cobertura de tests | M-6 |
| **M-8** | Informes y PDFs | Generación de reportes PDF (Conciliación, Arqueo, Anulaciones) | ✅ En prod | — | M-6, M-7 |
| **M-9** | Dashboard Programador + Métricas | KPIs, historial, métricas de ejecución | ✅ En prod | — | M-5, M-6 |
| **M-10** | Panel de Administración | Gestión de usuarios, roles, settings | ✅ Estable prod | — | M-1 |
| **M-11** | Gestión de Teams y Adaptadores | Teams, asignaciones, adapters bancarios (nuevo) | ❌ No implementado | Épicas E0-E1 pendientes | M-2 |
| **M-12** | Lead Enrichment | Enriquecimiento de datos de leads (propósito no claro) | ❓ Código existe, sin UI | Propósito de negocio no documentado | M-2 |

### 2.1 Dependencias entre módulos

```
M-1 (Auth)
  └─► M-2 (Clientes) ──────────────────────────────────────────┐
        └─► M-3 (Catálogo APIs) ──► [E1: Adaptadores]           │
  └─► M-5 (Workflows Core)                                       │
        ├─► M-6 (Conciliación) ◄────────────────────────────────┘
        │     └─► M-7 (Arqueo)
        │     └─► M-8 (PDFs)
        └─► M-9 (Dashboard)
  └─► M-10 (Admin)
  └─► M-11 (Teams & Adaptadores) [PENDIENTE — E0-E1]
```

### 2.2 Áreas de código legado / atención especial

- **Duplicación `ConciliacionDataService` / `ConciliationDataService`** (`app/Services/`): Dos servicios con nombres casi idénticos (uno en español, uno en inglés). Antes de modificar conciliación, identificar cuál está activo y deprecar el otro.
- **`workflow_id` nullable en `workflow_executions`**: Campo legacy marcado como `null` en código. No eliminar sin auditar si hay datos históricos.
- **M-12 (LeadEnrichmentService)**: Código sin UI ni documentación. No modificar sin entender su propósito (ver D-3).

---

## 3. Épicas de Producto

> **Backlog completo de la plataforma.** Cada épica incluye objetivo, historias de usuario con criterios de aceptación detallados y Definition of Done (DoD).
> **Orden de desarrollo sugerido:** E0 → E1 → E2 y E3 (paralelo) → E4 → E5 → E6

---

### E0: Gestión de Teams y Asignaciones (El Andamiaje Organizacional)

**Objetivo:** Proveer al Admin de las herramientas necesarias para estructurar la organización dentro de la plataforma: crear teams, asignar operadores y vincular clientes con sus sucursales, estableciendo la base sobre la que se construyen todas las demás funcionalidades.

---

#### US-0.1: Gestión de Teams

**Como** Admin, **quiero** crear, editar y desactivar teams desde un panel centralizado, **para** organizar a los operadores en unidades de trabajo que reflejen la estructura operativa real de la empresa.

- **AC-0.1.1:** El Admin puede crear un team completando: Nombre (obligatorio, único), Descripción (opcional) y color/etiqueta visual.
- **AC-0.1.2:** El sistema no permite dos teams con el mismo nombre (case-insensitive). Muestra: "Ya existe un team con ese nombre".
- **AC-0.1.3:** El Admin puede editar nombre y descripción sin afectar asignaciones de operadores ni clientes ya vinculados.
- **AC-0.1.4:** Al desactivar un team, el sistema solicita confirmación mostrando: cantidad de operadores, clientes y workflows activos afectados. Soft delete — no se elimina de la BD.
- **AC-0.1.5:** Si el team tiene workflows en estado "Activo" o "En Ejecución", el sistema bloquea la desactivación.
- **AC-0.1.6:** El panel lista todos los teams con: Nombre, cantidad de operadores, clientes, workflows activos y badge de estado (Activo/Inactivo).
- **AC-0.1.7:** El listado permite filtrar por estado y buscar por nombre.

**DoD:**
- [ ] Modelo `Team` con campos: nombre (unique), descripción, color, is_active, timestamps.
- [ ] API REST CRUD protegida por rol Admin.
- [ ] Validación de unicidad en frontend y backend.
- [ ] Soft delete con cascade check contra workflows activos.
- [ ] Tests unitarios: creación, edición, duplicado, desactivación con/sin workflows activos.

---

#### US-0.2: Asignación de Operadores a Teams

**Como** Admin, **quiero** asignar y reasignar operadores a un team específico, **para** que cada operador tenga claro a qué equipo pertenece y pueda ver los clientes y workflows correspondientes.

- **AC-0.2.1:** Desde la vista de detalle de un team, el Admin agrega operadores desde una lista de usuarios con rol "Operador" sin team asignado.
- **AC-0.2.2:** El Admin puede seleccionar varios operadores simultáneamente (bulk assign).
- **AC-0.2.3:** Un operador pertenece a un único team a la vez. Si ya pertenece a otro, el sistema muestra advertencia: "Este operador pertenece al Team X. ¿Desea reasignarlo?"
- **AC-0.2.4:** Al reasignar, el operador pierde acceso inmediato al team anterior y gana acceso al nuevo. No se borran registros históricos.
- **AC-0.2.5:** El Admin puede remover un operador, dejándolo en estado "Sin asignar". No puede operar hasta ser reasignado.
- **AC-0.2.6:** El operador ve en su sidebar el nombre de su team. Si no tiene team: "No estás asignado a ningún team. Contactá a tu administrador".
- **AC-0.2.7:** Cada cambio de asignación se registra con: operador, team origen, team destino, fecha, admin que ejecutó.

**DoD:**
- [ ] Relación `Operador → Team` (FK con null=True) en el modelo de usuario.
- [ ] Endpoint de asignación/desvinculación con validación de rol Admin.
- [ ] Log de auditoría persistente para cada cambio de asignación.
- [ ] Sidebar del operador muestra dinámicamente el team (o mensaje "sin team").
- [ ] Tests de integración: asignación, reasignación, desvinculación y acceso condicional.

---

#### US-0.3: Asignación de Clientes a Teams

**Como** Admin, **quiero** asignar clientes (con sus sucursales) a un team, **para** que el equipo operativo tenga visibilidad exclusiva sobre los clientes que debe gestionar.

- **AC-0.3.1:** El Admin puede crear un cliente con: Razón social, CUIT/identificador fiscal, y al menos una sucursal (Nombre + dirección o identificador).
- **AC-0.3.2:** Dentro del perfil de un cliente, el Admin puede agregar, editar o desactivar sucursales. Cada sucursal tiene nombre único dentro del cliente.
- **AC-0.3.3:** El Admin asigna un cliente a un team. A partir de ese momento, todos los operadores del team ven al cliente y sus sucursales en su dashboard.
- **AC-0.3.4:** Un mismo cliente puede asignarse a más de un team (distintas regiones). Cada team solo ve las sucursales que le fueron asignadas explícitamente.
- **AC-0.3.5:** Un operador solo ve los clientes asignados a su team. No puede buscar ni acceder a clientes de otros teams bajo ninguna circunstancia.
- **AC-0.3.6:** Al desvincular un cliente de un team, el sistema advierte si existen adaptadores o workflows activos vinculados y requiere desactivarlos primero.
- **AC-0.3.7:** El Admin tiene un panel con todos los clientes mostrando: Team(s) asignado(s), cantidad de sucursales, adaptadores configurados y estado general.

**DoD:**
- [ ] Modelos `Cliente` y `Sucursal` con relaciones correctas (1→N).
- [ ] Tabla pivote `TeamCliente` (o `TeamSucursal`) con constraints de integridad.
- [ ] Middleware/decorator que filtre querysets por team del operador autenticado.
- [ ] Validación de desvinculación con chequeo de adaptadores/workflows dependientes.
- [ ] Tests de permisos: operador de Team A no puede acceder a datos de Team B (HTTP 403).
- [ ] Panel de Admin con listado consolidado de clientes, filtrable por team.

---

### E1: Gestión de Adaptadores e Integraciones (El Ecosistema de Ingesta)

**Objetivo:** Empoderar al Operador para que pueda conectar las sucursales del cliente con sus fuentes de datos (bancos, pasarelas) mediante un proceso guiado, seguro y a prueba de errores humanos.

---

#### US-1.1: Configuración Guiada de Adaptador tipo API (Ej. Mercado Pago)

**Como** Operador, **quiero** configurar un adaptador de conexión directa vía API para una sucursal, **para** automatizar la ingesta de transacciones sin descargar archivos manualmente.

- **AC-1.1.1:** Si selecciono "Mercado Pago", el sistema autocompleta datos técnicos (Base URL, Content-Type) y solo solicita credenciales específicas (Access Token, User ID).
- **AC-1.1.2:** Cada campo de credencial incluye tooltip/modal con instrucciones exactas de dónde obtener el dato.
- **AC-1.1.3:** Al guardar, el sistema realiza una llamada de prueba (ej. `GET /v1/users/me`) para validar conectividad antes de persistir.
- **AC-1.1.4:** Si el token está mal escrito o expirado, el adaptador no se guarda: "Token inválido o expirado. Verifique sus credenciales".
- **AC-1.1.5:** Si el token es válido pero sin permisos de lectura financiera, indica explícitamente qué scope debe agregar el cliente.
- **AC-1.1.6:** Si la API devuelve error 429, el sistema sugiere reintentar en unos minutos en lugar de fallar genéricamente.
- **AC-1.1.7:** El formulario permite configurar credenciales independientes para "Sandbox" y "Producción" con un switch.
- **AC-1.1.8:** El operador puede cargar o definir el esquema de columnas esperado y reglas de limpieza. Se permite subir un Excel de referencia (drag & drop) para generar la metadata de columnas.
- **AC-1.1.9:** Si el usuario abandona el formulario, los datos generales (excepto tokens) se guardan como borrador local en el navegador.
- **AC-1.1.10:** No se permiten dos adaptadores con el mismo nombre para la misma sucursal.
- **AC-1.1.11:** Al editar un adaptador, los campos de token aparecen vacíos/ofuscados. Solo se sobreescriben si el usuario tipea algo nuevo.
- **AC-1.1.12:** Una vez guardadas, las credenciales se muestran como `APP_USR-****...` — nunca en texto plano.

**DoD:**
- [ ] Credenciales almacenadas encriptadas en BD.
- [ ] Variables sensibles excluidas de logs y payloads expuestos de la API.
- [ ] Tests unitarios para token válido, inválido y sin permisos.
- [ ] UI con validación de campos obligatorios y tooltips de ayuda.
- [ ] Log de auditoría inmutable (quién creó el adaptador, IP, fecha, sucursal).
- [ ] Peer Review aprobado.

---

#### US-1.2: Configuración de Adaptador tipo RPA / Carga Manual

**Como** Operador, **quiero** generar un endpoint/link único seguro para una sucursal, **para** que el RPA o el equipo envíe el archivo Excel automáticamente al sistema.

- **AC-1.2.1:** Al crear un adaptador tipo RPA, el sistema genera automáticamente una URL de API única vinculada a esa sucursal.
- **AC-1.2.2:** El sistema genera un Bearer/API Key de alta entropía exclusivo para autenticar la subida.
- **AC-1.2.3:** La interfaz provee botones de "Copiar al portapapeles" para el endpoint y el token.
- **AC-1.2.4:** El operador sube un "Excel de muestra". Si el RPA envía un archivo que no respeta las cabeceras, el sistema rechaza con HTTP 400.
- **AC-1.2.5:** Si un archivo es rechazado, el endpoint devuelve JSON estructurado indicando el error exacto: `{"error": "Missing column: 'Monto'"}`.
- **AC-1.2.6:** Al recibir un archivo correcto, el endpoint devuelve HTTP 201 y `{"status": "success", "rows_processed": X}`.
- **AC-1.2.7:** El endpoint y la UI rechazan archivos superiores a 50MB con mensaje claro.
- **AC-1.2.8:** Solo se aceptan extensiones `.xlsx` y `.csv`. Otros formatos devuelven HTTP 415.
- **AC-1.2.9:** Si el RPA envía dos archivos en menos de 5 segundos, el sistema procesa el primero e informa "Operación ya en curso" al segundo.
- **AC-1.2.10:** La vista del adaptador incluye botón de "Subida Manual" (drag & drop) como fallback si el RPA falla.
- **AC-1.2.11:** Historial de las últimas 10 cargas: Fecha, Método (RPA/Manual), Nombre de archivo, Filas y Estado.
- **AC-1.2.12:** El operador puede hacer clic en "Regenerar Token" para invalidar el acceso anterior inmediatamente.

**DoD:**
- [ ] Endpoint protegido aceptando solo POST con el token asociado válido.
- [ ] Módulo validador de estructura de Excel utilizable por API y carga web.
- [ ] Límites de subida configurados en aplicación y servidor web.
- [ ] Tests de integración: rechazos de archivos malformados y autenticación fallida.
- [ ] Componente visual de historial de cargas conectado al backend.
- [ ] Documentación Swagger/Postman del endpoint generada para devs de RPA.

---

### E2: Gestión de Tareas y Distribución en el Team (La Cadena de Responsabilidades)

**Objetivo:** Permitir que los operadores del team distribuyan las tareas de ingesta de datos entre los miembros del equipo, asegurando que cada archivo requerido tenga un responsable claro.

---

#### US-2.1: Tablero de Tareas del Team

**Como** Operador, **quiero** ver un tablero con todas las tareas de ingesta pendientes de mi team organizadas por estado, **para** entender de un vistazo qué archivos faltan, quién es responsable y cuáles están completados.

- **AC-2.1.1:** Tablero en columnas: "Pendiente", "En Progreso", "Completada", "Fallida". Cada tarjeta muestra: Nombre del adaptador, Sucursal, Responsable y deadline.
- **AC-2.1.2:** El operador puede alternar entre vista Kanban y vista tabla ordenable por cualquier campo.
- **AC-2.1.3:** Filtros por: Responsable (mis tareas / todas), Cliente, Sucursal, Tipo (API/RPA), Estado.
- **AC-2.1.4:** Tareas con deadline a menos de 1 hora se resaltan con badge rojo y se mueven al tope visual.
- **AC-2.1.5:** El sistema genera automáticamente las tareas de ingesta cada día según el schedule del workflow. No requiere intervención manual.
- **AC-2.1.6:** Cada tarea incluye `fecha_ciclo` (la fecha del día para el que se necesita el dato).
- **AC-2.1.7:** Contadores rápidos en la parte superior: "X pendientes", "Y completadas hoy", "Z fallidas".
- **AC-2.1.8:** Solo los operadores del team ven las tareas del team (acceso restringido).

**DoD:**
- [ ] Modelo `Tarea` con campos: adaptador (FK), workflow (FK), responsable (FK nullable), estado (enum), fecha_ciclo, deadline, timestamps.
- [ ] Servicio de generación automática de tareas (Celery beat/cron) según schedule de workflows activos.
- [ ] API de listado con filtros y ordenamiento, protegida por permisos de team.
- [ ] Componentes Kanban y tabla implementados y testeados.
- [ ] Tests: dado workflow con 3 adaptadores y schedule diario → se generan exactamente 3 tareas por día hábil.

---

#### US-2.2: Asignación y Reasignación de Tareas

**Como** Operador, **quiero** asignar tareas de ingesta a miembros específicos de mi team, **para** distribuir la carga de trabajo con un responsable claro por archivo.

- **AC-2.2.1:** En vista Kanban, el operador arrastra una tarjeta sobre el avatar de un compañero para asignarla.
- **AC-2.2.2:** En vista detalle de la tarea, un dropdown lista todos los operadores del team.
- **AC-2.2.3:** Un operador puede hacer clic en "Tomar esta tarea" para auto-asignársela.
- **AC-2.2.4:** Al reasignar, el sistema registra en historial: responsable anterior, nuevo, cuándo se cambió. Notifica al nuevo responsable.
- **AC-2.2.5:** "Auto-distribuir" sugiere una distribución equitativa de tareas sin asignar entre operadores activos. El operador puede ajustar antes de confirmar.
- **AC-2.2.6:** Al asignar, el operador recibe notificación in-app y por correo con: adaptador, sucursal, fecha de ciclo y deadline.
- **AC-2.2.7:** Tareas sin responsable muestran indicador diferenciado (avatar vacío o badge "Sin asignar").
- **AC-2.2.8:** Un operador no puede asignar tareas a operadores de otros teams.

**DoD:**
- [ ] Endpoint de asignación/reasignación con validación de pertenencia al team.
- [ ] Historial de asignaciones persistido (modelo `TareaAsignacionLog`).
- [ ] Algoritmo de auto-distribución (round-robin o balanceo por carga) como endpoint.
- [ ] Sistema de notificaciones in-app conectado al evento de asignación.
- [ ] Tests: asignar, reasignar, auto-asignar, asignar a operador de otro team (403).
- [ ] UI de drag & drop testeada en Chrome, Firefox, Edge.

---

#### US-2.3: Seguimiento y Cumplimiento de Tareas

**Como** Operador, **quiero** marcar tareas como completadas y ver el progreso global del team hacia la ejecución del workflow, **para** saber qué porcentaje de insumos están listos.

- **AC-2.3.1:** Cuando un archivo es subido exitosamente y pasa la validación, la tarea se marca automáticamente como "Completada".
- **AC-2.3.2:** El operador puede marcar manualmente una tarea como completada (caso excepcional) adjuntando una nota de justificación obligatoria.
- **AC-2.3.3:** Vista del workflow muestra barra de progreso: "3 de 5 archivos recibidos (60%)" con indicación de cuáles faltan, adaptador y responsable.
- **AC-2.3.4:** Si una carga falla, la tarea se marca "Fallida" con enlace al detalle del error. El responsable recibe notificación.
- **AC-2.3.5:** Desde una tarea fallida, el operador puede hacer clic en "Reintentar" → lo lleva al adaptador con subida manual pre-seleccionada.
- **AC-2.3.6:** Al final de la jornada, el tablero muestra resumen: tareas completadas a tiempo, con retraso y fallidas, por operador.
- **AC-2.3.7:** Si una tarea supera su deadline, se envía alerta al operador y al team: "El workflow X no podrá ejecutarse hasta que se complete esta tarea".

**DoD:**
- [ ] Listener/signal que detecta carga exitosa y actualiza automáticamente el estado de la tarea.
- [ ] Endpoint de progreso del workflow retornando estado de cada tarea asociada.
- [ ] Lógica de tareas vencidas (Celery periodic task) con envío de notificaciones.
- [ ] Campo `nota_justificacion` obligatorio en completado manual, validado en backend.
- [ ] Tests: carga exitosa → tarea completada; carga fallida → tarea fallida con detalle; deadline vencido → alerta emitida.
- [ ] Vista de resumen diario maquetada y conectada.

---

### E3: Solicitudes de Workflow (El Puente Ops → Tech)

**Objetivo:** Estandarizar la comunicación entre Operaciones y Analistas para que la creación de lógica a medida sea trazable, ordenada y rápida.

---

#### US-3.1: Creación de Solicitud de Workflow

**Como** Operador, **quiero** enviar una solicitud formal de workflow asociando los adaptadores correspondientes, **para** que el equipo de analistas tenga todo el contexto para programar la conciliación.

- **AC-3.1.1:** El operador puede seleccionar uno o varios adaptadores previamente creados (ej. Adapter MP + Adapter Galicia) que alimentarán el workflow.
- **AC-3.1.2:** El sistema bloquea el envío si algún adaptador tiene estado inválido (token no probado o expirado).
- **AC-3.1.3:** Campos obligatorios: "Resultado Esperado", "Reglas de Conciliación" y "Horarios/Turnos operativos del cliente".
- **AC-3.1.4:** Permite subir hasta 3 archivos adjuntos (PDFs, capturas) para dar contexto al analista.
- **AC-3.1.5:** Permite configurar la frecuencia de ejecución esperada (ej. Lunes a Viernes a las 09:00 AM).
- **AC-3.1.6:** Al configurar la frecuencia, la interfaz muestra en tiempo real: "Próxima ejecución esperada: Lunes 14, 09:00 AM".
- **AC-3.1.7:** Opción "Duplicar" una solicitud anterior: copia todos los campos (adaptadores, horarios), dejando vacío el campo de reglas.
- **AC-3.1.8:** Al enviarse, la solicitud queda en estado "Pendiente" en solo lectura — no se puede editar.
- **AC-3.1.9:** El operador puede "Anular" la solicitud únicamente mientras esté en estado "Pendiente".
- **AC-3.1.10:** El envío dispara alerta inmediata al equipo de Analistas (in-app + correo).
- **AC-3.1.11:** El sistema genera internamente el JSON (metadata) que empaqueta toda la información y estructura de los adaptadores.

**DoD:**
- [ ] Relación Solicitud → N Adaptadores (ManyToManyField) en modelo de datos.
- [ ] Validaciones frontend y backend bloqueando envíos con adaptadores inválidos.
- [ ] Servicio de subida (S3/Local) configurado para archivos adjuntos.
- [ ] Integración de notificaciones (Celery/colas) sin bloquear el hilo de UI.
- [ ] UX/UI: modo "Solo lectura" y botón "Anular" funcionando según el estado.

---

#### US-3.2: Gestión de Estados de la Solicitud

**Como** Sistema, **quiero** gestionar una máquina de estados clara y auditable para cada solicitud de workflow, **para** que operadores y analistas sepan en todo momento en qué fase está y qué acciones son posibles.

- **AC-3.2.1:** Estados: `Pendiente` → `En Revisión` → `Requiere Acción` (ida y vuelta con Pendiente) → `En Desarrollo` → `En Testing` → `Activo`. Estado terminal alternativo: `Anulada`.
- **AC-3.2.2:** Solo se permiten transiciones autorizadas. El sistema bloquea técnicamente cualquier cambio fuera de la tabla de transiciones válidas.
- **AC-3.2.3:** Cada cambio de estado genera registro inmutable: estado anterior, nuevo, usuario, fecha/hora y comentario obligatorio.
- **AC-3.2.4:** Operador y Analista pueden ver el estado actual y el historial completo de transiciones desde sus respectivos paneles.
- **AC-3.2.5:** El historial de cambios se muestra como timeline vertical cronológico con comentarios de ida y vuelta.
- **AC-3.2.6:** Solicitudes con más de 48 horas sin movimiento en un mismo estado se resaltan con badge de alerta.

**DoD:**
- [ ] Máquina de estados con validación de transiciones en backend (django-fsm o lógica custom).
- [ ] Modelo `SolicitudTransicion` (estado_anterior, estado_nuevo, usuario, timestamp, comentario).
- [ ] Endpoint de historial de transiciones disponible para frontend.
- [ ] Componente de timeline implementado y conectado.
- [ ] Tests unitarios para cada transición válida e inválida.

---

### E4: Panel del Analista (El Centro de Comando Técnico)

**Objetivo:** Proveer al Analista de un espacio de trabajo completo para recibir, analizar, desarrollar, testear y activar workflows, manteniendo trazabilidad completa y evitando colisiones entre analistas.

---

#### US-4.1: Inbox de Solicitudes del Analista

**Como** Analista, **quiero** ver un panel centralizado con todas las solicitudes pendientes y en progreso, **para** priorizar mi trabajo y evitar que dos analistas trabajen en la misma solicitud simultáneamente.

- **AC-4.1.1:** Panel Kanban/tabla con solicitudes ordenadas por urgencia/fecha. Muestra: Cliente, Team, Fecha de creación y Estado.
- **AC-4.1.2:** Solicitudes con más de 48h en estado "Pendiente" se resaltan con ícono rojo.
- **AC-4.1.3:** Si un Analista abre una solicitud para revisarla, los demás ven: "En revisión por [Nombre]" para evitar colisiones.
- **AC-4.1.4:** Si un analista tiene una solicitud en revisión por más de 1 hora sin acción, el lock se libera automáticamente.
- **AC-4.1.5:** Botón "Descargar Insumos" exporta un .zip con la metadata (JSON) de cada adaptador, Excels de muestra y archivos adjuntos de contexto.
- **AC-4.1.6:** Filtros: Estado, Cliente, Team, Tipo de workflow, Analista asignado.
- **AC-4.1.7:** Contadores: "X pendientes", "Y en desarrollo", "Z activadas este mes" y tiempo promedio de resolución.

**DoD:**
- [ ] Lógica de control de concurrencia (`locked_by`, `locked_at`) con timeout automático de 1 hora.
- [ ] Endpoint de generación de .zip con metadata + archivos, protegido por rol Analista.
- [ ] UI del inbox con filtros, ordenamiento y badges de SLA.
- [ ] Tests de concurrencia: dos analistas intentan tomar la misma solicitud simultáneamente.
- [ ] Revisión de seguridad: "Descargar Insumos" verifica rol Analista.

---

#### US-4.2: Integración, Testing y Activación del Workflow

**Como** Analista, **quiero** integrar la metadata al código Django, probar la ejecución y activar el workflow para el team operador, **para** que la conciliación corra automáticamente con confianza.

- **AC-4.2.1:** Para activar, el Analista debe introducir el `id_workflow`. El sistema verifica que esté registrado y disponible en el registry interno de workflows Django.
- **AC-4.2.2:** Botón "Simular Ejecución" (Dry Run) corre el script Django usando los Excels de muestra, mostrando output en consola web.
- **AC-4.2.3:** La consola web muestra: tiempo de ejecución, filas procesadas, errores/warnings y preview del resultado (primeras 20 filas).
- **AC-4.2.4:** Checklist pre-activación (bloqueante, validado en backend): ¿El código está desplegado en producción? ¿El dry run pasó sin errores? ¿Se verificó la correspondencia adaptadores/parámetros? El analista debe tildar todos los ítems.
- **AC-4.2.5:** Al activar exitosamente, la solicitud se mueve a "Workflows Operativos" del Team y se notifica al Operador que la creó.
- **AC-4.2.6:** El Analista puede usar "Requerir Cambios" con comentario obligatorio. La solicitud vuelve al Operador en estado "Requiere Acción".
- **AC-4.2.7:** La interfaz mantiene un timeline histórico de comentarios intercambiados entre Operador y Analista.
- **AC-4.2.8:** El sistema prohíbe cambiar el estado de un workflow "Activo" a "Pendiente". Solo permite pausarlo o desactivarlo.
- **AC-4.2.9:** Cada vez que un analista modifica y reactiva un workflow existente, el sistema crea una nueva versión (v1, v2, v3...) manteniendo el historial accesible.

**DoD:**
- [ ] Endpoint de validación del registry de workflows Django (lectura dinámica).
- [ ] Mecanismo de Dry Run seguro (entorno efímero o rollback de BD al terminar).
- [ ] Sistema de historial de comentarios (modelo `Comentario` asociado a la Solicitud).
- [ ] Checklist pre-activación como gate bloqueante en backend (no solo frontend).
- [ ] Modelo de versionado de workflows con referencia al historial.
- [ ] Tests: dry run exitoso → habilita activación; dry run fallido → bloquea activación; Activo → Pendiente → rechazado.

---

### E5: Ejecución de Workflows (El Motor de Orquestación) — *FUTURA*

**Objetivo:** Garantizar que los procesos de conciliación se ejecuten solo cuando la información esté íntegra, gestionando la asincronía de la llegada de datos con tolerancia a fallos.

---

#### US-5.1: Validación de Insumos y Ejecución Asíncrona

**Como** Sistema (Motor de Workflows), **quiero** verificar continuamente la disponibilidad de todos los archivos según el schedule, **para** ejecutar la conciliación de forma segura o alertar de archivos faltantes.

- **AC-5.1.1:** El sistema registra en tiempo real el evento de llegada de datos por cada adaptador (API o RPA), independientemente de la hora.
- **AC-5.1.2 (Barrier Logic):** Al llegar la hora del schedule, la ejecución NO comienza hasta que todos los archivos requeridos tengan registro exitoso con la `fecha_ciclo` correspondiente.
- **AC-5.1.3:** Si faltan datos al llegar la hora, el sistema inicia una ventana de gracia configurable (ej. 30 min), reintentando la verificación cada 5 minutos.
- **AC-5.1.4:** Si expira la ventana de gracia, el estado pasa a "Pausado por faltantes" y se emite alerta crítica al Operador indicando qué adaptador está en falta.
- **AC-5.1.5:** Si el workflow tiene flag "Solo días hábiles", el motor reconoce feriados y fines de semana sin generar falsas alarmas.
- **AC-5.1.6:** Si la tarea estaba pausada y el Operador carga el archivo manualmente, el sistema detecta que el checklist está completo y reanuda la ejecución sin intervención adicional.
- **AC-5.1.7:** Estando el checklist al 100%, el motor invoca asíncronamente (Celery worker) el script Django `id_workflow` con rutas seguras a los archivos validados.
- **AC-5.1.8:** Si la ejecución tarda más de 20 minutos (configurable), el motor la detiene y notifica un error técnico al equipo de Analistas.
- **AC-5.1.9 (Idempotencia):** El motor verifica antes de ejecutar si la ejecución del día ya está marcada como "En Progreso" o "Terminada", previniendo conciliaciones dobles.
- **AC-5.1.10:** Durante la ejecución, la pantalla del Operador muestra el estado en tiempo real: "Esperando Datos" → "Conciliando..." → "Finalizado con Éxito/Errores".

**DoD:**
- [ ] Orquestador de tareas asíncronas configurado (Celery + Redis o RabbitMQ).
- [ ] Tabla `WorkflowRun` con estados: Programado, Esperando, En Gracia, Pausado, Ejecutando, Exitoso, Fallido, Timeout.
- [ ] Tests: workflow se retiene si falta 1 de 5 archivos; se dispara inmediatamente al llegar el 5to.
- [ ] Idempotencia implementada con locks de BD transaccionales.
- [ ] Monitor de timeout (Soft/Hard Time Limits en Celery).
- [ ] Endpoint de estado (Polling o WebSocket) para actualizar el frontend en tiempo real.

---

#### US-5.2: Calendario y Configuración Avanzada de Schedules

**Como** Operador, **quiero** configurar y visualizar el calendario de ejecución de mis workflows, **para** planificar la operación diaria y anticipar días sin ejecuciones.

- **AC-5.2.1:** Vista calendario mensual donde cada día muestra los workflows programados con indicador de estado (programado/completado/pausado).
- **AC-5.2.2:** Schedule soporta: diario (L-V), diario (L-S), semanal (día específico), quincenal, mensual (día fijo o último día hábil), y personalizado (días específicos).
- **AC-5.2.3:** El Admin puede cargar un calendario de feriados nacionales por año. Los workflows con flag "Solo días hábiles" los respetan automáticamente.
- **AC-5.2.4:** El operador puede marcar un día específico como "No ejecutar" (ej. cierre por inventario) sin modificar el schedule general.
- **AC-5.2.5:** El detalle de cada workflow muestra las próximas 5 ejecuciones programadas calculadas según schedule y calendario de feriados.

**DoD:**
- [ ] Modelo `CalendarioFeriados` y lógica de cálculo de próxima ejecución implementados.
- [ ] Motor de schedule soportando todas las frecuencias definidas.
- [ ] Vista de calendario mensual conectada al backend.
- [ ] Tests: workflow L-V no se programa en feriado ni fin de semana; excepción manual respetada.

---

### E6: Visualización de Resultados de Conciliación/Arqueo — *FUTURA*

**Objetivo:** Proveer al equipo operador de una interfaz clara y accionable para analizar los resultados de cada ejecución de workflow, diferenciando por tipo de proceso y permitiendo la toma de decisiones sobre discrepancias.

---

#### US-6.1: Dashboard de Resultados por Ejecución

**Como** Operador, **quiero** ver un dashboard con el resultado de cada ejecución de workflow, **para** analizar si la conciliación fue exitosa o si existen discrepancias que requieren atención.

- **AC-6.1.1:** Desde la vista del workflow, el operador accede al historial de ejecuciones con: fecha, duración, estado (Éxito/Con discrepancias/Error) y botón "Ver Resultado".
- **AC-6.1.2:** Resumen ejecutivo (KPIs): Total de transacciones procesadas, Monto total, Transacciones conciliadas, Transacciones con discrepancia y Porcentaje de conciliación.
- **AC-6.1.3:** El layout del resultado se adapta según el tipo de workflow:
  - **Conciliación bancaria:** Vista de dos columnas (Sistema vs. Banco) con líneas matcheadas y discrepancias resaltadas.
  - **Arqueo:** Vista de totales agrupados por sucursal/caja con diferencias calculadas.
  - **Otros tipos:** Layout configurable por el analista al crear el workflow.
- **AC-6.1.4:** Tabla de discrepancias filtrable por: monto, fecha, sucursal, tipo de discrepancia (faltante en sistema, faltante en banco, diferencia de monto).
- **AC-6.1.5:** El operador puede descargar el resultado en Excel (.xlsx) o CSV con todas las hojas relevantes (Resumen, Conciliados, Discrepancias).
- **AC-6.1.6:** El operador puede seleccionar dos ejecuciones del mismo workflow y ver un comparativo lado a lado de métricas clave para detectar tendencias.

**DoD:**
- [ ] Modelo `WorkflowResultado` con estructura genérica soportando distintos tipos de output.
- [ ] Componente de KPIs con datos reales del backend.
- [ ] Al menos dos layouts implementados (conciliación bancaria y arqueo).
- [ ] Tabla de discrepancias con filtros y paginación server-side.
- [ ] Endpoint de exportación a Excel implementado y testeado.
- [ ] Tests: 0 discrepancias → badge "Exitoso"; con discrepancias → badge "Con discrepancias".

---

#### US-6.2: Gestión de Discrepancias

**Como** Operador, **quiero** marcar, clasificar y resolver las discrepancias encontradas en una conciliación, **para** llevar un registro auditable de las acciones tomadas y cerrar el ciclo de conciliación.

- **AC-6.2.1:** Cada fila de discrepancia permite seleccionar una acción: "Justificar" (nota obligatoria), "Escalar a supervisor", "Marcar como error del banco" o "Marcar como error del sistema".
- **AC-6.2.2:** El operador puede seleccionar múltiples discrepancias del mismo tipo y aplicar una acción en lote (ej. justificar 15 diferencias menores a $100 como "redondeo").
- **AC-6.2.3:** Una ejecución se considera "Cerrada" cuando el 100% de las discrepancias tienen acción asignada. Hasta entonces permanece "Abierta".
- **AC-6.2.4:** Cada acción sobre una discrepancia queda registrada con: operador, fecha/hora, acción y nota. Registro inmutable y accesible para auditores.
- **AC-6.2.5:** El historial de ejecuciones muestra badge de cierre: "Abierta (12 discrepancias sin resolver)" o "Cerrada".
- **AC-6.2.6:** Si una ejecución lleva más de 24 horas "Abierta" con discrepancias sin resolver, se envía alerta al team.

**DoD:**
- [ ] Modelo `DiscrepanciaResolucion` con campos: discrepancia (FK), accion (enum), nota, operador, timestamp.
- [ ] Endpoint de resolución individual y masiva.
- [ ] Lógica de cálculo de estado de cierre automática (trigger al resolver la última discrepancia).
- [ ] Log de auditoría inmutable por cada resolución.
- [ ] Alerta de cierre pendiente (Celery periodic task).
- [ ] Tests: resolver todas las discrepancias → "Cerrada"; resolver parcialmente → "Abierta" con conteo correcto.

---

### 3.1 Mapa de Dependencias entre Épicas

```
E0 (Teams & Asignaciones)
    └── E1 (Adaptadores) ← requiere clientes y sucursales asignados a un team
            ├── E2 (Tareas) ← requiere adaptadores para generar tareas
            └── E3 (Solicitudes) ← requiere adaptadores para armar la solicitud
                    └── E4 (Panel Analista) ← requiere solicitudes enviadas
                            └── E5 (Ejecución) ← requiere workflows activados [FUTURA]
                                    └── E6 (Visualización) ← requiere resultados de ejecución [FUTURA]
```

### 3.2 Mapeo a Tickets

| ID PRD | Historia | Bloquea | Estado |
|--------|----------|---------|--------|
| US-0.1 | Gestión de Teams | US-0.2, US-0.3 | Pendiente |
| US-0.2 | Asignación de Operadores | US-0.3 | Pendiente |
| US-0.3 | Asignación de Clientes | E1 completa | Pendiente |
| US-1.1 | Adaptador tipo API | US-2.1, US-3.1 | Pendiente |
| US-1.2 | Adaptador tipo RPA | US-2.1, US-3.1 | Pendiente |
| US-2.1 | Tablero de Tareas | US-2.2, US-2.3 | Pendiente |
| US-2.2 | Asignación de Tareas | US-2.3 | Pendiente |
| US-2.3 | Seguimiento de Tareas | E3 | Pendiente |
| US-3.1 | Creación de Solicitud | US-3.2 | Pendiente |
| US-3.2 | Gestión de Estados | E4 | Pendiente |
| US-4.1 | Inbox del Analista | US-4.2 | Pendiente |
| US-4.2 | Integración y Activación | E5 | Pendiente |
| US-5.1 | Ejecución Asíncrona | US-5.2, E6 | Pendiente (Futura) |
| US-5.2 | Calendario de Schedules | E6 | Pendiente (Futura) |
| US-6.1 | Dashboard de Resultados | US-6.2 | Pendiente (Futura) |
| US-6.2 | Gestión de Discrepancias | — | Pendiente (Futura) |

---

## 4. Requerimientos

### 4.1 Funcionales

| ID | Descripción | Origen | Épica | Impacto en legado |
|----|-------------|--------|-------|-------------------|
| FR-01 | CRUD de Teams con soft delete y cascade check contra workflows activos | US-0.1 | E0 | Nuevo — M-11 |
| FR-02 | Asignación y reasignación de Operadores a Teams con log de auditoría | US-0.2 | E0 | Nuevo — M-11 |
| FR-03 | Asignación de Clientes/Sucursales a Teams con filtro de permisos por operador | US-0.3 | E0 | Refactor M-2 |
| FR-04 | Configuración guiada de Adaptador tipo API con test de credenciales y offuscación | US-1.1 | E1 | Refactor M-3 |
| FR-05 | Configuración de Adaptador tipo RPA con endpoint único, Bearer token y validación de esquema | US-1.2 | E1 | Nuevo — M-3 |
| FR-06 | Tablero Kanban/tabla de tareas con generación automática por schedule | US-2.1 | E2 | Nuevo — M-11 |
| FR-07 | Asignación y reasignación de tareas con historial y auto-distribución | US-2.2 | E2 | Nuevo — M-11 |
| FR-08 | Seguimiento de cumplimiento de tareas con barra de progreso y alertas de deadline | US-2.3 | E2 | Nuevo — M-11 |
| FR-09 | Creación de Solicitud de Workflow con adaptadores, horarios, adjuntos y generación de JSON metadata | US-3.1 | E3 | Nuevo — M-5 |
| FR-10 | Máquina de estados de Solicitud con transiciones bloqueantes, log inmutable y timeline | US-3.2 | E3 | Nuevo — M-5 |
| FR-11 | Inbox del Analista con control de concurrencia (lock), filtros y descarga de insumos en .zip | US-4.1 | E4 | Nuevo |
| FR-12 | Integración, Dry Run, checklist pre-activación bloqueante y versionado de workflows | US-4.2 | E4 | Nuevo |
| FR-13 | Ejecución asíncrona con Barrier Logic, ventana de gracia e idempotencia | US-5.1 | E5 (Futura) | Nuevo |
| FR-14 | Calendario de schedules con feriados y excepciones por día | US-5.2 | E5 (Futura) | Nuevo |
| FR-15 | Dashboard de resultados de conciliación adaptable por tipo de workflow | US-6.1 | E6 (Futura) | Nuevo |
| FR-16 | Gestión de discrepancias con resolución individual y masiva y log de auditoría | US-6.2 | E6 (Futura) | Nuevo |

### 4.2 No Funcionales

| ID | Categoría | Descripción | Métrica objetivo | Módulo(s) afectados |
|----|-----------|-------------|-----------------|---------------------|
| NFR-01 | Performance | Carga del tablero de tareas y del inbox del analista | < 2s en VPS Hostinger | E2, E4 |
| NFR-02 | Performance | Ejecución end-to-end de un workflow (sin Python API, modo mock) | < 3s | M-5, M-6 |
| NFR-03 | Seguridad | Credenciales de adaptadores siempre encriptadas en BD | 0 credenciales en plaintext en BD o logs | E1, M-3 |
| NFR-04 | Seguridad | Endpoints RPA autenticados con Bearer token de alta entropía | 0 endpoints de adaptador sin auth | E1 |
| NFR-05 | Seguridad | Operadores solo acceden a datos de su propio team | 0 accesos cross-team posibles (HTTP 403) | E0, E1, E2 |
| NFR-06 | Disponibilidad | El sistema opera en modo degradado si Python API no responde | 100% disponibilidad en modo mock | M-5 |
| NFR-07 | Observabilidad | Logs de workflow con contexto completo (adaptador, sucursal, ciclo, error) | Ver FR-09 AC | M-5, M-6 |
| NFR-08 | Testing | Coverage de servicios core | ≥ 80% en FileValidation, Execution, ConciliationData | M-5, M-6 |
| NFR-09 | Compatibilidad | Todos los cambios corren en MySQL 8 + PHP 8.2 en Hostinger | 0 regressions en prod | Global |
| NFR-10 | UX | Acciones destructivas (desactivar team, anular solicitud) requieren confirmación con detalle de impacto | 0 eliminaciones accidentales | E0, E3 |
| NFR-11 | Auditoría | Cambios críticos de estado (asignaciones, solicitudes, discrepancias) generan logs inmutables | Log persistido en BD para cada evento | E0, E3, E4, E6 |
| NFR-12 | UI | La UI sigue el sistema de diseño Aurora Glass (dark theme, glassmorphism) sin regresión visual | Sin regresión visual detectada | Global |

---

## 5. [TODO-DECISIONES] Pendientes

> ⚠️ **Revisar y resolver en equipo antes de iniciar la épica bloqueada por cada decisión.**

| # | Decisión | Bloquea | Opciones / Estado |
|---|----------|---------|-------------------|
| D-1 | ¿La normalización de datos bancarios va en el Adapter o en ConciliationDataService? | US-1.1 | ✅ **RESUELTA (2026-03-18):** Normalización en `ConciliationDataService`. El Adapter retorna respuesta cruda. Los mapeos de campos por banco van en `config/workflows.php['bank_field_maps']`. |
| D-2 | ¿Disponibilidad de sandbox AFIP/Ualá/NX? | Adapters AFIP, Ualá, NX | Confirmar con proveedor |
| D-3 | ¿Qué hace `LeadEnrichmentService`? ¿Se mantiene o elimina? | — | Auditar uso real antes de cualquier refactor de M-2 |
| D-4 | ¿`ConciliacionDataService` o `ConciliationDataService` es el canónico? | Cualquier refactor de M-6 | Auditar qué clases inyectan cada uno; elegir canónico y deprecar el otro |
| D-9 | ¿Ayuda contextual de credenciales en formulario de adaptador: tooltip/modal inline o link a doc externa? | Diseño UI de US-1.1 | Decidir antes del diseño de pantalla |
| D-10 | ¿El link de tarea RPA expira? ¿Tiene TTL configurable? ¿Se puede regenerar con invalidación inmediata? | Seguridad de US-1.2 | AC-1.2.12 asume regeneración manual; definir si además hay TTL automático |
| D-11 | ¿Un cliente puede pertenecer a más de un team simultáneamente? | Modelo de datos E0 | ✅ **RESUELTA (2026-03-18) por AC-0.3.4:** Sí, con asignación explícita de sucursales por team |
| D-12 | ¿Las tareas se auto-asignan (round-robin) o el operador las toma libremente del pool? | US-2.2 | AC-2.2.5 asume ambos (auto-distribuir + toma libre); confirmar comportamiento por defecto |
| D-13 | ¿El operador puede editar el schedule del workflow activo, o solo el analista? | US-5.2 | Definir permisos antes de implementar US-5.2 |
| D-14 | ¿Qué pasa cuando un Excel llega repetido para la misma tarea? ¿Overwrite, reject o versioning? | Idempotencia US-1.2 / US-2.3 | Definir antes de implementar el endpoint RPA |
| D-15 | ¿Los templates de adaptadores (ej: "Mercado Pago") son del sistema o los gestiona el analista? | Catálogo de templates E1 | Impacta arquitectura del formulario de US-1.1 |

---

## 6. Oportunidades Futuras *(solo referencia — fuera de scope)*

1. **Procesamiento asíncrono de workflows** (Laravel Queue + Horizon) para archivos Excel grandes sin timeout HTTP.
2. **Dashboard de ROI por cliente**: mostrar cuánto tiempo ahorró la automatización vs proceso manual.
3. **Notificaciones en tiempo real** (Livewire polling o WebSockets) cuando un workflow termina.
4. **Multi-tenancy**: separar datos por empresa cliente (para escalar como SaaS a múltiples contadores).
5. **API pública REST** para que los clientes integren resultados de conciliación en sus propios sistemas.
6. **Artisan command `integration:make NombreBanco`** para generar skeleton de adapter bancario.
7. **Adapters AFIP, Ualá, Naranja X** (sujeto a disponibilidad de sandbox de cada proveedor).
