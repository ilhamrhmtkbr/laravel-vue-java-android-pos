# Business Requirements — POS

> Dokumen ini menjelaskan **kebutuhan bisnis tingkat tinggi (high-level business requirements)** yang melatarbelakangi pembangunan sistem POS ini. Dokumen ini adalah induk dari seluruh requirement teknis dan fungsional — setiap fitur yang dibangun harus dapat ditelusuri kembali (traceable) ke salah satu kebutuhan bisnis di sini.

---

## 1. Latar Belakang Masalah (Business Problem)

Perusahaan retail dengan banyak cabang — baik itu jaringan minimarket, restoran, apotek, maupun distributor — pada umumnya menghadapi masalah operasional berikut ketika masih menggunakan sistem kasir konvensional/terpisah per-cabang:

| Masalah | Dampak Bisnis |
|---|---|
| Data penjualan tiap cabang tidak terkonsolidasi secara real-time | Owner tidak dapat mengambil keputusan cepat berbasis data aktual |
| Harga & promo dikelola manual per cabang | Inkonsistensi harga antar cabang, potensi kerugian/kecurangan |
| Stok tidak sinkron antar cabang dan gudang pusat | Stockout di satu cabang, overstock di cabang lain |
| Tidak ada kontrol approval untuk transaksi sensitif (void, diskon manual) | Risiko kecurangan (fraud) kasir tinggi |
| Laporan disusun manual dari banyak sumber | Lambat, rawan human error, keputusan bisnis terlambat |
| Tidak ada audit trail perubahan data penting | Sulit menelusuri penyebab selisih stok/kas saat terjadi masalah |
| Sistem tidak mendukung karakteristik khusus per jenis usaha (F&B, apotek, distributor) | Perusahaan harus pakai banyak sistem berbeda yang tidak terintegrasi |

**Kesimpulan masalah inti:** perusahaan retail multi-cabang membutuhkan **satu platform terpusat** yang dapat mengakomodasi operasional harian di level cabang, sekaligus memberikan visibilitas dan kontrol penuh di level pusat — dengan fleksibilitas mendukung berbagai jenis model bisnis retail.

---

## 2. Tujuan Bisnis (Business Goals)

| ID | Tujuan Bisnis | Indikator Keberhasilan (Success Metric) |
|---|---|---|
| BG-01 | Konsolidasi data penjualan seluruh cabang secara real-time | Owner dapat melihat laporan penjualan seluruh cabang tanpa jeda manual |
| BG-02 | Standarisasi & kontrol harga terpusat | Perubahan harga di pusat otomatis berlaku ke seluruh cabang yang dituju |
| BG-03 | Visibilitas stok lintas cabang & gudang | Stok setiap cabang dapat dipantau dari pusat, transfer stok terekam sistem |
| BG-04 | Mengurangi risiko kecurangan transaksi | Setiap aksi sensitif (void, diskon manual, retur) memerlukan approval & tercatat |
| BG-05 | Mempercepat proses pelaporan manajemen | Laporan tersedia otomatis, tidak perlu rekap manual |
| BG-06 | Mendukung berbagai model bisnis retail dalam satu platform | Sistem dapat dikonfigurasi untuk minimarket, F&B, apotek, distributor tanpa mengubah arsitektur inti |
| BG-07 | Menjaga integritas data finansial & operasional | Setiap perubahan data penting memiliki audit trail (siapa, kapan, apa) |

---

## 3. Ruang Lingkup Bisnis (Business Scope)

### 3.1 Termasuk dalam Scope

- Manajemen data master (produk, kategori, harga, pelanggan, supplier, cabang, pegawai).
- Transaksi penjualan (kasir) dengan berbagai metode pembayaran.
- Transaksi pembelian dari supplier.
- Manajemen stok multi-cabang termasuk transfer dan stok opname.
- Approval workflow untuk transaksi sensitif.
- Promosi, diskon, dan program loyalitas pelanggan.
- Dashboard dan pelaporan manajemen (operasional & finansial dasar).
- Notifikasi sistem untuk kejadian penting operasional.
- Dukungan karakteristik khusus per domain (F&B: meja & open bill; Apotek: batch/expired & resep; Distributor: piutang & tiered pricing).

### 3.2 Tidak Termasuk dalam Scope (Out of Scope)

- Integrasi payment gateway pihak ketiga secara live/produksi (v1 didesain *extensible* untuk itu, namun tidak diimplementasikan penuh).
- Modul akuntansi/general ledger lengkap (hanya data dasar untuk diekspor ke sistem akuntansi eksternal).
- Integrasi marketplace/e-commerce.
- Manajemen HR & payroll pegawai (di luar data pegawai untuk kebutuhan akses sistem).

> Referensi silang: lihat juga [`docs/business/README.md`](README.md#4-batasan-out-of-scope) untuk konteks batasan scope secara umum.

---

## 4. Pemangku Kepentingan (Stakeholders)

| Stakeholder | Kepentingan Utama |
|---|---|
| Owner / Direktur | Visibilitas performa bisnis, kontrol strategis |
| Head Office Admin | Efisiensi pengelolaan data master & kebijakan pusat |
| Branch Manager | Kelancaran operasional cabang, kontrol staf |
| Kasir | Kemudahan & kecepatan transaksi harian |
| Staff Gudang/Purchasing | Akurasi stok, kemudahan proses pembelian & distribusi |
| Auditor/Finance | Keandalan data untuk rekonsiliasi dan audit |
| Tim Engineering (internal) | Sistem yang scalable, maintainable, dan dapat dikembangkan lanjut |

---

## 5. Kebutuhan Bisnis Utama (Key Business Requirements)

Setiap requirement berikut diberi ID unik (`BR-xxx`) agar dapat ditelusuri (*traceability*) ke functional requirement dan modul teknis terkait.

| ID | Kebutuhan Bisnis | Justifikasi |
|---|---|---|
| BR-01 | Sistem harus mendukung struktur organisasi Head Office → Branch → Outlet | Perusahaan besar tidak beroperasi sebagai entitas tunggal; laporan dan kontrol harus berjenjang |
| BR-02 | Sistem harus mampu melakukan transaksi penjualan walau koneksi ke pusat terganggu (local resilience di level arsitektur, didetailkan di dokumen teknis) | Operasional toko tidak boleh berhenti karena masalah jaringan/pusat |
| BR-03 | Setiap perubahan harga, stok, dan data master harus tercatat siapa & kapan melakukannya | Kebutuhan audit dan investigasi jika terjadi selisih |
| BR-04 | Transaksi sensitif (void, retur, diskon manual di luar batas) wajib melalui approval berjenjang | Mitigasi risiko kecurangan kasir/karyawan |
| BR-05 | Sistem harus menyediakan laporan konsolidasi lintas cabang secara real-time | Kecepatan pengambilan keputusan manajemen |
| BR-06 | Sistem harus mendukung berbagai metode pembayaran (tunai, kartu, e-wallet, transfer) | Karakteristik pembayaran retail modern bersifat multi-channel |
| BR-07 | Sistem harus mendukung konfigurasi per-domain bisnis (retail, F&B, apotek, distributor) tanpa mengubah struktur inti | Satu platform dapat digunakan lintas jenis usaha, mengurangi biaya pengembangan berulang |
| BR-08 | Sistem harus memberi notifikasi proaktif atas kejadian penting (stok menipis, approval pending, shift belum ditutup) | Mencegah masalah operasional terlambat diketahui |
| BR-09 | Sistem harus mendukung program loyalitas & promosi yang fleksibel | Retensi pelanggan adalah kebutuhan kompetitif di industri retail |
| BR-10 | Data pelanggan, transaksi, dan finansial harus dilindungi sesuai prinsip keamanan data | Kepatuhan terhadap ekspektasi perlindungan data pelanggan & transaksi finansial |

---

## 6. Asumsi (Assumptions)

- Setiap cabang memiliki koneksi internet, namun sistem tetap harus tangguh terhadap gangguan jaringan sementara.
- Pegawai memiliki akun individu (tidak berbagi akun) untuk menjaga akurasi audit trail.
- Perusahaan menggunakan struktur mata uang tunggal (single currency) pada versi awal.
- Kebijakan pajak (misal PPN) mengikuti regulasi umum yang berlaku dan dapat dikonfigurasi per produk/kategori.

## 7. Batasan (Constraints)

- Sistem harus dapat berjalan di infrastruktur cloud (AWS EC2) dengan biaya operasional yang wajar untuk skala UMKM hingga enterprise.
- Sistem harus dapat diakses melalui web (back-office & kasir berbasis web) dan aplikasi mobile Android (kasir/operasional lapangan).
- Dukungan perangkat keras kasir (printer struk, cash drawer, barcode scanner) dilakukan melalui integrasi standar (ESC/POS, USB/Bluetooth) — detail pada dokumentasi teknis.

## 8. Risiko Bisnis (Business Risks)

| Risiko | Mitigasi |
|---|---|
| Resistensi pengguna (kasir/staff) terhadap sistem baru | Desain UX kasir dibuat sesederhana mungkin, mendekati alur kasir konvensional |
| Kehilangan data akibat gangguan sistem | Strategi backup rutin (lihat [`backup-strategy.md`](../technical/backup-strategy.md)) |
| Kecurangan internal (fraud) | Approval workflow berjenjang dan audit trail menyeluruh |
| Kompleksitas multi-domain menyebabkan sistem terlalu generik dan sulit dipakai | Modul inti generik, namun setiap domain memiliki konfigurasi/ekstensi spesifik (lihat [`business-flow.md`](business-flow.md)) |

---

## 9. Traceability ke Dokumen Lain

| Business Requirement | Diturunkan Menjadi |
|---|---|
| BR-01 | [`master-data.md`](master-data.md) — struktur cabang; [`database-design.md`](../technical/database-design.md) |
| BR-02 | [`system-architecture.md`](../technical/system-architecture.md) |
| BR-03 | [`logging-strategy.md`](../technical/logging-strategy.md) |
| BR-04 | [`approval-flow.md`](approval-flow.md); [`authorization-design.md`](../technical/authorization-design.md) |
| BR-05 | [`dashboard.md`](dashboard.md); [`reports.md`](reports.md) |
| BR-06 | [`transactions.md`](transactions.md) |
| BR-07 | [`business-flow.md`](business-flow.md); [`module-design.md`](../technical/module-design.md) |
| BR-08 | [`notifications.md`](notifications.md) |
| BR-09 | [`business-rules.md`](business-rules.md) |
| BR-10 | [`security-design.md`](../technical/security-design.md) |

---

**Sebelumnya:** [`README.md`](README.md) — Index Dokumentasi Bisnis
**Selanjutnya:** [`functional-requirements.md`](functional-requirements.md) — Functional Requirements
