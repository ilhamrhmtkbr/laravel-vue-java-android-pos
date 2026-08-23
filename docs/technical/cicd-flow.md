# CI/CD Flow — POS

> Dokumen ini menjelaskan **alur Continuous Integration/Continuous Deployment** menggunakan GitHub Actions — bagaimana kode diverifikasi otomatis, di-build menjadi image Docker, dan dideploy ke AWS EC2. Dokumen ini mengoperasionalkan [`docker-architecture.md`](docker-architecture.md) dan [`deployment-architecture.md`](deployment-architecture.md), serta menegakkan [`testing-strategy.md`](testing-strategy.md) secara otomatis di setiap perubahan kode.

---

## 1. Prinsip Desain CI/CD

1. **Setiap perubahan kode diverifikasi otomatis sebelum dapat digabung (merge)** — tidak ada kode yang masuk ke branch utama tanpa lolos test dan pemeriksaan kualitas.
2. **Deployment ke production selalu melalui pipeline yang sama, tidak pernah manual** — mencegah *configuration drift* antara apa yang diuji dan apa yang benar-benar berjalan di production.
3. **Pipeline terpisah per komponen** (backend, frontend, mobile) — perubahan pada satu komponen tidak memicu build/deploy komponen lain yang tidak berubah, mempercepat siklus CI.
4. **Secret tidak pernah muncul di log atau kode** — seluruh kredensial dikelola melalui GitHub Secrets.

---

## 2. Struktur Branch & Trigger

| Branch | Trigger CI | Trigger CD |
|---|---|---|
| `feature/*` | Menjalankan test & lint saat push/PR | Tidak ada deploy otomatis |
| `develop` | Menjalankan test & lint saat merge | Deploy otomatis ke environment **Staging** |
| `main` | Menjalankan test & lint saat merge | Deploy otomatis ke environment **Production** (dengan manual approval gate) |

**Alasan manual approval gate pada production:** Meski pipeline sepenuhnya otomatis dari sisi teknis, keputusan **kapan** perubahan production-facing dirilis (terutama untuk sistem finansial yang digunakan banyak cabang secara live) tetap memerlukan persetujuan eksplisit dari penanggung jawab rilis — mencegah deployment tidak sengaja di luar jam yang direncanakan.

---

## 3. Pipeline: Backend (Laravel API)

### 3.1 Workflow CI (`.github/workflows/backend-ci.yml`)

```yaml
name: Backend CI

on:
  pull_request:
    paths: ['pos-backend/**']
  push:
    branches: [develop, main]
    paths: ['pos-backend/**']

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: pos_test
          POSTGRES_PASSWORD: test_secret
        ports: ['5432:5432']
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'

      - name: Install dependencies
        run: composer install --prefer-dist --no-progress
        working-directory: pos-backend

      - name: Run static analysis (PHPStan)
        run: vendor/bin/phpstan analyse
        working-directory: pos-backend

      - name: Run code style check (Laravel Pint)
        run: vendor/bin/pint --test
        working-directory: pos-backend

      - name: Run migrations (test database)
        run: php artisan migrate --force
        working-directory: pos-backend
        env:
          DB_HOST: localhost

      - name: Run unit & feature tests
        run: vendor/bin/phpunit --coverage-text
        working-directory: pos-backend

      - name: Check vulnerable dependencies
        run: composer audit
        working-directory: pos-backend
```

**Alasan setiap langkah:**

| Langkah | Alasan |
|---|---|
| PHPStan (static analysis) | Menangkap potensi bug tipe data dan error logika sebelum runtime, mendukung NFR-MT-03 |
| Laravel Pint (code style) | Menjaga konsistensi gaya kode di seluruh tim, mengurangi noise pada code review |
| Migration + Test dengan database sungguhan | Memastikan skema database dan query benar-benar berfungsi, bukan hanya mock (lihat [`testing-strategy.md`](testing-strategy.md)) |
| `composer audit` | Mendeteksi dependency dengan kerentanan keamanan yang diketahui (lihat [`security-design.md`](security-design.md#6-keamanan-level-container--infrastruktur)) |

### 3.2 Workflow CD — Build & Deploy (`.github/workflows/backend-cd.yml`)

```yaml
name: Backend CD

on:
  push:
    branches: [develop, main]
    paths: ['pos-backend/**']

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    needs: [test]   # hanya jalan jika CI (test) sukses
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t pos-registry/app:${{ github.sha }} -f pos-backend/Dockerfile pos-backend

      - name: Push to registry
        run: docker push pos-registry/app:${{ github.sha }}

  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    needs: [build-and-push]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to Staging EC2
        run: |
          ssh ${{ secrets.STAGING_SSH_TARGET }} \
            "cd /opt/pos && IMAGE_TAG=${{ github.sha }} docker compose pull && docker compose up -d"

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: [build-and-push]
    runs-on: ubuntu-latest
    environment: production   # environment ini dikonfigurasi dengan required reviewer di GitHub
    steps:
      - name: Run database migration
        run: |
          ssh ${{ secrets.PROD_SSH_TARGET }} \
            "cd /opt/pos && docker compose run --rm app php artisan migrate --force"

      - name: Rolling deploy to Production EC2
        run: |
          ssh ${{ secrets.PROD_SSH_TARGET }} \
            "cd /opt/pos && IMAGE_TAG=${{ github.sha }} docker compose up -d --no-deps app queue-worker"

      - name: Health check verification
        run: |
          curl -f https://api.pos-domain.com/health || exit 1
```

**Alasan `environment: production` dengan required reviewer:** Fitur GitHub Actions Environments memungkinkan konfigurasi *protection rules* — job tidak akan berjalan sampai reviewer yang ditunjuk menyetujui, mengimplementasikan manual approval gate yang disebutkan pada Bagian 2.

**Alasan migration dijalankan sebelum rolling deploy:** Sejalan dengan strategi migrasi backward-compatible pada [`deployment-architecture.md`](deployment-architecture.md#33-strategi-migrasi-database-yang-aman) — migrasi harus selesai sebelum kode baru yang bergantung padanya diaktifkan, namun kode lama tetap harus dapat berjalan sesaat selama proses rolling deployment berlangsung.

---

## 4. Pipeline: Frontend (Vue)

```yaml
name: Frontend CI/CD

on:
  push:
    branches: [develop, main]
    paths: ['pos-frontend/**']
  pull_request:
    paths: ['pos-frontend/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
        working-directory: pos-frontend
      - run: npm run lint
        working-directory: pos-frontend
      - run: npm run test:unit
        working-directory: pos-frontend
      - run: npm run build   # memastikan build production tidak error

  deploy:
    needs: [test]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Build & push image (berisi hasil build + Nginx)
        run: docker build -t pos-registry/nginx:${{ github.sha }} pos-frontend
      # Langkah deploy serupa dengan backend, menyesuaikan target environment
```

---

## 5. Pipeline: Mobile (Android)

```yaml
name: Mobile CI

on:
  push:
    branches: [develop, main]
    paths: ['pos-mobile/**']
  pull_request:
    paths: ['pos-mobile/**']

jobs:
  test-and-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run unit tests
        run: ./gradlew test
        working-directory: pos-mobile

      - name: Build debug APK (develop) / release APK (main)
        run: ./gradlew assembleRelease
        working-directory: pos-mobile

      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: pos-app-release
          path: pos-mobile/app/build/outputs/apk/release/*.apk
```

**Catatan:** Distribusi APK ke perangkat kasir lapangan (misal via MDM internal perusahaan atau distribusi manual terkontrol) berada di luar scope otomatisasi CI/CD pada versi ini — pipeline hanya bertanggung jawab hingga tahap build & artifact tersedia untuk diunduh oleh tim terkait.

---

## 6. Ringkasan Alur End-to-End

```
Developer push kode
    → Pull Request dibuka
        → CI berjalan: lint, static analysis, unit test, feature test
        → Review kode oleh rekan tim
    → Merge ke develop
        → CD otomatis: build image → deploy ke Staging
        → QA melakukan verifikasi di Staging
    → Merge develop ke main (setelah verifikasi Staging)
        → CD berjalan: build image → menunggu approval reviewer
        → Reviewer menyetujui
        → Migration dijalankan → Rolling deploy ke Production
        → Health check otomatis memverifikasi keberhasilan
```

---

## 7. Traceability

| Aspek CI/CD | Terkait Dokumen |
|---|---|
| Image yang dibangun & dideploy | [`docker-architecture.md`](docker-architecture.md) |
| Target infrastruktur deployment | [`deployment-architecture.md`](deployment-architecture.md) |
| Jenis test yang dijalankan | [`testing-strategy.md`](testing-strategy.md) |
| Pengelolaan secret | [`security-design.md`](security-design.md#6-keamanan-level-container--infrastruktur) |

---

**Sebelumnya:** [`docker-architecture.md`](docker-architecture.md) — Docker Architecture
**Selanjutnya:** [`logging-strategy.md`](logging-strategy.md) — Logging Strategy
