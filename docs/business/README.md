# Business Documentation — POS

> Dokumen ini adalah pintu masuk (index) untuk seluruh dokumentasi bisnis project POS. Ditujukan bagi Business Analyst, Product Owner, Stakeholder, dan Engineer yang ingin memahami **konteks bisnis** sebelum masuk ke detail teknis.

---

## 1. Tujuan Dokumentasi Bisnis

Dokumentasi bisnis pada project ini disusun untuk menjawab lima pertanyaan mendasar:

1. **Siapa** yang menggunakan sistem ini, dan dalam peran apa?
2. **Apa** proses bisnis nyata yang terjadi di lapangan (toko, gudang, kasir, kantor pusat)?
3. **Aturan** apa yang harus dipatuhi sistem agar sesuai dengan realita operasional dan regulasi?
4. **Bagaimana** proses tersebut diterjemahkan menjadi fitur (requirement) yang bisa diukur dan diverifikasi?
5. **Mengapa** setiap fitur/modul itu perlu ada — bukan sekadar "karena aplikasi POS pada umumnya begitu"?

Setiap dokumen turunan akan secara konsisten mengacu pada prinsip ini.

---

## 2. Daftar Isi Dokumentasi Bisnis

| No | Dokumen | Deskripsi Singkat |
|---|---|---|
| 1 | [`business-requirements.md`](business-requirements.md) | Kebutuhan bisnis tingkat tinggi — masalah yang ingin diselesaikan, tujuan strategis, ruang lingkup |
| 2 | [`functional-requirements.md`](functional-requirements.md) | Daftar fitur fungsional per modul, dipetakan ke kebutuhan bisnis |
| 3 | [`non-functional-requirements.md`](non-functional-requirements.md) | Kebutuhan non-fungsional: performa, skalabilitas, keamanan, ketersediaan |
| 4 | [`business-flow.md`](business-flow.md) | Alur proses bisnis end-to-end per domain (retail, F&B, distributor, apotek, dll) |
| 5 | [`sop.md`](sop.md) | Standard Operating Procedure operasional harian per peran |
| 6 | [`use-case.md`](use-case.md) | Use case per aktor, termasuk skenario utama dan alternatif |
| 7 | [`role-permission.md`](role-permission.md) | Definisi role, hak akses, dan matriks permission |
| 8 | [`business-rules.md`](business-rules.md) | Aturan bisnis eksplisit yang harus ditegakkan sistem |
| 9 | [`approval-flow.md`](approval-flow.md) | Alur persetujuan (approval) untuk transaksi/aksi sensitif |
| 10 | [`dashboard.md`](dashboard.md) | Kebutuhan dashboard & KPI untuk masing-masing peran |
| 11 | [`reports.md`](reports.md) | Daftar laporan yang dibutuhkan bisnis beserta tujuannya |
| 12 | [`notifications.md`](notifications.md) | Kebutuhan notifikasi sistem — pemicu, penerima, kanal |
| 13 | [`validation.md`](validation.md) | Aturan validasi data dari sudut pandang bisnis |
| 14 | [`edge-cases.md`](edge-cases.md) | Skenario kasus khusus/tidak umum yang harus diantisipasi |
| 15 | [`master-data.md`](master-data.md) | Definisi dan tata kelola data master |
| 16 | [`transactions.md`](transactions.md) | Definisi dan siklus hidup seluruh jenis transaksi |

---

## 3. Ruang Lingkup Bisnis (Business Scope)

### 3.1 Model Operasional yang Didukung

Sistem dirancang untuk perusahaan dengan struktur:

```
Head Office (Kantor Pusat)
    │
    ├── Branch A (Cabang Jakarta)
    │      ├── Outlet A1
    │      └── Outlet A2
    ├── Branch B (Cabang Surabaya)
    │      └── Outlet B1
    └── Warehouse (Gudang Pusat)
```

- **Head Office**: mengatur kebijakan harga, produk master, laporan konsolidasi seluruh cabang.
- **Branch/Outlet**: unit operasional yang melakukan transaksi penjualan harian, memiliki stok lokal.
- **Warehouse**: pusat distribusi stok ke cabang-cabang.

Alasan bisnis: perusahaan retail skala besar **tidak pernah beroperasi sebagai satu toko tunggal**. Kebutuhan konsolidasi data lintas cabang, transfer stok antar cabang, dan kontrol harga terpusat adalah kebutuhan riil sejak hari pertama — sehingga arsitektur multi-branch **tidak boleh menjadi fitur tambahan di kemudian hari**, melainkan pondasi sejak awal.

### 3.2 Aktor Bisnis Utama

| Aktor | Peran Umum |
|---|---|
| **Owner / Direktur** | Pemilik bisnis, melihat laporan konsolidasi, menentukan kebijakan besar |
| **Head Office Admin** | Mengelola data master, harga, promo, approval tingkat pusat |
| **Branch Manager** | Mengelola operasional cabang, approval tingkat cabang, laporan cabang |
| **Supervisor / Shift Leader** | Mengawasi kasir, approval transaksi tertentu (void, diskon manual) |
| **Kasir (Cashier)** | Melakukan transaksi penjualan harian |
| **Staff Gudang (Warehouse Staff)** | Mengelola stok, penerimaan barang, transfer stok |
| **Staff Purchasing** | Membuat pesanan pembelian ke supplier |
| **Apoteker (khusus domain Apotek)** | Validasi resep, kontrol obat keras |
| **Waiter/Kitchen Staff (khusus domain F&B)** | Input pesanan meja, mengelola dapur |
| **Auditor/Finance** | Rekonsiliasi keuangan, audit transaksi |

Detail hak akses tiap aktor dijelaskan pada [`role-permission.md`](role-permission.md).

### 3.3 Lingkup Modul Bisnis

| Kategori Modul | Cakupan |
|---|---|
| **Master Data** | Produk, kategori, satuan, harga, pelanggan, supplier, cabang, pegawai |
| **Transaksi Penjualan** | Penjualan tunai/non-tunai, retur, void, open bill (F&B) |
| **Transaksi Pembelian** | Purchase order, penerimaan barang, retur ke supplier |
| **Manajemen Stok** | Stok opname, transfer antar cabang, penyesuaian stok, batch/expired |
| **Keuangan** | Rekonsiliasi kas, laporan laba rugi sederhana, piutang/hutang |
| **Promosi & Loyalti** | Diskon, promo bundling, member/loyalty point |
| **Approval & Kontrol** | Approval diskon manual, void transaksi, penyesuaian stok |
| **Laporan & Dashboard** | Laporan penjualan, stok, keuangan; dashboard real-time per peran |
| **Notifikasi** | Stok menipis, approval pending, shift belum ditutup |

---

## 4. Batasan (Out of Scope)

Untuk menjaga fokus sebagai portfolio *production ready* namun tetap terarah, hal berikut **secara sadar tidak** dimasukkan ke dalam scope v1, dan akan disebutkan kembali di dokumen terkait bila relevan:

- Integrasi payment gateway pihak ketiga secara live (didesain agar *extensible*, namun implementasi awal menggunakan simulasi/manual reconciliation).
- Modul akuntansi penuh (general ledger lengkap) — sistem hanya menyediakan data keuangan dasar yang dapat diekspor/diintegrasikan ke sistem akuntansi.
- Marketplace/e-commerce integration.

Keputusan ini adalah keputusan *scope*, bukan keputusan *kualitas* — seluruh modul yang masuk ke dalam scope tetap dirancang dengan standar produksi penuh, tanpa penyederhanaan (no MVP shortcut).

---

## 5. Keterkaitan dengan Dokumentasi Teknis

Setiap dokumen bisnis memiliki pasangan teknis yang menjelaskan implementasinya:

| Dokumen Bisnis | Terkait Dokumen Teknis |
|---|---|
| `business-flow.md`, `use-case.md` | `module-design.md`, `service-layer-design.md` |
| `role-permission.md` | `authorization-design.md` |
| `master-data.md`, `transactions.md` | `database-design.md`, `erd.md` |
| `approval-flow.md` | `module-design.md`, `authorization-design.md` |
| `validation.md`, `edge-cases.md` | `api-contract.md`, `service-layer-design.md` |
| `reports.md`, `dashboard.md` | `database-design.md`, `api-contract.md` |
| `notifications.md` | `module-design.md`, `logging-strategy.md` |

---

**Sebelumnya:** [`../../README.md`](../../README.md) — Overview project
**Selanjutnya:** [`business-requirements.md`](business-requirements.md) — Business Requirements
