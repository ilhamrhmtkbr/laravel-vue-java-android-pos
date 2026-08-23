# Role & Permission — POS

> Dokumen ini mendefinisikan **role (peran)** pengguna sistem beserta **hak akses (permission)** yang melekat pada masing-masing role. Dokumen ini menjadi acuan langsung bagi [`authorization-design.md`](../technical/authorization-design.md) yang akan menjelaskan implementasi teknisnya (RBAC, middleware, policy).

---

## 1. Prinsip Desain Role & Permission

1. **Role bersifat granular, bukan biner (admin/user).** Setiap role memiliki cakupan tanggung jawab spesifik sesuai struktur organisasi retail nyata.
2. **Permission dipisah dari role secara konseptual.** Role adalah *kumpulan* permission — sehingga di masa depan dapat dibuat role kustom tanpa mengubah struktur inti sistem (lihat detail teknis di [`authorization-design.md`](../technical/authorization-design.md)).
3. **Data scoping mengikuti cabang.** Sebagian besar role hanya dapat mengakses data pada cabang tempat mereka ditempatkan, kecuali role level pusat.
4. **Prinsip least privilege.** Setiap role hanya diberi akses yang benar-benar dibutuhkan untuk menjalankan tugasnya — bukan akses penuh secara default.

---

## 2. Daftar Role

| Role | Level | Cakupan Data | Deskripsi Singkat |
|---|---|---|---|
| `owner` | Pusat | Seluruh cabang | Pemilik bisnis, akses penuh ke laporan & konfigurasi strategis |
| `ho_admin` | Pusat | Seluruh cabang | Mengelola data master, harga, promo tingkat pusat |
| `branch_manager` | Cabang | Cabang sendiri | Mengelola operasional & approval tingkat cabang |
| `supervisor` | Cabang | Cabang sendiri | Approval transaksi sensitif tingkat shift |
| `cashier` | Outlet | Shift sendiri (input), cabang sendiri (baca terbatas) | Transaksi penjualan harian |
| `warehouse_staff` | Cabang/Gudang | Cabang/gudang sendiri | Penerimaan barang, transfer stok, stok opname |
| `purchasing_staff` | Cabang/Pusat | Sesuai penempatan | Membuat & mengelola Purchase Order |
| `apoteker` | Outlet | Cabang sendiri | Validasi resep & kontrol obat (domain Apotek) |
| `waiter` | Outlet | Cabang sendiri | Input pesanan meja (domain F&B) |
| `kitchen_staff` | Outlet | Cabang sendiri | Mengelola status pesanan dapur (domain F&B) |
| `finance_auditor` | Pusat | Seluruh cabang (baca saja) | Rekonsiliasi & audit keuangan, akses baca menyeluruh |

**Alasan bisnis:** Struktur role ini mencerminkan hierarki organisasi retail sesungguhnya (lihat [`docs/business/README.md`](README.md#33-lingkup-modul-bisnis)) — role tidak dibuat generik ("admin", "user") karena tanggung jawab dan risiko tiap posisi berbeda signifikan.

---

## 3. Kategori Permission

Permission dikelompokkan berdasarkan modul, mengikuti struktur pada [`functional-requirements.md`](functional-requirements.md). Format permission: `[modul].[aksi]`.

| Kategori Aksi | Contoh |
|---|---|
| `view` | Melihat data (read-only) |
| `create` | Membuat data baru |
| `update` | Mengubah data yang ada |
| `delete` | Menghapus data |
| `approve` | Menyetujui aksi/permintaan tertentu |
| `void` | Membatalkan transaksi |
| `export` | Mengekspor laporan |

---

## 4. Matriks Role vs Permission

Legenda: ✅ = memiliki akses penuh, 🔶 = akses terbatas/dengan syarat, ➖ = tidak memiliki akses

### 4.1 Modul Master Data

| Permission | Owner | HO Admin | Branch Mgr | Supervisor | Kasir | W.Gudang | Purchasing |
|---|---|---|---|---|---|---|---|
| `product.view` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `product.create` | ✅ | ✅ | ➖ | ➖ | ➖ | ➖ | ➖ |
| `product.update` | ✅ | ✅ | 🔶 (stok lokal) | ➖ | ➖ | ➖ | ➖ |
| `price.update` | ✅ | ✅ | ➖ | ➖ | ➖ | ➖ | ➖ |
| `branch.manage` | ✅ | ✅ | ➖ | ➖ | ➖ | ➖ | ➖ |
| `customer.manage` | ✅ | ✅ | ✅ | 🔶 | 🔶 (buat baru) | ➖ | ➖ |
| `supplier.manage` | ✅ | ✅ | 🔶 | ➖ | ➖ | ➖ | 🔶 |

### 4.2 Modul Transaksi Penjualan

| Permission | Owner | Branch Mgr | Supervisor | Kasir |
|---|---|---|---|---|
| `sale.create` | ➖ | ➖ | ➖ | ✅ |
| `sale.view` | ✅ (semua cabang) | ✅ (cabang sendiri) | ✅ (cabang sendiri) | 🔶 (shift sendiri) |
| `sale.void` | ➖ | ✅ | ✅ | 🔶 (mengajukan, bukan menyetujui) |
| `sale.discount_manual` | ➖ | ✅ | ✅ | 🔶 (dalam batas threshold) |
| `sale.return` | ➖ | ✅ | ✅ | 🔶 (mengajukan) |

### 4.3 Modul Stok & Pembelian

| Permission | Owner | Branch Mgr | W.Gudang | Purchasing |
|---|---|---|---|---|
| `purchase_order.create` | ➖ | ✅ | ➖ | ✅ |
| `goods_receipt.create` | ➖ | 🔶 | ✅ | ➖ |
| `stock.transfer` | ➖ | ✅ (approval) | ✅ (eksekusi) | ➖ |
| `stock.opname` | ➖ | ✅ (approval selisih) | ✅ (eksekusi) | ➖ |
| `stock.adjust` | ➖ | ✅ (approval) | 🔶 (mengajukan) | ➖ |

### 4.4 Modul Approval

| Permission | Owner | Branch Mgr | Supervisor |
|---|---|---|---|
| `approval.view` | ✅ | ✅ (cabang sendiri) | ✅ (cabang sendiri) |
| `approval.decide` | ✅ | ✅ | ✅ (batas tertentu, lihat [`approval-flow.md`](approval-flow.md)) |

### 4.5 Modul Laporan & Dashboard

| Permission | Owner | HO Admin | Branch Mgr | Finance Auditor |
|---|---|---|---|---|
| `report.view_consolidated` | ✅ | ✅ | ➖ | ✅ |
| `report.view_branch` | ✅ | ✅ | ✅ (cabang sendiri) | ✅ |
| `report.export` | ✅ | ✅ | ✅ | ✅ |
| `dashboard.view` | ✅ | ✅ | ✅ | ✅ |

### 4.6 Modul Khusus Domain

| Permission | Apoteker | Waiter | Kitchen Staff |
|---|---|---|---|
| `prescription.validate` | ✅ | ➖ | ➖ |
| `table_order.create` | ➖ | ✅ | ➖ |
| `kitchen_order.update_status` | ➖ | ➖ | ✅ |

---

## 5. Aturan Data Scoping

| Role | Scoping Data |
|---|---|
| `owner`, `ho_admin`, `finance_auditor` | Akses seluruh cabang tanpa batasan |
| `branch_manager`, `supervisor` | Hanya data pada cabang tempat mereka terdaftar |
| `cashier`, `warehouse_staff`, `waiter`, `kitchen_staff`, `apoteker` | Hanya data pada outlet/cabang tempat mereka ditempatkan, dan untuk kasir dibatasi lebih lanjut pada shift aktif miliknya untuk operasi tulis |

**Alasan bisnis:** Sesuai BR-10 pada [`business-requirements.md`](business-requirements.md), kebocoran data antar cabang harus dicegah — seorang Branch Manager Cabang Jakarta tidak boleh dapat melihat detail transaksi Cabang Surabaya, kecuali levelnya adalah pusat.

---

## 6. Permission Bertingkat pada Threshold Tertentu

Beberapa permission tidak bersifat "boleh/tidak boleh" mutlak, melainkan **bertingkat berdasarkan nilai transaksi (threshold)** — detail lengkap ada di [`approval-flow.md`](approval-flow.md):

| Aksi | Kasir | Supervisor | Branch Manager |
|---|---|---|---|
| Diskon manual ≤ 10% | Boleh langsung | — | — |
| Diskon manual > 10% s.d. 25% | Perlu approval | Boleh approve | — |
| Diskon manual > 25% | Perlu approval | Perlu eskalasi | Boleh approve |

---

## 7. Traceability

| Role/Permission | Terkait Dokumen |
|---|---|
| Definisi role & tanggung jawab | [`sop.md`](sop.md), [`use-case.md`](use-case.md) |
| Threshold approval | [`approval-flow.md`](approval-flow.md) |
| Implementasi teknis RBAC | [`authorization-design.md`](../technical/authorization-design.md) |
| Autentikasi & sesi | [`authentication-design.md`](../technical/authentication-design.md) |

---

**Sebelumnya:** [`use-case.md`](use-case.md) — Use Case
**Selanjutnya:** [`business-rules.md`](business-rules.md) — Business Rules
