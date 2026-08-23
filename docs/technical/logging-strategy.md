# Logging Strategy — POS

> Dokumen ini menjelaskan **strategi logging dan observability** — bagaimana sistem mencatat aktivitas, error, dan audit trail secara terstruktur. Dokumen ini menjawab NFR-OB-01, NFR-OB-02, dan NFR-AU-01/NFR-AU-02 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md).

---

## 1. Prinsip Desain Logging

1. **Log terstruktur (structured logging), bukan free-text** — setiap entri log dalam format JSON agar dapat di-query dan dianalisis secara otomatis, bukan hanya dibaca manusia.
2. **Membedakan tiga jenis log secara eksplisit:** Application Log (debugging teknis), Audit Log (jejak perubahan data bisnis), dan Access Log (siapa mengakses apa).
3. **Data sensitif tidak pernah masuk log** — password, token, dan data kartu pembayaran (jika ada) di-redact otomatis.
4. **Log dapat ditelusuri lintas request** — setiap request memiliki `request_id` unik yang konsisten di seluruh log terkait, memudahkan investigasi.

---

## 2. Tiga Kategori Log

### 2.1 Application Log

**Tujuan:** Debugging teknis — error, warning, informasi proses sistem.

```json
{
  "timestamp": "2026-08-09T10:15:23Z",
  "level": "error",
  "request_id": "req-abc123",
  "message": "Failed to process queue job",
  "context": {
    "job": "GenerateSalesReport",
    "exception": "Illuminate\\Database\\QueryException",
    "branch_id": "uuid"
  }
}
```

**Level log yang digunakan:** `debug`, `info`, `warning`, `error`, `critical` — mengikuti standar PSR-3 yang didukung native oleh Laravel logging (Monolog).

### 2.2 Audit Log

**Tujuan:** Jejak perubahan data bisnis sensitif, menegakkan NFR-AU-01 dan NFR-AU-02.

```json
{
  "timestamp": "2026-08-09T10:20:00Z",
  "event": "price_updated",
  "entity_type": "Product",
  "entity_id": "uuid",
  "actor": { "employee_id": "uuid", "name": "Admin HO", "role": "ho_admin" },
  "changes": { "selling_price": { "old": 15000, "new": 17000 } },
  "branch_context": null
}
```

**Aksi yang wajib tercatat sebagai Audit Log** (mengacu pada [`business-rules.md`](../business/business-rules.md) dan [`approval-flow.md`](../business/approval-flow.md)):

| Aksi | Alasan |
|---|---|
| Perubahan harga produk | BRULE-PR-01 |
| Void transaksi | BRULE-TX-03, mitigasi fraud |
| Penyesuaian stok manual | BRULE-ST-05 |
| Keputusan approval (approve/reject) | BRULE-AP-03, keputusan final harus tercatat |
| Perubahan role/permission pegawai | NFR-SE-03 |
| Login & logout | FR-AU-04 |

**Prinsip immutability:** Audit Log **tidak dapat diubah atau dihapus** melalui aplikasi manapun — hanya dapat ditambah (append-only), sesuai NFR-AU-02. Secara teknis, tabel audit log tidak diekspos melalui endpoint UPDATE/DELETE apa pun.

### 2.3 Access Log

**Tujuan:** Mencatat siapa mengakses endpoint apa, kapan, dan hasilnya.

```json
{
  "timestamp": "2026-08-09T10:25:00Z",
  "request_id": "req-xyz789",
  "method": "POST",
  "endpoint": "/api/v1/sales/{id}/void",
  "employee_id": "uuid",
  "branch_id": "uuid",
  "status_code": 200,
  "duration_ms": 145,
  "ip_address": "10.0.1.5"
}
```

Access Log dihasilkan otomatis oleh middleware terpusat, tidak memerlukan penambahan kode di setiap Controller.

---

## 3. Implementasi Teknis

### 3.1 Konfigurasi Laravel Logging (Monolog)

```php
// config/logging.php
'channels' => [
    'application' => [
        'driver' => 'daily',
        'path' => storage_path('logs/application.log'),
        'level' => env('LOG_LEVEL', 'info'),
        'days' => 30,
    ],
    'audit' => [
        'driver' => 'daily',
        'path' => storage_path('logs/audit.log'),
        'days' => 1825,  // retensi 5 tahun sesuai NFR-AU-03
    ],
],
```

### 3.2 Middleware Request ID & Access Log

```php
// app/Http/Middleware/RequestLoggingMiddleware.php
public function handle($request, Closure $next)
{
    $requestId = (string) Str::uuid();
    $request->attributes->set('request_id', $requestId);

    $start = microtime(true);
    $response = $next($request);
    $duration = (microtime(true) - $start) * 1000;

    Log::channel('access')->info('request_completed', [
        'request_id' => $requestId,
        'method' => $request->method(),
        'endpoint' => $request->path(),
        'employee_id' => optional($request->user())->id,
        'status_code' => $response->status(),
        'duration_ms' => round($duration, 2),
    ]);

    return $response;
}
```

### 3.3 Redaction Data Sensitif

```php
// app/Logging/SensitiveDataProcessor.php
class SensitiveDataProcessor
{
    private array $sensitiveFields = ['password', 'token', 'password_confirmation'];

    public function __invoke(array $record): array
    {
        foreach ($this->sensitiveFields as $field) {
            if (isset($record['context'][$field])) {
                $record['context'][$field] = '[REDACTED]';
            }
        }
        return $record;
    }
}
```

**Alasan desain:** Redaction diterapkan sebagai **Monolog Processor global**, bukan mengandalkan setiap developer mengingat untuk tidak me-log field sensitif secara manual — pendekatan struktural ini mencegah kebocoran tidak sengaja (human error).

---

## 4. Audit Log via Event Listener (Bukan Manual di Setiap Service)

Mengacu pada desain event-driven pada [`module-design.md`](module-design.md#102-desain-event-driven):

```php
// app/Listeners/RecordAuditLog.php
class RecordAuditLog
{
    public function handle(ProductPriceUpdated $event): void
    {
        AuditLog::create([
            'event' => 'price_updated',
            'entity_type' => 'Product',
            'entity_id' => $event->product->id,
            'actor_id' => $event->actor->id,
            'changes' => $event->changes,
        ]);
    }
}
```

**Alasan desain:** Dengan mencatat audit log via Listener yang bereaksi terhadap domain event (bukan dipanggil manual di setiap Service), risiko developer **lupa** mencatat audit untuk aksi sensitif tertentu jauh berkurang — cukup pastikan event yang relevan selalu di-dispatch dari Service (lihat [`service-layer-design.md`](service-layer-design.md#7-domain-event-yang-di-dispatch-service)).

---

## 5. Health Check Endpoint

```php
// routes/api.php
Route::get('/health', function () {
    return response()->json([
        'status' => 'ok',
        'database' => DB::connection()->getPdo() ? 'connected' : 'disconnected',
        'redis' => Redis::ping() ? 'connected' : 'disconnected',
        'timestamp' => now()->toIso8601String(),
    ]);
});
```

Digunakan oleh Load Balancer dan Docker health check (lihat [`deployment-architecture.md`](deployment-architecture.md#6-ketersediaan--redundansi) dan [`docker-architecture.md`](docker-architecture.md#7-health-check-container)) untuk mendeteksi instance yang bermasalah.

---

## 6. Strategi Retensi Log

| Jenis Log | Retensi | Alasan |
|---|---|---|
| Application Log | 30 hari | Kebutuhan debugging jangka pendek, volume tinggi |
| Access Log | 90 hari | Kebutuhan investigasi keamanan jangka menengah |
| Audit Log | Minimal 5 tahun (dikonfigurasi) | Sejalan dengan NFR-AU-03 — kebutuhan retensi data finansial |

Log yang melewati masa retensi dipindahkan ke **cold storage** (S3 Glacier atau setara) sebelum dihapus dari storage utama, bukan langsung dihapus — mendukung kebutuhan audit historis jangka panjang tanpa membebani storage aktif.

---

## 7. Traceability

| Aspek Logging | Terkait Dokumen |
|---|---|
| Kebutuhan audit dari sisi bisnis | [`business-rules.md`](../business/business-rules.md), [`role-permission.md`](../business/role-permission.md) |
| Event yang memicu audit log | [`service-layer-design.md`](service-layer-design.md) |
| Infrastruktur monitoring | [`deployment-architecture.md`](deployment-architecture.md) |
| Strategi backup log jangka panjang | [`backup-strategy.md`](backup-strategy.md) |

---

**Sebelumnya:** [`cicd-flow.md`](cicd-flow.md) — CI/CD Flow
**Selanjutnya:** [`backup-strategy.md`](backup-strategy.md) — Backup Strategy
