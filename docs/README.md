# POS — Enterprise Point of Sale System

> Sistem Point of Sale (POS) enterprise multi-cabang, multi-domain bisnis, dirancang untuk skala perusahaan besar dengan kebutuhan operasional yang kompleks — mulai dari retail, minimarket, supermarket, restoran, distributor/grosir, apotek, hingga UMKM naik kelas.

[![Status](https://img.shields.io/badge/status-in--development-yellow)]()
[![License](https://img.shields.io/badge/license-Proprietary-blue)]()
[![Backend](https://img.shields.io/badge/backend-Laravel-red)]()
[![Frontend](https://img.shields.io/badge/frontend-Vue%203-42b883)]()
[![Mobile](https://img.shields.io/badge/mobile-Android%20Java-3ddc84)]()
[![Database](https://img.shields.io/badge/database-PostgreSQL-336791)]()

---

## 1. Tentang Project Ini

POS adalah sebuah **platform kasir dan manajemen bisnis retail terintegrasi**, dibangun sebagai contoh implementasi *production ready* dari sebuah sistem enterprise sesungguhnya — bukan sekadar aplikasi kasir sederhana (MVP).

Repository ini **bukan hanya berisi kode**, tetapi juga **dokumentasi lengkap ala perusahaan software house profesional**, yang mencakup:

- Bagaimana **proses bisnis** sebuah perusahaan retail multi-cabang berjalan.
- Bagaimana proses bisnis tersebut **diterjemahkan menjadi kebutuhan sistem**.
- Bagaimana sistem **dirancang secara arsitektural** agar scalable, secure, dan maintainable.
- **Alasan** di balik setiap keputusan desain — baik dari sisi bisnis maupun teknis.

Dokumentasi disusun agar siapa pun — baik business analyst, engineer baru, maupun reviewer portfolio — dapat **memahami dari alasan hingga implementasi**, dan langsung bisa mulai coding tanpa ambiguitas.

### 1.1 Mengapa Project Ini Dibuat

| Tujuan | Penjelasan |
|---|---|
| Memahami proses bisnis retail | Setiap domain (minimarket, restoran, apotek, distributor) punya alur operasional dan aturan bisnis yang berbeda. Dokumentasi bisnis menjelaskan proses tersebut secara nyata. |
| Memahami cara software dirancang | Dokumentasi teknis menunjukkan bagaimana keputusan arsitektur, desain database, dan desain API diambil — bukan hanya "apa" tapi "kenapa". |
| Memahami alasan setiap modul | Setiap modul memiliki justifikasi bisnis (business rationale) dan justifikasi teknis (technical rationale), bukan modul yang dibuat tanpa tujuan jelas. |
| Siap coding setelah membaca dokumentasi | Dokumentasi cukup detail (ERD, API contract, folder structure, service layer design) sehingga developer dapat langsung implementasi tanpa harus menebak-nebak. |

---

## 2. Domain Bisnis yang Didukung

Sistem dirancang generik namun cukup dalam (*deep enough*) untuk menangani karakteristik dari domain-domain berikut dalam satu arsitektur yang sama:

| Domain | Karakteristik Utama yang Diakomodasi |
|---|---|
| **Retail / Minimarket** | Transaksi cepat, barcode scanning, multi-payment, promo, member |
| **Supermarket** | Multi-kategori produk, satuan konversi, banyak SKU, rak & gudang |
| **Restoran / F&B** | Meja, open bill, kitchen order, modifier menu, split bill |
| **Distributor / Grosir** | Harga bertingkat (tiered pricing), piutang, term of payment, sales order |
| **Apotek** | Batch/expired tracking, resep, obat keras, regulasi stok |
| **UMKM** | Fleksibilitas skala kecil dengan opsi upgrade ke fitur enterprise |

Detail proses bisnis per-domain dijelaskan pada [`docs/business/business-flow.md`](docs/business/business-flow.md).

---

## 3. Technology Stack

| Layer | Teknologi | Alasan Pemilihan (ringkas) |
|---|---|---|
| Backend API | **Laravel (PHP)** | Ekosistem matang, ORM kuat (Eloquent), dukungan enterprise (queue, event, testing) sangat lengkap |
| Frontend Web | **Vue 3 + Tailwind CSS** | Reactive, component-based, learning curve rendah, cocok untuk dashboard back-office & kasir berbasis web |
| Mobile | **Android Native (Java)** | Performa tinggi untuk perangkat kasir low-end, akses hardware (printer, scanner, cash drawer) lebih stabil dibanding cross-platform |
| Database | **PostgreSQL** | ACID compliant, mendukung constraint kompleks, JSONB, cocok untuk data transaksional finansial |
| Cache / Queue | **Redis** | Session store, cache lapisan aplikasi, queue broker untuk job asynchronous |
| Container | **Docker** | Konsistensi environment dev-staging-production |
| Reverse Proxy | **Nginx** | Load balancing, SSL termination, static asset serving |
| Deployment | **AWS EC2** | Kontrol penuh infrastruktur, cocok untuk arsitektur multi-service |
| CI/CD | **GitHub Actions** | Terintegrasi langsung dengan repository, mendukung multi-stage pipeline |

Alasan teknis lengkap dijelaskan pada [`docs/technical/system-architecture.md`](docs/technical/system-architecture.md).

---

## 4. Struktur Dokumentasi

Seluruh dokumentasi dipisah menjadi dua kategori besar agar audiens yang berbeda (bisnis vs teknis) dapat fokus pada bagian yang relevan.

```
docs/
├── business/           # Dokumentasi bisnis: proses, aturan, requirement
│   ├── README.md
│   ├── business-requirements.md
│   ├── functional-requirements.md
│   ├── non-functional-requirements.md
│   ├── business-flow.md
│   ├── sop.md
│   ├── use-case.md
│   ├── role-permission.md
│   ├── business-rules.md
│   ├── approval-flow.md
│   ├── dashboard.md
│   ├── reports.md
│   ├── notifications.md
│   ├── validation.md
│   ├── edge-cases.md
│   ├── master-data.md
│   └── transactions.md
│
└── technical/           # Dokumentasi teknis: arsitektur, desain, deployment
    ├── README.md
    ├── system-architecture.md
    ├── folder-structure.md
    ├── module-design.md
    ├── database-design.md
    ├── erd.md
    ├── api-contract.md
    ├── authentication-design.md
    ├── authorization-design.md
    ├── service-layer-design.md
    ├── repository-layer-design.md
    ├── security-design.md
    ├── deployment-architecture.md
    ├── docker-architecture.md
    ├── cicd-flow.md
    ├── logging-strategy.md
    ├── backup-strategy.md
    └── testing-strategy.md
```

### 4.1 Panduan Membaca Dokumentasi

Urutan membaca yang direkomendasikan bagi pembaca baru:

1. **Business Documentation** dulu — untuk memahami *kenapa* sistem ini dibuat, dan bagaimana proses bisnis nyata di lapangan.
2. **Technical Documentation** setelahnya — untuk memahami *bagaimana* proses bisnis tersebut diimplementasikan menjadi arsitektur, database, dan API.

Setiap dokumen teknis akan secara eksplisit mereferensikan dokumen bisnis terkait, begitu juga sebaliknya, agar konsistensi antar dokumen terjaga.

---

## 5. Prinsip Desain Sistem

Sistem ini dirancang dengan asumsi **digunakan oleh perusahaan besar dengan banyak cabang**, sehingga prinsip berikut dipegang secara konsisten di seluruh dokumentasi dan (nantinya) implementasi:

1. **Multi-branch by design** — setiap entitas transaksional selalu terikat ke cabang (branch), bukan ditambahkan belakangan.
2. **Multi-tenant ready** — struktur data & arsitektur mendukung isolasi data antar perusahaan/tenant jika diperlukan di masa depan.
3. **Auditability** — setiap perubahan data penting (harga, stok, transaksi) dapat ditelusuri (siapa, kapan, apa).
4. **Security by default** — otentikasi & otorisasi granular berbasis role & permission, bukan hardcoded role check.
5. **Separation of concerns** — pemisahan tegas antara business logic (service layer), data access (repository layer), dan presentation layer.
6. **Idempotent & consistent transactions** — transaksi finansial menggunakan mekanisme yang mencegah duplikasi dan inkonsistensi data (database transaction, locking strategy).
7. **Observability** — sistem dirancang agar dapat dipantau (logging, monitoring) sejak awal, bukan ditambahkan setelah insiden terjadi.
8. **Tidak ada shortcut MVP** — setiap modul dirancang dengan standar produksi: validasi lengkap, penanganan edge case, dan dokumentasi alasan bisnis maupun teknis.

---

## 6. Status Dokumentasi

| Dokumen | Status |
|---|---|
| Business Documentation | 🟡 In Progress |
| Technical Documentation | ⚪ Not Started |

> Dokumentasi ini disusun secara bertahap, satu file per tahap, agar setiap dokumen dapat direview secara mendalam sebelum lanjut ke dokumen berikutnya.

---

## 7. Lisensi & Kontribusi

Project ini dibuat sebagai **portfolio teknis** untuk menunjukkan kemampuan desain sistem enterprise end-to-end. Struktur dan pendekatan dokumentasi dapat dijadikan referensi, namun disarankan untuk menyesuaikan dengan konteks bisnis masing-masing sebelum digunakan di lingkungan produksi nyata.

---

**Selanjutnya:** [`business/README.md`](business/README.md) — Ringkasan dokumentasi bisnis.
