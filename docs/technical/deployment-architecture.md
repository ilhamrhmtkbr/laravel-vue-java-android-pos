# Deployment Architecture — POS

> Dokumen ini menjelaskan **arsitektur deployment fisik** sistem ke AWS EC2 — topologi infrastruktur, konfigurasi jaringan, dan strategi deployment tanpa downtime signifikan. Dokumen ini adalah realisasi fisik dari [`system-architecture.md`](system-architecture.md) dan menjawab NFR-PD-01 s.d. NFR-PD-03 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md).

---

## 1. Topologi Infrastruktur

### 1.1 Diagram Topologi (Tahap Awal — Single Region)

```
                              Internet
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   Route 53 (DNS)        │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Application Load       │
                    │  Balancer (ALB) / atau  │
                    │  Elastic IP + Nginx     │
                    │  (tergantung skala)     │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
     ┌───────────────────┐┌─────────────────┐┌─────────────────┐
     │  EC2 Instance 1   ││  EC2 Instance 2 ││  EC2 Instance N │
     │  (App Server)     ││  (App Server)   ││  (App Server)   │
     │  - Nginx          ││  - Nginx        ││  - Nginx        │
     │  - PHP-FPM/Laravel││  - PHP-FPM      ││  - PHP-FPM      │
     │  - Queue Worker   ││  (opsional)     ││  (opsional)     │
     └───────────────────┘└─────────────────┘└─────────────────┘
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 ▼
              ┌───────────────────────────────────────┐
              │        VPC Private Subnet             │
              │  ┌──────────────┐   ┌───────────────┐ │
              │  │  EC2 Instance│   │  EC2 Instance │ │
              │  │  PostgreSQL  │   │  Redis        │ │
              │  │  (Primary)   │   │               │ │
              │  └──────┬───────┘   └───────────────┘ │
              │         │                             │
              │  ┌──────▼────────┐                    │
              │  │  EC2 Instance │  (opsional, tahap  │
              │  │  PostgreSQL   │   pertumbuhan)     │
              │  │  (Replica)    │                    │
              │  └───────────────┘                    │
              └───────────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  S3 (Backup & File      │
                    │  Storage untuk laporan  │
                    │  hasil ekspor)          │
                    └─────────────────────────┘
```

### 1.2 Tahapan Skala Infrastruktur

| Tahap | Konfigurasi | Kesesuaian |
|---|---|---|
| **Tahap 1 (UMKM/Startup)** | 1 EC2 instance (all-in-one: Nginx+Laravel+PostgreSQL+Redis via Docker Compose) | Portfolio demo, bisnis skala kecil |
| **Tahap 2 (Growth)** | EC2 App Server terpisah dari EC2 Database (PostgreSQL+Redis) | Bisnis menengah dengan beberapa cabang |
| **Tahap 3 (Enterprise)** | Multiple EC2 App Server di belakang Load Balancer, PostgreSQL dengan read replica, Redis terpisah | Perusahaan besar dengan banyak cabang (target utama project ini) |

**Alasan desain bertahap:** Sesuai prinsip *scalable by design* pada [`README.md`](../../README.md#5-prinsip-desain-sistem), arsitektur harus dapat bertumbuh **tanpa mengubah kode aplikasi** — hanya perubahan pada level infrastruktur/konfigurasi, karena aplikasi (Laravel API) sudah didesain stateless sejak Tahap 1 (lihat [`system-architecture.md`](system-architecture.md#42-skalabilitas-nfr-sc-01-sd-nfr-sc-04)).

---

## 2. Konfigurasi Jaringan (VPC)

| Komponen | Konfigurasi | Alasan |
|---|---|---|
| VPC | Satu VPC dengan public & private subnet | Isolasi jaringan dasar |
| Public Subnet | Berisi Load Balancer/EC2 App Server dengan akses internet | Hanya layer yang perlu diakses publik yang terekspos |
| Private Subnet | Berisi EC2 Database (PostgreSQL, Redis) | Database **tidak pernah** memiliki IP publik langsung, sesuai NFR-SE-01 dan prinsip *least privilege* pada [`security-design.md`](security-design.md) |
| Security Group — App Server | Inbound: 80/443 dari ALB/internet; SSH (22) hanya dari IP admin tertentu | Meminimalkan permukaan serangan |
| Security Group — Database | Inbound: 5432 (PostgreSQL)/6379 (Redis) hanya dari Security Group App Server | Database hanya dapat diakses oleh aplikasi, bukan dari luar VPC |

---

## 3. Strategi Deployment Tanpa Downtime Signifikan

Mengacu pada NFR-PD-02 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md).

### 3.1 Pendekatan: Rolling Deployment (Tahap Multi-Instance)

```
1. Instance baru dengan versi kode terbaru dijalankan (di samping instance lama)
2. Health check dijalankan terhadap instance baru
3. Jika sehat → Load Balancer mulai mengarahkan trafik ke instance baru secara bertahap
4. Instance lama dikeluarkan dari rotasi setelah trafik sepenuhnya berpindah
5. Instance lama dimatikan
```

### 3.2 Pendekatan untuk Tahap Single-Instance (Tahap 1)

Untuk konfigurasi single-instance (UMKM/skala kecil), digunakan pendekatan **graceful restart**:

```
1. Kode baru di-pull ke server
2. Migrasi database dijalankan (dengan strategi migrasi backward-compatible, lihat Bagian 4)
3. `php artisan queue:restart` — worker menyelesaikan job berjalan sebelum restart
4. PHP-FPM di-reload (bukan restart penuh) agar koneksi yang sedang berjalan tetap diselesaikan
```

**Alasan bisnis:** Sesuai catatan pada NFR-PD-02, operasional toko retail berjalan hampir sepanjang hari — downtime bahkan singkat saat jam sibuk berdampak langsung pada transaksi pelanggan yang sedang berjalan.

### 3.3 Strategi Migrasi Database yang Aman

| Prinsip | Penjelasan |
|---|---|
| Migrasi backward-compatible | Kolom baru ditambahkan sebagai nullable terlebih dahulu, tidak langsung mengubah/menghapus kolom yang sedang digunakan versi kode lama |
| Migrasi dijalankan terpisah dari deployment kode | Migrasi dijalankan sebagai langkah eksplisit di pipeline CI/CD sebelum kode baru sepenuhnya aktif (detail di [`cicd-flow.md`](cicd-flow.md)) |

---

## 4. Konfigurasi Nginx sebagai Reverse Proxy

```nginx
# Contoh konfigurasi dasar (disederhanakan)
server {
    listen 443 ssl http2;
    server_name api.pos-domain.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    location /api/ {
        proxy_pass http://laravel_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

upstream laravel_backend {
    server app-instance-1:9000;
    server app-instance-2:9000;  # ditambahkan seiring pertumbuhan (Tahap 3)
}
```

---

## 5. Resource Sizing (Rekomendasi Awal)

| Komponen | Instance Type (Rekomendasi) | Alasan |
|---|---|---|
| App Server (Tahap 1) | t3.medium (2 vCPU, 4GB RAM) | Cukup untuk Nginx + PHP-FPM + Queue Worker skala kecil-menengah |
| Database Server | t3.large (2 vCPU, 8GB RAM), storage EBS gp3 | PostgreSQL membutuhkan memori lebih besar untuk caching query yang efektif |
| Redis Server | t3.small (2 vCPU, 2GB RAM) | Redis efisien memori, kebutuhan cache & queue broker skala menengah tidak butuh instance besar |

**Catatan:** Sizing disesuaikan berdasarkan monitoring aktual (lihat [`logging-strategy.md`](logging-strategy.md)) setelah sistem berjalan di produksi — angka di atas adalah estimasi awal yang wajar, bukan angka final.

---

## 6. Ketersediaan & Redundansi

| Aspek | Strategi |
|---|---|
| Availability Zone | Database utama dan replica (jika ada) ditempatkan di Availability Zone berbeda dalam region yang sama |
| Auto Scaling (Tahap 3) | AWS Auto Scaling Group untuk App Server, menambah/mengurangi instance otomatis berdasarkan beban CPU/memory |
| Health Check | Load Balancer secara berkala memeriksa endpoint `/health` pada tiap instance, mengeluarkan instance yang tidak sehat dari rotasi |

---

## 7. Traceability

| Aspek Deployment | Terkait Dokumen |
|---|---|
| Containerization | [`docker-architecture.md`](docker-architecture.md) |
| Pipeline otomatisasi deployment | [`cicd-flow.md`](cicd-flow.md) |
| Strategi backup | [`backup-strategy.md`](backup-strategy.md) |
| Kebutuhan performa & skalabilitas asal | [`non-functional-requirements.md`](../business/non-functional-requirements.md#3-skalabilitas-scalability) |
| Konfigurasi keamanan jaringan | [`security-design.md`](security-design.md) |

---

**Sebelumnya:** [`security-design.md`](security-design.md) — Security Design
**Selanjutnya:** [`docker-architecture.md`](docker-architecture.md) — Docker Architecture
