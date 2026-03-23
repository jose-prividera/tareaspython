# Casos de Uso de la Plataforma Front-API

Este documento describe diez casos de uso representativos de las capacidades actuales de la plataforma (versión post US-4.2). Está dirigido a stakeholders, equipos de onboarding y demostraciones, por lo que se utiliza lenguaje natural sin jerga técnica innecesaria.

---

## CU-1: Conciliación financiera diaria de un comercio

**Roles involucrados:** Operador, Analista

Un operador sube los archivos Excel del día: ventas del sistema POS, transacciones de Getnet y pagos de MercadoPago. La plataforma guía el proceso a través de un wizard de cuatro pasos, valida la estructura de cada archivo y lo envía a la API de procesamiento. Al finalizar, se genera un reporte de conciliación que muestra transacciones conciliadas versus no conciliadas, diferencias por turno, movimientos de caja y devoluciones registradas. El analista puede consultar en cualquier momento el dashboard de conciliación con resúmenes por cliente y sucursal.

---

## CU-2: Generación de reportes PDF de anulaciones y no conciliados

**Rol involucrado:** Analista

Desde el wizard de informes, el analista selecciona el cliente, la sede, la fecha y el turno que desea reportar, y luego elige el tipo de documento: Anulaciones, No Conciliados, Egresos o Sucursal. La plataforma genera el PDF correspondiente, que puede previsualizarse directamente en el navegador o descargarse para compartir. El proceso es inmediato y no requiere intervención técnica adicional.

---

## CU-3: Configuración de un adaptador bancario por API directa (MercadoPago)

**Rol involucrado:** Operador

Un operador accede al wizard de configuración de adaptadores, selecciona la sucursal correspondiente, el servicio bancario (por ejemplo, MercadoPago), el entorno (pruebas o producción) y carga las credenciales de acceso. La plataforma encripta las credenciales de forma segura antes de guardarlas, ejecuta automáticamente una prueba de conexión para verificar que son válidas, y registra el adaptador como listo para operar. Todas las acciones quedan registradas en el log de auditoría del sistema.

---

## CU-4: Configuración de un adaptador RPA con endpoint dedicado

**Rol involucrado:** Operador

Para entidades bancarias que no ofrecen una API de integración directa, el operador configura un adaptador de tipo RPA: sube un archivo Excel de muestra, el sistema extrae automáticamente el esquema de columnas esperado, genera un token de autenticación único y crea un endpoint exclusivo para esa tarea. A partir de entonces, el bot RPA externo puede enviar los archivos Excel periódicamente a ese endpoint usando el token, sin intervención manual del operador.

---

## CU-5: Gestión de tareas diarias del equipo de operadores

**Roles involucrados:** Operador, Administrador

La plataforma genera automáticamente las tareas diarias que corresponden a cada adaptador activo de cada sucursal. Cada operador accede a su tablero personal con vista tipo Kanban, puede auto-asignarse las tareas disponibles y marcarlas como completadas al finalizarlas. Cuando una tarea se acerca a su fecha límite, el operador recibe una alerta visible en la interfaz. El administrador puede, en cualquier momento, redistribuir las tareas de forma equitativa entre todos los operadores del equipo usando la función de auto-distribución.

---

## CU-6: Solicitud de un nuevo workflow operativo

**Rol involucrado:** Operador

El operador crea una solicitud formal para poner en marcha un nuevo workflow de procesamiento: selecciona el cliente y la sucursal, elige los adaptadores que lo alimentarán, define las reglas de conciliación, el horario operativo, la frecuencia de ejecución y puede adjuntar hasta tres archivos de ejemplo. El sistema valida que los adaptadores seleccionados estén en estado válido y notifica automáticamente a todos los analistas disponibles. El operador puede consultar el estado de su solicitud en todo momento y cancelarla si aún no fue tomada.

---

## CU-7: Revisión y aprobación de solicitudes de workflow

**Rol involucrado:** Analista

El analista accede a su bandeja de solicitudes, donde puede filtrar por estado, cliente, equipo o tipo de workflow. Al abrir una solicitud, el sistema reserva un bloqueo exclusivo de una hora para ese analista, evitando que dos analistas trabajen sobre la misma solicitud simultáneamente. Desde esa pantalla, el analista puede descargar todos los insumos en un archivo comprimido (metadata y adjuntos), solicitar cambios al operador con un comentario explicativo, o aprobar la solicitud para dar inicio al desarrollo del workflow. Todas las interacciones quedan registradas en un timeline auditable visible para ambas partes.

---

## CU-8: Validación, simulación y activación de un workflow

**Rol involucrado:** Analista

Una vez que el workflow está en desarrollo, el analista registra el identificador del workflow en el sistema backend y lo valida contra el registro. Al avanzar a la fase de pruebas, aparece un checklist de pre-activación con los requisitos mínimos a cumplir. El analista ejecuta una simulación ("Dry Run") que procesa los datos de prueba y muestra en pantalla el tiempo de ejecución, la cantidad de filas procesadas, los errores o advertencias encontrados y una vista previa de los primeros resultados. Si la simulación es exitosa, el ítem del checklist se marca automáticamente. Una vez completado el checklist, el analista activa el workflow y el operador recibe una notificación de confirmación.

---

## CU-9: Gestión de equipos y asignación de sucursales

**Rol involucrado:** Administrador

El administrador crea equipos de trabajo, les asigna un nombre y un color identificatorio, e incorpora operadores a cada uno. Luego asigna las sucursales de clientes que corresponden a ese equipo, de modo que cada operador solo vea y trabaje con las sucursales que le fueron asignadas. Los equipos pueden activarse o desactivarse según la necesidad, con la validación de que no es posible desactivar un equipo que tenga workflows activos en ese momento.

---

## CU-10: Arqueo mensual de caja por sucursal

**Rol involucrado:** Analista

El analista selecciona el cliente, la sucursal, el año y el mes que desea arquear desde el wizard correspondiente. La plataforma consolida todos los datos de conciliación disponibles para ese período, organizados por turno, y los envía a la API de procesamiento (o en modo de simulación si no hay conexión). El resultado es el arqueo mensual completo, con totales de ventas, pagos electrónicos, efectivo registrado y diferencias detectadas. El reporte queda disponible dentro del detalle de conciliación de esa sucursal para consulta posterior.

---

## CU-11: Flujo operativo completo del operador Walter Jalaf (caso de prueba manual)

**Rol involucrado:** Operador

**Credenciales de acceso:** `walter.jalaf@demo.com` / `password`

**Setup requerido:** `php artisan db:seed --class=WalterJalafDemoSeeder`

### Pasos de verificación

**1. Login y redirección**
- Ir a `/login`, ingresar las credenciales de Walter.
- El sistema debe redirigir al dashboard del operador (`/dashboard`), no al de programador ni al de admin.

**2. Ver clientes del team (`/operador/clients`)**
- Walter debe ver exactamente 3 clientes pertenecientes a su team "Team Conciliaciones Walter":
  - `Supermercado Jalaf S.A.` (sede, sin sucursal padre)
  - `Sucursal Flores` (sucursal de Supermercado Jalaf S.A.)
  - `Sucursal Caballito` (sucursal de Supermercado Jalaf S.A.)
- No debe ver clientes de otros teams.

**3. Ver tareas del día (`/operador/tasks`)**
- Walter debe ver 2 tareas en estado **Pendiente**, asignadas a él:
  - `Conciliación Flores — <fecha de hoy>` (adaptador RPA)
  - `Conciliación Caballito — <fecha de hoy>` (adaptador Mercado Pago sandbox)
- El tablero Kanban debe mostrar ambas en la columna "Pendiente".

**4. Ver solicitudes de workflow (`/operador/solicitudes`)**
- Walter debe ver 1 solicitud en estado **Pendiente**:
  - `Conciliación diaria Sucursal Flores`
- Puede consultar el detalle y cancelarla si lo desea (mientras siga en estado Pendiente).

**5. Verificar acceso restringido**
- Intentar acceder a `/programadores/dashboard` → debe devolver **403 Forbidden**.
- Intentar acceder a `/admin/dashboard` → debe devolver **403 Forbidden**.
- Walter no debe ver ningún menú ni enlace de los roles Programador o Admin.

### Resultado esperado

| Pantalla | Resultado |
|----------|-----------|
| `/operador/tasks` | 2 tareas pendientes visibles |
| `/operador/clients` | 3 clientes del team |
| `/operador/solicitudes` | 1 solicitud pendiente |
| `/programadores/dashboard` | 403 |
| `/admin/dashboard` | 403 |
