# API Contract — POS

> Dokumen ini menjelaskan **kontrak API** — konvensi umum, struktur request/response, dan daftar endpoint utama per modul. Dokumen ini menjembatani backend Laravel dengan Frontend Vue dan Mobile Android, sesuai prinsip *API-first* pada [`system-architecture.md`](system-architecture.md#12-keputusan-api-first-architecture). Detail otentikasi dijelaskan pada [`authentication-design.md`](authentication-design.md).

---

## 1. Konvensi Umum

### 1.1 Base URL & Versioning

```
https://api.pos-domain.com/api/v1/
```

**Alasan versioning eksplisit (`/v1/`):** Memungkinkan evolusi API di masa depan (`/v2/`) tanpa memutus kompatibilitas klien mobile yang mungkin belum diupdate oleh seluruh pengguna cabang secara serentak — kondisi umum pada aplikasi mobile enterprise.

### 1.2 Format Request & Response

- Content-Type: `application/json` untuk seluruh request/response (kecuali upload file).
- Seluruh response mengikuti struktur *envelope* konsisten:

```json
// Response Sukses
{
  "success": true,
  "data": { ... },
  "meta": { "pagination": { "page": 1, "per_page": 20, "total": 150 } }
}

// Response Error
{
  "success": false,
  "message": "Stok tidak mencukupi untuk produk ini",
  "errors": {
    "quantity": ["Stok tersedia hanya 3, diminta 5"]
  },
  "error_code": "INSUFFICIENT_STOCK"
}
```

**Alasan desain `error_code`:** Selain pesan yang *human-readable* (mendukung kebutuhan pesan error actionable pada [`validation.md`](../business/validation.md)), kode error terstruktur memungkinkan klien (Vue/Android) menangani kasus tertentu secara programatik (misal menampilkan modal khusus untuk `INSUFFICIENT_STOCK`) tanpa parsing string pesan yang rapuh.

### 1.3 Kode Status HTTP yang Digunakan

| Kode | Kegunaan |
|---|---|
| 200 | Sukses (GET, PUT/PATCH, atau POST aksi non-create) |
| 201 | Sukses membuat resource baru (POST create) |
| 202 | Diterima untuk diproses asynchronous (mis. generate laporan besar) |
| 400 | Request tidak valid (validasi gagal) |
| 401 | Tidak terotentikasi |
| 403 | Terotentikasi namun tidak berwenang (permission/data scoping) |
| 404 | Resource tidak ditemukan |
| 409 | Konflik state (mis. mencoba membuka shift baru saat shift lain masih aktif — EC-SH-02) |
| 422 | Business rule violation (mis. stok tidak cukup, melebihi limit kredit) |
| 429 | Rate limit terlampaui |
| 500 | Error tidak terduga di server |

**Alasan pemisahan 400 vs 422:** Kode 400 digunakan untuk kegagalan validasi format/struktur data dasar, sedangkan 422 khusus untuk pelanggaran **business rule** ([`business-rules.md`](../business/business-rules.md)) — memudahkan klien membedakan "input Anda salah format" dari "input Anda valid tapi melanggar aturan bisnis".

### 1.4 Pagination

Seluruh endpoint list menggunakan pagination berbasis halaman:

```
GET /api/v1/products?page=1&per_page=20&sort=-created_at
```

### 1.5 Filtering

Konvensi query parameter filter menggunakan format `filter[field]`:

```
GET /api/v1/sales?filter[branch_id]=<uuid>&filter[date_from]=2026-08-01&filter[date_to]=2026-08-09
```

---

## 2. Endpoint: Autentikasi

| Method | Endpoint | Deskripsi | Terkait FR |
|---|---|---|---|
| POST | `/auth/login` | Login dengan email/username & password | FR-AU-01 |
| POST | `/auth/logout` | Logout, invalidasi token saat ini | FR-AU-01 |
| GET | `/auth/me` | Mendapatkan data pengguna yang sedang login beserta role & permission | FR-AU-02 |
| POST | `/auth/refresh` | Refresh token (jika menggunakan refresh token) | NFR-SE-05 |

**Contoh Request/Response — Login:**

```json
// POST /api/v1/auth/login
// Request
{ "email": "kasir1@cabang-a.com", "password": "********", "branch_code": "CAB-JKT-01" }

// Response 200
{
  "success": true,
  "data": {
    "token": "1|xxxxxxxx...",
    "employee": {
      "id": "uuid",
      "name": "Budi",
      "role": "cashier",
      "branch_id": "uuid",
      "permissions": ["sale.create", "sale.view"]
    }
  }
}
```

---

## 3. Endpoint: Master Data

| Method | Endpoint | Deskripsi | Terkait FR |
|---|---|---|---|
| GET | `/products` | Daftar produk (dengan filter cabang, kategori, kata kunci) | FR-MD-01 |
| GET | `/products/{id}` | Detail produk | FR-MD-01 |
| GET | `/products/search?barcode={code}` | Pencarian cepat berdasarkan barcode (untuk transaksi kasir) | FR-SL-01 |
| POST | `/products` | Membuat produk baru (HO Admin) | FR-MD-01 |
| PUT | `/products/{id}` | Update data produk | FR-MD-01 |
| PATCH | `/products/{id}/price` | Update harga khusus (endpoint terpisah agar dapat diaudit khusus) | FR-MD-08, BRULE-PR-01 |
| GET | `/branches` | Daftar cabang | FR-MD-04 |
| GET | `/customers` | Daftar pelanggan | FR-MD-05 |
| POST | `/customers` | Membuat pelanggan baru | FR-MD-05 |
| GET | `/suppliers` | Daftar supplier | FR-MD-06 |
| GET | `/employees` | Daftar pegawai (Branch Manager: cabang sendiri; HO Admin: semua) | FR-MD-07 |

---

## 4. Endpoint: Sales (Penjualan)

| Method | Endpoint | Deskripsi | Terkait FR |
|---|---|---|---|
| POST | `/sales` | Membuat transaksi penjualan baru | FR-SL-01 s.d. FR-SL-03 |
| GET | `/sales` | Daftar transaksi (filter cabang, periode, kasir) | FR-DB-02 |
| GET | `/sales/{id}` | Detail transaksi | FR-SL-01 |
| POST | `/sales/{id}/void` | Mengajukan void transaksi | FR-SL-05 |
| POST | `/sales/{id}/return` | Mengajukan retur | FR-SL-06 |
| POST | `/sales/{id}/print-receipt` | Mencetak ulang struk | FR-SL-07 |
| POST | `/shifts/open` | Membuka shift kasir | FR-SL-09 |
| POST | `/shifts/{id}/close` | Menutup shift kasir | FR-SL-09 |
| GET | `/shifts/{id}/summary` | Ringkasan shift (untuk rekonsiliasi) | FR-SL-09 |

**Contoh Request/Response — Membuat Transaksi Penjualan:**

```json
// POST /api/v1/sales
// Request
{
  "shift_id": "uuid",
  "customer_id": null,
  "items": [
    { "product_id": "uuid", "quantity": 2, "unit_price_override": null, "discount_amount": 0 }
  ],
  "payments": [
    { "payment_method": "cash", "amount": 50000 }
  ]
}

// Response 201
{
  "success": true,
  "data": {
    "id": "uuid",
    "transaction_number": "TRX-CAB01-20260809-0001",
    "status": "completed",
    "grand_total": 45000,
    "change": 5000
  }
}

// Response 422 (contoh business rule violation)
{
  "success": false,
  "message": "Stok tidak mencukupi untuk salah satu produk",
  "error_code": "INSUFFICIENT_STOCK",
  "errors": { "items[0].quantity": ["Stok tersedia 1, diminta 2"] }
}
```

### 4.1 Endpoint Khusus Domain F&B

| Method | Endpoint | Deskripsi |
|---|---|---|
| GET | `/tables` | Daftar meja beserta status |
| POST | `/tables/{id}/open-bill` | Membuka open bill baru di meja |
| POST | `/sales/{id}/items` | Menambah item ke open bill yang sedang berjalan |
| POST | `/sales/{id}/split-bill` | Membagi tagihan (split bill) |
| PATCH | `/kitchen-orders/{id}/status` | Update status pesanan dapur |

### 4.2 Endpoint Khusus Domain Apotek

| Method | Endpoint | Deskripsi |
|---|---|---|
| POST | `/sale-items/{id}/validate-prescription` | Apoteker memvalidasi item resep |

### 4.3 Endpoint Khusus Domain Distributor

| Method | Endpoint | Deskripsi |
|---|---|---|
| GET | `/customers/{id}/credit-status` | Cek sisa limit piutang pelanggan |
| GET | `/invoices` | Daftar invoice pelanggan |
| POST | `/invoices/{id}/payments` | Mencatat pelunasan piutang |

---

## 5. Endpoint: Purchasing

| Method | Endpoint | Deskripsi | Terkait FR |
|---|---|---|---|
| POST | `/purchase-orders` | Membuat PO baru | FR-PU-01 |
| GET | `/purchase-orders` | Daftar PO | FR-PU-01 |
| GET | `/purchase-orders/{id}` | Detail PO | FR-PU-01 |
| POST | `/purchase-orders/{id}/goods-receipts` | Mencatat penerimaan barang (mendukung partial) | FR-PU-02 |
| POST | `/goods-receipts/{id}/return` | Mengajukan retur pembelian | FR-PU-04 |

---

## 6. Endpoint: Inventory (Stok)

| Method | Endpoint | Deskripsi | Terkait FR |
|---|---|---|---|
| GET | `/stock` | Stok saat ini per produk per cabang | FR-IN-01 |
| GET | `/stock/{product_id}/ledger` | Kartu stok (histori mutasi) suatu produk | FR-IN-07 |
| POST | `/stock-transfers` | Membuat permintaan transfer stok | FR-IN-02 |
| POST | `/stock-transfers/{id}/ship` | Menandai transfer sebagai dikirim | FR-IN-02 |
| POST | `/stock-transfers/{id}/receive` | Konfirmasi penerimaan transfer | FR-IN-02 |
| POST | `/stock-opnames` | Memulai sesi stok opname | FR-IN-03 |
| POST | `/stock-opnames/{id}/items` | Input hasil hitung fisik | FR-IN-03 |
| POST | `/stock-opnames/{id}/finalize` | Finalisasi opname (memicu approval jika selisih signifikan) | FR-IN-03 |
| POST | `/stock-adjustments` | Mengajukan penyesuaian stok manual | FR-IN-04 |

---

## 7. Endpoint: Approval

| Method | Endpoint | Deskripsi | Terkait FR |
|---|---|---|---|
| GET | `/approval-requests` | Daftar permintaan approval (filter status, role) | FR-AP-01 |
| GET | `/approval-requests/{id}` | Detail permintaan approval | FR-AP-02 |
| POST | `/approval-requests/{id}/approve` | Menyetujui permintaan | FR-AP-01 |
| POST | `/approval-requests/{id}/reject` | Menolak permintaan (wajib alasan) | FR-AP-01 |

---

## 8. Endpoint: Promosi & Loyalitas

| Method | Endpoint | Deskripsi | Terkait FR |
|---|---|---|---|
| GET | `/promotions` | Daftar promo aktif | FR-PR-01 |
| POST | `/promotions` | Membuat promo baru (HO Admin) | FR-PR-01 |
| GET | `/customers/{id}/loyalty-points` | Cek poin loyalitas pelanggan | FR-PR-03 |
| POST | `/customers/{id}/redeem-points` | Menukar poin loyalitas | FR-PR-03 |

---

## 9. Endpoint: Dashboard & Laporan

| Method | Endpoint | Deskripsi | Terkait FR |
|---|---|---|---|
| GET | `/dashboard/summary` | Ringkasan dashboard sesuai role pengguna (scoped otomatis) | FR-DB-01 |
| GET | `/reports/sales` | Laporan penjualan (filter periode, cabang, produk) | FR-DB-02 |
| GET | `/reports/stock` | Laporan stok | FR-DB-03 |
| GET | `/reports/finance` | Laporan keuangan dasar | FR-DB-04 |
| POST | `/reports/{type}/export` | Memicu ekspor laporan (asynchronous, response 202) | FR-DB-05 |
| GET | `/reports/exports/{id}/status` | Cek status & unduh hasil ekspor | FR-DB-05 |

**Contoh Response — Ekspor Laporan Asynchronous:**

```json
// POST /api/v1/reports/sales/export
// Response 202
{
  "success": true,
  "data": { "export_id": "uuid", "status": "processing" },
  "message": "Laporan sedang diproses, Anda akan menerima notifikasi saat siap"
}
```

---

## 10. Endpoint: Notifikasi

| Method | Endpoint | Deskripsi | Terkait FR |
|---|---|---|---|
| GET | `/notifications` | Daftar notifikasi pengguna saat ini | FR-NT-01 s.d. FR-NT-04 |
| PATCH | `/notifications/{id}/read` | Menandai notifikasi sebagai dibaca | — |

---

## 11. Aturan Otorisasi pada Level Endpoint

Setiap endpoint menerapkan dua lapis pemeriksaan sebelum diproses (detail pada [`authorization-design.md`](authorization-design.md)):

1. **Permission check** — apakah role pengguna memiliki permission yang dibutuhkan (mis. `sale.void`).
2. **Data scoping check** — apakah data yang diakses berada dalam cakupan cabang pengguna (BRULE-MB-02), kecuali role level pusat.

Contoh respons ketika data scoping gagal:

```json
// Response 403
{
  "success": false,
  "message": "Anda tidak memiliki akses ke data cabang ini",
  "error_code": "BRANCH_SCOPE_VIOLATION"
}
```

---

## 12. Traceability

| Aspek API | Terkait Dokumen |
|---|---|
| Business logic di balik endpoint | [`service-layer-design.md`](service-layer-design.md) |
| Otentikasi | [`authentication-design.md`](authentication-design.md) |
| Otorisasi | [`authorization-design.md`](authorization-design.md) |
| Validasi request | [`validation.md`](../business/validation.md) |
| Kebutuhan fungsional asal | [`functional-requirements.md`](../business/functional-requirements.md) |

---

**Sebelumnya:** [`erd.md`](erd.md) — Entity Relationship Diagram
**Selanjutnya:** [`authentication-design.md`](authentication-design.md) — Authentication Design
