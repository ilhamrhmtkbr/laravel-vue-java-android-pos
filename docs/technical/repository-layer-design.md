# Repository Layer Design — POS

> Dokumen ini menjelaskan **desain lapisan Repository** — lapisan yang bertanggung jawab atas seluruh akses data ke database. Repository Layer memisahkan Service dari detail implementasi query (Eloquent, raw SQL), sesuai prinsip *separation of concerns* pada [`system-architecture.md`](system-architecture.md#5-prinsip-desain-sistem-yang-konsisten).

---

## 1. Prinsip Desain Repository Layer

1. **Repository hanya berisi logic akses data, tidak ada business logic.** Perhitungan, validasi bisnis, dan orkestrasi antar modul tetap berada di Service Layer.
2. **Setiap Repository mengimplementasikan Interface (Contract).** Service bergantung pada interface, bukan implementasi konkret — memungkinkan penggantian implementasi (misal untuk testing) tanpa mengubah Service.
3. **Query kompleks/reusable dienkapsulasi sebagai method bernama jelas**, bukan tersebar sebagai query builder mentah di banyak tempat.
4. **Repository tidak mengetahui konteks HTTP** — tidak menerima `Request`, hanya menerima parameter primitif/DTO.

---

## 2. Struktur Interface & Implementasi

```php
// app/Repositories/Contracts/SaleRepositoryInterface.php
namespace App\Repositories\Contracts;

interface SaleRepositoryInterface
{
    public function create(array $data): Sale;
    public function findById(string $id): ?Sale;
    public function getByShift(string $shiftId): Collection;
    public function getDailySummaryByBranch(string $branchId, Carbon $date): array;
}

// app/Repositories/Eloquent/EloquentSaleRepository.php
namespace App\Repositories\Eloquent;

class EloquentSaleRepository implements SaleRepositoryInterface
{
    public function create(array $data): Sale
    {
        return Sale::create($data);  // BranchScope otomatis berlaku (lihat authorization-design.md)
    }

    public function getDailySummaryByBranch(string $branchId, Carbon $date): array
    {
        return Sale::where('branch_id', $branchId)
            ->whereDate('created_at', $date)
            ->where('status', 'completed')
            ->selectRaw('COUNT(*) as total_transactions, SUM(grand_total) as total_amount')
            ->first()
            ->toArray();
    }
}
```

### 2.1 Binding Interface ke Implementasi

```php
// app/Providers/RepositoryServiceProvider.php
public function register()
{
    $this->app->bind(SaleRepositoryInterface::class, EloquentSaleRepository::class);
    $this->app->bind(StockRepositoryInterface::class, EloquentStockRepository::class);
    // ...
}
```

**Alasan desain:** Binding terpusat di Service Provider memungkinkan penggantian implementasi secara global (misal jika di masa depan sebagian data dipindah ke database analytics terpisah) tanpa mengubah kode Service yang memanggilnya.

---

## 3. Repository untuk Query Polymorphic (Modul Approval)

Mengacu pada desain polymorphic pada [`module-design.md`](module-design.md#72-desain-generik-polymorphic-approval) dan [`database-design.md`](database-design.md#45-modul-approval-polymorphic):

```php
class EloquentApprovalRepository implements ApprovalRepositoryInterface
{
    public function getPendingByApproverRole(string $role, string $branchId): Collection
    {
        return ApprovalRequest::where('current_approver_role', $role)
            ->where('branch_id', $branchId)
            ->where('status', 'pending')
            ->with('approvable')  // Eloquent morphTo relation
            ->get();
    }

    public function findApprovableEntity(ApprovalRequest $request): Model
    {
        // Resolusi dinamis berdasarkan approvable_type
        return $request->approvable_type::findOrFail($request->approvable_id);
    }
}
```

**Alasan desain:** Enkapsulasi logic polymorphic di dalam Repository mencegah Service Layer perlu memahami detail implementasi `morphTo`/resolusi tipe dinamis — Service hanya memanggil `getPendingByApproverRole()` dan menerima data yang sudah lengkap.

---

## 4. Repository dengan Locking untuk Konsistensi Stok

Mengacu pada EC-SL-02 pada [`edge-cases.md`](../business/edge-cases.md) dan implementasi pada [`service-layer-design.md`](service-layer-design.md#42-method-kunci-deductstockforsale):

```php
class EloquentStockRepository implements StockRepositoryInterface
{
    public function lockAndDeduct(string $batchId, float $quantity): void
    {
        DB::transaction(function () use ($batchId, $quantity) {
            $batch = ProductBatch::where('id', $batchId)
                ->lockForUpdate()   // SELECT ... FOR UPDATE — locking pesimis
                ->first();

            if ($batch->quantity_available < $quantity) {
                throw new InsufficientStockException();
            }

            $batch->decrement('quantity_available', $quantity);
        });
    }
}
```

**Alasan desain:** `lockForUpdate()` memastikan baris data batch **terkunci** selama transaksi berlangsung — transaksi konkuren lain yang mencoba mengurangi stok batch yang sama harus menunggu hingga lock dilepas, mencegah kondisi *race condition* yang dapat menyebabkan stok menjadi negatif secara tidak sengaja.

---

## 5. Repository untuk Kebutuhan Laporan (Read-Optimized Queries)

Mengacu pada desain modul Reporting yang bersifat read-only pada [`module-design.md`](module-design.md#9-modul-reporting--dashboard):

```php
class EloquentReportRepository implements ReportRepositoryInterface
{
    public function getSalesByProductCategory(string $branchId, Carbon $from, Carbon $to): Collection
    {
        return DB::table('sale_items')
            ->join('sales_transactions', 'sale_items.sale_id', '=', 'sales_transactions.id')
            ->join('products', 'sale_items.product_id', '=', 'products.id')
            ->join('categories', 'products.category_id', '=', 'categories.id')
            ->where('sales_transactions.branch_id', $branchId)
            ->whereBetween('sales_transactions.created_at', [$from, $to])
            ->groupBy('categories.name')
            ->selectRaw('categories.name, SUM(sale_items.subtotal) as total_revenue')
            ->get();
    }
}
```

**Alasan menggunakan Query Builder (`DB::table`) alih-alih Eloquent murni:** Untuk laporan dengan agregasi kompleks lintas beberapa tabel, Query Builder memberikan kontrol lebih presisi terhadap SQL yang dihasilkan dan performa lebih baik dibanding memuat banyak Eloquent Model beserta relasinya hanya untuk keperluan agregasi angka.

---

## 6. Strategi Testing Repository

Karena Repository bergantung pada database, pengujian dilakukan dengan dua pendekatan (detail lengkap pada [`testing-strategy.md`](testing-strategy.md)):

| Jenis Test | Target | Pendekatan |
|---|---|---|
| Unit Test Service | Menguji business logic tanpa database sungguhan | Mock `RepositoryInterface` |
| Feature/Integration Test Repository | Menguji query benar-benar menghasilkan data yang tepat | Database testing (SQLite in-memory atau PostgreSQL test database) dengan data seed |

---

## 7. Ringkasan Repository per Modul

| Repository | Tanggung Jawab Utama |
|---|---|
| `SaleRepository` | CRUD transaksi penjualan & query terkait |
| `SaleItemRepository` | Query detail item transaksi |
| `StockRepository` | Operasi baca/tulis `current_stock` dengan locking |
| `StockLedgerRepository` | Pencatatan & query kartu stok |
| `ProductBatchRepository` | Query alokasi FEFO |
| `ApprovalRepository` | Query polymorphic approval requests |
| `PurchaseOrderRepository` | CRUD PO & goods receipt |
| `ReportRepository` | Query agregasi read-only untuk laporan |
| `CustomerRepository` | CRUD pelanggan, query limit piutang |

---

## 8. Traceability

| Aspek Repository | Terkait Dokumen |
|---|---|
| Struktur tabel yang diakses | [`database-design.md`](database-design.md), [`erd.md`](erd.md) |
| Business logic pemanggil | [`service-layer-design.md`](service-layer-design.md) |
| Pemetaan modul | [`module-design.md`](module-design.md) |
| Strategi pengujian | [`testing-strategy.md`](testing-strategy.md) |

---

**Sebelumnya:** [`service-layer-design.md`](service-layer-design.md) — Service Layer Design
**Selanjutnya:** [`security-design.md`](security-design.md) — Security Design
