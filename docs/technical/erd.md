# Entity Relationship Diagram (ERD) — POS

> Dokumen ini menyajikan **visualisasi relasi antar entitas** secara lengkap, melengkapi definisi tabel pada [`database-design.md`](database-design.md). Diagram disajikan dalam notasi Mermaid ER Diagram agar dapat dirender langsung oleh GitHub/GitLab tanpa tools tambahan.

---

## 1. ERD: Master Data

```mermaid
erDiagram
    BRANCHES ||--o{ BRANCHES : "parent_id (hierarki outlet)"
    BRANCHES ||--o{ EMPLOYEES : "ditempatkan di"
    BRANCHES ||--o{ CUSTOMERS : "terdaftar di"

    CATEGORIES ||--o{ PRODUCTS : "mengelompokkan"
    UNITS ||--o{ PRODUCTS : "satuan dasar"
    PRODUCTS ||--o{ PRODUCT_UNITS : "memiliki konversi"
    UNITS ||--o{ PRODUCT_UNITS : "digunakan sebagai konversi"

    CUSTOMER_CLASSES ||--o{ CUSTOMERS : "menentukan tiered pricing"

    SUPPLIERS ||--o{ PURCHASE_ORDERS : "menerima pesanan"

    BRANCHES {
        uuid id PK
        uuid parent_id FK
        string code
        string name
        boolean is_active
    }
    PRODUCTS {
        uuid id PK
        string sku
        string name
        uuid category_id FK
        uuid base_unit_id FK
        numeric selling_price
        numeric cost_price
        boolean is_prescription_required
        boolean requires_batch_tracking
        jsonb attributes
    }
    CUSTOMERS {
        uuid id PK
        uuid branch_id FK
        uuid customer_class_id FK
        string name
        string type
        numeric credit_limit
        int loyalty_points
    }
    EMPLOYEES {
        uuid id PK
        uuid branch_id FK
        string name
        string role
        boolean is_active
    }
```

---

## 2. ERD: Modul Sales (Penjualan)

```mermaid
erDiagram
    BRANCHES ||--o{ SALES_TRANSACTIONS : "terjadi di"
    CASHIER_SHIFTS ||--o{ SALES_TRANSACTIONS : "dicatat dalam"
    EMPLOYEES ||--o{ CASHIER_SHIFTS : "membuka"
    CUSTOMERS ||--o{ SALES_TRANSACTIONS : "melakukan (opsional)"
    TABLES ||--o{ SALES_TRANSACTIONS : "open bill di (F&B, opsional)"

    SALES_TRANSACTIONS ||--|{ SALE_ITEMS : "terdiri dari"
    SALES_TRANSACTIONS ||--|{ SALE_PAYMENTS : "dibayar melalui"
    PRODUCTS ||--o{ SALE_ITEMS : "direferensikan (snapshot)"
    PRODUCT_BATCHES ||--o{ SALE_ITEMS : "dialokasikan dari (FEFO)"

    SALE_ITEMS ||--o| KITCHEN_ORDERS : "diproses di dapur (F&B)"
    SALE_ITEMS ||--o| PRESCRIPTION_VALIDATIONS : "divalidasi (Apotek)"

    SALES_TRANSACTIONS {
        uuid id PK
        uuid branch_id FK
        uuid shift_id FK
        uuid customer_id FK
        uuid table_id FK
        string transaction_number
        string status
        numeric grand_total
        boolean is_credit_sale
    }
    SALE_ITEMS {
        uuid id PK
        uuid sale_id FK
        uuid product_id FK
        uuid batch_id FK
        string product_name_snapshot
        numeric unit_price_snapshot
        numeric quantity
        numeric subtotal
    }
    SALE_PAYMENTS {
        uuid id PK
        uuid sale_id FK
        string payment_method
        numeric amount
    }
    CASHIER_SHIFTS {
        uuid id PK
        uuid branch_id FK
        uuid cashier_id FK
        numeric opening_balance
        numeric closing_balance_actual
        numeric variance
        string status
    }
```

---

## 3. ERD: Modul Inventory & Purchasing

```mermaid
erDiagram
    BRANCHES ||--o{ STOCK_LEDGER : "mencatat pergerakan di"
    PRODUCTS ||--o{ STOCK_LEDGER : "bergerak"
    PRODUCT_BATCHES ||--o{ STOCK_LEDGER : "dari batch"
    PRODUCTS ||--o{ CURRENT_STOCK : "diringkas stoknya"
    BRANCHES ||--o{ CURRENT_STOCK : "per cabang"

    SUPPLIERS ||--o{ PURCHASE_ORDERS : "dipesan dari"
    BRANCHES ||--o{ PURCHASE_ORDERS : "dipesan oleh"
    PURCHASE_ORDERS ||--|{ PURCHASE_ORDER_ITEMS : "terdiri dari"
    PRODUCTS ||--o{ PURCHASE_ORDER_ITEMS : "dipesan"

    PURCHASE_ORDERS ||--o{ GOODS_RECEIPTS : "diterima melalui"
    GOODS_RECEIPTS ||--|{ GOODS_RECEIPT_ITEMS : "terdiri dari"
    GOODS_RECEIPT_ITEMS ||--o| STOCK_LEDGER : "memicu entri"

    BRANCHES ||--o{ STOCK_TRANSFERS : "asal/tujuan"
    STOCK_TRANSFERS ||--|{ STOCK_TRANSFER_ITEMS : "terdiri dari"
    STOCK_TRANSFER_ITEMS ||--o| STOCK_LEDGER : "memicu entri (in & out)"

    BRANCHES ||--o{ STOCK_OPNAMES : "dilakukan di"
    STOCK_OPNAMES ||--|{ STOCK_OPNAME_ITEMS : "terdiri dari"
    STOCK_OPNAME_ITEMS ||--o| STOCK_LEDGER : "memicu entri adjustment"

    STOCK_LEDGER {
        uuid id PK
        uuid branch_id FK
        uuid product_id FK
        uuid batch_id FK
        string movement_type
        numeric quantity
        string reference_type
        uuid reference_id
        numeric balance_after
    }
    CURRENT_STOCK {
        uuid branch_id PK_FK
        uuid product_id PK_FK
        numeric quantity
        numeric reorder_point
    }
    PURCHASE_ORDERS {
        uuid id PK
        uuid branch_id FK
        uuid supplier_id FK
        string po_number
        string status
        numeric total_amount
    }
    PRODUCT_BATCHES {
        uuid id PK
        uuid product_id FK
        uuid branch_id FK
        string batch_number
        date expiry_date
        numeric quantity_available
    }
```

---

## 4. ERD: Modul Approval (Polymorphic)

```mermaid
erDiagram
    EMPLOYEES ||--o{ APPROVAL_REQUESTS : "mengajukan (requested_by)"
    EMPLOYEES ||--o{ APPROVAL_REQUESTS : "memutuskan (decided_by)"
    BRANCHES ||--o{ APPROVAL_REQUESTS : "terjadi di"

    SALES_TRANSACTIONS ||--o| APPROVAL_REQUESTS : "approvable (void/diskon)"
    STOCK_OPNAME_ITEMS ||--o| APPROVAL_REQUESTS : "approvable (selisih signifikan)"
    STOCK_ADJUSTMENTS ||--o| APPROVAL_REQUESTS : "approvable (penyesuaian manual)"
    SALES_TRANSACTIONS ||--o| APPROVAL_REQUESTS : "approvable (kredit melebihi limit)"

    APPROVAL_REQUESTS {
        uuid id PK
        uuid branch_id FK
        string approvable_type
        uuid approvable_id
        uuid requested_by FK
        string current_approver_role
        string status
        text reason
        uuid decided_by FK
        timestamp decided_at
    }
```

**Catatan notasi:** Relasi `approvable_type`/`approvable_id` bersifat **polymorphic** — tidak digambarkan sebagai foreign key literal pada database (karena dapat menunjuk ke beberapa tabel berbeda), namun divisualisasikan di sini untuk kejelasan konseptual. Implementasi query harus memfilter berdasarkan `approvable_type` terlebih dahulu, detail pada [`repository-layer-design.md`](repository-layer-design.md).

---

## 5. ERD: Domain Khusus (F&B, Apotek, Distributor)

```mermaid
erDiagram
    BRANCHES ||--o{ TABLES : "dimiliki"
    TABLES ||--o{ SALES_TRANSACTIONS : "open bill di"
    SALES_TRANSACTIONS ||--|{ SALE_ITEMS : "terdiri dari"
    SALE_ITEMS ||--|| KITCHEN_ORDERS : "diproses sebagai"

    SALE_ITEMS ||--o| PRESCRIPTION_VALIDATIONS : "divalidasi sebagai"
    EMPLOYEES ||--o{ PRESCRIPTION_VALIDATIONS : "memvalidasi (role apoteker)"

    CUSTOMERS ||--o{ CUSTOMER_INVOICES : "menerima invoice"
    SALES_TRANSACTIONS ||--|| CUSTOMER_INVOICES : "diterbitkan dari (kredit)"
    CUSTOMER_INVOICES ||--o{ INVOICE_PAYMENTS : "dilunasi melalui"

    KITCHEN_ORDERS {
        uuid id PK
        uuid sale_id FK
        uuid sale_item_id FK
        string status
        timestamp sent_at
        timestamp ready_at
    }
    PRESCRIPTION_VALIDATIONS {
        uuid id PK
        uuid sale_item_id FK
        uuid validated_by FK
        string status
        timestamp validated_at
    }
    CUSTOMER_INVOICES {
        uuid id PK
        uuid customer_id FK
        uuid sale_id FK
        numeric total_amount
        date due_date
        string status
    }
    INVOICE_PAYMENTS {
        uuid id PK
        uuid invoice_id FK
        numeric amount
        timestamp paid_at
    }
```

---

## 6. ERD: Relasi Auth & Access

```mermaid
erDiagram
    EMPLOYEES ||--o{ EMPLOYEE_ROLES : "memiliki"
    ROLES ||--o{ EMPLOYEE_ROLES : "diberikan ke"
    ROLES ||--o{ ROLE_PERMISSIONS : "memiliki"
    PERMISSIONS ||--o{ ROLE_PERMISSIONS : "diberikan pada"
    EMPLOYEES ||--o{ AUTH_TOKENS : "memiliki sesi"

    EMPLOYEES {
        uuid id PK
        uuid branch_id FK
        string name
        string email
        string password_hash
        boolean is_active
    }
    ROLES {
        uuid id PK
        string name
        string level
    }
    PERMISSIONS {
        uuid id PK
        string name
    }
    AUTH_TOKENS {
        uuid id PK
        uuid employee_id FK
        string token_hash
        timestamp expires_at
        timestamp revoked_at
    }
```

Detail desain otentikasi dan otorisasi dijelaskan lebih lanjut pada [`authentication-design.md`](authentication-design.md) dan [`authorization-design.md`](authorization-design.md).

---

## 7. Ringkasan Relasi Kunci Lintas Modul

| Relasi | Sifat | Alasan Desain |
|---|---|---|
| `sales_transactions` → `stock_ledger` | Tidak langsung (via `reference_type`/`reference_id`) | Stock ledger bersifat polymorphic terhadap berbagai sumber pergerakan, bukan hanya penjualan |
| `sale_items` → `product_batches` | Opsional (nullable) | Tidak semua produk memerlukan tracking batch (BRULE-ST-02) |
| `approval_requests` → berbagai entitas | Polymorphic | Satu modul Approval melayani banyak jenis aksi tanpa duplikasi struktur (lihat [`module-design.md`](module-design.md#72-desain-generik-polymorphic-approval)) |
| `sales_transactions` → `customer_invoices` | 1-to-1 (khusus transaksi kredit) | Hanya transaksi dengan `is_credit_sale = true` yang menghasilkan invoice |

---

## 8. Traceability

| Aspek ERD | Terkait Dokumen |
|---|---|
| Definisi kolom & tipe data lengkap | [`database-design.md`](database-design.md) |
| Definisi entitas dari sisi bisnis | [`master-data.md`](../business/master-data.md), [`transactions.md`](../business/transactions.md) |
| Implementasi akses data | [`repository-layer-design.md`](repository-layer-design.md) |

---

**Sebelumnya:** [`database-design.md`](database-design.md) — Database Design
**Selanjutnya:** [`api-contract.md`](api-contract.md) — API Contract
