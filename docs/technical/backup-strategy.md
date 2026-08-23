# Backup Strategy — POS

> Dokumen ini menjelaskan **strategi backup dan disaster recovery** — bagaimana data dilindungi dari kehilangan, dan bagaimana sistem dapat dipulihkan jika terjadi kegagalan. Dokumen ini menjawab NFR-AV-04 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md) dan risiko bisnis "kehilangan data akibat gangguan sistem" pada [`business-requirements.md`](../business/business-requirements.md#8-risiko-bisnis-business-risks).

---

## 1. Prinsip Strategi Backup

1. **3-2-1 Rule** — minimal 3 salinan data, disimpan di 2 media/lokasi berbeda, dengan 1 salinan di luar lokasi utama (off-site).
2. **Backup diuji secara berkala**, bukan hanya dibuat lalu diasumsikan berfungsi — backup yang tidak pernah diuji restore-nya sama saja tidak ada.
3. **RPO dan RTO didefinisikan eksplisit** — target maksimum data yang boleh hilang (Recovery Point Objective) dan waktu maksimum pemulihan (Recovery Time Objective) harus jelas, bukan asumsi implisit.
4. **Data transaksional finansial mendapat prioritas backup tertinggi** — sejalan dengan sifat kritis data pada [`transactions.md`](../business/transactions.md).

---

## 2. Target RPO & RTO

| Komponen | RPO (Recovery Point Objective) | RTO (Recovery Time Objective) |
|---|---|---|
| Database PostgreSQL (data transaksi) | ≤ 15 menit | ≤ 1 jam |
| Redis (cache & session) | Tidak kritis — dapat dibangun ulang dari database | ≤ 15 menit (restart service) |
| File hasil ekspor laporan | ≤ 24 jam | ≤ 4 jam |
| Konfigurasi & kode aplikasi | Real-time (via Git & CI/CD) | ≤ 30 menit (redeploy dari image terakhir) |

**Alasan RPO 15 menit untuk database:** Mengacu pada kebutuhan bisnis bahwa **data transaksi tidak boleh hilang** (BR-03 pada [`business-requirements.md`](../business/business-requirements.md)) — kehilangan data hingga 15 menit terakhir transaksi (skenario terburuk) dianggap batas maksimum yang dapat diterima secara operasional, sementara RPO mendekati nol akan memerlukan investasi infrastruktur (misal synchronous replication) yang belum sepadan pada skala project ini.

---

## 3. Strategi Backup PostgreSQL

### 3.1 Kombinasi Full Backup + Write-Ahead Log (WAL) Archiving

```
┌─────────────────────────────────────────────────────────────┐
│  Full Backup (pg_basebackup)                                  │
│  - Dijalankan setiap hari (jam operasional rendah, mis. 02:00)│
│  - Disimpan di S3, retensi 30 hari                             │
└─────────────────────────────────────────────────────────────┘
                            +
┌─────────────────────────────────────────────────────────────┐
│  WAL Archiving (Continuous)                                   │
│  - Setiap WAL segment yang selesai ditulis, langsung           │
│    disalin ke S3 (near real-time)                              │
│  - Memungkinkan Point-in-Time Recovery (PITR)                  │
└─────────────────────────────────────────────────────────────┘
```

**Alasan menggunakan WAL Archiving, bukan hanya full backup harian:** Full backup harian saja hanya memberikan RPO 24 jam (kehilangan hingga satu hari transaksi jika gagal tepat sebelum backup berikutnya) — jauh dari target RPO 15 menit. Dengan WAL archiving kontinu, sistem dapat dipulihkan ke titik waktu spesifik mendekati saat kegagalan terjadi (Point-in-Time Recovery).

### 3.2 Konfigurasi Contoh

```bash
# postgresql.conf
archive_mode = on
archive_command = 'aws s3 cp %p s3://pos-backup-bucket/wal-archive/%f'
wal_level = replica
```

### 3.3 Jadwal Backup

| Jenis Backup | Frekuensi | Retensi |
|---|---|---|
| Full backup | Harian (02:00 waktu setempat, jam operasional rendah) | 30 hari |
| WAL archiving | Kontinu (near real-time) | 7 hari (cukup untuk PITR ke full backup terdekat) |
| Backup mingguan (arsip jangka panjang) | Mingguan | 12 bulan, dipindah ke S3 Glacier setelah 30 hari |

---

## 4. Strategi Redundansi: Read Replica (Tahap Pertumbuhan)

Mengacu pada topologi Tahap 3 pada [`deployment-architecture.md`](deployment-architecture.md#12-tahapan-skala-infrastruktur), sistem dapat dikonfigurasi dengan **PostgreSQL streaming replication** (read replica) di Availability Zone berbeda:

```
Primary DB (AZ-1) ──streaming replication──▶ Replica DB (AZ-2)
```

**Manfaat ganda:**
1. **Disaster recovery** — jika Primary DB gagal total, replica dapat di-promote menjadi primary baru dengan downtime minimal.
2. **Load distribution** — query laporan berat (lihat [`repository-layer-design.md`](repository-layer-design.md#5-repository-untuk-kebutuhan-laporan-read-optimized-queries)) dapat diarahkan ke replica, mengurangi beban pada database primary yang melayani transaksi real-time.

---

## 5. Backup File Storage (Laporan & Aset)

| Aspek | Strategi |
|---|---|
| Lokasi penyimpanan utama | AWS S3 (durabilitas 99.999999999%) |
| Versioning | S3 Object Versioning diaktifkan — mencegah kehilangan akibat overwrite/delete tidak sengaja |
| Lifecycle policy | File berusia > 90 hari dipindah otomatis ke S3 Infrequent Access; > 1 tahun ke Glacier |

---

## 6. Prosedur Disaster Recovery

### 6.1 Skenario: Kegagalan Total Database Primary

```
1. Deteksi kegagalan (via health check & monitoring — lihat logging-strategy.md)
2. Jika ada replica → promote replica menjadi primary baru
3. Update konfigurasi DB_HOST aplikasi mengarah ke database baru
4. Jika tidak ada replica → restore dari full backup terakhir + replay WAL hingga
   titik waktu sedekat mungkin dengan kegagalan (PITR)
5. Verifikasi integritas data (jumlah transaksi, checksum tabel kritis)
6. Aktifkan kembali trafik aplikasi ke database yang telah pulih
7. Dokumentasikan insiden (root cause, durasi downtime, data yang mungkin hilang)
```

### 6.2 Skenario: Kegagalan EC2 App Server

```
1. Load Balancer secara otomatis mengeluarkan instance gagal dari rotasi (health check)
2. Traffic dialihkan ke instance sehat lainnya (jika konfigurasi multi-instance — Tahap 3)
3. Instance baru di-provision otomatis (Auto Scaling Group) menggantikan yang gagal
4. Untuk konfigurasi single-instance (Tahap 1) → restart manual/otomatis via Docker restart policy
```

### 6.3 Uji Coba Disaster Recovery (DR Drill)

**Kebijakan:** DR drill (simulasi pemulihan dari backup) dilakukan secara **berkala** (disarankan setiap kuartal) pada environment staging — memverifikasi bahwa backup benar-benar dapat direstore dan RTO yang ditargetkan realistis, bukan hanya asumsi di atas kertas.

---

## 7. Backup Konfigurasi & Secret

| Aspek | Strategi |
|---|---|
| Kode aplikasi | Tersimpan di GitHub (sudah terdistribusi secara inheren), tag/release untuk setiap versi produksi |
| Docker image | Tersimpan di container registry dengan tag versi (`IMAGE_TAG`), memungkinkan rollback cepat ke versi sebelumnya |
| Secret (kredensial DB, API key) | Dikelola melalui GitHub Secrets & environment variable server, didokumentasikan strukturnya (bukan nilainya) di runbook internal tim |

---

## 8. Traceability

| Aspek Backup | Terkait Dokumen |
|---|---|
| Kebutuhan ketersediaan asal | [`non-functional-requirements.md`](../business/non-functional-requirements.md#4-ketersediaan--keandalan-availability--reliability) |
| Infrastruktur tempat backup berjalan | [`deployment-architecture.md`](deployment-architecture.md) |
| Monitoring kegagalan | [`logging-strategy.md`](logging-strategy.md) |
| Rollback deployment | [`cicd-flow.md`](cicd-flow.md) |

---

**Sebelumnya:** [`logging-strategy.md`](logging-strategy.md) — Logging Strategy
**Selanjutnya:** [`testing-strategy.md`](testing-strategy.md) — Testing Strategy
