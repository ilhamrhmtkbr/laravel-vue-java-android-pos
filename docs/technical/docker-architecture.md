# Docker Architecture — POS

> Dokumen ini menjelaskan **desain containerization** sistem menggunakan Docker — bagaimana tiap komponen dikemas sebagai container, bagaimana container-container tersebut berkomunikasi, dan bagaimana konfigurasi berbeda antara development dan production. Dokumen ini menjawab NFR-PD-01 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md) dan menjadi dasar bagi [`cicd-flow.md`](cicd-flow.md).

---

## 1. Prinsip Desain Containerization

1. **Satu container, satu tanggung jawab** — tidak menggabungkan Nginx, PHP-FPM, dan PostgreSQL dalam satu container (kecuali untuk mode development all-in-one sederhana jika benar-benar diperlukan demo cepat).
2. **Image production seminimal mungkin (multi-stage build)** — dependency development (Xdebug, dll) tidak ikut masuk ke image production.
3. **Konfigurasi via environment variable**, bukan hardcoded di dalam image — image yang sama dapat dipakai di development, staging, maupun production hanya dengan mengubah environment variable.
4. **Container tidak menyimpan state penting** — data persisten (database, file upload) selalu menggunakan Docker Volume atau storage eksternal (S3), bukan disimpan di filesystem container yang bersifat sementara.

---

## 2. Struktur Container per Komponen

| Container | Image Dasar | Tanggung Jawab |
|---|---|---|
| `nginx` | `nginx:1.25-alpine` | Reverse proxy, static asset serving (build Vue) |
| `app` (Laravel) | `php:8.3-fpm-alpine` (custom) | Menjalankan PHP-FPM untuk Laravel API |
| `queue-worker` | Sama dengan image `app` | Menjalankan `php artisan queue:work` |
| `scheduler` | Sama dengan image `app` | Menjalankan `php artisan schedule:run` (via cron) |
| `postgres` | `postgres:16-alpine` | Database utama |
| `redis` | `redis:7-alpine` | Cache, session, queue broker |

**Alasan menggunakan image `app` yang sama untuk `queue-worker` dan `scheduler`:** Ketiganya menjalankan kode Laravel yang identik, hanya berbeda command yang dijalankan — menggunakan image yang sama menghindari duplikasi build dan memastikan konsistensi versi kode antar proses.

---

## 3. Dockerfile Backend (Multi-Stage Build)

```dockerfile
# Stage 1: Build dependencies (composer)
FROM composer:2 AS composer-build
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

# Stage 2: Production image
FROM php:8.3-fpm-alpine AS production
RUN apk add --no-cache postgresql-dev libzip-dev \
    && docker-php-ext-install pdo_pgsql zip opcache

# Menjalankan sebagai non-root user (lihat security-design.md)
RUN addgroup -g 1000 appgroup && adduser -u 1000 -G appgroup -D appuser

WORKDIR /var/www/html
COPY --from=composer-build /app/vendor ./vendor
COPY . .
RUN composer dump-autoload --optimize

USER appuser
EXPOSE 9000
CMD ["php-fpm"]
```

**Alasan multi-stage build:** Tahap `composer-build` yang membutuhkan tools composer lengkap **tidak ikut terbawa** ke image final — image production hanya berisi kode aplikasi dan dependency runtime, menghasilkan image yang lebih kecil dan permukaan serangan yang lebih sempit (sejalan dengan prinsip *least privilege* pada [`security-design.md`](security-design.md)).

---

## 4. Docker Compose — Development

```yaml
version: "3.8"
services:
  nginx:
    image: nginx:1.25-alpine
    ports: ["8000:80"]
    volumes:
      - ./nginx/dev.conf:/etc/nginx/conf.d/default.conf
      - ./pos-backend:/var/www/html
    depends_on: [app]

  app:
    build:
      context: ./pos-backend
      target: development   # stage terpisah dengan Xdebug aktif
    volumes:
      - ./pos-backend:/var/www/html   # bind mount untuk live-reload saat development
    environment:
      - APP_ENV=local
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on: [postgres, redis]

  queue-worker:
    build:
      context: ./pos-backend
      target: development
    command: php artisan queue:work --tries=3
    volumes:
      - ./pos-backend:/var/www/html
    depends_on: [postgres, redis]

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=pos_dev
      - POSTGRES_PASSWORD=secret_dev_only
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports: ["5432:5432"]   # terbuka untuk akses tools lokal (DBeaver, dsb.) saat development

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  postgres_data:
```

**Alasan `volumes` bind mount pada development:** Memungkinkan developer mengubah kode di mesin lokal dan langsung tercermin di container tanpa perlu rebuild image berulang kali, mempercepat siklus development.

---

## 5. Docker Compose — Production

```yaml
version: "3.8"
services:
  nginx:
    image: pos-registry/nginx:${IMAGE_TAG}
    restart: always
    ports: ["443:443"]
    volumes:
      - ./ssl:/etc/nginx/ssl:ro
    depends_on: [app]

  app:
    image: pos-registry/app:${IMAGE_TAG}
    restart: always
    environment:
      - APP_ENV=production
      - DB_HOST=${DB_HOST}          # dari secret, bukan hardcoded (lihat security-design.md)
      - REDIS_HOST=${REDIS_HOST}
    # Tidak ada bind mount — image bersifat immutable di production

  queue-worker:
    image: pos-registry/app:${IMAGE_TAG}
    restart: always
    command: php artisan queue:work --tries=3 --max-jobs=1000
    deploy:
      replicas: 2   # dapat diskalakan sesuai beban queue

  postgres:
    image: postgres:16-alpine
    restart: always
    volumes:
      - /mnt/ebs-data/postgres:/var/lib/postgresql/data   # EBS volume terpisah, bukan container-local
    # Tidak expose port ke luar — hanya diakses via internal network (lihat deployment-architecture.md)

  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - /mnt/ebs-data/redis:/data
```

**Perbedaan kunci Development vs Production:**

| Aspek | Development | Production |
|---|---|---|
| Source code | Bind mount (live reload) | Baked ke dalam image (immutable) |
| Xdebug | Aktif | Tidak diinstal sama sekali |
| Port database/Redis | Terekspos ke host untuk debugging | Tidak terekspos ke luar container network |
| Restart policy | Manual | `restart: always` untuk resiliensi (NFR-AV-03) |
| Image | Dibangun lokal | Ditarik dari container registry hasil CI/CD |

---

## 6. Container Networking

```
┌─────────────────────────────────────────────┐
│           Docker Network: pos_network         │
│                                                │
│  nginx ──▶ app ──▶ postgres                   │
│              │                                │
│              └───▶ redis                       │
│                                                │
│  queue-worker ──▶ postgres, redis              │
│  scheduler ──▶ postgres, redis                 │
└─────────────────────────────────────────────┘
```

Seluruh container berkomunikasi melalui **Docker bridge network internal** menggunakan nama service sebagai hostname (mis. `postgres`, `redis`) — tidak melalui IP publik/host, mendukung isolasi jaringan sesuai [`security-design.md`](security-design.md#6-keamanan-level-container--infrastruktur).

---

## 7. Health Check Container

```yaml
app:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9000/health"]
    interval: 30s
    timeout: 5s
    retries: 3
```

Mendukung NFR-OB-02 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md) — orkestrator (Docker Compose/deployment tooling) dapat mendeteksi container yang tidak sehat dan melakukan restart otomatis.

---

## 8. Traceability

| Aspek Docker | Terkait Dokumen |
|---|---|
| Topologi infrastruktur fisik | [`deployment-architecture.md`](deployment-architecture.md) |
| Build & push image otomatis | [`cicd-flow.md`](cicd-flow.md) |
| Keamanan container | [`security-design.md`](security-design.md) |
| Struktur kode yang di-build | [`folder-structure.md`](folder-structure.md) |

---

**Sebelumnya:** [`deployment-architecture.md`](deployment-architecture.md) — Deployment Architecture
**Selanjutnya:** [`cicd-flow.md`](cicd-flow.md) — CI/CD Flow
