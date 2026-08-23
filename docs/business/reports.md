# Reports — POS

> Dokumen ini mendaftar seluruh **laporan (reports)** yang dibutuhkan bisnis, berbeda dengan [`dashboard.md`](dashboard.md) yang bersifat ringkasan real-time interaktif — laporan pada dokumen ini bersifat **terstruktur, dapat difilter/diekspor, dan digunakan untuk analisis mendalam maupun kebutuhan formal (audit, rekonsiliasi, evaluasi manajemen)**.

---

## 1. Prinsip Desain Laporan

1. **Setiap laporan memiliki tujuan bisnis yang jelas** — bukan laporan yang dibuat "karena biasanya ada di sistem POS lain".
2. **Laporan harus dapat difilter** minimal berdasarkan: periode, cabang (jika berlaku), dan dimensi spesifik laporan (produk, kasir, pelanggan, dsb.).
3. **Laporan harus dapat diekspor** ke format umum (PDF untuk laporan formal, Excel/CSV untuk analisis lanjutan) sesuai FR-DB-05.
4. **Laporan besar diproses secara asynchronous** (background job) sesuai NFR-PF-04 agar tidak membebani sistem transaksional secara langsung.

---

## 2. Kategori Laporan Penjualan

| Nama Laporan | Deskripsi | Filter Utama | Konsumen Utama |
|---|---|---|---|
| Laporan Penjualan Harian | Rekap transaksi per hari, per cabang | Tanggal, cabang | Branch Manager, Owner |
| Laporan Penjualan per Produk | Kuantitas & omzet per produk dalam periode tertentu | Periode, kategori produk, cabang | Owner, HO Admin |
| Laporan Penjualan per Kasir | Performa transaksi per kasir (jumlah transaksi, omzet, rata-rata) | Periode, cabang, kasir | Branch Manager, Supervisor |
| Laporan Penjualan per Kategori | Kontribusi tiap kategori produk terhadap total omzet | Periode, cabang | Owner |
| Laporan Metode Pembayaran | Distribusi transaksi berdasarkan metode pembayaran (tunai, kartu, dsb.) | Periode, cabang | Finance Auditor, Branch Manager |
| Laporan Void & Diskon Manual | Rekap seluruh void dan diskon manual beserta approver-nya | Periode, cabang, kasir | Supervisor, Finance Auditor |
| Laporan Retur Penjualan | Rekap retur beserta alasan | Periode, cabang, produk | Branch Manager |

**Alasan bisnis:** Laporan Void & Diskon Manual secara khusus dipisahkan (bukan digabung ke laporan penjualan umum) karena fungsinya sebagai **alat kontrol anti-fraud**, bukan sekadar laporan performa — sejalan dengan BR-04 pada [`business-requirements.md`](business-requirements.md).

---

## 3. Kategori Laporan Stok & Inventori

| Nama Laporan | Deskripsi | Filter Utama | Konsumen Utama |
|---|---|---|---|
| Laporan Stok Saat Ini | Posisi stok terkini per produk per cabang | Cabang, kategori produk | Branch Manager, Staff Gudang |
| Laporan Mutasi Stok (Kartu Stok) | Histori pergerakan stok (masuk, keluar, transfer, penyesuaian) per produk | Periode, produk, cabang | Staff Gudang, Finance Auditor |
| Laporan Nilai Persediaan | Total nilai stok berdasarkan harga beli (untuk kebutuhan estimasi aset) | Cabang, kategori | Owner, Finance Auditor |
| Laporan Stok Opname | Hasil dan selisih tiap sesi stok opname | Periode, cabang | Branch Manager, Finance Auditor |
| Laporan Produk Mendekati/Sudah Kadaluarsa | Daftar produk berdasarkan batch & tanggal kadaluarsa | Cabang, rentang hari mendekati expired | Branch Manager, Apoteker |
| Laporan Transfer Stok Antar Cabang | Rekap transfer beserta status (dikirim/diterima) | Periode, cabang asal/tujuan | Staff Gudang, HO Admin |

---

## 4. Kategori Laporan Pembelian

| Nama Laporan | Deskripsi | Filter Utama | Konsumen Utama |
|---|---|---|---|
| Laporan Purchase Order | Status seluruh PO (draft, dikirim, diterima sebagian/penuh) | Periode, supplier, cabang | Branch Manager, Purchasing |
| Laporan Pembelian per Supplier | Total nilai pembelian per supplier dalam periode | Periode, supplier | HO Admin, Owner |
| Laporan Retur Pembelian | Rekap retur ke supplier beserta alasan | Periode, supplier | Purchasing |

---

## 5. Kategori Laporan Keuangan

| Nama Laporan | Deskripsi | Filter Utama | Konsumen Utama |
|---|---|---|---|
| Laporan Rekap Kas Harian | Rekonsiliasi kas seluruh shift dalam satu hari per cabang | Tanggal, cabang | Branch Manager, Finance Auditor |
| Laporan Laba Kotor | Estimasi laba kotor (omzet − HPP) per periode | Periode, cabang, kategori | Owner, Finance Auditor |
| Laporan Piutang Pelanggan (Distributor) | Status piutang per pelanggan, termasuk yang jatuh tempo | Periode, pelanggan, status | Finance Auditor, Branch Manager |
| Laporan Selisih Kas | Rekap seluruh selisih kas saat tutup shift beserta status approval | Periode, cabang, kasir | Finance Auditor |

---

## 6. Kategori Laporan Konsolidasi (Multi-Cabang)

| Nama Laporan | Deskripsi | Filter Utama | Konsumen Utama |
|---|---|---|---|
| Laporan Perbandingan Kinerja Antar Cabang | Perbandingan omzet, jumlah transaksi, rata-rata nilai transaksi antar cabang | Periode | Owner |
| Laporan Konsolidasi Penjualan Perusahaan | Total penjualan seluruh cabang tergabung menjadi satu laporan | Periode | Owner, HO Admin |
| Laporan Anomali Operasional | Cabang dengan tingkat void/retur/selisih kas di atas ambang batas normal | Periode | Owner, Finance Auditor |

---

## 7. Kategori Laporan Domain Khusus

| Nama Laporan | Domain | Deskripsi |
|---|---|---|
| Laporan Penggunaan Meja & Waktu Layanan | F&B | Rata-rata waktu meja terisi, tingkat perputaran meja |
| Laporan Item Menu Terlaris | F&B | Item menu dengan penjualan tertinggi, termasuk modifier populer |
| Laporan Validasi Resep | Apotek | Rekap transaksi obat resep beserta validasi Apoteker |
| Laporan Penjualan per Kelas Pelanggan | Distributor | Perbandingan omzet berdasarkan tingkatan/kelas pelanggan (tiered pricing) |

---

## 8. Format & Mekanisme Ekspor

| Format | Kegunaan |
|---|---|
| PDF | Laporan formal untuk kebutuhan cetak/arsip (misal laporan bulanan ke Owner) |
| Excel/CSV | Laporan untuk analisis lanjutan atau diimpor ke tools lain (misal Excel pivot table) |

**Alasan teknis:** Laporan dengan volume data besar (misal laporan mutasi stok tahunan) diproses melalui **background job** dan notifikasi dikirim saat file siap diunduh — mendukung NFR-PF-04 agar proses generasi laporan tidak memblokir performa sistem transaksional.

---

## 9. Ringkasan Pemetaan Laporan ke Role

| Role | Laporan Utama yang Diakses |
|---|---|
| Owner | Seluruh laporan konsolidasi & keuangan tingkat tinggi |
| HO Admin | Laporan pembelian, penjualan produk, konsolidasi |
| Branch Manager | Laporan operasional cabang (penjualan, stok, kas) |
| Supervisor | Laporan void & diskon manual |
| Staff Gudang | Laporan stok, mutasi, transfer |
| Finance Auditor | Seluruh laporan keuangan & audit |
| Apoteker | Laporan validasi resep, expired produk |

---

## 10. Traceability

| Kategori Laporan | Terkait Dokumen |
|---|---|
| Struktur data sumber laporan | [`database-design.md`](../technical/database-design.md) |
| Endpoint laporan | [`api-contract.md`](../technical/api-contract.md) |
| Kebutuhan performa laporan besar | [`non-functional-requirements.md`](non-functional-requirements.md#2-performa-performance) |
| Data yang mendasari laporan transaksi | [`transactions.md`](transactions.md) |

---

**Sebelumnya:** [`dashboard.md`](dashboard.md) — Dashboard
**Selanjutnya:** [`notifications.md`](notifications.md) — Notifications
