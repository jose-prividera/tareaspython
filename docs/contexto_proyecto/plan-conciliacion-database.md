# Plan: Almacenamiento y Visualización de Datos de Conciliación

## Resumen

Crear tablas de base de datos para almacenar **toda** la información del campo `data` de la respuesta del workflow "Conciliador", vistas completas con exportación a Excel/PDF, verificación del estado de la respuesta, y **filtrado de registros duplicados** para evitar almacenar información repetida entre ejecuciones.

---

## Estructura de Datos en `arqueo_resultado_test.json`

| Sección | Descripción | Campos clave |
|---------|-------------|--------------|
| `arqueo_por_turno` | Resumen completo por turno | ~60 campos: ventas, conciliación MP/Getnet/Efectivo, facturación, ventas por hora |
| `getnet_conciliado` | Transacciones Getnet | establecimiento, fecha, marca, tarjeta, monto, estado conciliación |
| `mp_conciliado` | Transacciones MercadoPago | id_operación, fecha, monto, estado, tipo_match |
| `sistema_conciliado` | Ventas sistema conciliadas | id_ticket, monto, método_pago, estado_conciliación |
| `ventas_sistema` | Todas las ventas | similar a sistema_conciliado |
| `turnos_procesados` | Info de turnos | fecha, hora apertura/cierre, encargado, comensales |
| `devoluciones` | Anulaciones | producto, monto, comentario, hora |
| `caja_adicion` | Movimientos caja | tipo, monto, comentario, usuario |
| `mp_negativos` | Movimientos negativos MP | id_operación, monto negativo |

---

## Estrategia de Filtrado de Duplicados

### Claves Únicas por Tabla

Para evitar registros duplicados entre ejecuciones, cada tabla tendrá una **clave única compuesta** basada en campos que identifican unívocamente cada registro:

| Tabla | Clave Única | Justificación |
|-------|-------------|---------------|
| `conciliation_summaries` | `fecha + turno + encargado` | Un turno por fecha/encargado es único |
| `conciliation_getnet_transactions` | `cod_transaccion` | ID único de Getnet |
| `conciliation_mp_transactions` | `id_operacion_mp` | ID único de MercadoPago |
| `conciliation_system_sales` | `id_ticket` | ID único del sistema POS |
| `conciliation_cash_movements` | `fecha_contable + fecha_modificacion + monto + comentario` | Combinación única |
| `conciliation_shifts` | `fecha_apertura + hora_apertura + turno` | Turno único por fecha/hora |
| `conciliation_refunds` | `fecha_hora_pedido + producto + precio` | Anulación única |
| `conciliation_mp_negatives` | `id_operacion_mp` | ID único de MercadoPago |

### Estrategia de Inserción

Se utilizará **`upsert`** de Laravel para:
1. Insertar registros nuevos
2. Actualizar registros existentes si cambian (ej: estado de conciliación)
3. Ignorar registros idénticos

```php
// Ejemplo de upsert con clave única
ConciliationGetnetTransaction::upsert(
    $records,                           // Datos a insertar
    ['cod_transaccion'],                // Columnas de clave única
    ['estado_conciliacion', 'tipo_match', 'id_venta_sistema'] // Columnas a actualizar si existe
);
```

---

## Plan de Implementación

### Fase 1: Migraciones de Base de Datos

Crear migración única `create_conciliation_tables.php` con **8 tablas** incluyendo índices únicos:

#### 1.1 `conciliation_summaries`
```php
Schema::create('conciliation_summaries', function (Blueprint $table) {
    $table->id();
    $table->foreignId('workflow_execution_id')->constrained()->onDelete('cascade');

    // Datos del turno
    $table->date('fecha');
    $table->string('dia', 20);
    $table->string('turno', 20);
    $table->string('encargado', 100)->nullable();
    $table->dateTime('apertura')->nullable();
    $table->dateTime('cierre')->nullable();
    $table->decimal('horas_trabajadas', 10, 4)->nullable();

    // Ventas generales
    $table->decimal('ventas_totales', 15, 2)->default(0);
    $table->integer('cantidad_comensales')->default(0);
    $table->decimal('ticket_promedio', 12, 2)->default(0);
    $table->integer('cantidad_tickets')->default(0);
    $table->decimal('propina', 12, 2)->default(0);

    // MercadoPago
    $table->decimal('mp_ventas_real', 15, 2)->default(0);
    $table->decimal('mp_ventas_sistema', 15, 2)->default(0);
    $table->decimal('mp_conciliado', 15, 2)->default(0);
    $table->decimal('mp_no_conciliado', 15, 2)->default(0);
    $table->decimal('mp_diferencia', 15, 2)->default(0);
    $table->decimal('mp_porcentaje', 8, 4)->default(0);
    $table->string('mp_estado', 50)->nullable();

    // Getnet
    $table->decimal('getnet_ventas_real', 15, 2)->default(0);
    $table->decimal('getnet_ventas_sistema', 15, 2)->default(0);
    $table->decimal('getnet_conciliado', 15, 2)->default(0);
    $table->decimal('getnet_no_conciliado', 15, 2)->default(0);
    $table->decimal('getnet_diferencia', 15, 2)->default(0);
    $table->decimal('getnet_porcentaje', 8, 4)->default(0);
    $table->string('getnet_estado', 50)->nullable();

    // Efectivo
    $table->decimal('efectivo_total', 15, 2)->default(0);
    $table->decimal('efectivo_apertura_caja', 15, 2)->default(0);
    $table->decimal('efectivo_recuento', 15, 2)->default(0);
    $table->decimal('efectivo_diferencia', 15, 2)->default(0);
    $table->decimal('efectivo_porcentaje', 8, 4)->default(0);
    $table->string('efectivo_estado', 50)->nullable();

    // Cuenta Corriente
    $table->decimal('cta_cte_total', 15, 2)->default(0);
    $table->decimal('otros', 15, 2)->default(0);

    // Facturación
    $table->decimal('descuentos', 15, 2)->default(0);
    $table->decimal('ventas_facturadas', 15, 2)->default(0);
    $table->decimal('ideal_facturacion', 15, 2)->default(0);
    $table->decimal('diferencia_facturacion', 15, 2)->default(0);
    $table->decimal('porcentaje_facturacion', 8, 4)->default(0);

    // Ventas por hora (24 slots)
    $table->json('ventas_por_hora')->nullable();

    $table->timestamps();

    // Índices
    $table->index(['workflow_execution_id', 'fecha']);
    $table->index('turno');

    // CLAVE ÚNICA para evitar duplicados
    $table->unique(['fecha', 'turno', 'encargado'], 'unique_summary_turno');
});
```

#### 1.2 `conciliation_getnet_transactions`
```php
Schema::create('conciliation_getnet_transactions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('workflow_execution_id')->constrained()->onDelete('cascade');

    $table->string('nro_establecimiento', 50)->nullable();
    $table->string('nombre_establecimiento', 200)->nullable();
    $table->dateTime('fecha_operacion')->nullable();
    $table->string('billetera', 50)->nullable();
    $table->string('marca', 50)->nullable();
    $table->string('tipo_tarjeta', 50)->nullable();
    $table->string('tarjeta_ultimos4', 10)->nullable();
    $table->string('tipo_transaccion', 50)->nullable();
    $table->string('canal', 50)->nullable();
    $table->string('modo_canal', 50)->nullable();
    $table->string('codigo_pos', 50)->nullable();
    $table->string('estado_venta', 50)->nullable();
    $table->string('cod_transaccion', 100)->nullable();
    $table->string('cod_transaccion_externo', 100)->nullable();
    $table->string('nro_cupon', 50)->nullable();
    $table->string('cod_autorizacion', 50)->nullable();
    $table->string('plan_cuotas', 20)->nullable();
    $table->string('moneda', 10)->default('ARS');
    $table->decimal('monto_bruto', 15, 2)->default(0);
    $table->decimal('arancel', 12, 2)->default(0);
    $table->decimal('iva_arancel', 12, 2)->default(0);
    $table->decimal('propina', 12, 2)->default(0);
    $table->decimal('monto_neto', 15, 2)->default(0);
    $table->date('fecha_liquidacion')->nullable();
    $table->date('fecha_pago')->nullable();
    $table->string('cod_liquidacion', 100)->nullable();
    $table->string('turno', 20)->nullable();
    $table->string('estado_conciliacion', 50)->nullable();
    $table->string('tipo_match', 100)->nullable();
    $table->string('id_venta_sistema', 50)->nullable();
    $table->date('fecha_solo')->nullable();

    $table->timestamps();

    // Índices
    $table->index(['workflow_execution_id', 'fecha_operacion']);
    $table->index('estado_conciliacion');
    $table->index('turno');

    // CLAVE ÚNICA - cod_transaccion es único en Getnet
    $table->unique('cod_transaccion', 'unique_getnet_transaction');
});
```

#### 1.3 `conciliation_mp_transactions`
```php
Schema::create('conciliation_mp_transactions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('workflow_execution_id')->constrained()->onDelete('cascade');

    $table->date('fecha')->nullable();
    $table->time('hora')->nullable();
    $table->string('id_operacion_mp', 50)->nullable();
    $table->decimal('monto_neto', 15, 2)->default(0);
    $table->string('tipo_movimiento', 100)->nullable();
    $table->string('medio_pago', 100)->nullable();
    $table->string('metodo_pago', 100)->nullable();
    $table->integer('cuotas')->nullable();
    $table->string('estado', 50)->nullable();
    $table->string('turno', 20)->nullable();
    $table->string('estado_conciliacion', 50)->nullable();
    $table->string('tipo_match', 100)->nullable();
    $table->string('id_venta_sistema', 50)->nullable();
    $table->date('fecha_solo')->nullable();

    $table->timestamps();

    // Índices
    $table->index(['workflow_execution_id', 'fecha']);
    $table->index('estado_conciliacion');
    $table->index('turno');

    // CLAVE ÚNICA - id_operacion_mp es único en MercadoPago
    $table->unique('id_operacion_mp', 'unique_mp_transaction');
});
```

#### 1.4 `conciliation_system_sales`
```php
Schema::create('conciliation_system_sales', function (Blueprint $table) {
    $table->id();
    $table->foreignId('workflow_execution_id')->constrained()->onDelete('cascade');

    $table->string('id_ticket', 50)->nullable();
    $table->dateTime('fecha_hora')->nullable();
    $table->time('hora')->nullable();
    $table->integer('items_count')->default(0);
    $table->decimal('monto_total', 15, 2)->default(0);
    $table->string('tipo_venta', 50)->nullable();
    $table->string('estado_venta', 50)->nullable();
    $table->string('metodo_pago', 100)->nullable();
    $table->string('turno', 20)->nullable();
    $table->string('estado_conciliacion', 50)->nullable();
    $table->string('tipo_match', 100)->nullable();
    $table->string('id_operacion_conciliado', 50)->nullable();
    $table->date('fecha_solo')->nullable();
    $table->boolean('conciliado')->default(false);
    $table->string('source', 20)->default('sistema'); // 'sistema' o 'conciliado'

    $table->timestamps();

    // Índices
    $table->index(['workflow_execution_id', 'fecha_hora']);
    $table->index('estado_conciliacion');
    $table->index('turno');

    // CLAVE ÚNICA - id_ticket es único en el sistema POS
    $table->unique(['id_ticket', 'source'], 'unique_system_sale');
});
```

#### 1.5 `conciliation_cash_movements`
```php
Schema::create('conciliation_cash_movements', function (Blueprint $table) {
    $table->id();
    $table->foreignId('workflow_execution_id')->constrained()->onDelete('cascade');

    $table->date('fecha_contable')->nullable();
    $table->dateTime('fecha_modificacion')->nullable();
    $table->string('origen', 50)->nullable();
    $table->string('clase', 100)->nullable();
    $table->string('proveedor_para', 200)->nullable();
    $table->decimal('monto', 15, 2)->default(0);
    $table->text('comentario')->nullable();
    $table->string('usuario', 200)->nullable();
    $table->string('tipo', 20)->nullable(); // Ingreso/Egreso
    $table->string('forma_pago', 50)->nullable();
    $table->string('cuenta_contable', 100)->nullable();
    $table->string('turno', 20)->nullable();

    // Hash único para identificar el movimiento
    $table->string('unique_hash', 64)->nullable();

    $table->timestamps();

    // Índices
    $table->index(['workflow_execution_id', 'fecha_contable']);
    $table->index('tipo');
    $table->index('turno');

    // CLAVE ÚNICA basada en hash de campos identificadores
    $table->unique('unique_hash', 'unique_cash_movement');
});
```

#### 1.6 `conciliation_shifts`
```php
Schema::create('conciliation_shifts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('workflow_execution_id')->constrained()->onDelete('cascade');

    $table->date('fecha_apertura')->nullable();
    $table->time('hora_apertura')->nullable();
    $table->date('fecha_cierre')->nullable();
    $table->time('hora_cierre')->nullable();
    $table->string('turno', 20)->nullable();
    $table->string('encargado', 100)->nullable();
    $table->integer('cantidad_comensales')->default(0);
    $table->decimal('recuento_efectivo', 15, 2)->default(0);
    $table->decimal('apertura_caja', 15, 2)->default(0);

    $table->timestamps();

    // Índices
    $table->index(['workflow_execution_id', 'fecha_apertura']);
    $table->index('turno');

    // CLAVE ÚNICA - turno único por fecha/hora
    $table->unique(['fecha_apertura', 'hora_apertura', 'turno'], 'unique_shift');
});
```

#### 1.7 `conciliation_refunds`
```php
Schema::create('conciliation_refunds', function (Blueprint $table) {
    $table->id();
    $table->foreignId('workflow_execution_id')->constrained()->onDelete('cascade');

    $table->string('producto', 200)->nullable();
    $table->decimal('precio', 12, 2)->default(0);
    $table->text('comentario')->nullable();
    $table->dateTime('fecha_hora_pedido')->nullable();
    $table->time('hora_pedido')->nullable();
    $table->string('hora_anulacion', 50)->nullable();
    $table->string('turno', 20)->nullable();

    // Hash único para identificar la devolución
    $table->string('unique_hash', 64)->nullable();

    $table->timestamps();

    // Índices
    $table->index(['workflow_execution_id']);
    $table->index('turno');

    // CLAVE ÚNICA basada en hash
    $table->unique('unique_hash', 'unique_refund');
});
```

#### 1.8 `conciliation_mp_negatives`
```php
Schema::create('conciliation_mp_negatives', function (Blueprint $table) {
    $table->id();
    $table->foreignId('workflow_execution_id')->constrained()->onDelete('cascade');

    $table->date('fecha')->nullable();
    $table->time('hora')->nullable();
    $table->string('id_operacion_mp', 50)->nullable();
    $table->decimal('monto_neto', 15, 2)->default(0);
    $table->string('turno', 20)->nullable();

    $table->timestamps();

    // Índices
    $table->index(['workflow_execution_id', 'fecha']);

    // CLAVE ÚNICA - id_operacion_mp es único
    $table->unique('id_operacion_mp', 'unique_mp_negative');
});
```

---

### Fase 2: Modelos Eloquent

Crear en `app/Models/Conciliation/`:

| Modelo | Relación | Clave Única |
|--------|----------|-------------|
| `ConciliationSummary` | BelongsTo WorkflowExecution | `fecha + turno + encargado` |
| `ConciliationGetnetTransaction` | BelongsTo WorkflowExecution | `cod_transaccion` |
| `ConciliationMpTransaction` | BelongsTo WorkflowExecution | `id_operacion_mp` |
| `ConciliationSystemSale` | BelongsTo WorkflowExecution | `id_ticket + source` |
| `ConciliationCashMovement` | BelongsTo WorkflowExecution | `unique_hash` |
| `ConciliationShift` | BelongsTo WorkflowExecution | `fecha_apertura + hora_apertura + turno` |
| `ConciliationRefund` | BelongsTo WorkflowExecution | `unique_hash` |
| `ConciliationMpNegative` | BelongsTo WorkflowExecution | `id_operacion_mp` |

#### Ejemplo de Modelo con Hash Único

```php
// app/Models/Conciliation/ConciliationCashMovement.php
class ConciliationCashMovement extends Model
{
    protected $fillable = [
        'workflow_execution_id',
        'fecha_contable',
        'fecha_modificacion',
        'origen',
        'clase',
        'proveedor_para',
        'monto',
        'comentario',
        'usuario',
        'tipo',
        'forma_pago',
        'cuenta_contable',
        'turno',
        'unique_hash',
    ];

    protected $casts = [
        'fecha_contable' => 'date',
        'fecha_modificacion' => 'datetime',
        'monto' => 'decimal:2',
    ];

    /**
     * Generar hash único basado en campos identificadores
     */
    public static function generateUniqueHash(array $data): string
    {
        $key = implode('|', [
            $data['fecha_contable'] ?? '',
            $data['fecha_modificacion'] ?? '',
            $data['monto'] ?? 0,
            substr($data['comentario'] ?? '', 0, 50),
            $data['tipo'] ?? '',
        ]);

        return hash('sha256', $key);
    }

    protected static function booted()
    {
        static::creating(function ($model) {
            if (empty($model->unique_hash)) {
                $model->unique_hash = self::generateUniqueHash($model->toArray());
            }
        });
    }

    public function workflowExecution()
    {
        return $this->belongsTo(WorkflowExecution::class);
    }
}
```

Modificar `WorkflowExecution.php` para agregar relaciones HasMany.

---

### Fase 3: Servicio de Persistencia con Filtrado de Duplicados

Crear `app/Services/ConciliationDataService.php`:

```php
<?php

namespace App\Services;

use App\Models\WorkflowExecution;
use App\Models\Conciliation\ConciliationSummary;
use App\Models\Conciliation\ConciliationGetnetTransaction;
use App\Models\Conciliation\ConciliationMpTransaction;
use App\Models\Conciliation\ConciliationSystemSale;
use App\Models\Conciliation\ConciliationCashMovement;
use App\Models\Conciliation\ConciliationShift;
use App\Models\Conciliation\ConciliationRefund;
use App\Models\Conciliation\ConciliationMpNegative;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class ConciliationDataService
{
    /**
     * Procesar y guardar datos de conciliación evitando duplicados
     */
    public function processAndSave(WorkflowExecution $execution, array $response): array
    {
        $stats = [
            'inserted' => 0,
            'updated' => 0,
            'skipped' => 0,
        ];

        // 1. Validar status
        if (!$this->validateResponse($response)) {
            Log::warning('Conciliation response validation failed', [
                'execution_id' => $execution->id,
                'status' => $response['status'] ?? 'unknown',
            ]);
            return $stats;
        }

        // 2. Persistir en transacción
        DB::transaction(function () use ($execution, $response, &$stats) {
            $data = $response['data'];

            $stats['summaries'] = $this->saveSummaries($execution, $data['arqueo_por_turno'] ?? []);
            $stats['getnet'] = $this->saveGetnetTransactions($execution, $data['getnet_conciliado'] ?? []);
            $stats['mp'] = $this->saveMpTransactions($execution, $data['mp_conciliado'] ?? []);
            $stats['system_conciliado'] = $this->saveSystemSales($execution, $data['sistema_conciliado'] ?? [], 'conciliado');
            $stats['system_ventas'] = $this->saveSystemSales($execution, $data['ventas_sistema'] ?? [], 'sistema');
            $stats['shifts'] = $this->saveShifts($execution, $data['turnos_procesados'] ?? []);
            $stats['cash'] = $this->saveCashMovements($execution, $data['caja_adicion'] ?? []);
            $stats['refunds'] = $this->saveRefunds($execution, $data['devoluciones'] ?? []);
            $stats['mp_negatives'] = $this->saveMpNegatives($execution, $data['mp_negativos'] ?? []);
        });

        Log::info('Conciliation data processed', [
            'execution_id' => $execution->id,
            'stats' => $stats,
        ]);

        return $stats;
    }

    /**
     * Validar que la respuesta sea exitosa
     */
    public function validateResponse(array $response): bool
    {
        return ($response['status'] ?? '') === 'success'
            && isset($response['data']);
    }

    /**
     * Guardar resúmenes por turno (upsert por fecha+turno+encargado)
     */
    private function saveSummaries(WorkflowExecution $execution, array $records): array
    {
        $stats = ['inserted' => 0, 'updated' => 0];

        foreach (array_chunk($records, 100) as $chunk) {
            $data = array_map(fn($r) => $this->mapSummary($execution, $r), $chunk);

            $result = ConciliationSummary::upsert(
                $data,
                ['fecha', 'turno', 'encargado'], // Unique keys
                $this->getSummaryUpdateColumns()  // Columns to update
            );

            $stats['processed'] = count($records);
        }

        return $stats;
    }

    /**
     * Guardar transacciones Getnet (upsert por cod_transaccion)
     */
    private function saveGetnetTransactions(WorkflowExecution $execution, array $records): array
    {
        $stats = ['processed' => 0];

        foreach (array_chunk($records, 500) as $chunk) {
            $data = array_map(fn($r) => $this->mapGetnetTransaction($execution, $r), $chunk);

            // Filtrar registros sin cod_transaccion (no se pueden deduplicar)
            $data = array_filter($data, fn($r) => !empty($r['cod_transaccion']));

            if (!empty($data)) {
                ConciliationGetnetTransaction::upsert(
                    array_values($data),
                    ['cod_transaccion'],
                    ['estado_conciliacion', 'tipo_match', 'id_venta_sistema', 'workflow_execution_id', 'updated_at']
                );
            }

            $stats['processed'] += count($chunk);
        }

        return $stats;
    }

    /**
     * Guardar transacciones MercadoPago (upsert por id_operacion_mp)
     */
    private function saveMpTransactions(WorkflowExecution $execution, array $records): array
    {
        $stats = ['processed' => 0];

        foreach (array_chunk($records, 500) as $chunk) {
            $data = array_map(fn($r) => $this->mapMpTransaction($execution, $r), $chunk);

            // Filtrar registros sin id_operacion_mp
            $data = array_filter($data, fn($r) => !empty($r['id_operacion_mp']));

            if (!empty($data)) {
                ConciliationMpTransaction::upsert(
                    array_values($data),
                    ['id_operacion_mp'],
                    ['estado_conciliacion', 'tipo_match', 'id_venta_sistema', 'workflow_execution_id', 'updated_at']
                );
            }

            $stats['processed'] += count($chunk);
        }

        return $stats;
    }

    /**
     * Guardar ventas del sistema (upsert por id_ticket + source)
     */
    private function saveSystemSales(WorkflowExecution $execution, array $records, string $source): array
    {
        $stats = ['processed' => 0];

        foreach (array_chunk($records, 500) as $chunk) {
            $data = array_map(fn($r) => $this->mapSystemSale($execution, $r, $source), $chunk);

            // Filtrar registros sin id_ticket
            $data = array_filter($data, fn($r) => !empty($r['id_ticket']));

            if (!empty($data)) {
                ConciliationSystemSale::upsert(
                    array_values($data),
                    ['id_ticket', 'source'],
                    ['estado_conciliacion', 'tipo_match', 'id_operacion_conciliado', 'conciliado', 'workflow_execution_id', 'updated_at']
                );
            }

            $stats['processed'] += count($chunk);
        }

        return $stats;
    }

    /**
     * Guardar turnos (upsert por fecha_apertura + hora_apertura + turno)
     */
    private function saveShifts(WorkflowExecution $execution, array $records): array
    {
        $stats = ['processed' => 0];

        $data = array_map(fn($r) => $this->mapShift($execution, $r), $records);

        ConciliationShift::upsert(
            $data,
            ['fecha_apertura', 'hora_apertura', 'turno'],
            ['cantidad_comensales', 'recuento_efectivo', 'apertura_caja', 'workflow_execution_id', 'updated_at']
        );

        $stats['processed'] = count($records);
        return $stats;
    }

    /**
     * Guardar movimientos de caja (upsert por unique_hash)
     */
    private function saveCashMovements(WorkflowExecution $execution, array $records): array
    {
        $stats = ['processed' => 0];

        foreach (array_chunk($records, 200) as $chunk) {
            $data = array_map(fn($r) => $this->mapCashMovement($execution, $r), $chunk);

            ConciliationCashMovement::upsert(
                $data,
                ['unique_hash'],
                ['workflow_execution_id', 'updated_at']
            );

            $stats['processed'] += count($chunk);
        }

        return $stats;
    }

    /**
     * Guardar devoluciones (upsert por unique_hash)
     */
    private function saveRefunds(WorkflowExecution $execution, array $records): array
    {
        $stats = ['processed' => 0];

        $data = array_map(fn($r) => $this->mapRefund($execution, $r), $records);

        ConciliationRefund::upsert(
            $data,
            ['unique_hash'],
            ['workflow_execution_id', 'updated_at']
        );

        $stats['processed'] = count($records);
        return $stats;
    }

    /**
     * Guardar MP negativos (upsert por id_operacion_mp)
     */
    private function saveMpNegatives(WorkflowExecution $execution, array $records): array
    {
        $stats = ['processed' => 0];

        $data = array_map(fn($r) => $this->mapMpNegative($execution, $r), $records);

        // Filtrar sin id_operacion_mp
        $data = array_filter($data, fn($r) => !empty($r['id_operacion_mp']));

        if (!empty($data)) {
            ConciliationMpNegative::upsert(
                array_values($data),
                ['id_operacion_mp'],
                ['workflow_execution_id', 'updated_at']
            );
        }

        $stats['processed'] = count($records);
        return $stats;
    }

    // ==================== MAPPERS ====================

    private function mapSummary(WorkflowExecution $execution, array $r): array
    {
        return [
            'workflow_execution_id' => $execution->id,
            'fecha' => $this->parseDate($r['Fecha'] ?? null),
            'dia' => $r['Dia'] ?? null,
            'turno' => $r['TURNO'] ?? null,
            'encargado' => $r['Encargado'] ?? null,
            'apertura' => $this->parseDateTime($r['Apertura'] ?? null),
            'cierre' => $this->parseDateTime($r['Cierre'] ?? null),
            'horas_trabajadas' => $r['Horas Trabajadas'] ?? 0,
            'ventas_totales' => $r['Ventas Totales'] ?? 0,
            'cantidad_comensales' => $r['Cantidad de comensales'] ?? 0,
            'ticket_promedio' => $r['Ticket Promedio'] ?? 0,
            'cantidad_tickets' => $r['Cantidad de ticket'] ?? 0,
            'propina' => $r['Propina'] ?? 0,
            // MercadoPago
            'mp_ventas_real' => $r['Ventas MP Real'] ?? 0,
            'mp_ventas_sistema' => $r['Ventas Sistema MP Real'] ?? 0,
            'mp_conciliado' => $r['Conciliado MP REAL'] ?? 0,
            'mp_no_conciliado' => $r['No conciliado MP REAL'] ?? 0,
            'mp_diferencia' => $r['Diferencia MP sistema vs real'] ?? 0,
            'mp_porcentaje' => $r['% diferencia MP'] ?? 0,
            'mp_estado' => $r['Estado Dif MP'] ?? null,
            // Getnet
            'getnet_ventas_real' => $r['Ventas Getnet Real'] ?? 0,
            'getnet_ventas_sistema' => $r['Ventas sistema Real'] ?? 0,
            'getnet_conciliado' => $r['Conciliado Getnet REAL'] ?? 0,
            'getnet_no_conciliado' => $r['No conciliado Getnet REAL'] ?? 0,
            'getnet_diferencia' => $r['Diferencia Getnet sistema vs real'] ?? 0,
            'getnet_porcentaje' => $r['% diferencia Getnet'] ?? 0,
            'getnet_estado' => $r['Estado Dif Getnet'] ?? null,
            // Efectivo
            'efectivo_total' => $r['Efectivo Total'] ?? 0,
            'efectivo_apertura_caja' => $r['APERTURA CAJA Efectivo'] ?? 0,
            'efectivo_recuento' => $r['Recuento Efectivo'] ?? 0,
            'efectivo_diferencia' => $r['Diferencia Efectivo'] ?? 0,
            'efectivo_porcentaje' => $r['% diferencia efectivo'] ?? 0,
            'efectivo_estado' => $r['Estado Diferencia Efectivo'] ?? null,
            // Otros
            'cta_cte_total' => $r['Cta Cte Total'] ?? 0,
            'otros' => $r['Otros'] ?? 0,
            'descuentos' => $r['Descuentos'] ?? 0,
            'ventas_facturadas' => $r['Ventas facturadas'] ?? 0,
            'ideal_facturacion' => $r['Ideal facturacion'] ?? 0,
            'diferencia_facturacion' => $r['Diferencia de facturacion'] ?? 0,
            'porcentaje_facturacion' => $r['% diferencia facturacion'] ?? 0,
            'ventas_por_hora' => json_encode($this->extractHourlySales($r)),
            'created_at' => now(),
            'updated_at' => now(),
        ];
    }

    private function mapGetnetTransaction(WorkflowExecution $execution, array $r): array
    {
        return [
            'workflow_execution_id' => $execution->id,
            'nro_establecimiento' => $r['Nro de Establecimiento'] ?? null,
            'nombre_establecimiento' => $r['Nombre Establecimiento'] ?? null,
            'fecha_operacion' => $this->parseDateTime($r['Fecha de operacion'] ?? null),
            'billetera' => $r['Billetera'] ?? null,
            'marca' => $r['Marca'] ?? null,
            'tipo_tarjeta' => $r['Tipo'] ?? null,
            'tarjeta_ultimos4' => $r['Tarjeta'] ?? null,
            'tipo_transaccion' => $r['Tipo de Transaccion'] ?? null,
            'canal' => $r['Canal'] ?? null,
            'modo_canal' => $r['Modo de canal'] ?? null,
            'codigo_pos' => $r['Codigo del POS'] ?? null,
            'estado_venta' => $r['Estado venta'] ?? null,
            'cod_transaccion' => $r['Cod de Transaccion'] ?? null,
            'cod_transaccion_externo' => $r['Cod. Transaccion Externo'] ?? null,
            'nro_cupon' => $r['Nro de cupon'] ?? null,
            'cod_autorizacion' => $r['Cod. Aut.'] ?? null,
            'plan_cuotas' => $r['Plan cuotas'] ?? null,
            'moneda' => $r['Moneda'] ?? 'ARS',
            'monto_bruto' => $r['Monto Bruto Transaccion'] ?? 0,
            'arancel' => $this->parseDecimal($r['Arancel'] ?? 0),
            'iva_arancel' => $this->parseDecimal($r['IVA Arancel'] ?? 0),
            'propina' => $this->parseDecimal($r['Propina'] ?? 0),
            'monto_neto' => $r['Monto Neto Transaccion'] ?? 0,
            'fecha_liquidacion' => $this->parseDate($r['Fecha de Liquidacion'] ?? null),
            'fecha_pago' => $this->parseDate($r['Fecha estimada de Pago'] ?? null),
            'cod_liquidacion' => $r['Cod. de Liquidacion'] ?? null,
            'turno' => $r['TURNO'] ?? null,
            'estado_conciliacion' => $r['Estado'] ?? null,
            'tipo_match' => $r['Tipo Match'] ?? null,
            'id_venta_sistema' => $this->cleanId($r['ID Venta Sistema (Conc.)'] ?? null),
            'fecha_solo' => $this->parseDate($r['Fecha (Solo)'] ?? null),
            'created_at' => now(),
            'updated_at' => now(),
        ];
    }

    private function mapMpTransaction(WorkflowExecution $execution, array $r): array
    {
        return [
            'workflow_execution_id' => $execution->id,
            'fecha' => $this->parseDate($r['Fecha'] ?? null),
            'hora' => $r['Hora'] ?? null,
            'id_operacion_mp' => $r['ID DE OPERACIÓN EN MERCADO PAGO'] ?? $r['id_operacion_mp'] ?? null,
            'monto_neto' => $r['MONTO NETO DE LA OPERACIÓN QUE IMPACTÓ TU DINERO'] ?? $r['monto_neto'] ?? 0,
            'tipo_movimiento' => $r['TIPO DE MOVIMIENTO'] ?? null,
            'medio_pago' => $r['MEDIO DE PAGO DE ORIGEN'] ?? null,
            'metodo_pago' => $r['MÉTODO DE PAGO'] ?? null,
            'cuotas' => $r['CUOTAS'] ?? null,
            'estado' => $r['Estado'] ?? null,
            'turno' => $r['TURNO'] ?? null,
            'estado_conciliacion' => $r['Estado Conciliacion'] ?? $r['Estado'] ?? null,
            'tipo_match' => $r['Tipo Match'] ?? null,
            'id_venta_sistema' => $this->cleanId($r['ID Venta Sistema (Conc.)'] ?? null),
            'fecha_solo' => $this->parseDate($r['Fecha (Solo)'] ?? $r['Fecha'] ?? null),
            'created_at' => now(),
            'updated_at' => now(),
        ];
    }

    private function mapSystemSale(WorkflowExecution $execution, array $r, string $source): array
    {
        return [
            'workflow_execution_id' => $execution->id,
            'id_ticket' => $this->cleanId($r['ID Ticket'] ?? $r['id_ticket'] ?? null),
            'fecha_hora' => $this->parseDateTime($r['Fecha Hora'] ?? null),
            'hora' => $r['Hora'] ?? null,
            'items_count' => $r['Items'] ?? 0,
            'monto_total' => $r['Monto Total'] ?? $r['monto'] ?? 0,
            'tipo_venta' => $r['Tipo Venta'] ?? null,
            'estado_venta' => $r['Estado'] ?? null,
            'metodo_pago' => $r['Metodo Pago'] ?? null,
            'turno' => $r['TURNO'] ?? null,
            'estado_conciliacion' => $r['Estado Conciliacion'] ?? null,
            'tipo_match' => $r['Tipo Match'] ?? null,
            'id_operacion_conciliado' => $this->cleanId($r['ID Operacion Conciliado'] ?? null),
            'fecha_solo' => $this->parseDate($r['Fecha (Solo)'] ?? null),
            'conciliado' => ($r['Estado Conciliacion'] ?? '') === 'Conciliado',
            'source' => $source,
            'created_at' => now(),
            'updated_at' => now(),
        ];
    }

    private function mapShift(WorkflowExecution $execution, array $r): array
    {
        return [
            'workflow_execution_id' => $execution->id,
            'fecha_apertura' => $this->parseDate($r['Fecha Apertura'] ?? null),
            'hora_apertura' => $r['Hs Ap. Caja'] ?? null,
            'fecha_cierre' => $this->parseDate($r['Fecha Cierre'] ?? null),
            'hora_cierre' => $r['Hs Cierre Caja'] ?? null,
            'turno' => $r['TURNO'] ?? null,
            'encargado' => $r['Encargado'] ?? null,
            'cantidad_comensales' => $r['Cantidad de comensales'] ?? 0,
            'recuento_efectivo' => $r['Recuento Efectivo'] ?? 0,
            'apertura_caja' => $r['APERTURA CAJA Efectivo'] ?? 0,
            'created_at' => now(),
            'updated_at' => now(),
        ];
    }

    private function mapCashMovement(WorkflowExecution $execution, array $r): array
    {
        $data = [
            'workflow_execution_id' => $execution->id,
            'fecha_contable' => $this->parseDate($r['Fecha Contable'] ?? null),
            'fecha_modificacion' => $this->parseDateTime($r['Fecha Modificación'] ?? null),
            'origen' => $r['Origen'] ?? null,
            'clase' => $r['Clase'] ?? null,
            'proveedor_para' => $r['Proveedor / Para'] ?? null,
            'monto' => $r['Monto'] ?? 0,
            'comentario' => $r['Comentario'] ?? null,
            'usuario' => $r['Usuario'] ?? null,
            'tipo' => $r['Tipo'] ?? null,
            'forma_pago' => $r['Forma de Pago'] ?? null,
            'cuenta_contable' => $r['Cuenta Contable'] ?? null,
            'turno' => $r['TURNO'] ?? null,
            'created_at' => now(),
            'updated_at' => now(),
        ];

        $data['unique_hash'] = ConciliationCashMovement::generateUniqueHash($data);

        return $data;
    }

    private function mapRefund(WorkflowExecution $execution, array $r): array
    {
        $data = [
            'workflow_execution_id' => $execution->id,
            'producto' => $r['Producto'] ?? null,
            'precio' => $r['Precios'] ?? 0,
            'comentario' => $r['Comentario'] ?? null,
            'fecha_hora_pedido' => $this->parseDateTime($r['Fecha Hora pedido'] ?? null),
            'hora_pedido' => $r['Hora pedido'] ?? null,
            'hora_anulacion' => $r['Hora Anulación'] ?? null,
            'turno' => $r['TURNO'] ?? null,
            'created_at' => now(),
            'updated_at' => now(),
        ];

        $data['unique_hash'] = ConciliationRefund::generateUniqueHash($data);

        return $data;
    }

    private function mapMpNegative(WorkflowExecution $execution, array $r): array
    {
        return [
            'workflow_execution_id' => $execution->id,
            'fecha' => $this->parseDate($r['Fecha'] ?? null),
            'hora' => $r['Hora'] ?? null,
            'id_operacion_mp' => $r['ID DE OPERACIÓN EN MERCADO PAGO'] ?? null,
            'monto_neto' => $r['MONTO NETO DE LA OPERACIÓN QUE IMPACTÓ TU DINERO'] ?? 0,
            'turno' => $r['TURNO'] ?? null,
            'created_at' => now(),
            'updated_at' => now(),
        ];
    }

    // ==================== HELPERS ====================

    private function parseDate($value): ?string
    {
        if (empty($value)) return null;

        try {
            // Formato "DD/MM/YYYY" o "YYYY-MM-DD"
            if (preg_match('/^\d{2}\/\d{2}\/\d{4}$/', $value)) {
                return \Carbon\Carbon::createFromFormat('d/m/Y', $value)->format('Y-m-d');
            }
            return \Carbon\Carbon::parse($value)->format('Y-m-d');
        } catch (\Exception $e) {
            return null;
        }
    }

    private function parseDateTime($value): ?string
    {
        if (empty($value)) return null;

        try {
            // Formato "DD/MM/YYYY HH:MM:SS"
            if (preg_match('/^\d{2}\/\d{2}\/\d{4}\s+\d{2}:\d{2}:\d{2}$/', $value)) {
                return \Carbon\Carbon::createFromFormat('d/m/Y H:i:s', $value)->format('Y-m-d H:i:s');
            }
            return \Carbon\Carbon::parse($value)->format('Y-m-d H:i:s');
        } catch (\Exception $e) {
            return null;
        }
    }

    private function parseDecimal($value): float
    {
        if (is_numeric($value)) return (float) $value;
        if (is_string($value)) {
            return (float) str_replace([',', '"'], ['', ''], $value);
        }
        return 0.0;
    }

    private function cleanId($value): ?string
    {
        if (empty($value)) return null;
        return trim(str_replace('"', '', $value));
    }

    private function extractHourlySales(array $r): array
    {
        $sales = [];
        for ($h = 0; $h < 24; $h++) {
            $key = sprintf('%02d:00 - %02d:00', $h, $h + 1);
            $sales[$key] = $r[$key] ?? 0;
        }
        return $sales;
    }

    private function getSummaryUpdateColumns(): array
    {
        return [
            'workflow_execution_id',
            'ventas_totales',
            'cantidad_comensales',
            'ticket_promedio',
            'cantidad_tickets',
            'propina',
            'mp_ventas_real',
            'mp_conciliado',
            'mp_diferencia',
            'mp_porcentaje',
            'getnet_ventas_real',
            'getnet_conciliado',
            'getnet_diferencia',
            'getnet_porcentaje',
            'efectivo_total',
            'efectivo_diferencia',
            'efectivo_porcentaje',
            'ventas_facturadas',
            'ventas_por_hora',
            'updated_at',
        ];
    }
}
```

---

### Fase 4: Modificar WorkflowExecutionService

En [WorkflowExecutionService.php](app/Services/WorkflowExecutionService.php):

```php
<?php

namespace App\Services;

use App\Models\WorkflowFileBatch;
use App\Models\WorkflowExecution;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class WorkflowExecutionService
{
    protected WorkflowJsonGeneratorService $jsonGenerator;
    protected ConciliationDataService $conciliationService;

    public function __construct(
        WorkflowJsonGeneratorService $jsonGenerator,
        ConciliationDataService $conciliationService
    ) {
        $this->jsonGenerator = $jsonGenerator;
        $this->conciliationService = $conciliationService;
    }

    public function execute(WorkflowFileBatch $batch): WorkflowExecution
    {
        $this->updateBatchStatus($batch, 'executing');

        try {
            $jsonData = $this->jsonGenerator->generateFromBatch($batch);
            $startTime = microtime(true);

            $useMock = config('workflows.python_api.use_mock', true);

            if ($useMock) {
                $response = $this->mockApiResponse($jsonData);
            } else {
                $response = $this->callExternalApi($jsonData);
            }

            $executionTime = (int) ((microtime(true) - $startTime) * 1000);

            $execution = $this->saveExecution($batch, $jsonData, $response, $executionTime);

            // ========== NUEVO: Verificar y guardar datos de conciliación ==========
            if ($this->isConciliationWorkflow($batch)) {
                $this->processConciliationResponse($execution, $response);
            }
            // ======================================================================

            $status = ($response['status'] ?? 'failed') === 'success' ? 'completed' : 'failed';
            $this->updateBatchStatus($batch, $status);

            return $execution;

        } catch (\Exception $e) {
            $this->updateBatchStatus($batch, 'failed');

            Log::error('Workflow execution failed', [
                'batch_id' => $batch->id,
                'error' => $e->getMessage(),
            ]);

            return $this->saveExecution(
                $batch,
                $jsonData ?? [],
                [
                    'status' => 'error',
                    'message' => $e->getMessage(),
                ],
                0
            );
        }
    }

    /**
     * Verificar si el workflow es de tipo Conciliador
     */
    protected function isConciliationWorkflow(WorkflowFileBatch $batch): bool
    {
        // Opción 1: Por nombre del tipo de workflow
        $workflowType = $batch->workflowType;
        if ($workflowType && str_contains(strtolower($workflowType->name), 'concilia')) {
            return true;
        }

        // Opción 2: Por ID de tipo configurado
        $conciliationTypeId = config('workflows.conciliation_type_id');
        if ($conciliationTypeId && $batch->workflow_type_id === $conciliationTypeId) {
            return true;
        }

        return false;
    }

    /**
     * Procesar y guardar respuesta de conciliación
     */
    protected function processConciliationResponse(WorkflowExecution $execution, array $response): void
    {
        // Verificar status de la respuesta
        if (!$this->conciliationService->validateResponse($response)) {
            Log::warning('Conciliation response invalid or failed', [
                'execution_id' => $execution->id,
                'status' => $response['status'] ?? 'unknown',
                'message' => $response['message'] ?? 'No message',
            ]);
            return;
        }

        // Guardar datos (con filtrado de duplicados)
        try {
            $stats = $this->conciliationService->processAndSave($execution, $response);

            Log::info('Conciliation data saved successfully', [
                'execution_id' => $execution->id,
                'stats' => $stats,
            ]);
        } catch (\Exception $e) {
            Log::error('Failed to save conciliation data', [
                'execution_id' => $execution->id,
                'error' => $e->getMessage(),
            ]);
            // No lanzar excepción - los datos JSON ya están guardados como backup
        }
    }

    // ... resto de métodos existentes sin cambios ...
}
```

---

### Fase 5: Vistas Livewire

#### 5.1 Controladores

**`app/Livewire/Conciliation/ConciliationDashboard.php`**
- Filtros: fecha desde/hasta, turno, encargado, cliente/sucursal
- KPIs: total ventas, % conciliación MP/Getnet/Efectivo
- Gráficos: tendencia conciliación, ventas por hora
- Lista de ejecuciones con alertas

**`app/Livewire/Conciliation/ConciliationDetail.php`**
- Tabs: Resumen, Getnet, MercadoPago, Sistema, Caja, Devoluciones
- Tablas paginadas con búsqueda
- Exportación Excel/PDF por tab

**`app/Livewire/Conciliation/Components/`**
- `SummaryCard.php` - Tarjeta de resumen por turno
- `TransactionsTable.php` - Tabla reutilizable con paginación
- `ExportButton.php` - Botón de exportación

#### 5.2 Vistas Blade

```
resources/views/livewire/conciliation/
├── dashboard.blade.php          # Vista principal
├── detail.blade.php             # Detalle de ejecución
└── components/
    ├── summary-card.blade.php
    ├── transactions-table.blade.php
    ├── kpi-cards.blade.php
    └── filters.blade.php
```

---

### Fase 6: Exportación

Crear `app/Exports/ConciliationExport.php` usando Laravel Excel:

- `ConciliationSummaryExport` - Resúmenes por turno
- `ConciliationTransactionsExport` - Transacciones con filtros
- `ConciliationFullReportExport` - Reporte completo multi-hoja

---

### Fase 7: Rutas

Agregar en `routes/web.php`:

```php
// Dentro del grupo de rutas Programador
Route::prefix('conciliacion')->name('conciliacion.')->group(function () {
    Route::get('/', ConciliationDashboard::class)->name('index');
    Route::get('/{execution}', ConciliationDetail::class)->name('show');
    Route::get('/{execution}/export/{type}', [ConciliationExportController::class, 'export'])
        ->name('export');
});
```

---

## Resumen de Archivos

### Crear

| # | Archivo | Descripción |
|---|---------|-------------|
| 1 | `database/migrations/2026_01_15_create_conciliation_tables.php` | Migración con 8 tablas + índices únicos |
| 2-9 | `app/Models/Conciliation/*.php` | 8 modelos Eloquent con métodos de hash |
| 10 | `app/Services/ConciliationDataService.php` | Servicio con upsert y filtrado |
| 11 | `app/Livewire/Conciliation/ConciliationDashboard.php` | Dashboard principal |
| 12 | `app/Livewire/Conciliation/ConciliationDetail.php` | Vista detalle |
| 13-16 | `resources/views/livewire/conciliation/*.blade.php` | Vistas Blade |
| 17 | `app/Exports/ConciliationExport.php` | Exportación Excel |
| 18 | `app/Http/Controllers/ConciliationExportController.php` | Controlador exportación |

### Modificar

| Archivo | Cambio |
|---------|--------|
| `app/Services/WorkflowExecutionService.php` | Inyectar servicio, agregar verificación y procesamiento |
| `app/Models/WorkflowExecution.php` | Agregar 8 relaciones HasMany |
| `routes/web.php` | Agregar rutas de conciliación |
| `config/workflows.php` | Agregar `conciliation_type_id` |

---

## Verificación

1. Ejecutar `php artisan migrate`
2. Ejecutar workflow Conciliador primera vez → verificar que se inserten todos los registros
3. Ejecutar workflow Conciliador segunda vez con datos superpuestos → verificar que:
   - Registros nuevos se inserten
   - Registros existentes se actualicen (si cambiaron)
   - No se creen duplicados
4. Verificar en BD: `SELECT COUNT(*) FROM conciliation_getnet_transactions`
5. Navegar a `/conciliacion` y verificar dashboard
6. Probar exportación Excel
7. Ejecutar tests: `php artisan test --filter=Conciliation`

---

## Notas Técnicas

### Performance con Grandes Volúmenes

- **Chunking**: Procesar en lotes de 500 registros para evitar memory overflow
- **Upsert**: Usar `upsert()` nativo de Laravel (MySQL 8.0+ / PostgreSQL)
- **Índices**: Crear índices en columnas de búsqueda frecuente
- **Transacciones**: Envolver todo en `DB::transaction()` para atomicidad

### Manejo de Errores

- Si falla el guardado normalizado, los datos JSON quedan en `workflow_executions.json_response` como backup
- Logs detallados para debugging
- No se interrumpe el flujo principal por errores en persistencia de conciliación
