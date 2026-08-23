# Master Data — POS

> Dokumen ini menjelaskan **definisi dan tata kelola data master** — data inti yang menjadi fondasi seluruh transaksi dan pelaporan dalam sistem. Master data bersifat relatif statis dibanding data transaksi, namun perubahannya berdampak luas ke seluruh sistem, sehingga tata kelolanya harus jelas. Dokumen ini menjadi acuan langsung bagi [`database-design.md`](../technical/database-design.md) dan [`erd.md`](../technical/erd.md).

---

## 1. Prinsip Tata Kelola Master Data

1. **Single source of truth** — setiap entitas master data memiliki satu sumber kebenaran (umumnya dikelola di Head Office), cabang tidak membuat data master secara independen tanpa kontrol.
2. **Histori perubahan tercatat** — perubahan pada data master kritis (harga, status aktif produk) harus memiliki jejak audit, bukan hanya nilai terkini.
3. **Data master tidak dihapus secara fisik (soft delete)** — dinonaktifkan (deactivate) untuk menjaga integritas data transaksi historis yang mereferensikannya.
4. **Data master mendukung ekstensi per domain** — beberapa entitas memiliki atribut tambahan tergantung domain bisnis (misal produk apotek memiliki atribut batch/expired, produk F&B memiliki modifier).

---

## 2. Entitas Master Data: Produk

### 2.1 Deskripsi

Produk adalah entitas master paling sentral — hampir seluruh transaksi (penjualan, pembelian, stok) mereferensikan produk.

### 2.2 Atribut Utama

| Atribut | Deskripsi | Wajib? |
|---|---|---|
| Nama Produk | Nama yang ditampilkan ke pengguna | Ya |
| SKU/Barcode | Identifier unik untuk pencarian & scanning | Ya |
| Kategori | Pengelompokan produk untuk laporan dan navigasi | Ya |
| Satuan Dasar | Satuan terkecil transaksi (pcs, kg, dll) | Ya |
| Satuan Konversi | Satuan turunan (misal 1 dus = 12 pcs) dengan rasio konversi | Tidak |
| Harga Jual | Harga default penjualan (dapat berbeda per cabang jika dikonfigurasi) | Ya |
| Harga Beli | Referensi harga beli terakhir dari supplier | Tidak |
| Pajak | Persentase pajak yang berlaku pada produk (jika ada) | Tidak |
| Status | Aktif/Nonaktif | Ya |

### 2.3 Atribut Ekstensi per Domain

| Domain | Atribut Tambahan |
|---|---|
| Apotek | Kategori regulasi (bebas, keras/resep), wajib tracking batch & expired |
| F&B | Modifier (misal level pedas, topping tambahan), waktu estimasi penyajian |
| Distributor | Harga bertingkat per kelas pelanggan (tiered pricing) |

**Alasan bisnis:** Struktur atribut inti + ekstensi domain memungkinkan satu tabel/entitas produk melayani seluruh domain bisnis tanpa duplikasi struktur data, sejalan dengan BR-07 pada [`business-requirements.md`](business-requirements.md).

---

## 3. Entitas Master Data: Kategori & Satuan

| Entitas | Deskripsi | Kepemilikan Data |
|---|---|---|
| Kategori Produk | Pengelompokan hierarkis (kategori & sub-kategori) | Dikelola Head Office |
| Satuan | Daftar satuan yang valid (pcs, kg, liter, dus, dll) beserta aturan konversi | Dikelola Head Office |

---

## 4. Entitas Master Data: Cabang & Outlet

| Atribut | Deskripsi | Wajib? |
|---|---|---|
| Kode Cabang | Identifier unik cabang | Ya |
| Nama Cabang | Nama tampilan cabang | Ya |
| Alamat | Alamat operasional cabang | Ya |
| Induk (Parent) | Referensi ke Head Office atau cabang induk (untuk struktur outlet di bawah branch) | Tidak |
| Status | Aktif/Nonaktif | Ya |

**Alasan bisnis:** Struktur cabang mengikuti hierarki Head Office → Branch → Outlet yang dijelaskan pada [`docs/business/README.md`](README.md#31-model-operasional-yang-didukung) — setiap transaksi dan laporan harus dapat ditelusuri ke cabang spesifik (BRULE-MB-01).

---

## 5. Entitas Master Data: Pelanggan (Customer)

| Atribut | Deskripsi | Wajib? |
|---|---|---|
| Nama | Nama pelanggan | Ya |
| Kontak (telepon/email) | Untuk keperluan komunikasi dan identifikasi member | Ya (untuk member) |
| Tipe Pelanggan | Umum, Member, Reseller/Distributor | Ya |
| Kelas Pelanggan (khusus Distributor) | Menentukan tiered pricing dan limit kredit | Ya (untuk tipe Reseller) |
| Limit Kredit | Batas maksimum piutang yang diizinkan | Ya (untuk pelanggan kredit) |
| Poin Loyalitas | Akumulasi poin dari transaksi (khusus member) | Otomatis dihitung sistem |

---

## 6. Entitas Master Data: Supplier

| Atribut | Deskripsi | Wajib? |
|---|---|---|
| Nama Supplier | Nama perusahaan/individu supplier | Ya |
| Kontak | Informasi komunikasi | Ya |
| Term of Payment | Ketentuan pembayaran (misal net 30 hari, COD) | Ya |
| Status | Aktif/Nonaktif | Ya |

---

## 7. Entitas Master Data: Pegawai (Employee)

| Atribut | Deskripsi | Wajib? |
|---|---|---|
| Nama | Nama pegawai | Ya |
| Akun Login | Kredensial individu (tidak boleh berbagi akun) | Ya |
| Role | Role sesuai [`role-permission.md`](role-permission.md) | Ya |
| Cabang Penempatan | Cabang tempat pegawai bertugas (menentukan data scoping) | Ya (kecuali role level pusat) |
| Status | Aktif/Nonaktif | Ya |

---

## 8. Ringkasan Kepemilikan & Otoritas Perubahan Data Master

| Entitas | Siapa yang Dapat Membuat/Mengubah | Alasan |
|---|---|---|
| Produk, Kategori, Satuan | Head Office Admin (create/update), Branch Manager (stok lokal saja) | Standarisasi produk di seluruh cabang (BRULE-PR-01) |
| Harga | Head Office Admin (harga dasar); cabang tidak dapat mengubah langsung | Kontrol harga terpusat sesuai BG-02 |
| Cabang/Outlet | Head Office Admin | Struktur organisasi adalah keputusan level pusat |
| Pelanggan | Head Office Admin, Branch Manager, Kasir (create dasar) | Pelanggan sering dibuat langsung saat transaksi berlangsung |
| Supplier | Head Office Admin, Branch Manager (terbatas) | Hubungan supplier dapat bersifat lokal maupun terpusat |
| Pegawai | Head Office Admin, Branch Manager (pegawai cabang sendiri) | Pengelolaan SDM operasional di level cabang tetap perlu fleksibilitas |

---

## 9. Siklus Hidup Data Master (Lifecycle)

```
DRAFT/CREATE → ACTIVE → (Opsional: UPDATED, dengan histori tersimpan) → INACTIVE (soft delete)
```

**Catatan penting:** Data master berstatus `INACTIVE` tidak muncul di pencarian transaksi baru, namun tetap dipertahankan di database karena data transaksi historis mereferensikannya (lihat EC-SL-06 pada [`edge-cases.md`](edge-cases.md)).

---

## 10. Traceability

| Master Data | Terkait Dokumen |
|---|---|
| Struktur tabel & relasi | [`database-design.md`](../technical/database-design.md), [`erd.md`](../technical/erd.md) |
| Aturan validasi | [`validation.md`](validation.md) |
| Aturan bisnis terkait | [`business-rules.md`](business-rules.md) |
| Endpoint pengelolaan | [`api-contract.md`](../technical/api-contract.md) |

---

**Sebelumnya:** [`edge-cases.md`](edge-cases.md) — Edge Cases
**Selanjutnya:** [`transactions.md`](transactions.md) — Transactions
