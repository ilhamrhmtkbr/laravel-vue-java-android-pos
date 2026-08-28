# System Architecture — POS

> Dokumen ini menjelaskan **arsitektur sistem secara keseluruhan** — pola arsitektur yang dipilih, komponen utama, bagaimana komponen berkomunikasi, serta alasan teknis di balik setiap keputusan. Dokumen ini adalah fondasi bagi seluruh dokumen teknis lainnya, dan secara langsung menjawab kebutuhan pada [`business-requirements.md`](../business/business-requirements.md) serta [`non-functional-requirements.md`](../business/non-functional-requirements.md).

---

## 1. Pola Arsitektur yang Dipilih

### 1.1 Keputusan: Modular Monolith (bukan Microservices)

Sistem ini dibangun sebagai **modular monolith** — satu aplikasi Laravel API dengan pemisahan modul yang tegas secara kode (lihat [`module-design.md`](module-design.md)), bukan microservices dengan banyak deployment terpisah.

| Pertimbangan | Modular Monolith (Dipilih) | Microservices (Tidak Dipilih untuk v1) |
|---|---|---|
| Kompleksitas operasional | Rendah — satu deployment unit | Tinggi — perlu service discovery, distributed tracing, banyak pipeline |
| Konsistensi transaksi lintas modul | Mudah (database transaction lokal) | Sulit (perlu saga pattern/distributed transaction) |
| Kecepatan development awal | Cepat | Lambat (overhead infrastruktur) |
| Skalabilitas | Scaling horizontal pada level aplikasi (banyak instance yang sama) | Scaling granular per service |
| Kesesuaian dengan tim & skala project | Sesuai untuk tim menengah, tahap awal-menengah | Lebih sesuai untuk organisasi besar dengan banyak tim independen |

**Alasan bisnis & teknis:** Modul-modul dalam sistem POS (penjualan, stok, approval, keuangan) **saling terkait erat dan sering membutuhkan transaksi database yang konsisten** (misal: transaksi penjualan harus atomically mengurangi stok). Microservices akan memperkenalkan kompleksitas distributed transaction yang tidak sepadan dengan manfaatnya pada skala project ini. Modular monolith tetap memungkinkan **migrasi bertahap ke microservices di masa depan** jika skala benar-benar membutuhkannya, karena modul sudah dipisah secara tegas sejak awal (prinsip *evolutionary architecture*).

### 1.2 Keputusan: API-First Architecture

Backend Laravel **murni berfungsi sebagai REST API**, tidak merender view HTML (tidak menggunakan Blade untuk UI aplikasi). Frontend (Vue) dan Mobile (Android) adalah **klien independen** yang mengonsumsi API yang sama.

**Alasan teknis:** Mendukung BR-06 dan kebutuhan multi-platform (web + mobile) tanpa duplikasi business logic — satu sumber kebenaran untuk seluruh aturan bisnis ada di backend API.

### 1.3 Keputusan: Layered Architecture pada Backend

```
Request → Route → Controller → Service (Business Logic) → Repository (Data Access) → Model/Database
                       │                    │
                       ▼                    ▼
                  Request Validation   Domain Events (untuk notifikasi, audit log)
```

Detail lengkap tiap layer dijelaskan pada [`service-layer-design.md`](service-layer-design.md) dan [`repository-layer-design.md`](repository-layer-design.md).

**Alasan teknis:** Sejalan dengan NFR-MT-01 — pemisahan lapisan memudahkan pengujian (service dapat diuji tanpa HTTP layer), memudahkan onboarding developer baru, dan mencegah "fat controller" yang sulit dipelihara.

---

## 2. Komponen Utama Sistem

### 2.1 Diagram Komponen

```
┌────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                              │
│  ┌───────────────┐       ┌──────────────┐        ┌──────────────┐      │
│  │  Vue Web App  │       │ Android App  │        │ (Future:     │      │
│  │  - Back Office│       │ - Kasir      │        │  Partner API)│      │
│  │  - Kasir Web  │       │   Lapangan   │        │              │      │
│  └───────┬───────┘       └───────┬──────┘        └───────┬──────┘      │
└──────────┼───────────────────────┼───────────────────────┼─────────────┘
           │                       │                       │
           └───────────────────────┼───────────────────────┘
                                    ▼
                        ┌─────────────────────────┐
                        │   Nginx (Reverse Proxy) │
                        │   - SSL Termination     │
                        │   - Load Balancing      │
                        │   - Rate Limiting (edge)│
                        └───────────┬─────────────┘
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        APPLICATION LAYER                               │
│                  ┌────────────────────────────┐                        │
│                  │      Laravel API (PHP-FPM) │                        │
│                  │  ┌──────────────────────┐  │                        │
│                  │  │ Controllers          │  │                        │
│                  │  ├──────────────────────┤  │                        │
│                  │  │ Services (Business   │  │                        │
│                  │  │ Logic)               │  │                        │
│                  │  ├──────────────────────┤  │                        │
│                  │  │ Repositories         │  │                        │
│                  │  ├──────────────────────┤  │                        │
│                  │  │ Models (Eloquent)    │  │                        │
│                  │  └──────────────────────┘  │                        │
│                  └───────────┬────────────────┘                        │
│                              │                                         │
│                  ┌───────────┴───────────┐                             │
│                  ▼                       ▼                             │
│         ┌──────────────────┐      ┌─────────────────┐                  │
│         │  Queue Worker    │      │  Scheduler      │                  │
│         │  (Laravel Queue) │      │  (Laravel       │                  │
│         │  - Notifikasi    │      │   Scheduler)    │                  │
│         │  - Laporan besar │      │  - Reminder     │                  │
│         │  - Sinkronisasi  │      │    kadaluarsa   │                  │
│         └──────────────────┘      └─────────────────┘                  │
└────────────────────────────────────────────────────────────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │ PostgreSQL       │  │      Redis       │  │  File Storage    │
    │(Primary Database)│  │  - Cache         │  │  (Local Volume/  │
    │                  │  │  - Session Store │  │   S3-compatible  │
    │                  │  │  - Queue Broker  │  │   for reports)   │
    └──────────────────┘  └──────────────────┘  └──────────────────┘
```

### 2.2 Tanggung Jawab Tiap Komponen

| Komponen | Tanggung Jawab |
|---|---|
| Nginx | Reverse proxy, terminasi SSL, load balancing antar instance Laravel, rate limiting dasar di edge |
| Laravel API (PHP-FPM) | Seluruh business logic, validasi, otorisasi, komunikasi dengan database |
| Queue Worker | Memproses job asynchronous: pengiriman notifikasi, generasi laporan besar, proses yang tidak boleh memblokir request utama |
| Scheduler | Menjalankan tugas terjadwal: pengecekan produk mendekati kadaluarsa, reminder SLA approval, pembersihan data sementara |
| PostgreSQL | Penyimpanan data utama (transaksional & master data) |
| Redis | Cache aplikasi, penyimpanan sesi (agar API stateless di level instance), broker untuk queue |
| File Storage | Penyimpanan file hasil ekspor laporan (PDF/Excel), didesain agar dapat dipindah ke S3-compatible storage saat skala bertambah |

---

## 3. Alasan Teknis per Keputusan Teknologi

### 3.1 Laravel (Backend)

- Ekosistem lengkap untuk kebutuhan enterprise: Eloquent ORM, Queue, Scheduler, Event/Listener, built-in testing tools.
- Dukungan native untuk Sanctum (token-based auth) yang sesuai kebutuhan API-first multi-klien (lihat [`authentication-design.md`](authentication-design.md)).
- Ekosistem package matang untuk kebutuhan enterprise seperti audit log, permission management, dan report generation.

### 3.2 PostgreSQL (Database)

- **ACID compliant penuh** — kritis untuk data transaksi finansial (BRULE-TX-02, BRULE-ST-01).
- Mendukung constraint kompleks (check constraint, foreign key dengan cascade rules) yang menegakkan business rule di level database sebagai lapisan pertahanan tambahan.
- Mendukung tipe data JSONB untuk atribut fleksibel per domain (misal atribut ekstensi produk per domain — lihat [`master-data.md`](../business/master-data.md#22-atribut-ekstensi-per-domain)) tanpa mengorbankan kemampuan query relasional.
- Mendukung partitioning tabel besar (misal tabel transaksi) untuk menjaga performa seiring pertumbuhan data (NFR-SC-04).

### 3.3 Redis (Cache & Queue)

- Latency sangat rendah untuk operasi cache, penting untuk NFR-PF-01 (respons transaksi kasir ≤ 1 detik).
- Digunakan sebagai **session store terpusat** — memungkinkan Laravel API berjalan sebagai multiple instance tanpa sticky session (mendukung NFR-SC-03: horizontal scaling).
- Digunakan sebagai **queue broker** untuk job asynchronous (laporan besar, notifikasi), mendukung NFR-PF-04.

### 3.4 Docker & Docker Compose

- Konsistensi environment dari development hingga production (NFR-PD-01).
- Memudahkan onboarding developer baru — environment lengkap dapat dijalankan dengan satu perintah.

### 3.5 Nginx

- Reverse proxy yang matang dan efisien untuk terminasi SSL dan load balancing antar instance PHP-FPM.
- Dapat menangani static asset serving untuk build frontend Vue secara efisien.

### 3.6 AWS EC2

- Kontrol penuh terhadap konfigurasi infrastruktur, sesuai kebutuhan arsitektur multi-service (Nginx, Laravel, PostgreSQL, Redis) yang dijelaskan lebih detail pada [`deployment-architecture.md`](deployment-architecture.md).

---

## 4. Strategi Menjawab Kebutuhan Non-Fungsional Kritis

### 4.1 Performa (NFR-PF-01 s.d. NFR-PF-04)

- Query database dioptimalkan dengan indexing yang tepat (detail di [`database-design.md`](database-design.md)).
- Redis cache digunakan untuk data yang sering diakses namun jarang berubah (misal daftar produk aktif per cabang).
- Laporan besar diproses melalui queue worker secara asynchronous, tidak memblokir request real-time.

### 4.2 Skalabilitas (NFR-SC-01 s.d. NFR-SC-04)

- Laravel API bersifat **stateless** — session disimpan di Redis, memungkinkan penambahan instance API di belakang Nginx load balancer tanpa perubahan arsitektur.
- Struktur database dirancang dengan `branch_id` sebagai bagian dari indexing strategy di hampir seluruh tabel transaksional, mendukung query performa tinggi meski jumlah cabang bertambah.
- Tabel dengan volume tinggi (misal `sales_transactions`) dirancang mendukung partitioning berdasarkan waktu (lihat [`database-design.md`](database-design.md#7-strategi-partitioning)).

### 4.3 Ketersediaan & Resiliensi Lokal (NFR-AV-01 s.d. NFR-AV-04, BR-02)

- Aplikasi kasir (web maupun mobile) dirancang mendukung **local resilience**: transaksi disimpan sementara di local storage klien saat koneksi terputus, dan disinkronkan ke server saat koneksi pulih (mendukung EC-SL-01). Detail mekanisme sinkronisasi dijelaskan pada [`module-design.md`](module-design.md).
- Redis dan PostgreSQL dikonfigurasi dengan strategi backup rutin (lihat [`backup-strategy.md`](backup-strategy.md)) sehingga data tidak hilang meski terjadi kegagalan sistem mendadak.

### 4.4 Keamanan (NFR-SE-01 s.d. NFR-SE-06)

Dijelaskan detail pada [`security-design.md`](security-design.md), namun secara arsitektural:
- Seluruh komunikasi klien-server melewati Nginx dengan TLS termination.
- Backend API menerapkan otorisasi berbasis role & data scoping di level Service Layer (lihat [`authorization-design.md`](authorization-design.md)), bukan hanya di level route middleware.

### 4.5 Observability (NFR-OB-01, NFR-OB-02)

- Setiap request API dan error dicatat sebagai log terstruktur (lihat [`logging-strategy.md`](logging-strategy.md)).
- Endpoint health check disediakan untuk setiap komponen kritis (API, queue worker) guna mendukung monitoring otomatis.

---

## 5. Batasan Arsitektur pada Versi Ini

Sejalan dengan scope pada [`business-requirements.md`](../business/business-requirements.md#32-tidak-termasuk-dalam-scope-out-of-scope):

- Arsitektur **tidak** menggunakan message broker terpisah (Kafka/RabbitMQ) — Redis queue dianggap cukup untuk skala volume transaksi yang ditargetkan pada versi ini. Migrasi ke message broker khusus dapat dilakukan di masa depan tanpa mengubah struktur modul.
- Arsitektur **tidak** menerapkan multi-region deployment pada versi awal — seluruh infrastruktur berada pada satu region AWS, dengan strategi backup lintas availability zone (lihat [`deployment-architecture.md`](deployment-architecture.md)).
- Database menggunakan **single database dengan `branch_id` sebagai kolom scoping** (bukan database terpisah per cabang/tenant) — pendekatan ini dipilih karena kebutuhan konsolidasi laporan real-time lintas cabang (BR-05) jauh lebih sering terjadi dibanding kebutuhan isolasi data fisik antar cabang.

---

## 6. Diagram Alur Request (Contoh: Transaksi Penjualan)

```
Kasir (Vue/Android) 
    → POST /api/v1/sales (via Nginx)
    → SalesController::store()
    → SalesService::createSale()
        ├─ Validasi input (Request Validation)
        ├─ Cek stok via ProductRepository
        ├─ Hitung total, diskon, pajak
        ├─ Simpan transaksi via SalesRepository (dalam DB Transaction)
        ├─ Kurangi stok via StockRepository (dalam DB Transaction yang sama)
        ├─ Dispatch Event: SaleCompleted
        │       └─ Listener: Kirim notifikasi (async via Queue)
        │       └─ Listener: Catat audit log
        └─ Return response
    → Response ke Klien (struk data)
```

**Alasan teknis:** Pengurangan stok dan pencatatan transaksi terjadi **dalam satu database transaction** untuk menjamin konsistensi (BRULE-ST-01, EC-SL-02) — jika salah satu gagal, keduanya di-rollback.

---

## 7. Traceability

| Aspek Arsitektur | Terkait Dokumen |
|---|---|
| Struktur kode | [`folder-structure.md`](folder-structure.md) |
| Desain modul | [`module-design.md`](module-design.md) |
| Struktur data | [`database-design.md`](database-design.md), [`erd.md`](erd.md) |
| Keamanan | [`security-design.md`](security-design.md) |
| Deployment fisik | [`deployment-architecture.md`](deployment-architecture.md), [`docker-architecture.md`](docker-architecture.md) |

---

**Sebelumnya:** [`README.md`](README.md) — Index Dokumentasi Teknis
**Selanjutnya:** [`folder-structure.md`](folder-structure.md) — Folder Structure
