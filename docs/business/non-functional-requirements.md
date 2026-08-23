# Non-Functional Requirements — POS

> Dokumen ini menjelaskan **kebutuhan non-fungsional (NFR)** — yaitu kualitas sistem yang tidak berkaitan langsung dengan fitur, namun menentukan apakah sistem **layak digunakan perusahaan besar dengan banyak cabang** atau tidak. NFR pada dokumen ini menjadi acuan langsung bagi keputusan arsitektur pada [`docs/technical/system-architecture.md`](../technical/system-architecture.md).

---

## 1. Mengapa NFR Penting untuk Project Ini

Functional Requirement menjawab "apa yang bisa dilakukan sistem". Non-Functional Requirement menjawab "**seberapa baik** sistem melakukannya, dalam kondisi nyata seperti apa, dan dengan jaminan apa". Untuk sistem yang menangani **transaksi finansial** dan digunakan **serentak oleh banyak cabang**, NFR yang lemah dapat berakibat langsung pada kerugian bisnis — bukan sekadar bug teknis.

Setiap NFR diberi ID (`NFR-xxx`), dikelompokkan per kategori kualitas (mengacu pendekatan umum: performa, skalabilitas, keandalan, keamanan, dst.).

---

## 2. Performa (Performance)

| ID | Requirement | Target Terukur | Justifikasi Bisnis |
|---|---|---|---|
| NFR-PF-01 | Waktu respons transaksi kasir (submit penjualan) | ≤ 1 detik pada kondisi normal | Antrian kasir berdampak langsung pada kepuasan pelanggan di toko |
| NFR-PF-02 | Waktu muat (load) dashboard ringkasan | ≤ 2 detik untuk data harian, ≤ 5 detik untuk data konsolidasi multi-cabang | Manajemen membutuhkan data cepat untuk keputusan operasional harian |
| NFR-PF-03 | Waktu respons API rata-rata (p95) | ≤ 300ms untuk operasi baca (read), ≤ 800ms untuk operasi tulis (write) | Konsistensi pengalaman pengguna di seluruh cabang |
| NFR-PF-04 | Generasi laporan bulanan/konsolidasi besar | Dapat diproses secara asynchronous (background job), notifikasi saat selesai | Laporan besar tidak boleh memblokir operasional real-time sistem |

---

## 3. Skalabilitas (Scalability)

| ID | Requirement | Justifikasi Bisnis |
|---|---|---|
| NFR-SC-01 | Sistem harus mampu menangani penambahan cabang baru tanpa perubahan arsitektur | Ekspansi bisnis adalah hal wajar bagi perusahaan retail yang berkembang |
| NFR-SC-02 | Sistem harus mampu menangani transaksi konkuren dari ratusan kasir secara bersamaan pada jam sibuk | Jam sibuk (peak hour) adalah kondisi normal, bukan kondisi ekstrem, bagi retail |
| NFR-SC-03 | Arsitektur backend harus mendukung horizontal scaling (penambahan instance/server) | Beban sistem meningkat seiring pertumbuhan jumlah cabang & transaksi |
| NFR-SC-04 | Lapisan database harus mendukung strategi optimasi untuk volume data transaksi yang terus bertambah (indexing, partitioning) | Data transaksi retail bertumbuh secara linear terhadap waktu dan jumlah cabang |

---

## 4. Ketersediaan & Keandalan (Availability & Reliability)

| ID | Requirement | Target | Justifikasi Bisnis |
|---|---|---|---|
| NFR-AV-01 | Ketersediaan sistem (uptime) | ≥ 99.5% per bulan | Downtime langsung menghentikan transaksi penjualan di seluruh cabang |
| NFR-AV-02 | Sistem kasir harus tetap dapat mencatat transaksi meski koneksi ke server pusat terputus sementara, dan melakukan sinkronisasi otomatis saat pulih | Operasional toko tidak boleh berhenti karena gangguan jaringan |
| NFR-AV-03 | Sistem harus memiliki mekanisme pemulihan otomatis (auto-recovery) untuk service yang gagal | Meminimalkan campur tangan manual saat terjadi gangguan |
| NFR-AV-04 | Data transaksi tidak boleh hilang meski terjadi kegagalan sistem mendadak | Data transaksi finansial bersifat kritis dan tidak dapat direkonstruksi begitu saja |

---

## 5. Keamanan (Security)

| ID | Requirement | Justifikasi Bisnis |
|---|---|---|
| NFR-SE-01 | Seluruh komunikasi data harus terenkripsi (HTTPS/TLS) | Data transaksi dan pelanggan bersifat sensitif |
| NFR-SE-02 | Password pengguna harus disimpan dalam bentuk hash yang aman, tidak pernah dalam bentuk plain text | Standar keamanan dasar untuk sistem enterprise |
| NFR-SE-03 | Sistem harus menerapkan otorisasi berbasis role & permission pada setiap endpoint API | Mencegah akses tidak sah ke data/fitur sensitif |
| NFR-SE-04 | Sistem harus memiliki mekanisme proteksi terhadap serangan umum (SQL Injection, XSS, CSRF, brute force login) | Kepatuhan terhadap prinsip keamanan aplikasi web standar industri |
| NFR-SE-05 | Sesi login harus memiliki mekanisme expiry & dapat dicabut (revoke) oleh admin | Mitigasi risiko akun disalahgunakan setelah pegawai resign/berpindah |
| NFR-SE-06 | Setiap akses ke data harus dibatasi sesuai cabang penempatan pengguna (data scoping) | Mencegah kebocoran data antar cabang yang tidak berkepentingan |

---

## 6. Audit & Kepatuhan (Auditability & Compliance)

| ID | Requirement | Justifikasi Bisnis |
|---|---|---|
| NFR-AU-01 | Setiap perubahan pada data master, harga, dan stok harus tercatat dalam audit log (siapa, kapan, nilai sebelum/sesudah) | Kebutuhan investigasi saat terjadi selisih atau kecurangan |
| NFR-AU-02 | Log audit tidak boleh dapat diubah atau dihapus oleh pengguna biasa | Menjaga integritas bukti audit |
| NFR-AU-03 | Sistem harus menyimpan histori transaksi minimal sesuai kebutuhan retensi bisnis (dikonfigurasi, default disarankan ≥ 5 tahun untuk data finansial) | Kepatuhan terhadap praktik retensi data keuangan yang umum berlaku |

---

## 7. Maintainability & Extensibility

| ID | Requirement | Justifikasi Bisnis/Teknis |
|---|---|---|
| NFR-MT-01 | Kode backend harus mengikuti pemisahan lapisan (layered architecture: controller, service, repository) | Memudahkan pemeliharaan dan onboarding developer baru |
| NFR-MT-02 | Sistem harus mendukung penambahan domain bisnis baru (mis. domain usaha baru) tanpa mengubah struktur inti | Fleksibilitas produk untuk berbagai jenis usaha adalah kebutuhan bisnis inti (BR-07) |
| NFR-MT-03 | Setiap modul harus memiliki cakupan pengujian otomatis (automated test) yang memadai | Perubahan pada satu modul tidak boleh merusak modul lain tanpa terdeteksi |
| NFR-MT-04 | Dokumentasi API harus selalu sinkron dengan implementasi | Mengurangi miskomunikasi antara tim backend, frontend, dan mobile |

---

## 8. Usability

| ID | Requirement | Justifikasi Bisnis |
|---|---|---|
| NFR-US-01 | Antarmuka kasir harus dapat dioperasikan dengan pelatihan minimal (< 1 hari) | Turnover kasir di industri retail tergolong tinggi, sistem harus cepat dipelajari |
| NFR-US-02 | Antarmuka harus responsif pada berbagai ukuran layar (desktop, tablet) untuk kebutuhan back-office maupun kasir | Perangkat di lapangan bervariasi tergantung skala cabang |
| NFR-US-03 | Aplikasi mobile harus tetap dapat digunakan pada perangkat Android kelas menengah-bawah yang umum dipakai di toko | Efisiensi biaya investasi perangkat bagi pemilik usaha |

---

## 9. Portabilitas & Deployment

| ID | Requirement | Justifikasi Bisnis/Teknis |
|---|---|---|
| NFR-PD-01 | Seluruh komponen sistem harus dapat dijalankan dalam container (Docker) | Konsistensi environment dari development hingga production |
| NFR-PD-02 | Proses deployment harus dapat dilakukan tanpa downtime signifikan (mendekati zero-downtime deployment) | Operasional toko berjalan hampir 24/7, deployment tidak boleh mengganggu jam operasional |
| NFR-PD-03 | Sistem harus mendukung deployment terpisah antar layer (backend, frontend, database) untuk fleksibilitas scaling | Setiap layer memiliki karakteristik beban yang berbeda |

---

## 10. Observability

| ID | Requirement | Justifikasi Bisnis/Teknis |
|---|---|---|
| NFR-OB-01 | Sistem harus mencatat log terstruktur untuk seluruh request API dan error | Mempercepat investigasi saat terjadi insiden produksi |
| NFR-OB-02 | Sistem harus menyediakan mekanisme monitoring kesehatan service (health check) | Memungkinkan deteksi dini sebelum gangguan dirasakan pengguna |

---

## 11. Ringkasan Prioritas NFR Kritis

Untuk konteks "perusahaan besar dengan banyak cabang", NFR berikut dianggap **non-negotiable** (harus dipenuhi sejak versi awal, bukan iterasi berikutnya):

- NFR-PF-01, NFR-PF-03 (performa transaksi kasir)
- NFR-AV-01, NFR-AV-02, NFR-AV-04 (ketersediaan & resiliensi data)
- NFR-SE-01 s.d. NFR-SE-06 (keamanan menyeluruh)
- NFR-AU-01, NFR-AU-02 (audit trail)

---

## 12. Traceability ke Dokumentasi Teknis

| Kategori NFR | Diimplementasikan Pada |
|---|---|
| Performa & Skalabilitas | [`system-architecture.md`](../technical/system-architecture.md), [`deployment-architecture.md`](../technical/deployment-architecture.md) |
| Ketersediaan & Keandalan | [`system-architecture.md`](../technical/system-architecture.md), [`backup-strategy.md`](../technical/backup-strategy.md) |
| Keamanan | [`security-design.md`](../technical/security-design.md), [`authentication-design.md`](../technical/authentication-design.md), [`authorization-design.md`](../technical/authorization-design.md) |
| Audit & Kepatuhan | [`logging-strategy.md`](../technical/logging-strategy.md), [`database-design.md`](../technical/database-design.md) |
| Maintainability | [`module-design.md`](../technical/module-design.md), [`service-layer-design.md`](../technical/service-layer-design.md), [`testing-strategy.md`](../technical/testing-strategy.md) |
| Portabilitas & Deployment | [`docker-architecture.md`](../technical/docker-architecture.md), [`cicd-flow.md`](../technical/cicd-flow.md) |
| Observability | [`logging-strategy.md`](../technical/logging-strategy.md) |

---

**Sebelumnya:** [`functional-requirements.md`](functional-requirements.md) — Functional Requirements
**Selanjutnya:** [`business-flow.md`](business-flow.md) — Business Flow
