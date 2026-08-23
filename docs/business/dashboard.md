# Dashboard — POS

> Dokumen ini menjelaskan kebutuhan **dashboard dan KPI (Key Performance Indicator)** untuk masing-masing peran pengguna, mengacu pada [`role-permission.md`](role-permission.md) dan [`business-requirements.md`](business-requirements.md#2-tujuan-bisnis-business-goals) (khususnya BG-01 dan BG-05: konsolidasi data real-time dan percepatan pelaporan). Dokumen ini menjadi acuan untuk desain API laporan pada [`api-contract.md`](../technical/api-contract.md) dan struktur data pada [`database-design.md`](../technical/database-design.md).

---

## 1. Prinsip Desain Dashboard

1. **Setiap role melihat dashboard yang relevan dengan tanggung jawabnya** — bukan satu dashboard generik untuk semua orang.
2. **Data harus real-time atau near real-time** — sesuai BR-05, laporan manual/batch tidak dapat memenuhi kebutuhan kecepatan pengambilan keputusan.
3. **KPI harus actionable** — setiap angka yang ditampilkan harus dapat ditindaklanjuti (misal: stok menipis → tombol langsung ke buat PO), bukan sekadar informasi pasif.
4. **Data discoping sesuai cabang** — dashboard cabang hanya menampilkan data cabang tersebut kecuali untuk role level pusat (lihat [`role-permission.md`](role-permission.md#5-aturan-data-scoping)).

---

## 2. Dashboard: Owner / Direktur

**Tujuan:** Visibilitas strategis performa seluruh perusahaan.

| KPI/Widget | Deskripsi | Alasan Bisnis |
|---|---|---|
| Total Penjualan (Hari Ini/Minggu Ini/Bulan Ini) — Konsolidasi | Total omzet seluruh cabang, dapat difilter per cabang | Indikator utama kesehatan bisnis harian |
| Perbandingan Penjualan Antar Cabang | Ranking cabang berdasarkan omzet/pertumbuhan | Identifikasi cabang berkinerja baik/buruk |
| Tren Penjualan (Grafik) | Grafik penjualan 30/90 hari terakhir | Melihat pola musiman dan tren pertumbuhan |
| Produk Terlaris (Top Products) | Ranking produk berdasarkan kuantitas/omzet | Insight untuk strategi stok dan promosi |
| Nilai Persediaan Total | Total nilai stok seluruh cabang | Indikator modal kerja yang tertahan di stok |
| Cabang dengan Anomali | Cabang dengan tingkat void/retur di atas rata-rata | Deteksi dini potensi kecurangan atau masalah operasional |

### Alur Interaksi
```
Owner login → Dashboard Konsolidasi tampil default →
Filter periode/cabang (opsional) → Drill down ke laporan detail bila diperlukan
```

---

## 3. Dashboard: Head Office Admin

**Tujuan:** Kontrol data master dan kebijakan pusat.

| KPI/Widget | Deskripsi | Alasan Bisnis |
|---|---|---|
| Status Perubahan Harga Aktif | Daftar perubahan harga yang sedang berjalan/terjadwal | Memastikan kebijakan harga terpusat berjalan sesuai rencana |
| Promo Aktif & Performanya | Daftar promo berjalan beserta jumlah penggunaan | Evaluasi efektivitas promo |
| Produk Baru/Perubahan Master Data Terbaru | Log aktivitas perubahan data master | Kontrol kualitas data master |
| Ringkasan Konsolidasi Multi-Cabang | Sama seperti Owner namun dengan akses ke detail konfigurasi | Kebutuhan operasional pusat sehari-hari |

---

## 4. Dashboard: Branch Manager

**Tujuan:** Kontrol operasional harian cabang.

| KPI/Widget | Deskripsi | Alasan Bisnis |
|---|---|---|
| Penjualan Hari Ini (Cabang) | Total omzet, jumlah transaksi, rata-rata nilai transaksi | Pemantauan performa harian cabang |
| Status Shift Kasir Aktif | Daftar shift yang sedang berjalan dan yang belum ditutup | Memastikan tidak ada shift "menggantung" |
| Approval Pending | Daftar permintaan approval yang menunggu keputusan | Aksi cepat mengurangi bottleneck operasional |
| Stok Menipis (Reorder Alert) | Produk yang mencapai batas minimum stok | Trigger pembuatan PO tepat waktu |
| Produk Mendekati Kadaluarsa | Daftar produk dengan expired date dekat | Mitigasi kerugian akibat barang kadaluarsa |
| Perbandingan Penjualan vs Target (jika ada) | Progress terhadap target bulanan cabang | Motivasi dan evaluasi kinerja cabang |

### Alur Interaksi
```
Branch Manager login → Dashboard cabang tampil →
Lihat approval pending → Klik untuk review & putuskan →
Lihat stok menipis → Klik untuk buat PO langsung
```

---

## 5. Dashboard: Supervisor

**Tujuan:** Pengawasan shift dan transaksi real-time.

| KPI/Widget | Deskripsi | Alasan Bisnis |
|---|---|---|
| Shift Aktif Saat Ini | Daftar kasir yang sedang bertugas | Pengawasan langsung operasional kasir |
| Antrian Approval | Permintaan void/diskon yang menunggu keputusan Supervisor | Aksi cepat, mengurangi waktu tunggu pelanggan |
| Transaksi dengan Frekuensi Void/Diskon Tinggi | Highlight kasir dengan pola transaksi tidak biasa | Deteksi dini potensi kecurangan |

---

## 6. Dashboard: Kasir

**Tujuan:** Fokus pada transaksi, minim distraksi data analitik.

| Widget | Deskripsi | Alasan Bisnis |
|---|---|---|
| Ringkasan Shift Berjalan | Total transaksi & total penjualan sejak shift dibuka | Kasir dapat memantau progress shift sendiri |
| Notifikasi Approval Masuk | Status permintaan void/diskon yang diajukan | Transparansi status pengajuan tanpa perlu bertanya ke Supervisor |

**Alasan bisnis:** Kasir tidak membutuhkan dashboard analitik kompleks — kebutuhan utamanya adalah kecepatan transaksi (sesuai NFR-US-01), sehingga dashboard dibuat seminimal mungkin.

---

## 7. Dashboard: Staff Gudang

**Tujuan:** Kontrol pergerakan stok.

| KPI/Widget | Deskripsi | Alasan Bisnis |
|---|---|---|
| PO Menunggu Penerimaan | Daftar Purchase Order yang belum sepenuhnya diterima | Prioritas kerja harian staff gudang |
| Transfer Stok Masuk/Keluar Pending | Status transfer yang perlu ditindaklanjuti | Mempercepat siklus distribusi stok |
| Jadwal Stok Opname Berikutnya | Pengingat jadwal opname sesuai kebijakan cabang | Kepatuhan terhadap SOP stok opname |

---

## 8. Dashboard: Finance Auditor

**Tujuan:** Rekonsiliasi dan audit lintas cabang.

| KPI/Widget | Deskripsi | Alasan Bisnis |
|---|---|---|
| Rekap Selisih Kas Seluruh Cabang | Daftar selisih kas dari seluruh shift yang ditutup | Fokus audit pada cabang dengan pola selisih berulang |
| Log Audit Perubahan Data Sensitif | Perubahan harga, void, penyesuaian stok, dsb. | Kebutuhan investigasi dan kepatuhan (lihat NFR-AU-01) |
| Status Piutang Jatuh Tempo (Distributor) | Ringkasan piutang yang mendekati/lewat jatuh tempo | Pemantauan risiko kredit pelanggan |

---

## 9. Dashboard Domain Khusus

### 9.1 F&B — Dashboard Status Meja (Waiter/Kasir)

| Widget | Deskripsi |
|---|---|
| Peta Status Meja | Visualisasi meja kosong/terisi/menunggu pembayaran |
| Antrian Pesanan Dapur | Daftar pesanan yang sedang diproses/selesai di dapur |

### 9.2 Apotek — Dashboard Kepatuhan (Apoteker/Branch Manager)

| Widget | Deskripsi |
|---|---|
| Produk Mendekati Kadaluarsa (Prioritas Tinggi) | Highlight khusus untuk obat, dengan urgensi lebih tinggi dibanding retail umum |
| Log Validasi Resep | Riwayat validasi resep yang telah dilakukan |

---

## 10. Ringkasan Pemetaan Dashboard per Role

| Role | Dashboard Utama |
|---|---|
| Owner | Konsolidasi seluruh cabang |
| HO Admin | Kontrol data master & kebijakan pusat |
| Branch Manager | Operasional harian cabang |
| Supervisor | Pengawasan shift & approval real-time |
| Kasir | Ringkasan shift pribadi |
| Staff Gudang | Pergerakan stok & PO |
| Finance Auditor | Rekonsiliasi & audit lintas cabang |
| Apoteker | Kepatuhan & expired produk |
| Waiter | Status meja & pesanan |

---

## 11. Traceability

| Dashboard | Terkait Dokumen |
|---|---|
| Sumber data KPI | [`reports.md`](reports.md), [`transactions.md`](transactions.md) |
| Implementasi API | [`api-contract.md`](../technical/api-contract.md) |
| Struktur data pendukung | [`database-design.md`](../technical/database-design.md) |
| Batasan akses data | [`role-permission.md`](role-permission.md) |

---

**Sebelumnya:** [`approval-flow.md`](approval-flow.md) — Approval Flow
**Selanjutnya:** [`reports.md`](reports.md) — Reports
