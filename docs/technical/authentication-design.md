# Authentication Design — POS

> Dokumen ini menjelaskan **desain otentikasi** — bagaimana pengguna login, bagaimana sesi/token dikelola, dan bagaimana keamanan otentikasi ditegakkan di seluruh klien (Web, Mobile). Dokumen ini mengimplementasikan FR-AU-01 pada [`functional-requirements.md`](../business/functional-requirements.md) dan NFR-SE-02, NFR-SE-05 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md).

---

## 1. Keputusan: Laravel Sanctum (Token-Based Authentication)

Sistem menggunakan **Laravel Sanctum** untuk otentikasi API, bukan session-based cookie authentication tradisional maupun OAuth2 penuh (Passport).

| Pertimbangan | Sanctum (Dipilih) | Session Cookie | OAuth2 (Passport) |
|---|---|---|---|
| Kesesuaian multi-klien (Web + Mobile) | Baik — token sederhana, mudah digunakan di Android | Buruk — cookie tidak ideal untuk native mobile app | Baik, namun overkill untuk kebutuhan internal |
| Kompleksitas implementasi | Rendah | Rendah (namun terbatas untuk mobile) | Tinggi |
| Kesesuaian kebutuhan | Sesuai — sistem internal perusahaan, bukan platform pihak ketiga yang butuh OAuth scope kompleks | — | Fitur seperti scope/consent screen tidak relevan untuk sistem internal |

**Alasan teknis:** Sanctum dirancang khusus untuk kasus **SPA (Vue) dan mobile app native** yang mengakses API milik sendiri (first-party), sesuai persis dengan kebutuhan sistem ini. OAuth2 penuh (Passport) menambahkan kompleksitas (client credentials, scope, consent) yang tidak dibutuhkan karena seluruh klien (Vue, Android) adalah aplikasi resmi perusahaan sendiri, bukan aplikasi pihak ketiga.

---

## 2. Alur Otentikasi

### 2.1 Login

```
Klien (Vue/Android) → POST /api/v1/auth/login {email, password, branch_code}
    → AuthController::login()
    → AuthService::authenticate()
        ├─ Validasi kredensial (password_hash dicocokkan menggunakan bcrypt/argon2)
        ├─ Cek status pegawai aktif (is_active = true)
        ├─ Cek cabang yang dipilih sesuai penempatan pegawai (kecuali role level pusat)
        ├─ Generate token Sanctum (personal access token)
        └─ Catat log login (mendukung NFR-AU-01, FR-AU-04)
    → Response: { token, employee data, permissions }
```

### 2.2 Penggunaan Token pada Request Berikutnya

```
Klien menyertakan header: Authorization: Bearer <token>
    → Middleware auth:sanctum memvalidasi token
    → Middleware EnsureBranchScope menetapkan konteks branch_id dari data pegawai yang login
    → Request diteruskan ke Controller
```

### 2.3 Logout

```
Klien → POST /api/v1/auth/logout
    → Token yang digunakan saat ini dihapus/direvoke dari tabel auth_tokens
    → Klien menghapus token yang tersimpan secara lokal
```

---

## 3. Penyimpanan Password

| Aspek | Keputusan | Alasan |
|---|---|---|
| Algoritma hashing | bcrypt (default Laravel) atau argon2id | Standar industri, tahan terhadap brute force (NFR-SE-02) |
| Password minimum | Minimal 8 karakter, kombinasi huruf & angka (dikonfigurasi via validation rule) | Mitigasi dasar terhadap password lemah |
| Password tidak pernah di-log | Log aplikasi memfilter/redact field password secara eksplisit | Mencegah kebocoran kredensial melalui log (lihat [`logging-strategy.md`](logging-strategy.md)) |

---

## 4. Manajemen Sesi & Token

### 4.1 Karakteristik Token

- Token bersifat **personal access token** (satu token per sesi login, bukan token tunggal per pengguna) — memungkinkan pengguna login dari beberapa perangkat (misal kasir login di terminal berbeda) dengan token terpisah yang dapat direvoke secara independen.
- Token memiliki **masa berlaku (expiry)** yang dikonfigurasi (misal 8 jam, mendekati durasi shift kerja) untuk memenuhi NFR-SE-05.
- Token disimpan sebagai **hash** di database (`auth_tokens.token_hash`), bukan plain text — sejalan dengan prinsip database design pada [`erd.md`](erd.md#6-erd-relasi-auth--access).

### 4.2 Revokasi Token

| Skenario | Mekanisme |
|---|---|
| Logout manual | Token yang sedang digunakan dihapus |
| Admin menghentikan akses pegawai (resign/pelanggaran) | HO Admin/Branch Manager dapat merevoke seluruh token milik pegawai tertentu melalui endpoint admin |
| Token kadaluarsa | Otomatis ditolak oleh middleware saat validasi (`expires_at < now()`) |

**Alasan bisnis:** Kemampuan revokasi token oleh admin secara langsung menjawab NFR-SE-05 — penting khusus untuk kasus pegawai retail yang memiliki turnover tinggi (disebutkan pada NFR-US-01), di mana akses harus dapat dicabut segera tanpa menunggu token expired secara alami.

---

## 5. Otentikasi Khusus Mode Offline (Mobile)

Mengacu pada kebutuhan BR-02 dan arsitektur *offline-first* pada [`folder-structure.md`](folder-structure.md#41-penjelasan-keputusan-struktur):

```
Mobile App (Android)
    → Saat login berhasil (online), token & data pegawai di-cache secara lokal (Room database, terenkripsi)
    → Saat koneksi terputus, aplikasi menggunakan token yang di-cache untuk operasi lokal
    → Transaksi yang dibuat offline disimpan lokal dengan status "pending_sync"
    → Saat koneksi pulih, SyncService mengirim ulang request dengan token yang sama (jika belum expired)
        ├─ Jika token masih valid → sinkronisasi berhasil
        └─ Jika token sudah expired → aplikasi meminta re-login sebelum melanjutkan sinkronisasi
```

**Alasan desain:** Data pegawai yang di-cache lokal **dienkripsi** menggunakan mekanisme keamanan Android (Android Keystore) untuk mencegah kebocoran kredensial jika perangkat hilang/dicuri, mendukung NFR-SE-01/NFR-SE-02 meski dalam konteks mobile offline.

---

## 6. Rate Limiting pada Endpoint Otentikasi

| Endpoint | Batasan | Alasan |
|---|---|---|
| `/auth/login` | Maks. 5 percobaan gagal per 15 menit per kombinasi email+IP | Mitigasi brute force login (NFR-SE-04) |
| `/auth/refresh` | Maks. 10 request per menit per token | Mencegah abuse pada mekanisme refresh |

Implementasi detail rate limiting menggunakan Redis sebagai storage counter, sejalan dengan peran Redis pada [`system-architecture.md`](system-architecture.md#33-redis-cache--queue).

---

## 7. Diagram Sekuens Login Lengkap

```
Klien          Nginx        Laravel API      Redis        PostgreSQL
  │              │               │             │               │
  │─Login req──▶│               │             │               │
  │              │─forward─────▶│             │               │
  │              │               │─rate limit check─▶│         │
  │              │               │◀─OK─────────│               │
  │              │               │─cek kredensial──────────────▶│
  │              │               │◀─data pegawai─────────────────│
  │              │               │─generate token───────────────▶│ (simpan hash)
  │              │               │─catat log login──────────────▶│
  │              │◀─response─────│             │               │
  │◀─token + data│               │             │               │
```

---

## 8. Traceability

| Aspek Otentikasi | Terkait Dokumen |
|---|---|
| Otorisasi setelah login | [`authorization-design.md`](authorization-design.md) |
| Keamanan menyeluruh | [`security-design.md`](security-design.md) |
| Struktur tabel terkait | [`erd.md`](erd.md#6-erd-relasi-auth--access) |
| Kebutuhan bisnis asal | [`role-permission.md`](../business/role-permission.md) |
| Logging login | [`logging-strategy.md`](logging-strategy.md) |

---

**Sebelumnya:** [`api-contract.md`](api-contract.md) — API Contract
**Selanjutnya:** [`authorization-design.md`](authorization-design.md) — Authorization Design
