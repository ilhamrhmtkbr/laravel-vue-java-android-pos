# Functional Requirements — POS

> Dokumen ini menerjemahkan [`business-requirements.md`](business-requirements.md) menjadi **kebutuhan fungsional (functional requirements)** yang konkret dan dapat diverifikasi. Setiap requirement diberi ID unik (`FR-xxx`), dikelompokkan per modul, dan ditelusuri kembali (traceable) ke Business Requirement (`BR-xxx`) yang melatarbelakanginya.

---

## 1. Cara Membaca Dokumen Ini

Setiap baris requirement memiliki format:

| Kolom | Keterangan |
|---|---|
| **ID** | Identifier unik, format `FR-[MODUL]-[nomor]` |
| **Requirement** | Pernyataan kebutuhan fungsional, harus dapat diuji (testable) |
| **Prioritas** | `Must Have` / `Should Have` / `Could Have` (mengacu MoSCoW) |
| **Sumber (BR)** | Business Requirement asal |

---

## 2. Modul: Manajemen Data Master

| ID | Requirement | Prioritas | Sumber |
|---|---|---|---|
| FR-MD-01 | Sistem harus dapat mengelola data produk (nama, SKU, kategori, satuan, harga jual, harga beli, pajak) | Must Have | BR-01, BR-07 |
| FR-MD-02 | Sistem harus mendukung produk dengan multi-satuan (misal: 1 dus = 12 pcs) beserta konversi harga otomatis | Must Have | BR-07 |
| FR-MD-03 | Sistem harus mendukung produk dengan varian (misal ukuran, rasa) tanpa duplikasi data produk induk | Should Have | BR-07 |
| FR-MD-04 | Sistem harus dapat mengelola data cabang (branch) dan outlet dengan relasi hierarki ke Head Office | Must Have | BR-01 |
| FR-MD-05 | Sistem harus dapat mengelola data pelanggan (member) termasuk histori transaksi & poin loyalitas | Must Have | BR-09 |
| FR-MD-06 | Sistem harus dapat mengelola data supplier termasuk term of payment | Must Have | BR-01 |
| FR-MD-07 | Sistem harus dapat mengelola data pegawai beserta role dan cabang penempatan | Must Have | BR-03, BR-04 |
| FR-MD-08 | Sistem harus mencatat histori perubahan harga produk (price history) | Must Have | BR-03 |
| FR-MD-09 | Sistem harus mendukung data khusus domain: batch & tanggal kadaluarsa produk (untuk Apotek/Minimarket) | Must Have | BR-07 |
| FR-MD-10 | Sistem harus mendukung struktur menu & modifier (untuk domain F&B) | Must Have | BR-07 |

---

## 3. Modul: Transaksi Penjualan (Sales / POS)

| ID | Requirement | Prioritas | Sumber |
|---|---|---|---|
| FR-SL-01 | Sistem harus dapat memproses transaksi penjualan dengan pemindaian barcode maupun pencarian manual produk | Must Have | BR-06 |
| FR-SL-02 | Sistem harus mendukung multi-metode pembayaran dalam satu transaksi (split payment) | Must Have | BR-06 |
| FR-SL-03 | Sistem harus dapat menghitung otomatis diskon, pajak, dan biaya layanan (service charge) | Must Have | BR-06, BR-09 |
| FR-SL-04 | Sistem harus mendukung transaksi open bill (pesanan belum dibayar, khusus F&B) yang dapat ditambah item sebelum pembayaran final | Must Have | BR-07 |
| FR-SL-05 | Sistem harus mendukung void transaksi/item dengan approval sesuai role | Must Have | BR-04 |
| FR-SL-06 | Sistem harus mendukung retur penjualan dengan alasan retur wajib diisi | Must Have | BR-04 |
| FR-SL-07 | Sistem harus mencetak struk transaksi ke printer thermal (ESC/POS) | Must Have | BR-06 |
| FR-SL-08 | Sistem harus mendukung transaksi tanpa internet (offline) dan sinkronisasi otomatis saat koneksi pulih | Should Have | BR-02 |
| FR-SL-09 | Sistem harus mencatat kasir & shift yang bertanggung jawab atas setiap transaksi | Must Have | BR-03 |
| FR-SL-10 | Sistem harus mendukung pembulatan otomatis sesuai kebijakan pembulatan yang dikonfigurasi | Should Have | BR-06 |
| FR-SL-11 | Sistem harus mendukung penerapan harga bertingkat (tiered pricing) berdasarkan tipe pelanggan (khusus Distributor) | Must Have | BR-07 |
| FR-SL-12 | Sistem harus mendukung transaksi piutang (credit sale) untuk pelanggan tertentu dengan limit kredit (khusus Distributor) | Must Have | BR-07 |

---

## 4. Modul: Transaksi Pembelian (Purchasing)

| ID | Requirement | Prioritas | Sumber |
|---|---|---|---|
| FR-PU-01 | Sistem harus dapat membuat Purchase Order (PO) ke supplier | Must Have | BR-01 |
| FR-PU-02 | Sistem harus mendukung proses penerimaan barang (goods receipt) berdasarkan PO, termasuk penerimaan sebagian (partial receipt) | Must Have | BR-01 |
| FR-PU-03 | Sistem harus otomatis memperbarui stok saat penerimaan barang dikonfirmasi | Must Have | BR-03 |
| FR-PU-04 | Sistem harus mendukung retur pembelian ke supplier | Must Have | BR-04 |
| FR-PU-05 | Sistem harus mencatat histori harga beli per supplier untuk perbandingan | Should Have | BR-03 |

---

## 5. Modul: Manajemen Stok (Inventory)

| ID | Requirement | Prioritas | Sumber |
|---|---|---|---|
| FR-IN-01 | Sistem harus mencatat stok secara real-time per cabang/outlet | Must Have | BR-01, BR-05 |
| FR-IN-02 | Sistem harus mendukung transfer stok antar cabang dengan status pengiriman (dikirim, diterima) | Must Have | BR-01 |
| FR-IN-03 | Sistem harus mendukung stok opname (stock take) dengan pencatatan selisih otomatis | Must Have | BR-03 |
| FR-IN-04 | Sistem harus mendukung penyesuaian stok manual dengan approval dan alasan wajib | Must Have | BR-04 |
| FR-IN-05 | Sistem harus memberi peringatan otomatis saat stok mencapai batas minimum (reorder point) | Must Have | BR-08 |
| FR-IN-06 | Sistem harus melacak stok berdasarkan batch dan tanggal kadaluarsa, serta memperingatkan produk mendekati expired | Must Have | BR-07, BR-08 |
| FR-IN-07 | Sistem harus mencatat mutasi stok (kartu stok) untuk setiap produk sebagai jejak audit | Must Have | BR-03 |

---

## 6. Modul: Approval & Kontrol

| ID | Requirement | Prioritas | Sumber |
|---|---|---|---|
| FR-AP-01 | Sistem harus mendukung alur approval berjenjang (misal: Supervisor → Branch Manager) untuk aksi sensitif | Must Have | BR-04 |
| FR-AP-02 | Sistem harus mencatat status approval (pending, approved, rejected) beserta pemberi approval dan waktunya | Must Have | BR-03, BR-04 |
| FR-AP-03 | Sistem harus mengirim notifikasi ke approver saat ada permintaan approval baru | Must Have | BR-08 |
| FR-AP-04 | Sistem harus mendukung konfigurasi batas nilai (threshold) yang memicu kebutuhan approval (misal diskon > 10%) | Should Have | BR-04 |

---

## 7. Modul: Promosi & Loyalitas

| ID | Requirement | Prioritas | Sumber |
|---|---|---|---|
| FR-PR-01 | Sistem harus mendukung berbagai jenis promo: diskon persentase, nominal, beli-X-gratis-Y, bundling | Must Have | BR-09 |
| FR-PR-02 | Sistem harus mendukung periode promo (tanggal mulai & berakhir) dan penerapan otomatis saat transaksi | Must Have | BR-09 |
| FR-PR-03 | Sistem harus mendukung program poin loyalitas pelanggan (akumulasi & penukaran poin) | Must Have | BR-09 |
| FR-PR-04 | Sistem harus mendukung promo yang dibatasi per cabang tertentu | Should Have | BR-09 |

---

## 8. Modul: Dashboard & Laporan

| ID | Requirement | Prioritas | Sumber |
|---|---|---|---|
| FR-DB-01 | Sistem harus menyediakan dashboard ringkasan penjualan harian per cabang dan konsolidasi | Must Have | BR-05 |
| FR-DB-02 | Sistem harus menyediakan laporan penjualan per produk, kategori, kasir, dan periode | Must Have | BR-05 |
| FR-DB-03 | Sistem harus menyediakan laporan stok (stok saat ini, mutasi, nilai persediaan) | Must Have | BR-05 |
| FR-DB-04 | Sistem harus menyediakan laporan keuangan dasar (rekap kas, laba kotor per periode) | Must Have | BR-05 |
| FR-DB-05 | Sistem harus mendukung ekspor laporan ke format umum (PDF, Excel/CSV) | Should Have | BR-05 |
| FR-DB-06 | Sistem harus menyediakan laporan piutang pelanggan (khusus Distributor) | Must Have | BR-05, BR-07 |

---

## 9. Modul: Notifikasi

| ID | Requirement | Prioritas | Sumber |
|---|---|---|---|
| FR-NT-01 | Sistem harus mengirim notifikasi stok menipis ke Branch Manager/Purchasing | Must Have | BR-08 |
| FR-NT-02 | Sistem harus mengirim notifikasi approval pending ke approver terkait | Must Have | BR-08 |
| FR-NT-03 | Sistem harus mengirim notifikasi shift kasir yang belum ditutup pada akhir hari | Must Have | BR-08 |
| FR-NT-04 | Sistem harus mengirim notifikasi produk mendekati tanggal kadaluarsa | Must Have | BR-08 |

---

## 10. Modul: Otentikasi & Otorisasi

| ID | Requirement | Prioritas | Sumber |
|---|---|---|---|
| FR-AU-01 | Sistem harus mendukung login individual per pegawai (tidak ada akun bersama) | Must Have | BR-03, BR-10 |
| FR-AU-02 | Sistem harus mendukung role & permission granular per fitur/aksi | Must Have | BR-04, BR-10 |
| FR-AU-03 | Sistem harus mendukung pembatasan akses berdasarkan cabang penempatan pegawai | Must Have | BR-01, BR-10 |
| FR-AU-04 | Sistem harus mencatat log aktivitas login dan aksi penting pengguna | Must Have | BR-03, BR-10 |

---

## 11. Ringkasan Pemetaan Modul ke Business Requirement

| Modul | BR Terkait |
|---|---|
| Manajemen Data Master | BR-01, BR-03, BR-07 |
| Transaksi Penjualan | BR-02, BR-04, BR-06, BR-07, BR-09 |
| Transaksi Pembelian | BR-01, BR-03, BR-04 |
| Manajemen Stok | BR-01, BR-03, BR-04, BR-05, BR-07, BR-08 |
| Approval & Kontrol | BR-03, BR-04, BR-08 |
| Promosi & Loyalitas | BR-09 |
| Dashboard & Laporan | BR-05, BR-07 |
| Notifikasi | BR-08 |
| Otentikasi & Otorisasi | BR-03, BR-04, BR-10 |

---

## 12. Traceability Lanjutan

Setiap FR pada dokumen ini akan diturunkan lebih lanjut menjadi:

- **Use case detail** → [`use-case.md`](use-case.md)
- **Aturan bisnis spesifik** → [`business-rules.md`](business-rules.md)
- **Kontrak API** → [`api-contract.md`](../technical/api-contract.md)
- **Desain modul teknis** → [`module-design.md`](../technical/module-design.md)

---

**Sebelumnya:** [`business-requirements.md`](business-requirements.md) — Business Requirements
**Selanjutnya:** [`non-functional-requirements.md`](non-functional-requirements.md) — Non-Functional Requirements
