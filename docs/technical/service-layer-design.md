# Service Layer Design — POS

> Dokumen ini menjelaskan **desain lapisan Service** — tempat seluruh business logic sistem berada. Service Layer adalah implementasi konkret dari aturan bisnis pada [`business-rules.md`](../business/business-rules.md) dan validasi pada [`validation.md`](../business/validation.md), serta menjadi penghubung antara Controller (HTTP layer) dan Repository (data access layer) sesuai [`system-architecture.md`](system-architecture.md#13-keputusan-layered-architecture-pada-backend).

---

## 1. Prinsip Desain Service Layer

1. **Service adalah satu-satunya tempat business logic berada** — Controller hanya menangani HTTP concern (parsing request, memanggil Service, membungkus response), tidak pernah mengandung logika bisnis.
2. **Service tidak bergantung pada HTTP context** — Service dapat dipanggil dari Controller, Command (Artisan), atau Job (Queue) tanpa modifikasi, karena tidak menerima objek `Request` secara langsung, melainkan DTO/array data yang sudah tervalidasi.
3. **Satu Service, satu tanggung jawab utama** — mengikuti pemetaan pada [`module-design.md`](module-design.md), setiap Service fokus pada satu domain proses bisnis.
4. **Service memanggil Repository untuk akses data, tidak pernah Eloquent Model langsung untuk query kompleks** — menjaga pemisahan tegas dengan Repository Layer ([`repository-layer-design.md`](repository-layer-design.md)).
5. **Transaksi database (DB Transaction) dikelola di level Service**, bukan Repository, karena Service yang memahami batas atomicity dari suatu proses bisnis (contoh: penjualan + pengurangan stok harus atomic).

---

## 2. Struktur Umum Sebuah Service

```php
namespace App\Services\Sales;

class SalesService
{
    public function __construct(
        private SaleRepositoryInterface $saleRepository,
        private StockService $stockService,
        private PromotionService $promotionService,
        private ApprovalService $approvalService,
    ) {}

    public function createSale(array $data, Employee $cashier): Sale
    {
        return DB::transaction(function () use ($data, $cashier) {
            // 1. Validasi business rule (di luar validasi format request)
            $this->assertShiftIsActive($data['shift_id']);

            // 2. Hitung total, terapkan promo
            $calculatedItems = $this->promotionService->applyPromotions($data['items']);

            // 3. Cek & kurangi stok (dalam transaction yang sama — BRULE-ST-01)
            $this->stockService->deductStockForSale($calculatedItems, $cashier->branch_id);

            // 4. Simpan transaksi (snapshot data — EC-SL-06)
            $sale = $this->saleRepository->create([...]);

            // 5. Dispatch domain event (efek samping: notifikasi, audit — asynchronous)
            event(new SaleCompleted($sale));

            return $sale;
        });
    }
}
```

**Alasan pola ini:** Dependency Injection terhadap **interface** (`SaleRepositoryInterface`, bukan implementasi konkret) memungkinkan Service diuji dengan mock tanpa menyentuh database sungguhan, sejalan dengan kebutuhan [`testing-strategy.md`](testing-strategy.md).

---

## 3. Contoh Detail: SalesService

### 3.1 Tanggung Jawab

Mengorkestrasi pembuatan transaksi penjualan sesuai alur pada [`business-flow.md`](../business/business-flow.md#12-alur-umum-transaksi-penjualan) dan menegakkan aturan pada [`business-rules.md`](../business/business-rules.md#4-aturan-terkait-transaksi-penjualan).

### 3.2 Method Utama

| Method | Business Rule yang Ditegakkan |
|---|---|
| `createSale()` | BRULE-TX-01 (shift aktif), BRULE-TX-02 (total pembayaran sesuai) |
| `requestVoid()` | BRULE-TX-03 (hanya shift aktif), meneruskan ke `ApprovalService` |
| `processReturn()` | BRULE-TX-05 (retur tidak melebihi jumlah asal) |

### 3.3 Contoh Penegakan BRULE-TX-02

```php
private function assertPaymentMatchesTotal(array $payments, float $grandTotal): void
{
    $totalPaid = array_sum(array_column($payments, 'amount'));
    if (round($totalPaid, 2) !== round($grandTotal, 2)) {
        throw new PaymentMismatchException(
            "Total pembayaran ({$totalPaid}) tidak sesuai dengan total tagihan ({$grandTotal})"
        );
    }
}
```

`PaymentMismatchException` di-*mapping* ke response HTTP 422 dengan `error_code: PAYMENT_MISMATCH` sesuai konvensi pada [`api-contract.md`](api-contract.md#13-kode-status-http-yang-digunakan).

---

## 4. Contoh Detail: StockService

### 4.1 Tanggung Jawab

Satu-satunya titik masuk untuk mengubah nilai stok, sesuai desain pada [`module-design.md`](module-design.md#6-modul-inventory-stok).

### 4.2 Method Kunci: `deductStockForSale()`

```php
public function deductStockForSale(array $items, string $branchId): void
{
    foreach ($items as $item) {
        $availableBatches = $this->batchRepository->getAvailableBatchesFEFO(
            $item['product_id'], $branchId
        );

        $remainingQty = $item['quantity'];
        foreach ($availableBatches as $batch) {
            if ($remainingQty <= 0) break;

            $deductQty = min($remainingQty, $batch->quantity_available);

            // Locking pesimis untuk mencegah race condition (EC-SL-02)
            $this->stockRepository->lockAndDeduct($batch->id, $deductQty);

            $this->ledgerRepository->recordMovement([
                'movement_type' => 'sale_out',
                'batch_id' => $batch->id,
                'quantity' => -$deductQty,
                'reference_type' => 'Sale',
            ]);

            $remainingQty -= $deductQty;
        }

        if ($remainingQty > 0) {
            throw new InsufficientStockException($item['product_id'], $item['quantity']);
        }
    }
}
```

**Alasan desain locking pesimis:** Mengacu pada EC-SL-02 pada [`edge-cases.md`](../business/edge-cases.md) — dua kasir menjual produk yang sama dengan stok tersisa 1 secara bersamaan. Penggunaan `SELECT ... FOR UPDATE` (locking pesimis) di level Repository memastikan hanya satu transaksi yang berhasil mengurangi stok terakhir; transaksi lain akan menunggu lock dilepas, lalu menemukan stok sudah 0 dan menerima exception `InsufficientStockException`.

**Alasan alokasi FEFO otomatis:** Menegakkan BRULE-ST-03 secara struktural — kasir tidak perlu (dan tidak bisa) memilih batch secara manual, sistem selalu mengalokasikan batch dengan expiry terdekat terlebih dahulu.

---

## 5. Contoh Detail: ApprovalService

### 5.1 Tanggung Jawab

Mengelola siklus hidup permintaan approval secara generik lintas modul, sesuai [`approval-flow.md`](../business/approval-flow.md).

### 5.2 Method Kunci: `escalateIfNeeded()`

```php
public function escalateIfNeeded(ApprovalRequest $request): void
{
    $currentLevel = $request->current_approver_role;
    $nextLevel = $this->getNextApprovalLevel($currentLevel, $request->approvable_type);

    if ($this->exceedsAuthority($request, $currentLevel)) {
        $request->update([
            'status' => 'escalated',
            'current_approver_role' => $nextLevel,
        ]);
        event(new ApprovalEscalated($request));
    }
}
```

Method ini menegakkan BRULE-AP-02 — eskalasi otomatis tanpa memerlukan pengajuan ulang oleh pemohon.

---

## 6. Exception Handling Sebagai Bagian dari Kontrak Service

Setiap Service melempar **custom exception** yang bermakna bisnis (bukan exception generik), memungkinkan Controller/Exception Handler memetakannya ke response HTTP yang sesuai (lihat [`api-contract.md`](api-contract.md#13-kode-status-http-yang-digunakan)):

| Exception | Kapan Dilempar | HTTP Status | error_code |
|---|---|---|---|
| `InsufficientStockException` | Stok tidak cukup | 422 | `INSUFFICIENT_STOCK` |
| `PaymentMismatchException` | Total pembayaran tidak sesuai | 422 | `PAYMENT_MISMATCH` |
| `CreditLimitExceededException` | Piutang melebihi limit tanpa approval | 422 | `CREDIT_LIMIT_EXCEEDED` |
| `SelfApprovalNotAllowedException` | Approver = pengaju | 403 | `SELF_APPROVAL_FORBIDDEN` |
| `ShiftNotActiveException` | Transaksi dibuat tanpa shift aktif | 409 | `SHIFT_NOT_ACTIVE` |

**Alasan desain:** Exception spesifik memungkinkan **satu titik penanganan terpusat** (Laravel Exception Handler) untuk memetakan seluruh error bisnis ke format response konsisten, tanpa setiap Controller perlu menulis blok try-catch berulang.

---

## 7. Domain Event yang Di-dispatch Service

| Event | Dipicu oleh Service | Konsumen (Listener) |
|---|---|---|
| `SaleCompleted` | `SalesService::createSale()` | Notification (jika relevan), Audit Log |
| `SaleVoided` | `SalesService::executeVoid()` | Notification, Audit Log |
| `StockLow` | `StockService` (setelah pengurangan, cek reorder point) | Notification (NOTIF-ST-01) |
| `ApprovalRequested` | `ApprovalService::create()` | Notification (NOTIF-AP-01) |
| `ApprovalEscalated` | `ApprovalService::escalateIfNeeded()` | Notification (NOTIF-AP-02) |

Detail pola event-driven dijelaskan pada [`module-design.md`](module-design.md#102-desain-event-driven).

---

## 8. Traceability

| Aspek Service Layer | Terkait Dokumen |
|---|---|
| Aturan bisnis yang ditegakkan | [`business-rules.md`](../business/business-rules.md) |
| Skenario edge case yang ditangani | [`edge-cases.md`](../business/edge-cases.md) |
| Data access yang dipanggil | [`repository-layer-design.md`](repository-layer-design.md) |
| Pemetaan ke modul | [`module-design.md`](module-design.md) |
| Test case turunan | [`testing-strategy.md`](testing-strategy.md) |

---

**Sebelumnya:** [`authorization-design.md`](authorization-design.md) — Authorization Design
**Selanjutnya:** [`repository-layer-design.md`](repository-layer-design.md) — Repository Layer Design
