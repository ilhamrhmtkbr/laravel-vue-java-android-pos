# Authorization Design — POS

> Dokumen ini menjelaskan **desain otorisasi** — bagaimana role, permission, dan data scoping ditegakkan secara teknis. Dokumen ini mengimplementasikan langsung [`role-permission.md`](../business/role-permission.md) dan menjadi lapisan keamanan utama setelah otentikasi ([`authentication-design.md`](authentication-design.md)) berhasil dilakukan.

---

## 1. Keputusan: Role-Based Access Control (RBAC) dengan Permission Granular

Sistem menggunakan **RBAC** — pengguna diberi satu atau lebih **role**, dan setiap role memiliki kumpulan **permission**. Ini bukan model role hardcoded (`if role == 'admin'`), melainkan **data-driven**: role dan permission tersimpan di database dan dapat dikonfigurasi tanpa mengubah kode aplikasi.

| Pertimbangan | RBAC Data-Driven (Dipilih) | Hardcoded Role Check |
|---|---|---|
| Fleksibilitas menambah role baru | Tinggi — cukup insert data, tanpa deploy ulang | Rendah — perlu mengubah kode setiap kali |
| Auditability perubahan hak akses | Baik — perubahan permission tercatat di database | Buruk — perubahan hanya terlihat di riwayat kode |
| Kesesuaian dengan kebutuhan (BR-04, BR-10) | Sesuai — mendukung kebutuhan role granular yang bervariasi per domain bisnis | Tidak sesuai untuk skala enterprise multi-domain |

**Alasan teknis:** Mengacu pada [`module-design.md`](module-design.md#11-modul-auth--access), modul Auth & Access bersifat cross-cutting dan digunakan seluruh modul — desain data-driven memastikan penambahan role kustom (misal role spesifik untuk domain baru di masa depan) tidak memerlukan perubahan pada modul-modul lain.

---

## 2. Struktur Implementasi RBAC

```
Employee ──belongs to many──▶ Role ──belongs to many──▶ Permission
```

Implementasi mengikuti struktur tabel pada [`erd.md`](erd.md#6-erd-relasi-auth--access): `employees`, `roles`, `permissions`, `employee_roles`, `role_permissions`.

### 2.1 Pemeriksaan Permission di Level Kode

```php
// Middleware: CheckPermission.php
public function handle($request, Closure $next, string $permission)
{
    if (!$request->user()->hasPermission($permission)) {
        throw new UnauthorizedException('Anda tidak memiliki izin untuk aksi ini');
    }
    return $next($request);
}

// Penggunaan pada route
Route::post('/sales/{id}/void', [SalesController::class, 'void'])
    ->middleware('permission:sale.void');
```

**Alasan desain:** Pemeriksaan permission dilakukan di **middleware level route**, bukan di dalam Controller/Service — memastikan tidak ada endpoint yang lupa memeriksa otorisasi (fail-safe by design), sekaligus tetap memungkinkan pemeriksaan tambahan yang lebih granular di Service Layer untuk kasus kompleks (lihat Bagian 4).

---

## 3. Data Scoping (Pembatasan Akses per Cabang)

### 3.1 Implementasi: Global Query Scope

Mengacu pada BRULE-MB-02 pada [`business-rules.md`](../business/business-rules.md), seluruh query terhadap tabel yang memiliki `branch_id` secara otomatis dibatasi sesuai cabang pengguna yang login, kecuali role level pusat (`owner`, `ho_admin`, `finance_auditor`).

```php
// app/Models/Scopes/BranchScope.php
class BranchScope implements Scope
{
    public function apply(Builder $builder, Model $model)
    {
        $user = auth()->user();
        if ($user && !$user->hasCentralAccess()) {
            $builder->where('branch_id', $user->branch_id);
        }
    }
}

// Diterapkan otomatis pada model transaksional
class SalesTransaction extends Model
{
    protected static function booted()
    {
        static::addGlobalScope(new BranchScope);
    }
}
```

**Alasan desain:** Menerapkan scoping sebagai **Global Scope Eloquent** (bukan filter manual di setiap query pada Controller/Service) memastikan **tidak ada satu pun query yang bisa lupa** menerapkan pembatasan cabang — ini adalah pertahanan struktural terhadap risiko kebocoran data antar cabang (BR-10), sejalan dengan filosofi "constraint di level yang tidak bisa dilewati" pada [`database-design.md`](database-design.md#6-constraint-level-database-sebagai-pertahanan-lapis-kedua).

### 3.2 Bypass Scoping untuk Role Pusat

```php
// app/Models/Employee.php
public function hasCentralAccess(): bool
{
    return in_array($this->role->level, ['owner', 'ho_admin', 'finance_auditor']);
}
```

---

## 4. Otorisasi Kontekstual via Laravel Policy

Untuk kasus yang lebih kompleks dari sekadar permission dan branch scoping — misalnya **larangan self-approval** (BRULE-AP-01) — sistem menggunakan Laravel Policy sebagai lapisan tambahan di Service Layer.

```php
// app/Policies/ApprovalRequestPolicy.php
public function decide(Employee $employee, ApprovalRequest $request): bool
{
    // Tidak boleh approve permintaan sendiri (BRULE-AP-01)
    if ($request->requested_by === $employee->id) {
        return false;
    }
    // Approver harus sesuai role yang berwenang saat ini
    return $employee->role->name === $request->current_approver_role;
}
```

**Alasan desain:** Middleware permission generik (`sale.void`) tidak dapat menangkap aturan kontekstual seperti "tidak boleh approve milik sendiri" — Policy memungkinkan pemeriksaan yang bergantung pada **data spesifik record**, dipanggil secara eksplisit di dalam Service (`ApprovalService::decide()`) sebelum eksekusi.

---

## 5. Permission Bertingkat Berdasarkan Threshold

Mengacu pada [`role-permission.md`](../business/role-permission.md#6-permission-bertingkat-pada-threshold-tertentu) — beberapa aksi memerlukan pemeriksaan tambahan di luar RBAC statis, karena kewenangannya bergantung pada **nilai transaksi**.

```php
// app/Services/Sales/DiscountAuthorizationService.php
public function canApplyDirectly(Employee $employee, float $discountPercentage): bool
{
    $threshold = config('business.discount_threshold.' . $employee->role->name);
    return $discountPercentage <= $threshold;
}
```

Nilai threshold disimpan di **konfigurasi yang dapat diubah tanpa deploy ulang kode** (misal melalui tabel `system_configurations`), mendukung fleksibilitas kebijakan berbeda antar cabang jika diperlukan di masa depan.

---

## 6. Matriks Implementasi vs Dokumen Bisnis

| Konsep Bisnis | Implementasi Teknis |
|---|---|
| Role & Permission ([`role-permission.md`](../business/role-permission.md)) | Tabel `roles`, `permissions`, middleware `CheckPermission` |
| Data scoping per cabang | Eloquent Global Scope `BranchScope` |
| Larangan self-approval (BRULE-AP-01) | Laravel Policy `ApprovalRequestPolicy` |
| Threshold approval bertingkat | `DiscountAuthorizationService` + konfigurasi dinamis |
| Eskalasi approval (BRULE-AP-02) | Logic di `ApprovalService`, memeriksa kewenangan level saat ini vs nilai aksi |

---

## 7. Contoh Alur Otorisasi Lengkap (Void Transaksi)

```
Request: POST /api/v1/sales/{id}/void
    → Middleware auth:sanctum (cek token valid)
    → Middleware permission:sale.void (cek Kasir/Supervisor memiliki permission ini)
    → Controller memanggil SalesVoidService::requestVoid()
        ├─ BranchScope otomatis membatasi query transaksi hanya milik cabang pengguna
        ├─ Cek shift transaksi masih aktif (BRULE-TX-03)
        ├─ Buat ApprovalRequest (status: pending)
        └─ Dispatch event ApprovalRequested
    → Approver (Supervisor) menerima notifikasi
    → Request: POST /api/v1/approval-requests/{id}/approve
        ├─ Middleware permission:approval.decide
        ├─ ApprovalRequestPolicy::decide() — cek bukan self-approval, cek role sesuai
        └─ Jika lolos → SalesVoidService::executeVoid()
```

---

## 8. Traceability

| Aspek Otorisasi | Terkait Dokumen |
|---|---|
| Definisi role & permission dari sisi bisnis | [`role-permission.md`](../business/role-permission.md) |
| Aturan kontekstual (self-approval, dll) | [`business-rules.md`](../business/business-rules.md) |
| Struktur tabel pendukung | [`erd.md`](erd.md#6-erd-relasi-auth--access) |
| Implementasi business logic terkait | [`service-layer-design.md`](service-layer-design.md) |
| Keamanan menyeluruh | [`security-design.md`](security-design.md) |

---

**Sebelumnya:** [`authentication-design.md`](authentication-design.md) — Authentication Design
**Selanjutnya:** [`service-layer-design.md`](service-layer-design.md) — Service Layer Design
