# Validation — POS

> Dokumen ini menjelaskan **aturan validasi data dari sudut pandang bisnis** — yaitu data apa yang wajib diisi, format apa yang berlaku, dan kondisi apa yang membuat suatu input dianggap tidak valid. Dokumen ini adalah jembatan antara [`business-rules.md`](business-rules.md) (aturan logika bisnis) dengan implementasi validasi teknis pada [`api-contract.md`](../technical/api-contract.md) dan [`service-layer-design.md`](../technical/service-layer-design.md).

---

## 1. Prinsip Validasi

1. **Validasi bisnis dilakukan di server, bukan hanya di client.** Validasi di sisi frontend/mobile hanya untuk pengalaman pengguna (UX), validasi final dan otoritatif selalu terjadi di backend.
2. **Pesan error harus jelas dan actionable** bagi pengguna (misal "Stok tidak mencukupi, tersedia 3, diminta 5" bukan sekadar "Error").
3. **Validasi mengikuti konteks role dan data scoping** — misal validasi limit piutang hanya relevan untuk transaksi kredit distributor.
4. **Validasi tidak menggantikan business rule**, melainkan menegakkannya di titik input data (lihat [`business-rules.md`](business-rules.md) untuk daftar aturan lengkap).

---

## 2. Validasi Data Master

| Entitas | Field | Aturan Validasi | Alasan Bisnis |
|---|---|---|---|
| Produk | Nama | Wajib diisi, unik dalam satu cabang/tenant | Mencegah duplikasi data yang membingungkan kasir |
| Produk | SKU/Barcode | Wajib diisi, unik secara global | Barcode adalah kunci utama pencarian produk saat transaksi |
| Produk | Harga Jual | Wajib diisi, harus ≥ 0 | Mencegah kesalahan input yang berdampak langsung ke transaksi |
| Produk | Satuan Dasar | Wajib memiliki minimal 1 satuan sebelum dapat ditransaksikan (BRULE-PR-02) | Produk tanpa satuan tidak dapat dihitung dalam transaksi |
| Pelanggan | Nomor Telepon/Identitas | Wajib diisi untuk pelanggan member; format sesuai standar nomor yang berlaku | Digunakan untuk identifikasi program loyalitas |
| Supplier | Nama & Kontak | Wajib diisi | Kebutuhan komunikasi proses pembelian |
| Cabang | Kode Cabang | Wajib diisi, unik | Digunakan sebagai referensi di seluruh transaksi (BRULE-MB-01) |
| Pegawai | Role | Wajib dipilih dari daftar role yang valid, tidak boleh kosong | Setiap pengguna harus memiliki cakupan akses yang jelas |

---

## 3. Validasi Transaksi Penjualan

| Field/Kondisi | Aturan Validasi | Alasan Bisnis |
|---|---|---|
| Item transaksi | Minimal 1 item sebelum transaksi dapat difinalisasi | Transaksi kosong tidak memiliki nilai bisnis |
| Kuantitas item | Harus > 0 dan tidak melebihi stok tersedia (kecuali kebijakan stok minus diaktifkan — BRULE-ST-01) | Mencegah penjualan barang yang secara fisik tidak tersedia |
| Total pembayaran | Harus sama dengan total tagihan (BRULE-TX-02) | Mencegah transaksi tidak seimbang secara finansial |
| Diskon manual | Tidak boleh melebihi 100% dari subtotal item | Mencegah kesalahan input yang menyebabkan kerugian besar |
| Void/Retur | Wajib mengisi alasan sebelum dapat diajukan | Kebutuhan audit trail (BRULE-TX-05) |
| Shift aktif | Transaksi hanya dapat dibuat jika kasir memiliki shift aktif (BRULE-TX-01) | Setiap transaksi harus dapat ditelusuri ke shift tertentu |

---

## 4. Validasi Stok

| Field/Kondisi | Aturan Validasi | Alasan Bisnis |
|---|---|---|
| Penerimaan barang | Jumlah diterima tidak boleh melebihi jumlah pada PO tanpa catatan/approval khusus | Mencegah penerimaan barang di luar kesepakatan dengan supplier |
| Batch/expired | Wajib diisi untuk produk yang dikonfigurasi wajib tracking batch (BRULE-ST-02) | Kebutuhan regulasi dan kontrol FEFO |
| Tanggal kadaluarsa | Tidak boleh berupa tanggal yang telah lewat saat input penerimaan barang baru | Mencegah kesalahan input data yang tidak masuk akal secara bisnis |
| Transfer stok | Jumlah transfer tidak boleh melebihi stok tersedia di cabang asal | Mencegah stok di cabang asal menjadi negatif akibat transfer |
| Penyesuaian stok | Wajib mengisi alasan, nilai penyesuaian tidak boleh 0 | Penyesuaian tanpa perubahan nilai tidak memiliki tujuan bisnis |

---

## 5. Validasi Approval

| Field/Kondisi | Aturan Validasi | Alasan Bisnis |
|---|---|---|
| Approver | Tidak boleh sama dengan pengaju (BRULE-AP-01) | Mencegah self-approval |
| Approver | Harus memiliki role dan cakupan cabang yang sesuai dengan permintaan | Mencegah approval oleh pihak yang tidak berwenang |
| Alasan penolakan | Wajib diisi jika keputusan adalah "Reject" | Kebutuhan transparansi bagi pengaju dan audit |

---

## 6. Validasi Khusus Domain

### 6.1 Apotek

| Field/Kondisi | Aturan Validasi | Alasan Bisnis |
|---|---|---|
| Produk obat keras/resep | Tidak dapat masuk status "selesai transaksi" tanpa validasi Apoteker tercatat (BRULE-PH-01) | Kepatuhan regulasi kefarmasian |
| Batch expired | Batch dengan tanggal kadaluarsa terlewati tidak muncul sebagai opsi alokasi stok saat penjualan (BRULE-PH-02) | Mencegah penjualan produk kadaluarsa |

### 6.2 F&B

| Field/Kondisi | Aturan Validasi | Alasan Bisnis |
|---|---|---|
| Open bill per meja | Satu meja hanya boleh memiliki satu open bill berstatus aktif (BRULE-FB-01) | Mencegah tumpang tindih pesanan pada satu meja |
| Split bill | Total nilai hasil split harus sama dengan total bill asal (BRULE-FB-03) | Mencegah kehilangan/kelebihan nilai saat pembagian tagihan |

### 6.3 Distributor

| Field/Kondisi | Aturan Validasi | Alasan Bisnis |
|---|---|---|
| Transaksi kredit | Total transaksi + piutang berjalan tidak boleh melebihi limit kredit pelanggan tanpa approval (BRULE-CR-01) | Kontrol risiko piutang tak tertagih |
| Pelunasan piutang | Wajib mereferensikan invoice yang valid dan belum lunas penuh (BRULE-CR-02) | Mencegah pencatatan pembayaran yang tidak jelas alokasinya |

---

## 7. Klasifikasi Tingkat Keparahan Validasi

| Tingkat | Deskripsi | Contoh | Perilaku Sistem |
|---|---|---|---|
| **Blocking Error** | Transaksi/aksi tidak dapat dilanjutkan sama sekali | Stok tidak cukup tanpa kebijakan minus, total pembayaran tidak sesuai | Sistem menahan proses hingga input diperbaiki |
| **Warning (dapat dilanjutkan dengan konfirmasi)** | Input tidak ideal namun masih dapat diproses dengan konfirmasi eksplisit | Harga jual di bawah harga beli tanpa flag promo | Sistem menampilkan peringatan, pengguna harus konfirmasi untuk lanjut |
| **Requires Approval** | Input valid secara teknis namun memerlukan otorisasi tambahan | Diskon manual melebihi threshold | Sistem meneruskan ke alur approval, bukan menolak langsung |

---

## 8. Traceability

| Kategori Validasi | Terkait Dokumen |
|---|---|
| Aturan dasar | [`business-rules.md`](business-rules.md) |
| Skenario tidak umum terkait validasi | [`edge-cases.md`](edge-cases.md) |
| Implementasi validasi API | [`api-contract.md`](../technical/api-contract.md) |
| Implementasi logika validasi | [`service-layer-design.md`](../technical/service-layer-design.md) |

---

**Sebelumnya:** [`notifications.md`](notifications.md) — Notifications
**Selanjutnya:** [`edge-cases.md`](edge-cases.md) — Edge Cases
