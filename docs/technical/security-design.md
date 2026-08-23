# Security Design — POS

> Dokumen ini menjelaskan **desain keamanan menyeluruh** yang melengkapi [`authentication-design.md`](authentication-design.md) dan [`authorization-design.md`](authorization-design.md). Dokumen ini mengimplementasikan NFR-SE-01 s.d. NFR-SE-06 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md) dan menjawab kebutuhan BR-10 pada [`business-requirements.md`](../business/business-requirements.md).

---

## 1. Prinsip Keamanan Menyeluruh

1. **Defense in depth** — keamanan tidak bergantung pada satu lapisan saja; validasi, otorisasi, dan constraint database bekerja bersama sebagai lapisan berlapis.
2. **Least privilege** — setiap komponen (pengguna, service, container) hanya diberi akses minimum yang dibutuhkan.
3. **Secure by default** — konfigurasi default sistem harus aman tanpa perlu pengaturan tambahan manual.
4. **Tidak ada data sensitif dalam bentuk plain text** — baik saat disimpan (at rest) maupun saat dikirim (in transit).

---

## 2. Keamanan Komunikasi (In-Transit)

| Aspek | Implementasi | Terkait NFR |
|---|---|---|
| Enkripsi transport | Seluruh trafik klien-server menggunakan HTTPS/TLS 1.2+ (terminasi di Nginx) | NFR-SE-01 |
| HSTS (HTTP Strict Transport Security) | Header `Strict-Transport-Security` diaktifkan di konfigurasi Nginx | NFR-SE-01 |
| Sertifikat SSL | Menggunakan Let's Encrypt (auto-renewal) atau sertifikat dari AWS Certificate Manager | NFR-SE-01 |

Detail konfigurasi Nginx dijelaskan pada [`deployment-architecture.md`](deployment-architecture.md).

---

## 3. Keamanan Data (At Rest)

| Aspek | Implementasi |
|---|---|
| Password | Hash menggunakan bcrypt/argon2id, tidak pernah disimpan/di-log dalam bentuk plain text (NFR-SE-02) |
| Token otentikasi | Disimpan sebagai hash di database (lihat [`authentication-design.md`](authentication-design.md#41-karakteristik-token)) |
| Data sensitif pelanggan (jika ada, mis. nomor identitas) | Dipertimbangkan enkripsi kolom (`encrypted` cast Laravel) untuk data yang secara regulasi memerlukan proteksi tambahan |
| Cache lokal mobile (Android) | Data pegawai & token dienkripsi menggunakan Android Keystore (lihat [`authentication-design.md`](authentication-design.md#5-otentikasi-khusus-mode-offline-mobile)) |

---

## 4. Proteksi Terhadap Serangan Umum

### 4.1 SQL Injection

**Mitigasi:** Seluruh akses database menggunakan Eloquent ORM/Query Builder dengan parameter binding otomatis (lihat [`repository-layer-design.md`](repository-layer-design.md)). Raw SQL yang tidak dapat dihindari (misal query agregasi laporan kompleks) tetap menggunakan parameter binding (`DB::select($sql, $bindings)`), **tidak pernah** melakukan concatenation string input pengguna langsung ke query.

### 4.2 Cross-Site Scripting (XSS)

**Mitigasi:**
- Backend API mengembalikan data sebagai JSON murni (bukan HTML) — risiko XSS klasik dari server-side rendering tidak relevan pada arsitektur API-first ini.
- Frontend Vue secara default melakukan escaping otomatis pada binding template (`{{ }}`), mencegah eksekusi skrip dari data yang ditampilkan. Penggunaan `v-html` dihindari kecuali untuk konten yang benar-benar terpercaya dan telah disanitasi.

### 4.3 Cross-Site Request Forgery (CSRF)

**Mitigasi:** Karena API bersifat *token-based* (Sanctum, bukan cookie-session untuk mobile), risiko CSRF klasik diminimalkan. Untuk skenario SPA Vue yang mungkin menggunakan cookie (opsional), Sanctum menyediakan mekanisme CSRF token bawaan yang diaktifkan sesuai konfigurasi domain.

### 4.4 Brute Force Login

**Mitigasi:** Rate limiting pada endpoint `/auth/login` (lihat [`authentication-design.md`](authentication-design.md#6-rate-limiting-pada-endpoint-otentikasi)) — maksimal 5 percobaan gagal per 15 menit per kombinasi email+IP, disimpan counter-nya di Redis.

### 4.5 Mass Assignment

**Mitigasi:** Seluruh Eloquent Model mendefinisikan `$fillable` secara eksplisit (whitelist), bukan `$guarded = []` — mencegah pengguna menyisipkan field tak terduga (misal `branch_id` atau `role` yang seharusnya tidak dapat diubah langsung dari request) melalui manipulasi payload.

### 4.6 Insecure Direct Object Reference (IDOR)

**Mitigasi:** Kombinasi UUID (tidak mudah ditebak — lihat [`database-design.md`](database-design.md#3-keputusan-uuid-sebagai-primary-key)) dan **data scoping otomatis** via `BranchScope` (lihat [`authorization-design.md`](authorization-design.md#3-data-scoping-pembatasan-akses-per-cabang)) — meski seseorang mengetahui/menebak UUID milik cabang lain, query akan tetap mengembalikan "not found" karena scope otomatis membatasi hasil.

---

## 5. Keamanan Input & Validasi

Seluruh input divalidasi di **Form Request** Laravel sebelum mencapai Controller/Service:

```php
// app/Http/Requests/Sales/StoreSaleRequest.php
public function rules(): array
{
    return [
        'shift_id' => 'required|uuid|exists:cashier_shifts,id',
        'items' => 'required|array|min:1',
        'items.*.product_id' => 'required|uuid|exists:products,id',
        'items.*.quantity' => 'required|numeric|min:0.01',
        'payments' => 'required|array|min:1',
        'payments.*.amount' => 'required|numeric|min:0.01',
    ];
}
```

**Alasan desain:** Validasi format/tipe data (Form Request) dipisah tegas dari validasi business rule (Service Layer, lihat [`service-layer-design.md`](service-layer-design.md)) — keduanya diperlukan namun untuk tujuan berbeda (lihat pembagian error 400 vs 422 pada [`api-contract.md`](api-contract.md#13-kode-status-http-yang-digunakan)).

---

## 6. Keamanan Level Container & Infrastruktur

| Aspek | Implementasi | Terkait Dokumen |
|---|---|---|
| Container tidak berjalan sebagai root | Dockerfile menetapkan non-root user untuk proses PHP-FPM | [`docker-architecture.md`](docker-architecture.md) |
| Firewall/Security Group AWS | Hanya port 80/443 terbuka ke publik; port database & Redis hanya dapat diakses dari internal network | [`deployment-architecture.md`](deployment-architecture.md) |
| Environment variable sensitif | Kredensial (DB password, API keys) disimpan sebagai secret, tidak pernah di-commit ke repository | [`cicd-flow.md`](cicd-flow.md) |
| Update dependency rutin | CI pipeline menjalankan pemeriksaan vulnerability pada dependency (Composer/NPM audit) | [`cicd-flow.md`](cicd-flow.md) |

---

## 7. Audit Trail Sebagai Kontrol Keamanan

Mengacu pada NFR-AU-01, NFR-AU-02 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md), seluruh perubahan pada data sensitif dicatat:

```php
// Contoh: mencatat perubahan harga (BRULE-PR-01)
class ProductPriceHistory extends Model
{
    protected $fillable = ['product_id', 'old_price', 'new_price', 'changed_by', 'changed_at'];
}
```

**Prinsip:** Tabel audit **tidak memiliki endpoint update/delete** yang diekspos ke API manapun — hanya dapat diisi melalui event listener otomatis saat perubahan data sensitif terjadi (menegakkan NFR-AU-02 secara struktural, bukan hanya kebijakan).

---

## 8. Kebijakan Sesi & Perangkat

| Kebijakan | Implementasi |
|---|---|
| Satu pegawai dapat memiliki beberapa sesi aktif (multi-device) | Sesuai kebutuhan operasional (kasir dapat login di lebih dari satu terminal secara bergantian) |
| Admin dapat melihat & mencabut sesi aktif pegawai tertentu | Endpoint admin memanfaatkan tabel `auth_tokens` untuk menampilkan daftar sesi dan opsi revoke |
| Sesi otomatis expired sesuai durasi shift kerja | Mendukung NFR-SE-05, mengurangi risiko token yang lupa di-logout aktif tanpa batas waktu |

---

## 9. Ringkasan Pemetaan Keamanan ke NFR

| NFR | Kontrol Keamanan Terkait |
|---|---|
| NFR-SE-01 (Enkripsi transport) | TLS di Nginx, HSTS |
| NFR-SE-02 (Password hashing) | bcrypt/argon2id |
| NFR-SE-03 (Otorisasi per endpoint) | Middleware permission, Policy |
| NFR-SE-04 (Proteksi serangan umum) | Eloquent parameter binding, escaping Vue, rate limiting |
| NFR-SE-05 (Sesi expiry & revoke) | Token expiry, admin revoke endpoint |
| NFR-SE-06 (Data scoping cabang) | `BranchScope` Eloquent |

---

## 10. Traceability

| Aspek Keamanan | Terkait Dokumen |
|---|---|
| Otentikasi | [`authentication-design.md`](authentication-design.md) |
| Otorisasi | [`authorization-design.md`](authorization-design.md) |
| Infrastruktur & network | [`deployment-architecture.md`](deployment-architecture.md), [`docker-architecture.md`](docker-architecture.md) |
| Audit & logging | [`logging-strategy.md`](logging-strategy.md) |
| Kebutuhan bisnis asal | [`non-functional-requirements.md`](../business/non-functional-requirements.md#5-keamanan-security) |

---

**Sebelumnya:** [`repository-layer-design.md`](repository-layer-design.md) — Repository Layer Design
**Selanjutnya:** [`deployment-architecture.md`](deployment-architecture.md) — Deployment Architecture
