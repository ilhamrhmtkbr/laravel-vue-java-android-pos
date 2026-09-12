# Folder Structure — POS

> Dokumen ini menjelaskan **struktur direktori** untuk setiap komponen sistem (Backend Laravel, Frontend Vue, Mobile Android). Struktur ini adalah implementasi konkret dari prinsip *Layered Architecture* yang dijelaskan pada [`system-architecture.md`](system-architecture.md), dan menjadi acuan wajib sebelum developer mulai menulis kode agar konsisten di seluruh tim.

---

## 1. Prinsip Penataan Folder

1. **Predictable location** — developer harus bisa menebak lokasi file berdasarkan tanggung jawabnya (mis. business logic → `Services/`, bukan tersebar).
2. **Group by layer, bukan by feature murni** — pada backend, pemisahan utama mengikuti layer arsitektur (Controller, Service, Repository), dengan sub-pengelompokan per modul bisnis di dalamnya.
3. **Konsisten lintas komponen** — pola penamaan folder/file serupa antara backend, frontend, dan mobile sedapat mungkin, memudahkan context-switching developer.
4. **Tidak ada "God folder"** — folder seperti `Helpers/` atau `Utils/` dibatasi penggunaannya dan didokumentasikan tujuannya secara eksplisit.

---

## 2. Struktur Backend — Laravel API

```
pos-backend/
├── Application
├── Domain
├── Helpers
├── Infrastructure
├── Models
│       └── User.php
├── Presentation
│       └── Http
│           └── Controllers
│               └── Controller.php
└── Providers
    └── AppServiceProvider.php
├── config/
├── database/
│   ├── factories/                     # Model factory untuk testing & seeding
│   ├── migrations/                    # Skema database bertahap (lihat database-design.md)
│   └── seeders/                       # Data awal (role default, kategori dasar, dll)
├── routes/
│   ├── api.php                        # Entry point, meng-include file route per modul
│   └── api/
│       ├── sales.php
│       ├── inventory.php
│       ├── purchasing.php
│       └── ...
├── tests/
│   ├── Unit/                          # Unit test (Service, Repository)
│   └── Feature/                       # Feature test (endpoint API end-to-end)
├── docker/                            # Dockerfile & konfigurasi khusus container (lihat docker-architecture.md)
├── .github/workflows/                 # GitHub Actions CI/CD (lihat cicd-flow.md)
├── .env.example
└── composer.json
```

### 2.1 Penjelasan Keputusan Struktur

| Keputusan | Alasan |
|---|---|
| Controller dikelompokkan per modul bisnis (`Sales/`, `Inventory/`, dll), bukan flat | Mempermudah navigasi seiring bertambahnya jumlah controller, konsisten dengan pembagian modul pada [`module-design.md`](module-design.md) |
| `Repositories/Contracts/` dan `Repositories/Eloquent/` dipisah | Memungkinkan dependency injection ke interface (bukan implementasi konkret), memudahkan unit testing dengan mock (lihat [`repository-layer-design.md`](repository-layer-design.md)) |
| `Exceptions/Business/` terpisah dari exception umum Laravel | Business exception (mis. stok tidak cukup) memerlukan handling response yang berbeda dari exception teknis (500 error), sesuai kebutuhan pesan error actionable pada [`validation.md`](../business/validation.md) |
| `routes/api/` dipecah per modul, di-include dari `api.php` | File route tunggal yang sangat panjang sulit dipelihara pada sistem dengan puluhan endpoint |

---

## 3. Struktur Frontend — Vue 3 + Tailwind

```
pos-frontend/
├── src/
│   ├── assets/                        # Static asset (logo, ikon kustom)
│   ├── components/
│   │   ├── common/                    # Komponen reusable lintas modul (Button, Modal, DataTable)
│   │   ├── sales/                     # Komponen spesifik modul penjualan (ProductGrid, CartSummary)
│   │   ├── inventory/
│   │   ├── approval/
│   │   └── dashboard/
│   ├── composables/                   # Vue Composition API reusable logic (useCart, usePermission)
│   ├── layouts/
│   │   ├── BackOfficeLayout.vue       # Layout untuk halaman back-office (sidebar, header)
│   │   └── CashierLayout.vue          # Layout khusus kasir (minim distraksi, sesuai NFR-US-01)
│   ├── pages/                         # Satu file per halaman/route (mirroring struktur routing)
│   │   ├── sales/
│   │   │   ├── CashierPage.vue
│   │   │   └── SalesHistoryPage.vue
│   │   ├── inventory/
│   │   ├── purchasing/
│   │   ├── approval/
│   │   ├── reports/
│   │   └── dashboard/
│   ├── router/
│   │   └── index.js                   # Definisi route, termasuk route guard berbasis permission
│   ├── stores/                        # State management (Pinia)
│   │   ├── authStore.js
│   │   ├── cartStore.js
│   │   └── notificationStore.js
│   ├── services/                      # API client layer (axios instance per modul)
│   │   ├── apiClient.js               # Konfigurasi dasar axios (base URL, interceptor token)
│   │   ├── salesService.js
│   │   ├── inventoryService.js
│   │   └── ...
│   ├── utils/                         # Helper murni (formatCurrency, formatDate) — tanpa business logic
│   ├── App.vue
│   └── main.js
├── public/
├── tests/
│   ├── unit/                          # Unit test komponen (Vitest)
│   └── e2e/                           # End-to-end test (Playwright/Cypress)
├── docker/
├── .github/workflows/
├── tailwind.config.js
├── vite.config.js
└── package.json
```

### 3.1 Penjelasan Keputusan Struktur

| Keputusan | Alasan |
|---|---|
| `pages/` mengikuti struktur modul yang sama dengan backend (`sales/`, `inventory/`, dll) | Konsistensi mental model antara frontend dan backend, memudahkan tracing fitur end-to-end |
| `services/` sebagai satu-satunya layer yang memanggil API | Komponen Vue tidak pernah memanggil `axios` langsung — memastikan satu titik kontrol untuk error handling, retry, dan interceptor token (lihat [`authentication-design.md`](authentication-design.md)) |
| `CashierLayout.vue` terpisah dari `BackOfficeLayout.vue` | Kebutuhan UX kasir (kecepatan, minim elemen) berbeda signifikan dari back-office (kaya data, navigasi kompleks), sesuai NFR-US-01 |
| `composables/` untuk logic reusable | Mengikuti pola idiomatis Vue 3 Composition API, memisahkan logic dari presentasi |

---

## 4. Struktur Mobile — Android (Java)

```
pos-mobile/
├── app/
│   ├── src/main/java/com/company/pos/
│   │   ├── data/
│   │   │   ├── remote/                # API service interface (Retrofit)
│   │   │   │   ├── SalesApiService.java
│   │   │   │   └── ...
│   │   │   ├── local/                 # Local database (Room) untuk mode offline (BR-02)
│   │   │   │   ├── entity/
│   │   │   │   ├── dao/
│   │   │   │   └── AppDatabase.java
│   │   │   └── repository/            # Repository yang menggabungkan remote & local (offline-first logic)
│   │   │       ├── SalesRepository.java
│   │   │       └── ...
│   │   ├── domain/
│   │   │   ├── model/                 # Model domain (POJO), terpisah dari entity database & response API
│   │   │   └── usecase/               # Use case/interactor per aksi bisnis (CreateSaleUseCase, dll)
│   │   ├── ui/
│   │   │   ├── cashier/               # Activity/Fragment layar kasir
│   │   │   ├── inventory/
│   │   │   ├── approval/
│   │   │   ├── auth/
│   │   │   └── common/                # Adapter, custom view reusable
│   │   ├── service/
│   │   │   ├── SyncService.java       # Background service sinkronisasi data offline (EC-SL-01)
│   │   │   └── PrinterService.java    # Integrasi printer thermal (ESC/POS)
│   │   ├── util/
│   │   └── di/                        # Dependency injection setup (mis. Hilt/Dagger module)
│   ├── src/test/                      # Unit test (JUnit)
│   ├── src/androidTest/               # Instrumented test (Espresso)
│   └── build.gradle
├── docker/                            # (opsional, untuk build environment terkontainer jika diperlukan CI)
├── .github/workflows/
└── build.gradle (project-level)
```

### 4.1 Penjelasan Keputusan Struktur

| Keputusan | Alasan |
|---|---|
| Struktur `data/remote`, `data/local`, `data/repository` terpisah | Mengikuti pola *offline-first architecture* — Repository menjadi satu-satunya sumber kebenaran yang memutuskan mengambil data dari local (Room) atau remote (API), mendukung BR-02 dan EC-SL-01 |
| `domain/usecase/` terpisah dari `ui/` | Business logic (mis. validasi sebelum submit transaksi) tidak tercampur dengan lifecycle Activity/Fragment, memudahkan unit testing |
| `service/SyncService.java` sebagai komponen eksplisit | Sinkronisasi offline-to-online adalah kebutuhan arsitektural kritis (NFR-AV-02), bukan logic tambahan yang ditempel di UI layer |
| `service/PrinterService.java` terpisah | Integrasi hardware (printer struk) adalah tanggung jawab terisolasi, memudahkan penggantian jenis printer tanpa menyentuh business logic |

---

## 5. Konvensi Penamaan Lintas Komponen

| Elemen | Konvensi | Contoh |
|---|---|---|
| Nama Controller (Laravel) | PascalCase + suffix `Controller` | `SalesController.php` |
| Nama Service (Laravel) | PascalCase + suffix `Service` | `SalesService.php` |
| Nama Repository (Laravel) | PascalCase + suffix `Repository`/`RepositoryInterface` | `SalesRepositoryInterface.php` |
| Nama Komponen Vue | PascalCase + `.vue` | `CartSummary.vue` |
| Nama Composable Vue | camelCase, prefix `use` | `useCart.js` |
| Nama Class Java (Android) | PascalCase | `SalesRepository.java` |
| Nama Endpoint API | kebab-case, plural resource | `/api/v1/sales-transactions` |
| Nama Kolom Database | snake_case | `branch_id`, `created_at` |

Detail konvensi database dijelaskan lebih lanjut pada [`database-design.md`](database-design.md).

---

## 6. Traceability

| Struktur | Terkait Dokumen |
|---|---|
| Alasan pemisahan layer | [`system-architecture.md`](system-architecture.md) |
| Detail isi tiap Service | [`service-layer-design.md`](service-layer-design.md) |
| Detail isi tiap Repository | [`repository-layer-design.md`](repository-layer-design.md) |
| Pemetaan modul ke folder | [`module-design.md`](module-design.md) |

---

**Sebelumnya:** [`system-architecture.md`](system-architecture.md) — System Architecture
**Selanjutnya:** [`module-design.md`](module-design.md) — Module Design
