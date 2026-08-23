# Notifications — POS

> Dokumen ini menjelaskan kebutuhan **notifikasi sistem** — pemicu (trigger), penerima, kanal, dan urgensi tiap notifikasi. Notifikasi adalah mekanisme utama agar masalah operasional **diketahui sedini mungkin**, sejalan dengan BR-08 pada [`business-requirements.md`](business-requirements.md) dan mendukung modul [`approval-flow.md`](approval-flow.md) serta [`dashboard.md`](dashboard.md).

---

## 1. Prinsip Desain Notifikasi

1. **Setiap notifikasi harus actionable** — penerima tahu persis apa yang harus dilakukan setelah menerima notifikasi, bukan sekadar informasi pasif.
2. **Notifikasi ditargetkan ke role/individu yang tepat** — tidak broadcast ke semua orang, untuk mencegah "notification fatigue" yang membuat notifikasi penting terabaikan.
3. **Urgensi menentukan kanal** — notifikasi kritis (approval real-time) menggunakan kanal in-app/push, sedangkan notifikasi ringkasan (laporan harian) dapat menggunakan email.
4. **Notifikasi tidak menggantikan approval flow** — notifikasi adalah pengingat/pemicu tindakan, keputusan tetap dilakukan melalui alur approval resmi di sistem.

---

## 2. Kanal Notifikasi

| Kanal | Kegunaan |
|---|---|
| In-App Notification | Notifikasi real-time dalam aplikasi web/mobile (ikon lonceng, badge) |
| Push Notification (Mobile) | Notifikasi ke perangkat mobile untuk urgensi tinggi (approval pending, stok kritis) |
| Email | Ringkasan berkala (laporan harian/mingguan), notifikasi non-urgent |

Detail implementasi kanal dan strategi pengiriman dijelaskan pada [`module-design.md`](../technical/module-design.md) dan [`logging-strategy.md`](../technical/logging-strategy.md).

---

## 3. Notifikasi Terkait Approval

| ID | Trigger | Penerima | Kanal | Urgensi |
|---|---|---|---|---|
| NOTIF-AP-01 | Permintaan void/diskon manual baru diajukan | Supervisor bertugas di cabang tersebut | In-App, Push | Tinggi (real-time) |
| NOTIF-AP-02 | Permintaan dieskalasi ke level berikutnya | Approver level berikutnya (Branch Manager) | In-App, Push | Tinggi |
| NOTIF-AP-03 | Permintaan disetujui/ditolak | Pengaju (kasir/staff terkait) | In-App | Tinggi |
| NOTIF-AP-04 | Permintaan approval pending melebihi SLA (lihat [`approval-flow.md`](approval-flow.md#6-sla-approval-service-level-expectation)) | Approver saat ini + tembusan Branch Manager | In-App, Push, Email | Tinggi |
| NOTIF-AP-05 | Penyesuaian stok/piutang memerlukan approval | Branch Manager | In-App, Push | Sedang |

---

## 4. Notifikasi Terkait Stok

| ID | Trigger | Penerima | Kanal | Urgensi |
|---|---|---|---|---|
| NOTIF-ST-01 | Stok produk mencapai reorder point (batas minimum) | Branch Manager, Purchasing Staff | In-App, Email | Sedang |
| NOTIF-ST-02 | Produk mendekati tanggal kadaluarsa (sesuai ambang hari yang dikonfigurasi) | Branch Manager, Apoteker (domain Apotek) | In-App, Email | Sedang–Tinggi (tergantung sisa hari) |
| NOTIF-ST-03 | Transfer stok diterima/dikonfirmasi oleh cabang tujuan | Staff Gudang cabang asal | In-App | Rendah |
| NOTIF-ST-04 | Hasil stok opname menunjukkan selisih signifikan | Branch Manager | In-App, Push | Tinggi |

---

## 5. Notifikasi Terkait Kasir & Shift

| ID | Trigger | Penerima | Kanal | Urgensi |
|---|---|---|---|---|
| NOTIF-SH-01 | Shift kasir belum ditutup melewati jam operasional toko | Branch Manager, Supervisor | In-App, Push | Tinggi |
| NOTIF-SH-02 | Selisih kas saat tutup shift melebihi batas toleransi | Supervisor | In-App, Push | Tinggi |
| NOTIF-SH-03 | Kasir dengan frekuensi void/diskon tidak biasa dalam satu shift | Supervisor, Branch Manager | In-App | Sedang |

---

## 6. Notifikasi Terkait Pembelian

| ID | Trigger | Penerima | Kanal | Urgensi |
|---|---|---|---|---|
| NOTIF-PU-01 | Purchase Order berhasil dikirim ke supplier | Branch Manager (pembuat PO) | In-App | Rendah |
| NOTIF-PU-02 | Barang diterima sebagian (partial receipt) | Purchasing Staff, Branch Manager | In-App | Sedang |

---

## 7. Notifikasi Terkait Piutang (Distributor)

| ID | Trigger | Penerima | Kanal | Urgensi |
|---|---|---|---|---|
| NOTIF-CR-01 | Piutang pelanggan mendekati/melewati limit | Branch Manager, Sales terkait | In-App, Email | Sedang |
| NOTIF-CR-02 | Invoice piutang mendekati/melewati jatuh tempo | Finance Auditor, Branch Manager | In-App, Email | Sedang |

---

## 8. Notifikasi Terkait Domain Khusus

| ID | Trigger | Penerima | Kanal | Urgensi | Domain |
|---|---|---|---|---|---|
| NOTIF-FB-01 | Pesanan dapur baru masuk | Kitchen Staff | In-App (kitchen display) | Tinggi | F&B |
| NOTIF-FB-02 | Pesanan dapur selesai diproses | Waiter terkait meja tersebut | In-App | Tinggi | F&B |
| NOTIF-PH-01 | Transaksi obat resep menunggu validasi Apoteker | Apoteker bertugas | In-App, Push | Tinggi | Apotek |

---

## 9. Notifikasi Ringkasan Berkala (Digest)

| ID | Trigger | Penerima | Kanal | Frekuensi |
|---|---|---|---|---|
| NOTIF-DG-01 | Ringkasan penjualan harian cabang | Branch Manager | Email | Harian (akhir hari operasional) |
| NOTIF-DG-02 | Ringkasan konsolidasi seluruh cabang | Owner, HO Admin | Email | Harian/Mingguan (dapat dikonfigurasi) |
| NOTIF-DG-03 | Ringkasan anomali operasional (void tinggi, selisih kas, dsb.) | Owner, Finance Auditor | Email | Mingguan |

**Alasan bisnis:** Notifikasi digest mencegah manajemen harus membuka dashboard setiap saat untuk mendapatkan gambaran umum — sejalan dengan prinsip *actionable* dan mengurangi beban kognitif pengguna level manajemen yang memiliki banyak tanggung jawab.

---

## 10. Matriks Urgensi vs Kanal

| Urgensi | Kanal yang Digunakan | Contoh |
|---|---|---|
| Tinggi | In-App + Push (real-time) | Approval pending, shift belum ditutup, pesanan dapur |
| Sedang | In-App + Email | Stok reorder point, piutang mendekati limit |
| Rendah | In-App saja | PO terkirim, transfer stok diterima |
| Digest | Email terjadwal | Ringkasan harian/mingguan |

---

## 11. Traceability

| Kategori Notifikasi | Terkait Dokumen |
|---|---|
| Approval | [`approval-flow.md`](approval-flow.md) |
| Stok | [`transactions.md`](transactions.md), [`master-data.md`](master-data.md) |
| Shift & Kasir | [`sop.md`](sop.md) |
| Implementasi teknis notifikasi | [`module-design.md`](../technical/module-design.md), [`logging-strategy.md`](../technical/logging-strategy.md) |

---

**Sebelumnya:** [`reports.md`](reports.md) — Reports
**Selanjutnya:** [`validation.md`](validation.md) — Validation
