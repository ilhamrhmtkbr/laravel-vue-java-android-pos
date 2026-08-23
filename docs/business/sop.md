# Standard Operating Procedure (SOP) — POS

> Dokumen ini menjelaskan **prosedur operasional standar** yang harus diikuti oleh masing-masing peran pengguna sistem sehari-hari. SOP ini adalah penerjemahan operasional dari [`business-flow.md`](business-flow.md), dan menjadi acuan untuk memastikan sistem yang dibangun benar-benar mendukung cara kerja nyata di lapangan — bukan sebaliknya (memaksa pengguna mengikuti keterbatasan sistem).

---

## 1. Tujuan SOP

- Menstandarkan cara kerja pengguna di seluruh cabang agar konsisten.
- Menjadi acuan pelatihan pegawai baru.
- Menjadi dasar validasi bahwa desain sistem (UI/UX, alur, aturan) benar-benar dapat diikuti di lapangan.
- Menjadi rujukan saat terjadi penyimpangan prosedur (untuk investigasi/audit).

---

## 2. SOP: Kasir (Cashier)

### 2.1 SOP Buka Shift

1. Login ke sistem kasir menggunakan akun individu.
2. Pilih meja kasir/terminal yang digunakan (jika lebih dari satu terminal per outlet).
3. Input jumlah kas awal (modal) secara fisik dan catat di sistem.
4. Sistem mencatat waktu mulai shift dan kas awal sebagai baseline rekonsiliasi.
5. Pastikan perangkat pendukung (printer struk, laci kas, scanner) berfungsi normal sebelum melayani transaksi pertama.

**Alasan:** Modal awal yang tercatat menjadi dasar perhitungan selisih kas di akhir shift — tanpa langkah ini, rekonsiliasi kas tidak dapat dilakukan secara akurat.

### 2.2 SOP Transaksi Penjualan Normal

1. Scan/pilih produk yang dibeli pelanggan.
2. Konfirmasi jumlah (quantity) setiap item ke pelanggan.
3. Terapkan diskon/promo (otomatis oleh sistem, atau manual jika ada dan sesuai kewenangan kasir).
4. Tanyakan metode pembayaran ke pelanggan.
5. Input pembayaran diterima, sistem menghitung kembalian (jika tunai).
6. Cetak struk dan serahkan ke pelanggan.
7. Transaksi tercatat otomatis di sistem sebagai bagian dari shift kasir yang sedang berjalan.

### 2.3 SOP Void Transaksi / Item

1. Kasir mengidentifikasi kebutuhan void (misal salah input, pelanggan batal beli).
2. Kasir mengajukan void melalui sistem, wajib mengisi alasan.
3. Sistem mengirim permintaan approval ke Supervisor/Branch Manager yang sedang bertugas.
4. Kasir menunggu approval sebelum transaksi dianggap batal.
5. Setelah disetujui, sistem mencatat void beserta siapa yang mengajukan dan menyetujui.

**Alasan:** Void adalah salah satu titik rawan kecurangan (misal void fiktif untuk mengambil uang tunai) — approval wajib adalah kontrol utama terhadap risiko ini, sesuai [`business-rules.md`](business-rules.md).

### 2.4 SOP Tutup Shift

1. Kasir menghentikan penerimaan transaksi baru pada terminal tersebut.
2. Kasir menghitung kas fisik yang ada di laci.
3. Kasir input jumlah kas fisik ke sistem.
4. Sistem menampilkan perbandingan: kas seharusnya (modal awal + penjualan tunai − retur tunai) vs kas fisik.
5. Jika ada selisih, kasir wajib mencantumkan catatan; jika selisih melebihi batas toleransi, sistem meminta approval Supervisor.
6. Shift ditutup, laporan shift otomatis tergenerate.

---

## 3. SOP: Supervisor / Shift Leader

### 3.1 SOP Approval Void/Diskon Manual

1. Menerima notifikasi permintaan approval dari kasir.
2. Meninjau detail transaksi dan alasan yang diajukan.
3. Melakukan verifikasi fisik bila diperlukan (misal cek barang yang diretur).
4. Menyetujui atau menolak permintaan melalui sistem, dengan opsi menambahkan catatan.

### 3.2 SOP Pengawasan Shift

1. Memantau status seluruh shift kasir yang aktif melalui dashboard supervisor.
2. Menindaklanjuti shift yang belum ditutup melewati jam operasional.
3. Melakukan spot-check transaksi mencurigakan (frekuensi void tinggi, diskon manual berulang).

---

## 4. SOP: Branch Manager

### 4.1 SOP Buka & Tutup Toko

1. **Buka toko:** memastikan seluruh terminal kasir dan perangkat siap sebelum jam operasional dimulai.
2. **Tutup toko:** memastikan seluruh shift kasir hari itu sudah ditutup, meninjau rekap penjualan harian, dan menindaklanjuti selisih kas (jika ada) sebelum laporan harian difinalisasi.

### 4.2 SOP Approval Penyesuaian Stok

1. Menerima pengajuan penyesuaian stok dari staff gudang/kasir (misal akibat barang rusak, hilang, atau selisih stok opname).
2. Meninjau alasan dan besaran penyesuaian.
3. Menyetujui/menolak; jika nilai penyesuaian melebihi batas kewenangan cabang, sistem meneruskan ke Head Office Admin.

### 4.3 SOP Pengajuan Purchase Order

1. Meninjau laporan stok menipis (reorder point) dari sistem.
2. Membuat/menyetujui Purchase Order ke supplier terkait.
3. Memantau status penerimaan barang hingga PO selesai.

---

## 5. SOP: Staff Gudang (Warehouse Staff)

### 5.1 SOP Penerimaan Barang

1. Menerima barang fisik dari supplier sesuai Purchase Order.
2. Mencocokkan jumlah & kondisi barang fisik dengan dokumen PO.
3. Input penerimaan barang ke sistem (termasuk batch & tanggal kadaluarsa bila relevan).
4. Jika ada ketidaksesuaian (kurang/rusak), catat sebagai catatan penerimaan dan ajukan retur ke supplier bila perlu.
5. Sistem otomatis memperbarui stok setelah penerimaan dikonfirmasi.

### 5.2 SOP Transfer Stok Antar Cabang

1. Membuat permintaan transfer stok di sistem (cabang tujuan, produk, jumlah).
2. Menyiapkan barang fisik sesuai permintaan.
3. Mengirim barang dan mengubah status transfer menjadi "dikirim" di sistem.
4. Cabang penerima melakukan konfirmasi penerimaan fisik dan mengubah status menjadi "diterima".
5. Stok otomatis berkurang di cabang asal dan bertambah di cabang tujuan setelah konfirmasi.

### 5.3 SOP Stok Opname

1. Menjadwalkan periode stok opname (harian/mingguan/bulanan sesuai kebijakan cabang).
2. Menghitung stok fisik per produk.
3. Input hasil perhitungan ke sistem.
4. Sistem menampilkan selisih antara stok sistem dan stok fisik.
5. Selisih signifikan memerlukan approval Branch Manager sebelum stok sistem disesuaikan.

---

## 6. SOP: Domain Khusus

### 6.1 SOP Apoteker (Domain Apotek)

1. Menerima permintaan pembelian obat dari pelanggan.
2. Jika produk termasuk kategori obat keras/resep, memvalidasi resep fisik terlebih dahulu.
3. Mencatat validasi resep di sistem sebelum transaksi dapat diproses kasir.
4. Memastikan produk yang dijual mengikuti prinsip FEFO (First Expired First Out) berdasarkan rekomendasi sistem.

### 6.2 SOP Waiter & Kitchen (Domain F&B)

1. **Waiter:** mencatat pesanan pelanggan per meja, termasuk catatan khusus (modifier), lalu mengirim pesanan ke sistem dapur.
2. **Kitchen Staff:** menerima pesanan masuk di layar dapur (kitchen display/print), memproses sesuai urutan, menandai status selesai di sistem.
3. **Waiter:** mengantar pesanan yang sudah ditandai selesai, dan dapat menambah item baru ke bill yang sama (open bill) sesuai permintaan pelanggan lanjutan.
4. Saat pelanggan meminta bill, waiter/kasir memproses pembayaran (termasuk opsi split bill) dan menutup status meja.

---

## 7. SOP: Head Office Admin

### 7.1 SOP Pengelolaan Harga & Promo Terpusat

1. Membuat/mengubah harga produk di tingkat pusat.
2. Menentukan cabang mana saja yang menerima perubahan harga (dapat berlaku global atau spesifik cabang).
3. Mempublikasikan perubahan; sistem otomatis menerapkannya ke cabang terkait berdasarkan jadwal berlaku (efektif sejak tanggal tertentu bila diperlukan).

### 7.2 SOP Peninjauan Laporan Konsolidasi

1. Mengakses dashboard konsolidasi seluruh cabang secara berkala.
2. Mengidentifikasi anomali (misal cabang dengan tingkat void/retur tidak wajar).
3. Menindaklanjuti temuan ke Branch Manager terkait.

---

## 8. Ringkasan Frekuensi Prosedur

| SOP | Frekuensi |
|---|---|
| Buka/Tutup Shift Kasir | Setiap shift (harian, bisa lebih dari sekali/hari) |
| Transaksi Penjualan | Berkelanjutan sepanjang shift |
| Penerimaan Barang | Sesuai jadwal kedatangan supplier |
| Stok Opname | Sesuai kebijakan cabang (harian/mingguan/bulanan) |
| Transfer Stok | Sesuai kebutuhan operasional |
| Peninjauan Laporan Konsolidasi | Harian/mingguan oleh Head Office |
| Approval Void/Diskon/Penyesuaian | Real-time saat dibutuhkan |

---

## 9. Keterkaitan dengan Dokumen Lain

| SOP | Terkait Dokumen |
|---|---|
| Buka/Tutup Shift | [`transactions.md`](transactions.md) |
| Void/Diskon/Approval | [`approval-flow.md`](approval-flow.md), [`business-rules.md`](business-rules.md) |
| Penerimaan Barang & Stok | [`master-data.md`](master-data.md), [`transactions.md`](transactions.md) |
| SOP per Domain | [`business-flow.md`](business-flow.md) |
| Role terkait tiap SOP | [`role-permission.md`](role-permission.md) |

---

**Sebelumnya:** [`business-flow.md`](business-flow.md) — Business Flow
**Selanjutnya:** [`use-case.md`](use-case.md) — Use Case
