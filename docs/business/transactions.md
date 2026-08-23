# Transactions — POS

> Dokumen ini menjelaskan **definisi dan siklus hidup (lifecycle)** seluruh jenis transaksi dalam sistem — berbeda dengan [`master-data.md`](master-data.md) yang bersifat relatif statis, transaksi adalah data yang **terus bertambah setiap hari** dan menjadi inti nilai bisnis dari sistem POS ini. Dokumen ini menjadi acuan utama bagi [`database-design.md`](../technical/database-design.md) dan [`service-layer-design.md`](../technical/service-layer-design.md).

---

## 1. Prinsip Umum Transaksi

1. **Immutable by default** — setelah transaksi selesai (completed), data intinya tidak diubah langsung; koreksi dilakukan melalui transaksi baru (retur, void, adjustment) agar jejak audit utuh (sejalan dengan BRULE-AP-03).
2. **Setiap transaksi terikat konteks yang jelas** — cabang, pengguna yang membuat, dan waktu (BRULE-MB-01).
3. **Snapshot data terkait** — transaksi menyimpan salinan data penting saat transaksi terjadi (nama produk, harga saat itu), tidak hanya referensi ke master data yang bisa berubah (EC-SL-06).
4. **Status transaksi eksplisit** — setiap transaksi memiliki status yang jelas sepanjang siklus hidupnya, tidak ambigu.

---

## 2. Transaksi Penjualan (Sales)

### 2.1 Definisi

Transaksi yang mencatat penjualan produk/jasa kepada pelanggan, baik retail langsung, open bill (F&B), maupun kredit (Distributor).

### 2.2 Siklus Hidup

```
DRAFT (opsional, untuk open bill) → COMPLETED → (opsional: VOIDED / RETURNED (sebagian/penuh))
```

| Status | Deskripsi |
|---|---|
| `DRAFT` | Transaksi sedang disusun, belum final (relevan untuk open bill F&B) |
| `COMPLETED` | Pembayaran diterima penuh, transaksi selesai dan tercatat final |
| `VOIDED` | Transaksi dibatalkan melalui approval sebelum shift ditutup |
| `PARTIALLY_RETURNED` | Sebagian item pada transaksi diretur |
| `RETURNED` | Seluruh transaksi diretur |

### 2.3 Atribut Kunci

- Nomor transaksi (unik, dapat ditelusuri)
- Cabang & terminal kasir
- Kasir & shift terkait
- Daftar item (dengan snapshot nama, harga, kuantitas, diskon per item)
- Total sebelum diskon, total diskon, total pajak, total akhir
- Metode pembayaran (dapat lebih dari satu — split payment)
- Pelanggan (opsional, untuk transaksi member/kredit)
- Waktu transaksi

**Alasan bisnis:** Snapshot item mencegah laporan historis berubah nilai akibat perubahan harga produk di kemudian hari — kebutuhan integritas laporan keuangan (lihat BRULE-PR-04).

---

## 3. Transaksi Pembelian (Purchase Order & Goods Receipt)

### 3.1 Definisi

Rangkaian transaksi yang mencatat proses pemesanan barang ke supplier hingga barang diterima di gudang/cabang.

### 3.2 Siklus Hidup Purchase Order

```
DRAFT → SENT → PARTIALLY_RECEIVED → RECEIVED (selesai)
                        │
                        └──▶ CANCELLED (jika dibatalkan sebelum diterima)
```

| Status | Deskripsi |
|---|---|
| `DRAFT` | PO sedang disusun |
| `SENT` | PO telah dikirim/disetujui, menunggu barang dari supplier |
| `PARTIALLY_RECEIVED` | Sebagian barang telah diterima |
| `RECEIVED` | Seluruh barang pada PO telah diterima |
| `CANCELLED` | PO dibatalkan sebelum penerimaan barang |

### 3.3 Siklus Hidup Goods Receipt (Penerimaan Barang)

```
CREATED → CONFIRMED (stok diperbarui otomatis)
```

Setiap goods receipt mereferensikan PO terkait dan mencatat jumlah aktual diterima per item (dapat berbeda dari jumlah dipesan — lihat EC-ST-05 pada [`edge-cases.md`](edge-cases.md)).

---

## 4. Transaksi Stok (Inventory Movement)

### 4.1 Jenis Transaksi Stok

| Jenis | Deskripsi | Efek pada Stok |
|---|---|---|
| Stok Masuk (Goods Receipt) | Penerimaan barang dari supplier | Menambah stok |
| Stok Keluar (Penjualan) | Pengurangan akibat transaksi penjualan | Mengurangi stok |
| Transfer Keluar | Stok dikirim ke cabang lain | Mengurangi stok cabang asal |
| Transfer Masuk | Stok diterima dari cabang lain | Menambah stok cabang tujuan |
| Penyesuaian (Adjustment) | Koreksi manual (rusak, hilang, selisih opname) | Menambah/mengurangi sesuai nilai penyesuaian |
| Retur Penjualan | Barang kembali dari pelanggan | Menambah stok (jika kondisi baik) |
| Retur Pembelian | Barang dikembalikan ke supplier | Mengurangi stok |

### 4.2 Prinsip Pencatatan (Kartu Stok)

Setiap pergerakan stok, apa pun jenisnya, **selalu dicatat sebagai entri pada kartu stok (stock ledger)** dengan referensi ke transaksi sumbernya — tidak ada perubahan stok yang terjadi tanpa jejak transaksi (BRULE-ST-04, mendukung NFR-AU-01).

### 4.3 Siklus Hidup Transfer Stok

```
REQUESTED → SHIPPED → RECEIVED (selesai)
                │
                └──▶ RECEIVED_WITH_DISCREPANCY (jika ada selisih saat diterima — EC-ST-02)
```

---

## 5. Transaksi Approval

Didefinisikan detail pada [`approval-flow.md`](approval-flow.md). Ringkasan siklus hidup:

```
PENDING → APPROVED / REJECTED / ESCALATED
```

Setiap transaksi approval mereferensikan **transaksi sumber** yang memicunya (void, diskon, penyesuaian stok, dll.), sehingga membentuk relasi induk-anak yang dapat ditelusuri penuh.

---

## 6. Transaksi Keuangan Dasar

### 6.1 Rekonsiliasi Kas Shift

```
SHIFT_OPENED (kas awal dicatat) → TRANSACTIONS_RECORDED (berjalan) → SHIFT_CLOSED (kas akhir dicatat, selisih dihitung)
```

### 6.2 Piutang (Distributor)

```
INVOICE_ISSUED → PARTIALLY_PAID → PAID (lunas)
                       │
                       └──▶ OVERDUE (jika melewati jatuh tempo tanpa pelunasan penuh)
```

Setiap pembayaran piutang mereferensikan invoice spesifik (BRULE-CR-02), mendukung pelacakan piutang per pelanggan pada [`reports.md`](reports.md#5-kategori-laporan-keuangan).

---

## 7. Transaksi Domain F&B

### 7.1 Open Bill & Kitchen Order

```
TABLE_OPENED → ITEMS_ADDED (dapat berulang) → SENT_TO_KITCHEN → KITCHEN_IN_PROGRESS → KITCHEN_READY → SERVED
        │
        └──▶ (setelah semua item selesai) → BILL_REQUESTED → PAID → TABLE_CLOSED
```

Status kitchen order berjalan paralel dengan status open bill, namun keduanya saling terhubung — open bill tidak dapat ditutup (`TABLE_CLOSED`) sebelum pembayaran diterima (`PAID`), sejalan dengan BRULE-FB-01.

---

## 8. Transaksi Domain Apotek

### 8.1 Validasi Resep

```
PENDING_VALIDATION → VALIDATED (oleh Apoteker) → dapat dilanjutkan ke transaksi penjualan
                │
                └──▶ REJECTED (resep tidak valid, item tidak dapat dijual)
```

Transaksi penjualan dengan item resep tidak dapat berstatus `COMPLETED` sebelum seluruh item terkait memiliki status validasi `VALIDATED` (BRULE-PH-01).

---

## 9. Ringkasan Relasi Antar Jenis Transaksi

```
Purchase Order ──▶ Goods Receipt ──▶ Stock Ledger (stok masuk)
                                            │
                                            ▼
Sales Transaction ──▶ Stock Ledger (stok keluar) ──▶ Return (jika ada)
        │
        ▼
Shift Reconciliation (kas)
        │
        ▼
Approval (jika ada aksi sensitif: void, diskon, adjustment)
```

**Alasan bisnis:** Seluruh jenis transaksi pada akhirnya bermuara pada dua hal yang paling dijaga integritasnya di bisnis retail: **stok** dan **kas** — struktur relasi ini memastikan keduanya selalu dapat direkonsiliasi kembali ke transaksi sumbernya.

---

## 10. Traceability

| Kategori Transaksi | Terkait Dokumen |
|---|---|
| Struktur data transaksi | [`database-design.md`](../technical/database-design.md), [`erd.md`](../technical/erd.md) |
| Aturan bisnis transaksi | [`business-rules.md`](business-rules.md) |
| Validasi transaksi | [`validation.md`](validation.md) |
| Skenario tidak umum | [`edge-cases.md`](edge-cases.md) |
| Endpoint transaksi | [`api-contract.md`](../technical/api-contract.md) |
| Implementasi service | [`service-layer-design.md`](../technical/service-layer-design.md) |

---

**Sebelumnya:** [`master-data.md`](master-data.md) — Master Data
**Selanjutnya:** [`../technical/README.md`](../technical/README.md) — Index Dokumentasi Teknis

---

> 🎉 **Business Documentation selesai.** Seluruh 16 dokumen bisnis (README hingga Transactions) telah tersusun secara konsisten dan saling terhubung. Dokumentasi teknis akan dimulai dari [`docs/technical/README.md`](../technical/README.md) pada tahap berikutnya.
