# Database Design — POS

> Dokumen ini menjelaskan **desain skema database** — konvensi penamaan, struktur tabel per modul, strategi indexing, dan strategi partitioning. Dokumen ini adalah implementasi teknis dari [`master-data.md`](../business/master-data.md) dan [`transactions.md`](../business/transactions.md), serta dilengkapi visualisasi relasi lengkap pada [`erd.md`](erd.md).

---

## 1. Prinsip Desain Database

1. **PostgreSQL sebagai single source of truth** — seluruh data transaksional dan master tersimpan terpusat, dengan `branch_id` sebagai kolom scoping (sesuai keputusan pada [`system-architecture.md`](system-architecture.md#5-batasan-arsitektur-pada-versi-ini)).
2. **Normalized untuk data master, denormalized secukupnya untuk data transaksi (snapshot)** — sesuai kebutuhan [`transactions.md`](../business/transactions.md#1-prinsip-umum-transaksi) di mana transaksi menyimpan snapshot harga/nama produk.
3. **Soft delete pada data master** — kolom `deleted_at` (nullable) digunakan untuk menonaktifkan tanpa menghapus fisik, menjaga integritas data historis (EC-SL-06).
4. **Audit trail melekat di level tabel** — kolom `created_by`, `updated_by`, `created_at`, `updated_at` wajib ada di seluruh tabel yang relevan dengan NFR-AU-01.
5. **Constraint di level database sebagai pertahanan lapis kedua** — business rule kritis (BRULE-ST-01, BRULE-TX-02) ditegakkan juga melalui `CHECK constraint` dan foreign key, bukan hanya validasi di aplikasi.

---

## 2. Konvensi Penamaan

| Elemen | Konvensi | Contoh |
|---|---|---|
| Nama tabel | snake_case, plural | `sales_transactions`, `products` |
| Nama kolom | snake_case | `branch_id`, `total_amount` |
| Primary key | `id` (UUID, lihat Bagian 3) | `id` |
| Foreign key | `<nama_tabel_singular>_id` | `product_id`, `branch_id` |
| Kolom boolean | prefix `is_`/`has_` | `is_active`, `has_prescription` |
| Kolom timestamp | suffix `_at` | `created_at`, `approved_at` |
| Kolom status | `status` (enum/string terkontrol) | `status` |
| Tabel pivot/polymorphic | `<tabelA>_<tabelB>` atau nama deskriptif | `approval_requests` (polymorphic) |

---

## 3. Keputusan: UUID sebagai Primary Key

Seluruh tabel menggunakan **UUID** (bukan auto-increment integer) sebagai primary key.

| Pertimbangan | UUID (Dipilih) | Auto-increment Integer |
|---|---|---|
| Keamanan (tidak mudah ditebak) | Baik — ID tidak berurutan, sulit dieksploitasi via enumerasi | Buruk — ID berurutan mudah ditebak |
| Merge data multi-cabang/offline (mobile) | Baik — ID dapat digenerate di klien (Android) sebelum sinkronisasi tanpa risiko konflik | Buruk — konflik ID saat sinkronisasi data offline |
| Ukuran index | Lebih besar dibanding integer | Lebih kecil |

**Alasan bisnis & teknis:** Karena sistem mendukung **transaksi offline di mobile** (BR-02, EC-SL-01) yang perlu membuat ID transaksi sebelum tersinkron ke server, UUID (khususnya UUIDv7 yang tetap dapat diurutkan berdasarkan waktu) adalah pilihan yang jauh lebih aman dibanding auto-increment yang rawan konflik saat sinkronisasi banyak perangkat.

---

## 4. Struktur Tabel per Modul

### 4.1 Modul Master Data

```sql
-- branches: struktur cabang, mendukung hierarki Head Office → Branch → Outlet
branches (
    id UUID PRIMARY KEY,
    parent_id UUID NULL REFERENCES branches(id),
    code VARCHAR UNIQUE NOT NULL,
    name VARCHAR NOT NULL,
    address TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at, updated_at, deleted_at
)

-- products
products (
    id UUID PRIMARY KEY,
    sku VARCHAR UNIQUE NOT NULL,
    name VARCHAR NOT NULL,
    category_id UUID REFERENCES categories(id),
    base_unit_id UUID REFERENCES units(id),
    selling_price NUMERIC(15,2) NOT NULL CHECK (selling_price >= 0),
    cost_price NUMERIC(15,2),
    tax_rate NUMERIC(5,2) DEFAULT 0,
    is_prescription_required BOOLEAN DEFAULT FALSE,   -- ekstensi domain Apotek
    requires_batch_tracking BOOLEAN DEFAULT FALSE,     -- ekstensi domain Apotek/Minimarket
    attributes JSONB,                                  -- atribut ekstensi per domain (mis. modifier F&B)
    is_active BOOLEAN DEFAULT TRUE,
    created_at, updated_at, deleted_at
)

-- product_units: konversi multi-satuan
product_units (
    id UUID PRIMARY KEY,
    product_id UUID REFERENCES products(id),
    unit_id UUID REFERENCES units(id),
    conversion_ratio NUMERIC(15,4) NOT NULL,  -- mis. 1 dus = 12 pcs → ratio 12
    price_override NUMERIC(15,2) NULL
)

-- customers
customers (
    id UUID PRIMARY KEY,
    branch_id UUID REFERENCES branches(id),  -- cabang tempat pelanggan pertama terdaftar
    name VARCHAR NOT NULL,
    phone VARCHAR,
    type VARCHAR CHECK (type IN ('general','member','reseller')),
    customer_class_id UUID NULL REFERENCES customer_classes(id),  -- untuk tiered pricing
    credit_limit NUMERIC(15,2) DEFAULT 0,
    loyalty_points INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at, updated_at, deleted_at
)

-- suppliers, employees, categories, units → struktur serupa dengan pola di atas
```

**Alasan desain kolom `attributes JSONB` pada `products`:** Menghindari duplikasi tabel produk per domain (misal `pharmacy_products`, `fnb_products`), sesuai keputusan pada [`master-data.md`](../business/master-data.md#22-atribut-ekstensi-per-domain). Atribut yang **sering di-query** (seperti `is_prescription_required`) tetap dijadikan kolom eksplisit untuk mendukung indexing, sementara atribut yang jarang di-query disimpan di JSONB.

### 4.2 Modul Sales (Penjualan)

```sql
-- sales_transactions
sales_transactions (
    id UUID PRIMARY KEY,
    branch_id UUID NOT NULL REFERENCES branches(id),
    shift_id UUID NOT NULL REFERENCES cashier_shifts(id),
    customer_id UUID NULL REFERENCES customers(id),
    transaction_number VARCHAR UNIQUE NOT NULL,
    status VARCHAR CHECK (status IN ('draft','completed','voided','partially_returned','returned')),
    subtotal NUMERIC(15,2) NOT NULL,
    discount_total NUMERIC(15,2) DEFAULT 0,
    tax_total NUMERIC(15,2) DEFAULT 0,
    grand_total NUMERIC(15,2) NOT NULL CHECK (grand_total >= 0),
    table_id UUID NULL REFERENCES tables(id),  -- khusus domain F&B
    is_credit_sale BOOLEAN DEFAULT FALSE,       -- khusus domain Distributor
    created_by UUID REFERENCES employees(id),
    created_at, updated_at
) PARTITION BY RANGE (created_at)  -- lihat Bagian 7

-- sale_items: snapshot produk saat transaksi (EC-SL-06)
sale_items (
    id UUID PRIMARY KEY,
    sale_id UUID NOT NULL REFERENCES sales_transactions(id),
    product_id UUID REFERENCES products(id),      -- referensi, boleh null jika produk dihapus
    product_name_snapshot VARCHAR NOT NULL,        -- snapshot, tidak berubah walau produk diubah
    unit_price_snapshot NUMERIC(15,2) NOT NULL,
    quantity NUMERIC(15,4) NOT NULL CHECK (quantity > 0),
    discount_amount NUMERIC(15,2) DEFAULT 0,
    batch_id UUID NULL REFERENCES product_batches(id),  -- alokasi FEFO
    subtotal NUMERIC(15,2) NOT NULL
)

-- sale_payments: mendukung split payment (FR-SL-02)
sale_payments (
    id UUID PRIMARY KEY,
    sale_id UUID NOT NULL REFERENCES sales_transactions(id),
    payment_method VARCHAR NOT NULL,
    amount NUMERIC(15,2) NOT NULL CHECK (amount > 0)
)

-- cashier_shifts
cashier_shifts (
    id UUID PRIMARY KEY,
    branch_id UUID NOT NULL REFERENCES branches(id),
    cashier_id UUID NOT NULL REFERENCES employees(id),
    opening_balance NUMERIC(15,2) NOT NULL,
    closing_balance_expected NUMERIC(15,2),
    closing_balance_actual NUMERIC(15,2),
    variance NUMERIC(15,2),
    status VARCHAR CHECK (status IN ('open','closed','pending_approval')),
    opened_at TIMESTAMP NOT NULL,
    closed_at TIMESTAMP NULL
)
```

### 4.3 Modul Inventory (Stok)

```sql
-- stock_ledger: kartu stok, satu-satunya sumber kebenaran pergerakan stok (BRULE-ST-04)
stock_ledger (
    id UUID PRIMARY KEY,
    branch_id UUID NOT NULL REFERENCES branches(id),
    product_id UUID NOT NULL REFERENCES products(id),
    batch_id UUID NULL REFERENCES product_batches(id),
    movement_type VARCHAR CHECK (movement_type IN
        ('purchase_in','sale_out','transfer_in','transfer_out','adjustment','return_in','return_out')),
    quantity NUMERIC(15,4) NOT NULL,  -- positif untuk masuk, negatif untuk keluar
    reference_type VARCHAR NOT NULL,   -- polymorphic: 'Sale','GoodsReceipt','StockTransfer', dll
    reference_id UUID NOT NULL,
    balance_after NUMERIC(15,4) NOT NULL,  -- saldo stok setelah pergerakan ini, untuk audit cepat
    created_by UUID REFERENCES employees(id),
    created_at
) PARTITION BY RANGE (created_at)

-- product_batches: tracking batch & expired (FR-MD-09)
product_batches (
    id UUID PRIMARY KEY,
    product_id UUID NOT NULL REFERENCES products(id),
    branch_id UUID NOT NULL REFERENCES branches(id),
    batch_number VARCHAR,
    expiry_date DATE,
    quantity_available NUMERIC(15,4) NOT NULL CHECK (quantity_available >= 0),
    created_at
)

-- current_stock: tabel ringkasan (denormalized) untuk kebutuhan pembacaan cepat saat transaksi
current_stock (
    branch_id UUID NOT NULL REFERENCES branches(id),
    product_id UUID NOT NULL REFERENCES products(id),
    quantity NUMERIC(15,4) NOT NULL DEFAULT 0,
    reorder_point NUMERIC(15,4) DEFAULT 0,
    updated_at,
    PRIMARY KEY (branch_id, product_id)
)

-- stock_transfers, stock_opnames, stock_adjustments → mengikuti pola serupa dengan status lifecycle
```

**Alasan desain tabel `current_stock` (denormalized):** Meski `stock_ledger` adalah sumber kebenaran utama, menghitung total stok dari seluruh histori ledger pada setiap transaksi penjualan **terlalu lambat** untuk memenuhi NFR-PF-01 (respons transaksi ≤ 1 detik). `current_stock` adalah tabel ringkasan yang diperbarui **dalam transaction yang sama** setiap kali `stock_ledger` mendapat entri baru, menjaga konsistensi tanpa mengorbankan performa baca.

### 4.4 Modul Purchasing

```sql
-- purchase_orders
purchase_orders (
    id UUID PRIMARY KEY,
    branch_id UUID NOT NULL REFERENCES branches(id),
    supplier_id UUID NOT NULL REFERENCES suppliers(id),
    po_number VARCHAR UNIQUE NOT NULL,
    status VARCHAR CHECK (status IN ('draft','sent','partially_received','received','cancelled')),
    total_amount NUMERIC(15,2),
    created_by UUID REFERENCES employees(id),
    created_at, updated_at
)

-- purchase_order_items, goods_receipts, goods_receipt_items → struktur serupa pola transaksi umum
```

### 4.5 Modul Approval (Polymorphic)

```sql
-- approval_requests: desain generik lintas modul (lihat module-design.md)
approval_requests (
    id UUID PRIMARY KEY,
    branch_id UUID NOT NULL REFERENCES branches(id),
    approvable_type VARCHAR NOT NULL,   -- 'Sale','StockAdjustment','CreditSale', dll
    approvable_id UUID NOT NULL,
    requested_by UUID NOT NULL REFERENCES employees(id),
    current_approver_role VARCHAR NOT NULL,  -- role yang berwenang saat ini
    status VARCHAR CHECK (status IN ('pending','approved','rejected','escalated','cancelled')),
    reason TEXT,
    decision_note TEXT,
    decided_by UUID NULL REFERENCES employees(id),
    decided_at TIMESTAMP NULL,
    created_at
)

-- index polymorphic wajib ada, lihat Bagian 5
```

### 4.6 Modul Domain Khusus

```sql
-- tables (F&B)
tables (
    id UUID PRIMARY KEY,
    branch_id UUID NOT NULL REFERENCES branches(id),
    table_number VARCHAR NOT NULL,
    status VARCHAR CHECK (status IN ('empty','occupied','awaiting_payment'))
)

-- kitchen_orders (F&B)
kitchen_orders (
    id UUID PRIMARY KEY,
    sale_id UUID NOT NULL REFERENCES sales_transactions(id),
    sale_item_id UUID NOT NULL REFERENCES sale_items(id),
    status VARCHAR CHECK (status IN ('queued','in_progress','ready','served')),
    sent_at, ready_at
)

-- prescription_validations (Apotek)
prescription_validations (
    id UUID PRIMARY KEY,
    sale_item_id UUID NOT NULL REFERENCES sale_items(id),
    validated_by UUID REFERENCES employees(id),  -- role apoteker
    status VARCHAR CHECK (status IN ('pending_validation','validated','rejected')),
    validated_at
)

-- customer_invoices, invoice_payments (Distributor) → mengelola siklus piutang
```

---

## 5. Strategi Indexing

| Tabel | Index | Alasan |
|---|---|---|
| `sales_transactions` | `(branch_id, created_at)` | Query laporan penjualan harian per cabang adalah query paling sering dijalankan (NFR-PF-01, NFR-PF-02) |
| `sales_transactions` | `(shift_id)` | Query rekonsiliasi kas per shift |
| `sale_items` | `(sale_id)`, `(product_id)` | Detail transaksi dan laporan penjualan per produk |
| `stock_ledger` | `(branch_id, product_id, created_at)` | Perhitungan mutasi stok dan kartu stok per produk per cabang |
| `current_stock` | Primary key composite `(branch_id, product_id)` | Lookup stok saat transaksi harus sangat cepat (kritikal untuk NFR-PF-01) |
| `products` | `(sku)` unique index | Pencarian barcode saat scan produk harus instan |
| `approval_requests` | `(approvable_type, approvable_id)` | Query polymorphic relation |
| `approval_requests` | `(status, current_approver_role, branch_id)` | Query dashboard approval pending per role/cabang |
| `product_batches` | `(product_id, branch_id, expiry_date)` | Query FEFO dan deteksi produk mendekati kadaluarsa |
| `customers` | `(phone)` | Pencarian pelanggan member saat transaksi |

---

## 6. Constraint Level Database Sebagai Pertahanan Lapis Kedua

| Constraint | Tabel | Menegakkan |
|---|---|---|
| `CHECK (grand_total >= 0)` | `sales_transactions` | Mencegah nilai transaksi negatif akibat bug aplikasi |
| `CHECK (quantity_available >= 0)` | `product_batches` | Mencegah stok batch negatif (BRULE-ST-01) |
| `CHECK (amount > 0)` | `sale_payments` | Mencegah pembayaran bernilai nol/negatif |
| `UNIQUE (sku)` | `products` | Menegakkan keunikan SKU di level database, tidak hanya aplikasi |
| Foreign Key dengan `ON DELETE RESTRICT` | Sebagian besar relasi ke `sales_transactions` | Mencegah penghapusan data yang masih direferensikan transaksi (menjaga integritas historis) |

**Alasan desain:** Validasi di level aplikasi (Service Layer) dapat memiliki bug atau terlewat pada endpoint baru yang lupa memanggilnya — constraint database menjadi jaring pengaman terakhir yang **tidak bisa dilewati** apa pun jalur aksesnya.

---

## 7. Strategi Partitioning

Tabel dengan volume tinggi dan terus bertambah — `sales_transactions` dan `stock_ledger` — menggunakan **range partitioning berdasarkan bulan** (`created_at`):

```sql
CREATE TABLE sales_transactions (
    ...
) PARTITION BY RANGE (created_at);

CREATE TABLE sales_transactions_2026_01 PARTITION OF sales_transactions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- dst., dibuat otomatis via scheduled job/migration bulanan
```

**Alasan teknis:** Mendukung NFR-SC-04 — seiring pertumbuhan jumlah cabang dan transaksi, query yang difilter berdasarkan rentang tanggal (laporan bulanan, dashboard harian) hanya perlu memindai partisi yang relevan, bukan seluruh tabel historis bertahun-tahun.

---

## 8. Traceability

| Aspek Database | Terkait Dokumen |
|---|---|
| Visualisasi relasi lengkap | [`erd.md`](erd.md) |
| Definisi entitas bisnis | [`master-data.md`](../business/master-data.md), [`transactions.md`](../business/transactions.md) |
| Aturan yang ditegakkan constraint | [`business-rules.md`](../business/business-rules.md) |
| Implementasi akses data | [`repository-layer-design.md`](repository-layer-design.md) |
| Strategi backup | [`backup-strategy.md`](backup-strategy.md) |

---

**Sebelumnya:** [`module-design.md`](module-design.md) — Module Design
**Selanjutnya:** [`erd.md`](erd.md) — Entity Relationship Diagram
