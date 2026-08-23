# Edge Cases — POS

> Dokumen ini mendaftar **skenario kasus khusus (edge cases)** yang jarang terjadi namun berdampak signifikan jika tidak diantisipasi. Berbeda dari [`validation.md`](validation.md) yang berfokus pada aturan input data, dokumen ini berfokus pada **kondisi operasional tidak umum** yang harus tetap ditangani sistem secara benar. Dokumen ini menjadi acuan penting bagi [`testing-strategy.md`](../technical/testing-strategy.md) untuk menyusun test case non-trivial.

---

## 1. Mengapa Edge Cases Penting untuk Sistem Ini

Sistem POS enterprise beroperasi di lingkungan nyata yang penuh ketidakpastian — koneksi internet terputus, perangkat keras gagal, pengguna melakukan kesalahan input, hingga kondisi bisnis yang jarang tapi valid (retur sebagian, transaksi lintas hari). Kegagalan menangani edge case pada sistem finansial **berarti kerugian nyata**, bukan sekadar bug kosmetik.

---

## 2. Edge Cases: Transaksi Penjualan

| ID | Skenario | Penanganan yang Diharapkan |
|---|---|---|
| EC-SL-01 | Koneksi internet terputus saat transaksi sedang berlangsung | Transaksi tersimpan lokal (jika mode offline diaktifkan — NFR-AV-02), disinkronkan otomatis saat koneksi pulih tanpa duplikasi |
| EC-SL-02 | Dua kasir mencoba menjual produk yang sama secara bersamaan dengan stok tersisa 1 | Sistem menerapkan mekanisme locking/atomicity pada level database agar hanya satu transaksi berhasil mengurangi stok terakhir, transaksi lain menerima notifikasi stok habis |
| EC-SL-03 | Pelanggan membatalkan transaksi setelah pembayaran non-tunai (kartu/e-wallet) berhasil namun sebelum struk tercetak | Transaksi tetap tercatat sebagai selesai (karena pembayaran sudah diproses), pembatalan harus melalui alur retur, bukan void |
| EC-SL-04 | Struk gagal tercetak (printer error) meskipun transaksi sudah berhasil di sistem | Transaksi tetap sah, sistem menyediakan opsi cetak ulang struk tanpa membuat transaksi duplikat |
| EC-SL-05 | Transaksi terjadi mendekati tengah malam, melewati pergantian hari kalender sebelum shift ditutup | Transaksi tetap tercatat dalam shift yang sedang berjalan (bukan dipotong berdasarkan tanggal kalender), laporan harian mengikuti periode shift, bukan hanya tanggal kalender |
| EC-SL-06 | Produk yang dijual dihapus/dinonaktifkan dari master data setelah transaksi tercatat | Data historis transaksi tetap menyimpan snapshot nama & harga produk saat transaksi terjadi, tidak terpengaruh perubahan/penghapusan master data selanjutnya |
| EC-SL-07 | Diskon promo otomatis dan diskon manual diterapkan bersamaan pada transaksi yang sama | Sistem mengikuti aturan prioritas/kombinasi promo yang dikonfigurasi eksplisit (misal: dapat digabung atau tidak), tidak boleh diterapkan ambigu |

---

## 3. Edge Cases: Stok & Inventori

| ID | Skenario | Penanganan yang Diharapkan |
|---|---|---|
| EC-ST-01 | Stok opname dilakukan saat masih ada transaksi penjualan berjalan di cabang yang sama | Sistem mengunci sesi opname terhadap perubahan stok baru selama proses berlangsung, atau mencatat transaksi yang terjadi selama opname sebagai penyesuaian terpisah setelah opname selesai |
| EC-ST-02 | Transfer stok dikirim namun barang hilang/rusak dalam perjalanan sebelum diterima cabang tujuan | Cabang tujuan dapat mengonfirmasi penerimaan sebagian dengan catatan selisih, yang kemudian memerlukan approval sebagai penyesuaian |
| EC-ST-03 | Produk dengan multi-batch, sebagian batch expired namun sebagian lain masih valid | Sistem hanya mengalokasikan batch yang valid untuk penjualan (BRULE-PH-02), batch expired ditandai terpisah untuk proses write-off |
| EC-ST-04 | Retur pembelian dilakukan setelah sebagian barang dari PO yang sama sudah terjual ke pelanggan | Sistem memvalidasi retur hanya terhadap stok yang masih tersedia, tidak dapat meretur barang yang sudah terjual |
| EC-ST-05 | Penerimaan barang dengan jumlah melebihi PO (supplier mengirim lebih dari yang dipesan) | Sistem meminta konfirmasi eksplisit dan/atau approval untuk kelebihan penerimaan sebelum stok diperbarui |

---

## 4. Edge Cases: Shift & Kas

| ID | Skenario | Penanganan yang Diharapkan |
|---|---|---|
| EC-SH-01 | Kasir lupa menutup shift dan langsung logout/perangkat mati | Shift tetap berstatus "belum ditutup", sistem mengirim notifikasi eskalasi ke Supervisor (NOTIF-SH-01), shift dapat ditutup paksa oleh Supervisor dengan pencatatan khusus |
| EC-SH-02 | Dua kasir bergantian pada satu terminal dalam satu hari tanpa menutup shift sebelumnya | Sistem tidak mengizinkan shift baru dibuka pada terminal yang sama sebelum shift sebelumnya ditutup, mencegah pencampuran tanggung jawab kas |
| EC-SH-03 | Selisih kas bernilai negatif besar (kekurangan signifikan) saat tutup shift | Selisih dicatat apa adanya (tidak disembunyikan/dibulatkan), memicu approval wajib dan notifikasi prioritas tinggi ke Supervisor |

---

## 5. Edge Cases: Approval

| ID | Skenario | Penanganan yang Diharapkan |
|---|---|---|
| EC-AP-01 | Approver satu-satunya yang berwenang sedang tidak aktif/cuti dalam waktu lama | Sistem mendukung eskalasi ke level di atasnya setelah SLA terlampaui (BRULE-AP-02), mencegah permintaan menggantung tanpa batas waktu |
| EC-AP-02 | Permintaan approval dibatalkan oleh pengaju sebelum diputuskan | Status berubah menjadi "Dibatalkan", bukan dihapus, agar tetap tercatat dalam audit trail |
| EC-AP-03 | Dua approval untuk aksi yang saling terkait diputuskan dengan urutan yang tidak konsisten (misal approval stok disetujui, namun approval kas terkait ditolak) | Sistem memperlakukan setiap approval sebagai unit independen sesuai konteksnya masing-masing; ketidaksesuaian ditinjau secara manual melalui laporan anomali, bukan otomatis saling membatalkan |

---

## 6. Edge Cases: Domain F&B

| ID | Skenario | Penanganan yang Diharapkan |
|---|---|---|
| EC-FB-01 | Pelanggan pindah meja di tengah pesanan berjalan | Sistem mendukung perpindahan open bill dari satu meja ke meja lain tanpa kehilangan data pesanan |
| EC-FB-02 | Item pesanan sudah dikirim ke dapur namun pelanggan membatalkan sebelum selesai dimasak | Pembatalan dicatat sebagai void dengan approval (BRULE-FB-02), bukan penghapusan langsung, agar dapur mengetahui pembatalan |
| EC-FB-03 | Dua meja digabung menjadi satu bill (misal rombongan pelanggan pindah tempat duduk) | Sistem mendukung penggabungan open bill dengan pencatatan referensi ke meja asal untuk keperluan audit |

---

## 7. Edge Cases: Domain Apotek

| ID | Skenario | Penanganan yang Diharapkan |
|---|---|---|
| EC-PH-01 | Resep yang diajukan pelanggan mencakup kombinasi obat resep dan non-resep dalam satu transaksi | Sistem menahan hanya item yang memerlukan validasi resep, item non-resep dapat diproses tanpa menunggu (partial hold) |
| EC-PH-02 | Apoteker sedang tidak berada di lokasi saat pelanggan membutuhkan obat resep mendesak | Transaksi item resep tertahan hingga validasi tersedia; sistem tidak memperbolehkan bypass tanpa validasi tercatat, sesuai kepatuhan regulasi (BRULE-PH-01) |

---

## 8. Edge Cases: Domain Distributor

| ID | Skenario | Penanganan yang Diharapkan |
|---|---|---|
| EC-CR-01 | Pelanggan melakukan pelunasan piutang melebihi jumlah yang tertagih (overpayment) | Sistem mencatat kelebihan sebagai saldo kredit pelanggan untuk transaksi berikutnya, bukan menolak pembayaran |
| EC-CR-02 | Pelanggan dengan piutang jatuh tempo mencoba melakukan transaksi kredit baru | Sistem menahan transaksi baru dan mengarahkan ke proses approval/penagihan sesuai kebijakan cabang (terkait BRULE-CR-01) |

---

## 9. Edge Cases: Multi-Cabang & Sinkronisasi

| ID | Skenario | Penanganan yang Diharapkan |
|---|---|---|
| EC-MB-01 | Perubahan harga terpusat dipublikasikan saat transaksi sedang berjalan di cabang menggunakan harga lama | Transaksi yang sudah dimulai tetap menggunakan harga saat transaksi dibuat (BRULE-PR-04), perubahan berlaku untuk transaksi baru setelahnya |
| EC-MB-02 | Pegawai dipindahkan ke cabang lain di tengah shift yang sedang berjalan | Sistem tidak mengizinkan perpindahan cabang aktif selama shift berjalan; perubahan penempatan berlaku efektif setelah shift ditutup |

---

## 10. Ringkasan Prioritas Penanganan Edge Case

| Prioritas | Kategori | Alasan |
|---|---|---|
| Kritis | EC-SL-02, EC-SL-05, EC-ST-01, EC-SH-01 | Berkaitan langsung dengan integritas data finansial dan stok |
| Tinggi | EC-SL-01, EC-ST-02, EC-AP-01, EC-PH-02 | Berdampak pada operasional harian jika tidak ditangani |
| Sedang | Sisanya | Tetap penting namun frekuensi kejadian lebih rendah |

---

## 11. Traceability

| Kategori Edge Case | Terkait Dokumen |
|---|---|
| Aturan dasar yang relevan | [`business-rules.md`](business-rules.md) |
| Validasi terkait | [`validation.md`](validation.md) |
| Implementasi teknis penanganan | [`service-layer-design.md`](../technical/service-layer-design.md), [`system-architecture.md`](../technical/system-architecture.md) |
| Test case turunan | [`testing-strategy.md`](../technical/testing-strategy.md) |

---

**Sebelumnya:** [`validation.md`](validation.md) — Validation
**Selanjutnya:** [`master-data.md`](master-data.md) — Master Data
