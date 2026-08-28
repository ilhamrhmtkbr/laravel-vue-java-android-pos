# Technical Documentation — POS

> Dokumen ini adalah pintu masuk (index) untuk seluruh dokumentasi teknis project POS. Ditujukan bagi Software Architect, Database Architect, Backend/Frontend/Mobile Tech Lead, DevOps Engineer, dan QA Engineer yang akan mengimplementasikan sistem berdasarkan kebutuhan bisnis yang telah didefinisikan di [`docs/business/`](../business/README.md).

---

## 1. Tujuan Dokumentasi Teknis

Dokumentasi teknis pada project ini disusun untuk menjawab:

1. **Bagaimana** sistem dirancang secara arsitektural agar memenuhi kebutuhan bisnis dan non-fungsional yang telah ditetapkan?
2. **Mengapa** setiap keputusan teknis diambil — trade-off apa yang dipertimbangkan?
3. **Bagaimana** developer dapat langsung mulai implementasi tanpa ambiguitas struktural?
4. **Bagaimana** sistem dijalankan, dipantau, dan dipelihara di lingkungan produksi?

Setiap dokumen teknis secara eksplisit mereferensikan balik ke dokumen bisnis yang melatarbelakanginya, menjaga *traceability* dua arah di seluruh repository.

---

## 2. Daftar Isi Dokumentasi Teknis

| No | Dokumen | Deskripsi Singkat |
|---|---|---|
| 1 | [`system-architecture.md`](system-architecture.md) | Arsitektur sistem secara keseluruhan — komponen, komunikasi antar layer, keputusan arsitektural |
| 2 | [`folder-structure.md`](folder-structure.md) | Struktur direktori tiap komponen (Laravel API, Vue, Android) |
| 3 | [`module-design.md`](module-design.md) | Desain tiap modul fungsional dan tanggung jawabnya |
| 4 | [`database-design.md`](database-design.md) | Desain skema database, konvensi penamaan, strategi indexing |
| 5 | [`erd.md`](erd.md) | Entity Relationship Diagram lengkap |
| 6 | [`api-contract.md`](api-contract.md) | Kontrak API — endpoint, request/response, konvensi |
| 7 | [`authentication-design.md`](authentication-design.md) | Desain otentikasi (login, token, sesi) |
| 8 | [`authorization-design.md`](authorization-design.md) | Desain otorisasi (RBAC, permission, data scoping) |
| 9 | [`service-layer-design.md`](service-layer-design.md) | Desain lapisan business logic |
| 10 | [`repository-layer-design.md`](repository-layer-design.md) | Desain lapisan akses data |
| 11 | [`security-design.md`](security-design.md) | Desain keamanan menyeluruh |
| 12 | [`deployment-architecture.md`](deployment-architecture.md) | Arsitektur deployment ke AWS EC2 |
| 13 | [`docker-architecture.md`](docker-architecture.md) | Desain containerization |
| 14 | [`cicd-flow.md`](cicd-flow.md) | Alur CI/CD dengan GitHub Actions |
| 15 | [`logging-strategy.md`](logging-strategy.md) | Strategi logging & observability |
| 16 | [`backup-strategy.md`](backup-strategy.md) | Strategi backup & disaster recovery |
| 17 | [`testing-strategy.md`](testing-strategy.md) | Strategi pengujian di semua layer |

---

## 3. Technology Stack Detail

| Layer | Teknologi | Versi Acuan (rekomendasi minimum) |
|---|---|---|
| Backend API | Laravel | 11.x (PHP 8.3+) |
| Frontend Web | Vue | 3.x (Composition API) + Tailwind CSS 3.x |
| Mobile | Android Native | Java, minSdkVersion 24 (Android 7.0) ke atas |
| Database | PostgreSQL | 16.x |
| Cache/Queue | Redis | 7.x |
| Container | Docker & Docker Compose | Docker Engine 24.x+ |
| Reverse Proxy | Nginx | 1.25.x |
| Deployment | AWS EC2 | Ubuntu Server 22.04 LTS |
| CI/CD | GitHub Actions | — |

Justifikasi pemilihan versi dan alasan teknis mendalam dijelaskan pada [`system-architecture.md`](system-architecture.md).

---

## 4. Gambaran Arsitektur Tingkat Tinggi

```
┌─────────────┐     ┌─────────────┐     ┌──────────────────┐
│  Vue Web    │     │  Android    │     │  (Future) Third- │
│  (Back-office│    │  Mobile App │     │  Party Clients   │
│  & Kasir Web)│    │  (Kasir     │     │                  │
│             │     │  Lapangan)  │     │                  │
└──────┬──────┘     └──────┬──────┘     └─────────┬────────┘
       │                   │                      │
       └───────────────────┼──────────────────────┘
                           ▼
                    ┌───────────────┐
                    │     Nginx     │  (Reverse Proxy + SSL Termination)
                    └──────┬────────┘
                           ▼
                    ┌───────────────┐
                    │  Laravel API  │  (Stateless, horizontally scalable)
                    └──────┬────────┘
              ┌────────────┼──────────────┐
              ▼            ▼              ▼
      ┌──────────────┐┌──────────┐┌────────────────┐
      │ PostgreSQL   ││  Redis   ││ Queue Worker   │
      │ (Primary DB) ││ (Cache & ││ (Async Jobs:   │
      │              ││  Session)││ Reports, Notif)│
      └──────────────┘└──────────┘└────────────────┘
```

Detail komponen, alasan pemilihan pola arsitektur, dan strategi scaling dijelaskan penuh pada [`system-architecture.md`](system-architecture.md).

---

## 5. Panduan Membaca Dokumentasi Teknis

Urutan membaca yang direkomendasikan untuk developer baru:

1. **`system-architecture.md`** — memahami gambaran besar sistem terlebih dahulu.
2. **`folder-structure.md`** — memahami tata letak kode sebelum mulai coding.
3. **`database-design.md`** dan **`erd.md`** — memahami struktur data yang menjadi fondasi seluruh fitur.
4. **`module-design.md`**, **`service-layer-design.md`**, **`repository-layer-design.md`** — memahami bagaimana business logic diorganisir.
5. **`api-contract.md`** — kontrak yang menjembatani backend dengan frontend/mobile.
6. **`authentication-design.md`** dan **`authorization-design.md`** — memahami bagaimana akses dikontrol.
7. **`security-design.md`** — memahami lapisan keamanan tambahan.
8. **`docker-architecture.md`**, **`deployment-architecture.md`**, **`cicd-flow.md`** — memahami bagaimana sistem dibangun dan dijalankan di produksi.
9. **`logging-strategy.md`**, **`backup-strategy.md`**, **`testing-strategy.md`** — memahami operasional jangka panjang sistem.

---

## 6. Prinsip Desain Teknis yang Konsisten

Seluruh dokumen teknis pada repository ini mengikuti prinsip berikut secara konsisten, sejalan dengan prinsip desain bisnis pada [`README.md`](../../README.md#5-prinsip-desain-sistem):

1. **Layered Architecture** — pemisahan tegas Controller → Service → Repository → Model, tidak ada business logic yang "bocor" ke Controller.
2. **API-first** — backend Laravel murni sebagai REST API, tidak mencampur logic rendering view; frontend dan mobile adalah konsumen API independen.
3. **Statelessness pada layer API** — session/state disimpan di Redis, bukan di memori server, agar backend dapat di-scale secara horizontal.
4. **Explicit over implicit** — struktur folder, penamaan, dan konvensi kode dibuat eksplisit dan terdokumentasi, mengurangi "tacit knowledge" yang hanya diketahui sebagian tim.
5. **Security by design** — otentikasi, otorisasi, dan validasi diterapkan di setiap layer yang relevan, bukan hanya di titik masuk (API Gateway/frontend).
6. **Observability sejak awal** — logging terstruktur dan health check menjadi bagian dari desain modul, bukan ditambahkan belakangan.
7. **Konsistensi dengan kebutuhan bisnis** — setiap keputusan teknis dapat ditelusuri ke Business Requirement atau Non-Functional Requirement terkait.

---

## 7. Traceability ke Dokumentasi Bisnis

| Dokumen Teknis | Berangkat dari Dokumen Bisnis |
|---|---|
| `system-architecture.md` | [`business-requirements.md`](../business/business-requirements.md), [`non-functional-requirements.md`](../business/non-functional-requirements.md) |
| `database-design.md`, `erd.md` | [`master-data.md`](../business/master-data.md), [`transactions.md`](../business/transactions.md) |
| `module-design.md` | [`business-flow.md`](../business/business-flow.md), [`use-case.md`](../business/use-case.md) |
| `authorization-design.md` | [`role-permission.md`](../business/role-permission.md) |
| `service-layer-design.md` | [`business-rules.md`](../business/business-rules.md), [`validation.md`](../business/validation.md) |
| `api-contract.md` | [`functional-requirements.md`](../business/functional-requirements.md), [`dashboard.md`](../business/dashboard.md), [`reports.md`](../business/reports.md) |
| `testing-strategy.md` | [`edge-cases.md`](../business/edge-cases.md) |

---

**Sebelumnya:** [`../business/transactions.md`](../business/transactions.md) — Transactions (Business Documentation)
**Selanjutnya:** [`system-architecture.md`](system-architecture.md) — System Architecture
