# Business Rules — POS

> Dokumen ini mendaftar **aturan bisnis eksplisit (business rules)** yang harus ditegakkan oleh sistem secara konsisten, terlepas dari siapa penggunanya atau modul mana yang sedang berjalan. Business rules berbeda dari functional requirement — FR menjelaskan *fitur apa yang ada*, sedangkan business rule menjelaskan *batasan/logika yang tidak boleh dilanggar* oleh fitur tersebut. Dokumen ini menjadi acuan langsung bagi [`validation.md`](validation.md) dan [`service-layer-design.md`](../technical/service-layer-design.md).

---

## 1. Cara Membaca Dokumen Ini

Setiap rule diberi ID (`BRULE-xxx`), dikelompokkan per area bisnis, dan mencantumkan **konsekuensi jika dilanggar** untuk kejelasan implementasi validasi.

---

## 2. Aturan Terkait Harga & Produk

| ID | Aturan | Konsekuensi Jika Dilanggar |
|---|---|---|
| BRULE-PR-01 | Harga jual produk tidak boleh diubah langsung oleh cabang; hanya Head Office yang berwenang mengubah harga dasar | Sistem menolak permintaan update harga dari role selain `owner`/`ho_admin` |
| BRULE-PR-02 | Setiap produk harus memiliki minimal satu satuan dasar (base unit) sebelum dapat ditransaksikan | Produk tanpa satuan dasar tidak dapat muncul di pencarian transaksi penjualan |
| BRULE-PR-03 | Harga jual tidak boleh lebih rendah dari harga beli terakhir kecuali ditandai sebagai "harga promo" secara eksplisit | Sistem memunculkan peringatan (warning) saat harga jual disetel di bawah harga beli tanpa flag promo |
| BRULE-PR-04 | Perubahan harga memiliki tanggal efektif dan tidak berlaku retroaktif terhadap transaksi yang sudah terjadi | Transaksi lama tetap menggunakan harga yang berlaku saat transaksi dibuat |

---

## 3. Aturan Terkait Stok

| ID | Aturan | Konsekuensi Jika Dilanggar |
|---|---|---|
| BRULE-ST-01 | Stok tidak boleh menjadi negatif akibat transaksi penjualan, kecuali cabang mengaktifkan kebijakan "izinkan stok minus" secara eksplisit | Transaksi penjualan ditolak jika stok tidak mencukupi dan kebijakan tidak diaktifkan |
| BRULE-ST-02 | Produk dengan tracking batch/expired wajib menyertakan informasi batch saat stok masuk | Penerimaan barang tanpa data batch untuk produk yang wajib tracking ditolak sistem |
| BRULE-ST-03 | Penjualan produk dengan multi-batch harus mengikuti prinsip FEFO (First Expired First Out) | Sistem otomatis mengalokasikan batch dengan tanggal kadaluarsa terdekat terlebih dahulu |
| BRULE-ST-04 | Transfer stok antar cabang tidak final hingga cabang tujuan mengonfirmasi penerimaan fisik | Stok di cabang asal berstatus "dialokasikan" (bukan hilang) selama transfer belum dikonfirmasi |
| BRULE-ST-05 | Penyesuaian stok manual wajib disertai alasan dan tunduk pada alur approval sesuai [`approval-flow.md`](approval-flow.md) | Sistem menolak penyesuaian stok tanpa alasan atau tanpa approval yang diperlukan |

---

## 4. Aturan Terkait Transaksi Penjualan

| ID | Aturan | Konsekuensi Jika Dilanggar |
|---|---|---|
| BRULE-TX-01 | Transaksi hanya dapat dibuat oleh kasir dengan shift yang sedang aktif | Sistem menolak transaksi jika tidak ada shift aktif untuk kasir tersebut |
| BRULE-TX-02 | Total pembayaran (termasuk kombinasi metode pembayaran) harus sama dengan total tagihan sebelum transaksi dapat difinalisasi | Sistem menahan finalisasi transaksi hingga jumlah sesuai |
| BRULE-TX-03 | Void hanya dapat dilakukan terhadap transaksi dalam shift yang masih aktif (belum ditutup) | Void terhadap transaksi pada shift yang sudah ditutup harus melalui proses retur, bukan void |
| BRULE-TX-04 | Diskon manual yang melebihi threshold yang dikonfigurasi wajib melalui approval sebelum diterapkan | Sistem menahan penerapan diskon hingga approval diberikan |
| BRULE-TX-05 | Retur penjualan wajib mereferensikan transaksi asal dan tidak boleh melebihi jumlah/nilai item pada transaksi asal | Sistem menolak retur yang jumlahnya melebihi jumlah pembelian asli |
| BRULE-TX-06 | Transaksi open bill (F&B) tidak dianggap selesai hingga pembayaran final diterima dan meja ditutup | Laporan penjualan harian tidak menghitung open bill yang belum dibayar sebagai penjualan final |

---

## 5. Aturan Terkait Piutang (Distributor)

| ID | Aturan | Konsekuensi Jika Dilanggar |
|---|---|---|
| BRULE-CR-01 | Transaksi kredit (piutang) baru tidak dapat dibuat jika pelanggan telah melewati limit piutang yang disetujui | Sistem menahan transaksi dan mengarahkan ke alur approval kelebihan limit |
| BRULE-CR-02 | Setiap pelunasan piutang harus mereferensikan invoice yang jelas, tidak boleh berupa pembayaran umum tanpa alokasi | Sistem menolak pencatatan pembayaran piutang tanpa referensi invoice |
| BRULE-CR-03 | Term of payment (jatuh tempo) dihitung sejak tanggal invoice diterbitkan, bukan sejak barang dikirim | Perhitungan status piutang (lewat jatuh tempo/tidak) mengacu pada tanggal invoice |

---

## 6. Aturan Terkait Domain Apotek

| ID | Aturan | Konsekuensi Jika Dilanggar |
|---|---|---|
| BRULE-PH-01 | Produk berkategori obat keras/resep tidak dapat dijual tanpa validasi resep tercatat di sistem | Transaksi tertahan pada item tersebut hingga validasi Apoteker tercatat |
| BRULE-PH-02 | Produk dengan tanggal kadaluarsa yang telah lewat tidak boleh dijual | Sistem memblokir penjualan batch yang sudah expired, meskipun stok tercatat tersedia |

---

## 7. Aturan Terkait Domain F&B

| ID | Aturan | Konsekuensi Jika Dilanggar |
|---|---|---|
| BRULE-FB-01 | Satu meja hanya dapat memiliki satu open bill aktif pada satu waktu | Sistem menolak pembuatan open bill baru jika meja masih memiliki bill aktif |
| BRULE-FB-02 | Item pada open bill tidak dapat dihapus setelah dikirim ke dapur, kecuali melalui proses void dengan approval | Perubahan pesanan yang sudah diproses dapur wajib tercatat sebagai void, bukan penghapusan langsung |
| BRULE-FB-03 | Split bill total nilainya harus sama dengan total bill asal | Sistem menolak split bill yang totalnya tidak sesuai dengan bill asal |

---

## 8. Aturan Terkait Approval Berjenjang

| ID | Aturan | Konsekuensi Jika Dilanggar |
|---|---|---|
| BRULE-AP-01 | Approver tidak boleh menyetujui permintaan yang diajukan oleh dirinya sendiri | Sistem menolak self-approval, permintaan diteruskan ke approver lain/level di atasnya |
| BRULE-AP-02 | Permintaan approval yang melebihi kewenangan level saat ini otomatis dieskalasi ke level berikutnya | Permintaan tidak "menggantung" tanpa approver yang berwenang |
| BRULE-AP-03 | Keputusan approval bersifat final dan tidak dapat diubah setelah dieksekusi; jika terjadi kesalahan, harus dibuat transaksi koreksi baru | Menjaga integritas audit trail — histori tidak boleh berubah setelah tercatat |

---

## 9. Aturan Terkait Multi-Cabang

| ID | Aturan | Konsekuensi Jika Dilanggar |
|---|---|---|
| BRULE-MB-01 | Setiap transaksi harus terasosiasi dengan tepat satu cabang/outlet, tidak boleh tanpa cabang | Sistem menolak pembuatan transaksi tanpa konteks cabang yang jelas |
| BRULE-MB-02 | Pengguna hanya dapat login dan bertransaksi pada cabang tempat mereka terdaftar, kecuali role level pusat | Percobaan akses transaksi cabang lain ditolak di level otorisasi |
| BRULE-MB-03 | Laporan konsolidasi harus mencerminkan data real-time dari seluruh cabang tanpa proses sinkronisasi manual | Arsitektur data harus mendukung agregasi langsung, bukan proses batch harian |

---

## 10. Ringkasan Rule Kritis (Non-Negotiable)

Rule berikut tidak boleh dilewati/dinonaktifkan dalam kondisi apa pun karena berkaitan langsung dengan integritas finansial dan kepatuhan:

- BRULE-TX-02 (kesesuaian total pembayaran)
- BRULE-ST-01 (stok tidak boleh negatif tanpa kebijakan eksplisit)
- BRULE-PH-01, BRULE-PH-02 (kepatuhan regulasi apotek)
- BRULE-AP-01 (larangan self-approval)
- BRULE-CR-01 (kontrol limit piutang)

---

## 11. Traceability

| Kategori Rule | Diimplementasikan Pada |
|---|---|
| Harga & Produk | [`master-data.md`](master-data.md), [`api-contract.md`](../technical/api-contract.md) |
| Stok | [`transactions.md`](transactions.md), [`service-layer-design.md`](../technical/service-layer-design.md) |
| Transaksi Penjualan | [`validation.md`](validation.md), [`edge-cases.md`](edge-cases.md) |
| Approval | [`approval-flow.md`](approval-flow.md), [`authorization-design.md`](../technical/authorization-design.md) |
| Multi-Cabang | [`database-design.md`](../technical/database-design.md) |

---

**Sebelumnya:** [`role-permission.md`](role-permission.md) — Role & Permission
**Selanjutnya:** [`approval-flow.md`](approval-flow.md) — Approval Flow
