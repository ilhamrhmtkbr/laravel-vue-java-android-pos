# Module Design — POS

> Dokumen ini menjelaskan **desain tiap modul fungsional** — tanggung jawab, batasan (boundary), dan interaksi antar modul. Modul di sini adalah unit logis di dalam modular monolith (lihat [`system-architecture.md`](system-architecture.md#11-keputusan-modular-monolith-bukan-microservices)), setiap modul memetakan langsung ke domain bisnis pada [`business-flow.md`](../business/business-flow.md) dan [`functional-requirements.md`](../business/functional-requirements.md).

---

## 1. Prinsip Desain Modul

1. **High cohesion, low coupling** — setiap modul memiliki tanggung jawab tunggal yang jelas, dan berkomunikasi dengan modul lain melalui interface/event yang eksplisit, bukan pemanggilan langsung antar internal class.
2. **Modul berkomunikasi via Service Layer, bukan Repository langsung** — mencegah satu modul mengakses data modul lain tanpa melalui business logic yang seharusnya.
3. **Cross-module interaction melalui Domain Event bila memungkinkan** — mengurangi ketergantungan langsung (tight coupling) antar modul, terutama untuk efek samping seperti notifikasi dan audit log.
4. **Setiap modul memetakan ke satu atau lebih Business Requirement** — menjaga traceability sesuai prinsip pada [`README.md`](../../README.md).

---

## 2. Daftar Modul

| Modul | Tanggung Jawab Utama | Terkait FR |
|---|---|---|
| Master Data | Mengelola data inti (produk, kategori, cabang, pelanggan, supplier, pegawai) | FR-MD-xx |
| Sales (Penjualan) | Memproses transaksi penjualan, void, retur, open bill | FR-SL-xx |
| Purchasing (Pembelian) | Mengelola PO dan penerimaan barang | FR-PU-xx |
| Inventory (Stok) | Mengelola pergerakan stok, transfer, opname | FR-IN-xx |
| Approval | Mengelola alur persetujuan lintas modul | FR-AP-xx |
| Promotion (Promosi & Loyalitas) | Mengelola promo dan poin loyalitas | FR-PR-xx |
| Reporting & Dashboard | Menyediakan data agregasi untuk laporan dan dashboard | FR-DB-xx |
| Notification | Mengelola pengiriman notifikasi lintas modul | FR-NT-xx |
| Auth & Access | Otentikasi dan otorisasi pengguna | FR-AU-xx |

---

## 3. Modul: Master Data

### 3.1 Tanggung Jawab
Menyediakan dan menjaga integritas data referensi yang digunakan seluruh modul lain: produk, kategori, satuan, cabang, pelanggan, supplier, pegawai.

### 3.2 Batasan Modul
- **Tidak** menangani logika transaksi (stok, penjualan) — hanya menyediakan data referensi.
- **Bertanggung jawab** atas validasi keunikan (SKU, kode cabang) dan siklus hidup soft-delete (lihat [`master-data.md`](../business/master-data.md#9-siklus-hidup-data-master-lifecycle)).

### 3.3 Interaksi dengan Modul Lain

```
Master Data ──(dikonsumsi via Service)──▶ Sales, Purchasing, Inventory, Reporting
```

Modul lain **tidak boleh** mengubah data master secara langsung; perubahan harga misalnya, hanya dapat dilakukan melalui `MasterDataService::updatePrice()` yang menegakkan BRULE-PR-01.

---

## 4. Modul: Sales (Penjualan)

### 4.1 Tanggung Jawab
Memproses seluruh siklus hidup transaksi penjualan: pembuatan transaksi, penerapan promo, void, retur, dan transaksi khusus domain (open bill F&B, kredit Distributor).

### 4.2 Sub-komponen

| Sub-komponen | Fungsi |
|---|---|
| `SalesService` | Orkestrasi pembuatan transaksi (validasi, hitung total, simpan) |
| `SaleVoidService` | Menangani permintaan void, terintegrasi dengan modul Approval |
| `SaleReturnService` | Menangani retur, validasi terhadap transaksi asal (BRULE-TX-05) |
| `OpenBillService` (khusus F&B) | Mengelola siklus open bill per meja |
| `CreditSaleService` (khusus Distributor) | Mengelola transaksi kredit dan validasi limit piutang |

### 4.3 Interaksi dengan Modul Lain

```
Sales
  ├──▶ Master Data (baca harga & data produk)
  ├──▶ Inventory (kurangi stok saat transaksi selesai — dalam 1 DB transaction)
  ├──▶ Approval (kirim permintaan saat void/diskon melebihi threshold)
  ├──▶ Promotion (baca & terapkan promo aktif)
  └──▶ Notification (event SaleCompleted, SaleVoided memicu notifikasi/audit)
```

**Alasan desain:** Pengurangan stok terjadi **dalam service yang sama** dengan pembuatan transaksi penjualan (bukan event asynchronous terpisah) untuk menjamin konsistensi data secara atomic (BRULE-ST-01, EC-SL-02). Notifikasi dan audit log, sebaliknya, **dilakukan via event asynchronous** karena tidak memerlukan konsistensi transaksional yang sama ketatnya.

---

## 5. Modul: Purchasing (Pembelian)

### 5.1 Tanggung Jawab
Mengelola pembuatan Purchase Order hingga penerimaan barang dari supplier.

### 5.2 Sub-komponen

| Sub-komponen | Fungsi |
|---|---|
| `PurchaseOrderService` | Pembuatan dan pengelolaan status PO |
| `GoodsReceiptService` | Pencatatan penerimaan barang, termasuk penerimaan sebagian (EC-ST-05) |

### 5.3 Interaksi dengan Modul Lain

```
Purchasing
  ├──▶ Master Data (baca data supplier & produk)
  └──▶ Inventory (memicu penambahan stok saat goods receipt dikonfirmasi)
```

---

## 6. Modul: Inventory (Stok)

### 6.1 Tanggung Jawab
Mengelola seluruh pergerakan stok: penambahan (dari pembelian), pengurangan (dari penjualan), transfer antar cabang, penyesuaian, dan stok opname. Mengelola kartu stok (stock ledger) sebagai jejak audit.

### 6.2 Sub-komponen

| Sub-komponen | Fungsi |
|---|---|
| `StockService` | Fungsi dasar penambahan/pengurangan stok, dipanggil modul lain |
| `StockTransferService` | Mengelola siklus hidup transfer antar cabang |
| `StockOpnameService` | Mengelola proses stok opname dan perhitungan selisih |
| `StockAdjustmentService` | Mengelola penyesuaian manual, terintegrasi dengan modul Approval |
| `BatchExpiryService` | Mengelola alokasi FEFO dan deteksi produk mendekati kadaluarsa |

### 6.3 Interaksi dengan Modul Lain

```
Inventory
  ├──◀── dipanggil oleh Sales (kurangi stok)
  ├──◀── dipanggil oleh Purchasing (tambah stok)
  ├──▶ Approval (kirim permintaan untuk penyesuaian & selisih opname signifikan)
  └──▶ Notification (event StockLow, ProductExpiringSoon)
```

**Alasan desain:** `StockService` menjadi **satu-satunya titik masuk** untuk mengubah nilai stok di database — modul lain (Sales, Purchasing) tidak pernah memanipulasi tabel stok secara langsung. Ini menegakkan BRULE-ST-04 (setiap perubahan stok tercatat di kartu stok) secara struktural, bukan hanya konvensi.

---

## 7. Modul: Approval

### 7.1 Tanggung Jawab
Mengelola seluruh alur persetujuan lintas modul secara generik, sesuai [`approval-flow.md`](../business/approval-flow.md).

### 7.2 Desain Generik (Polymorphic Approval)

Modul Approval dirancang **generik** — dapat menangani berbagai jenis aksi (void, diskon, penyesuaian stok, piutang) tanpa membuat modul Approval terpisah untuk tiap jenis, menggunakan pola *polymorphic relation* pada database (detail di [`database-design.md`](database-design.md)).

```
ApprovalRequest {
    approvable_type: "Sale" | "StockAdjustment" | "CreditSale" | ...
    approvable_id: <id transaksi terkait>
    requested_by, status, threshold_level, ...
}
```

**Alasan desain:** Tanpa desain generik ini, setiap modul baru yang membutuhkan approval (misal fitur masa depan) akan memerlukan duplikasi logic approval — desain polymorphic memungkinkan modul Approval **dipakai ulang** oleh modul mana pun tanpa modifikasi.

### 7.3 Interaksi dengan Modul Lain

```
Approval
  ├──◀── dipanggil oleh Sales, Inventory (membuat permintaan approval)
  ├──▶ Auth & Access (cek kewenangan approver sesuai role & threshold)
  └──▶ Notification (event ApprovalRequested, ApprovalDecided)
```

---

## 8. Modul: Promotion (Promosi & Loyalitas)

### 8.1 Tanggung Jawab
Mengelola definisi promo (diskon, bundling), penerapan otomatis saat transaksi, dan program poin loyalitas pelanggan.

### 8.2 Interaksi dengan Modul Lain

```
Promotion
  ├──◀── dipanggil oleh Sales (evaluasi promo aktif saat transaksi)
  └──▶ Master Data (baca data pelanggan untuk poin loyalitas)
```

---

## 9. Modul: Reporting & Dashboard

### 9.1 Tanggung Jawab
Menyediakan data agregasi untuk kebutuhan laporan ([`reports.md`](../business/reports.md)) dan dashboard ([`dashboard.md`](../business/dashboard.md)).

### 9.2 Desain: Read Model Terpisah untuk Laporan Berat

Untuk laporan dengan agregasi kompleks/volume besar, modul ini menggunakan **query yang dioptimalkan terpisah** (bukan menggunakan Eloquent model transaksional yang sama dengan modul Sales), dan untuk laporan sangat besar diproses via queue worker (sesuai NFR-PF-04).

### 9.3 Interaksi dengan Modul Lain

```
Reporting
  └──◀── membaca data (read-only) dari Sales, Inventory, Purchasing, Approval
```

**Alasan desain:** Modul Reporting **tidak pernah menulis** data ke modul lain — murni read-only, mencegah efek samping tak terduga terhadap data transaksional.

---

## 10. Modul: Notification

### 10.1 Tanggung Jawab
Menerima event dari modul lain dan mengirimkan notifikasi sesuai kanal yang sesuai ([`notifications.md`](../business/notifications.md)).

### 10.2 Desain: Event-Driven

```
Modul Sumber (Sales, Inventory, Approval, dll)
    → Dispatch Domain Event (mis. StockLow, ApprovalRequested)
    → Listener di Modul Notification menangkap event
    → NotificationService menentukan kanal & penerima
    → Dikirim secara asynchronous via Queue
```

**Alasan desain:** Modul sumber **tidak perlu tahu** bagaimana/ke mana notifikasi dikirim — cukup men-dispatch event. Ini menjaga low coupling dan memudahkan penambahan kanal notifikasi baru di masa depan tanpa mengubah modul sumber.

---

## 11. Modul: Auth & Access

### 11.1 Tanggung Jawab
Otentikasi pengguna (login, token) dan otorisasi (role, permission, data scoping), sesuai [`authentication-design.md`](authentication-design.md) dan [`authorization-design.md`](authorization-design.md).

### 11.2 Interaksi dengan Modul Lain

```
Auth & Access
  └──◀── digunakan oleh SEMUA modul lain (via Middleware & Policy)
```

**Alasan desain:** Modul ini bersifat **cross-cutting concern** — diakses oleh hampir seluruh modul melalui middleware terpusat, bukan dipanggil secara eksplisit di tiap Service.

---

## 12. Diagram Interaksi Modul Menyeluruh

```
                         ┌─────────────┐
                         │  Auth &     │◀───────── (cross-cutting, semua modul)
                         │  Access     │
                         └─────────────┘

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Master Data │◀───▶│    Sales    │◀───▶│  Promotion  │
└─────────────┘     └──────┬──────┘     └─────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
      ┌─────────────┐┌─────────────┐┌─────────────┐
      │  Inventory  ││  Approval   ││Notification │
      └──────┬──────┘└─────────────┘└─────────────┘
             │
             ▼
      ┌─────────────┐
      │ Purchasing  │
      └─────────────┘

      ┌─────────────────────────────────┐
      │   Reporting & Dashboard         │  (read-only dari seluruh modul di atas)
      └─────────────────────────────────┘
```

---

## 13. Traceability

| Modul | Terkait Dokumen |
|---|---|
| Seluruh modul | [`business-flow.md`](../business/business-flow.md), [`use-case.md`](../business/use-case.md) |
| Implementasi service | [`service-layer-design.md`](service-layer-design.md) |
| Implementasi data access | [`repository-layer-design.md`](repository-layer-design.md) |
| Kontrak API per modul | [`api-contract.md`](api-contract.md) |

---

**Sebelumnya:** [`folder-structure.md`](folder-structure.md) — Folder Structure
**Selanjutnya:** [`database-design.md`](database-design.md) — Database Design
