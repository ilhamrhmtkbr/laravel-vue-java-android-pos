# Use Case — POS

> Dokumen ini menjelaskan **use case** sistem dari sudut pandang tiap aktor, mengacu pada [`business-flow.md`](business-flow.md) dan [`sop.md`](sop.md). Setiap use case memiliki skenario utama (main flow) dan skenario alternatif (alternative/exception flow) sebagai dasar penyusunan test case pada [`testing-strategy.md`](../technical/testing-strategy.md).

---

## 1. Diagram Aktor & Ruang Lingkup

```
                    ┌────────────────────────────────────────────┐
                    │                  SISTEM POS                 │
                    │                                              │
   Owner/Direktur ──┼──▶ Lihat Dashboard Konsolidasi               │
                    │                                              │
   HO Admin ────────┼──▶ Kelola Master Data & Harga Terpusat       │
                    │                                              │
   Branch Manager ──┼──▶ Approval, Kelola Cabang, Lihat Laporan    │
                    │                                              │
   Supervisor ──────┼──▶ Approval Transaksi Sensitif               │
                    │                                              │
   Kasir ───────────┼──▶ Transaksi Penjualan, Buka/Tutup Shift     │
                    │                                              │
   Staff Gudang ────┼──▶ Penerimaan Barang, Transfer, Stok Opname  │
                    │                                              │
   Apoteker ────────┼──▶ Validasi Resep                            │
                    │                                              │
   Waiter/Kitchen ──┼──▶ Kelola Pesanan Meja, Kitchen Order        │
                    │                                              │
   Pelanggan ───────┼──▶ (Aktor tidak langsung — dilayani kasir)   │
                    └────────────────────────────────────────────┘
```

---

## 2. Use Case: Kasir

### UC-01 — Melakukan Transaksi Penjualan

| Atribut | Detail |
|---|---|
| Aktor | Kasir |
| Prasyarat | Kasir sudah login dan shift sudah dibuka |
| Terkait FR | FR-SL-01, FR-SL-02, FR-SL-03, FR-SL-07 |

**Main Flow:**
1. Kasir memilih/scan produk yang dibeli pelanggan.
2. Sistem menampilkan harga dan menambahkan ke keranjang.
3. Kasir menerapkan promo/member (jika ada).
4. Kasir memilih metode pembayaran.
5. Kasir mengonfirmasi pembayaran diterima.
6. Sistem mencetak struk dan menyelesaikan transaksi.

**Alternative Flow:**
- 2a. Produk tidak ditemukan → sistem menampilkan pesan error, kasir dapat mencari manual.
- 4a. Pembayaran gabungan (split payment) → kasir input lebih dari satu metode dengan totalnya harus sama dengan total tagihan.
- 5a. Pembayaran tunai kurang dari total → sistem menolak konfirmasi hingga jumlah mencukupi.

### UC-02 — Void Transaksi/Item

| Atribut | Detail |
|---|---|
| Aktor | Kasir (pengaju), Supervisor (approver) |
| Prasyarat | Transaksi/item yang akan divoid belum termasuk dalam shift yang sudah ditutup |
| Terkait FR | FR-SL-05, FR-AP-01, FR-AP-02 |

**Main Flow:**
1. Kasir memilih transaksi/item yang akan divoid.
2. Kasir mengisi alasan void.
3. Sistem mengirim permintaan approval ke Supervisor yang bertugas.
4. Supervisor meninjau dan menyetujui.
5. Sistem membatalkan transaksi/item dan mencatat log void.

**Alternative Flow:**
- 4a. Supervisor menolak → transaksi tetap berlaku, kasir menerima notifikasi penolakan.
- 3a. Tidak ada Supervisor yang bertugas/online → permintaan diteruskan ke Branch Manager.

### UC-03 — Buka & Tutup Shift

| Atribut | Detail |
|---|---|
| Aktor | Kasir |
| Terkait FR | FR-SL-09 |

**Main Flow (Buka Shift):**
1. Kasir login dan memilih terminal kasir.
2. Kasir input kas awal (modal).
3. Sistem mencatat waktu mulai shift.

**Main Flow (Tutup Shift):**
1. Kasir memilih akhiri shift.
2. Kasir input kas fisik akhir.
3. Sistem menghitung selisih otomatis.
4. Jika selisih dalam batas toleransi → shift ditutup langsung.
5. Jika selisih melebihi batas toleransi → sistem meminta approval Supervisor sebelum shift resmi ditutup.

---

## 3. Use Case: Supervisor / Branch Manager

### UC-04 — Approval Aksi Sensitif

| Atribut | Detail |
|---|---|
| Aktor | Supervisor, Branch Manager |
| Terkait FR | FR-AP-01 s.d. FR-AP-04 |

**Main Flow:**
1. Aktor menerima notifikasi permintaan approval (void, diskon manual, penyesuaian stok, dll).
2. Aktor membuka detail permintaan.
3. Aktor menyetujui atau menolak, dengan opsi menambahkan catatan.
4. Sistem mengeksekusi/membatalkan aksi sesuai keputusan dan mencatat log approval.

**Alternative Flow:**
- 3a. Permintaan melebihi batas kewenangan approver saat ini → sistem meneruskan ke level approval berikutnya (eskalasi).

### UC-05 — Mengelola Purchase Order

| Atribut | Detail |
|---|---|
| Aktor | Branch Manager, Staff Purchasing |
| Terkait FR | FR-PU-01 |

**Main Flow:**
1. Aktor meninjau daftar produk dengan stok di bawah reorder point.
2. Aktor memilih supplier dan produk yang akan dipesan.
3. Aktor membuat Purchase Order.
4. Sistem mengirim PO (status: menunggu penerimaan) dan mencatatnya untuk pelacakan.

### UC-06 — Meninjau Laporan Cabang

| Atribut | Detail |
|---|---|
| Aktor | Branch Manager |
| Terkait FR | FR-DB-01 s.d. FR-DB-04 |

**Main Flow:**
1. Aktor membuka dashboard cabang.
2. Sistem menampilkan ringkasan penjualan, stok, dan kas harian.
3. Aktor dapat men-drill down ke laporan detail atau mengekspor laporan.

---

## 4. Use Case: Staff Gudang

### UC-07 — Penerimaan Barang

| Atribut | Detail |
|---|---|
| Aktor | Staff Gudang |
| Prasyarat | Purchase Order berstatus aktif |
| Terkait FR | FR-PU-02, FR-PU-03 |

**Main Flow:**
1. Staff Gudang membuka daftar PO yang menunggu penerimaan.
2. Staff Gudang mencocokkan barang fisik dengan PO.
3. Staff Gudang input jumlah diterima (termasuk batch/expired bila relevan).
4. Sistem memperbarui stok dan status PO.

**Alternative Flow:**
- 2a. Barang diterima tidak lengkap → status PO menjadi "diterima sebagian", sisa tetap tercatat menunggu.
- 2b. Barang rusak/tidak sesuai → Staff Gudang mengajukan retur ke supplier.

### UC-08 — Transfer Stok Antar Cabang

| Atribut | Detail |
|---|---|
| Aktor | Staff Gudang (cabang asal & tujuan) |
| Terkait FR | FR-IN-02 |

**Main Flow:**
1. Staff Gudang cabang asal membuat permintaan transfer.
2. Sistem mengurangi stok "dialokasikan" di cabang asal (belum dikurangi penuh hingga dikonfirmasi terkirim).
3. Staff Gudang cabang asal menandai barang "dikirim".
4. Staff Gudang cabang tujuan menandai barang "diterima" setelah verifikasi fisik.
5. Sistem memperbarui stok final di kedua cabang.

### UC-09 — Stok Opname

| Atribut | Detail |
|---|---|
| Aktor | Staff Gudang, Branch Manager (approval selisih) |
| Terkait FR | FR-IN-03 |

**Main Flow:**
1. Staff Gudang menghitung stok fisik.
2. Staff Gudang input hasil ke sistem.
3. Sistem menampilkan selisih dengan stok tercatat.
4. Jika selisih signifikan → memerlukan approval Branch Manager sebelum stok sistem disesuaikan.

---

## 5. Use Case: Domain Khusus

### UC-10 — Validasi Resep (Apotek)

| Atribut | Detail |
|---|---|
| Aktor | Apoteker |
| Terkait FR | FR-MD-09 |

**Main Flow:**
1. Kasir memindai produk kategori obat keras/resep.
2. Sistem menahan transaksi dan meminta validasi Apoteker.
3. Apoteker memeriksa resep fisik dan mencatat validasi di sistem.
4. Kasir melanjutkan transaksi setelah validasi tercatat.

### UC-11 — Kelola Pesanan Meja (F&B)

| Atribut | Detail |
|---|---|
| Aktor | Waiter, Kitchen Staff |
| Terkait FR | FR-SL-04, FR-MD-10 |

**Main Flow:**
1. Waiter memilih meja dan menginput pesanan (item + modifier).
2. Sistem mengirim pesanan ke tampilan/printer dapur.
3. Kitchen Staff memproses dan menandai pesanan selesai.
4. Waiter dapat menambah item baru ke bill yang sama (open bill).
5. Saat pelanggan selesai, kasir/waiter memproses pembayaran dan menutup meja.

**Alternative Flow:**
- 5a. Pelanggan meminta split bill → sistem membagi total tagihan sesuai item/porsi yang ditentukan.

### UC-12 — Transaksi Piutang (Distributor)

| Atribut | Detail |
|---|---|
| Aktor | Kasir/Sales, Branch Manager (approval limit) |
| Terkait FR | FR-SL-11, FR-SL-12 |

**Main Flow:**
1. Pelanggan (reseller) melakukan pemesanan.
2. Sistem menghitung harga sesuai kelas pelanggan (tiered pricing).
3. Sistem memeriksa sisa limit piutang pelanggan.
4. Jika dalam limit → transaksi dicatat sebagai piutang dan invoice diterbitkan.

**Alternative Flow:**
- 3a. Melebihi limit piutang → transaksi ditahan, memerlukan approval Branch Manager untuk melanjutkan.

---

## 6. Use Case: Head Office Admin & Owner

### UC-13 — Kelola Harga & Promo Terpusat

| Atribut | Detail |
|---|---|
| Aktor | Head Office Admin |
| Terkait FR | FR-MD-01, FR-MD-08, FR-PR-01, FR-PR-02 |

**Main Flow:**
1. HO Admin membuat/mengubah harga atau promo.
2. HO Admin menentukan cabang target dan periode berlaku.
3. Sistem mempublikasikan perubahan sesuai jadwal efektif.
4. Sistem mencatat histori perubahan harga.

### UC-14 — Melihat Dashboard Konsolidasi

| Atribut | Detail |
|---|---|
| Aktor | Owner/Direktur |
| Terkait FR | FR-DB-01 |

**Main Flow:**
1. Owner login ke sistem.
2. Sistem menampilkan dashboard konsolidasi seluruh cabang secara real-time.
3. Owner dapat memfilter berdasarkan periode, cabang, atau kategori produk.

---

## 7. Matriks Use Case vs Aktor

| Use Case | Kasir | Supervisor | Branch Mgr | Staff Gudang | Apoteker | Waiter | HO Admin | Owner |
|---|---|---|---|---|---|---|---|---|
| UC-01 Transaksi Penjualan | ✅ | | | | | | | |
| UC-02 Void Transaksi | ✅ | ✅ | ✅ | | | | | |
| UC-03 Buka/Tutup Shift | ✅ | | | | | | | |
| UC-04 Approval Aksi Sensitif | | ✅ | ✅ | | | | | |
| UC-05 Purchase Order | | | ✅ | | | | | |
| UC-06 Laporan Cabang | | | ✅ | | | | | |
| UC-07 Penerimaan Barang | | | | ✅ | | | | |
| UC-08 Transfer Stok | | | | ✅ | | | | |
| UC-09 Stok Opname | | | ✅ | ✅ | | | | |
| UC-10 Validasi Resep | | | | | ✅ | | | |
| UC-11 Kelola Pesanan Meja | | | | | | ✅ | | |
| UC-12 Transaksi Piutang | ✅ | | ✅ | | | | | |
| UC-13 Harga & Promo Terpusat | | | | | | | ✅ | |
| UC-14 Dashboard Konsolidasi | | | | | | | | ✅ |

---

**Sebelumnya:** [`sop.md`](sop.md) — Standard Operating Procedure
**Selanjutnya:** [`role-permission.md`](role-permission.md) — Role & Permission
