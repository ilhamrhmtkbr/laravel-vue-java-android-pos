# Approval Flow — POS

> Dokumen ini menjelaskan secara rinci **alur persetujuan (approval workflow)** untuk aksi-aksi sensitif dalam sistem. Approval flow adalah kontrol utama terhadap risiko kecurangan (fraud) dan kesalahan operasional, sebagaimana dijelaskan pada [`business-requirements.md`](business-requirements.md#8-risiko-bisnis-business-risks) dan ditegakkan melalui aturan pada [`business-rules.md`](business-rules.md#8-aturan-terkait-approval-berjenjang).

---

## 1. Prinsip Umum Approval

1. **Approval berjenjang mengikuti hierarki organisasi** — dari Supervisor → Branch Manager → Head Office, sesuai besaran risiko/nilai aksi yang diajukan.
2. **Tidak ada self-approval** (BRULE-AP-01) — pengaju tidak pernah menjadi penyetuju atas pengajuannya sendiri.
3. **Eskalasi otomatis** — jika suatu permintaan melebihi kewenangan level approver saat ini, sistem meneruskan ke level berikutnya tanpa perlu pengajuan ulang oleh pemohon.
4. **Semua keputusan approval final dan tercatat permanen** (BRULE-AP-03) — koreksi dilakukan melalui transaksi baru, bukan mengubah histori approval.
5. **Approval bersifat auditable** — setiap keputusan mencatat siapa, kapan, dan alasan (jika ditolak).

---

## 2. Jenis Aksi yang Memerlukan Approval

| Jenis Aksi | Trigger Approval | Level Approval Awal |
|---|---|---|
| Void transaksi/item | Selalu memerlukan approval | Supervisor |
| Diskon manual di luar batas otomatis | Melebihi threshold terkonfigurasi | Supervisor/Branch Manager (tergantung besaran) |
| Retur penjualan | Selalu memerlukan approval | Supervisor |
| Penyesuaian stok manual | Selalu memerlukan approval | Branch Manager |
| Selisih kas saat tutup shift | Selisih melebihi batas toleransi | Supervisor |
| Transaksi piutang melebihi limit | Melebihi limit piutang pelanggan | Branch Manager |
| Perubahan harga produk (level cabang, jika diizinkan) | Selalu memerlukan approval | Head Office Admin |

---

## 3. Alur Umum Approval (State Diagram)

```
                ┌───────────┐
                │  DRAFT    │  (permintaan dibuat oleh pengaju)
                └─────┬─────┘
                      ▼
                ┌───────────┐
                │  PENDING  │──────────────┐
                └─────┬─────┘              │
           ┌──────────┼──────────┐         │ (melebihi kewenangan
           ▼          ▼          ▼         │  approver saat ini)
     ┌──────────┐┌──────────┐┌──────────┐  │
     │ APPROVED ││ REJECTED ││ESCALATED │◀─┘
     └──────────┘└──────────┘└─────┬────┘
                                    │
                                    ▼
                              (kembali ke PENDING
                               pada level approver
                               berikutnya)
```

**Penjelasan status:**

| Status | Deskripsi |
|---|---|
| `DRAFT` | Permintaan baru dibuat, belum dikirim untuk approval (jarang dipakai — umumnya langsung ke PENDING) |
| `PENDING` | Menunggu keputusan dari approver yang berwenang saat ini |
| `APPROVED` | Disetujui, aksi terkait dieksekusi otomatis oleh sistem |
| `REJECTED` | Ditolak, aksi terkait dibatalkan, pengaju menerima notifikasi beserta alasan |
| `ESCALATED` | Permintaan diteruskan ke level approval berikutnya karena melebihi kewenangan approver saat ini |

---

## 4. Alur Detail per Jenis Aksi

### 4.1 Void Transaksi

```
Kasir ajukan void (dengan alasan)
    → Sistem cek: apakah ada Supervisor aktif di cabang tsb?
        Ya → Kirim ke Supervisor
        Tidak → Kirim ke Branch Manager
    → Approver review detail transaksi
    → Approve → Transaksi dibatalkan, stok dikembalikan (jika relevan), log tercatat
    → Reject → Transaksi tetap berlaku, kasir diberi notifikasi alasan penolakan
```

### 4.2 Diskon Manual Bertingkat

Mengacu pada threshold di [`role-permission.md`](role-permission.md#6-permission-bertingkat-pada-threshold-tertentu):

```
Kasir input diskon manual
    → ≤ 10%  → Diterapkan langsung tanpa approval
    → 10%–25% → PENDING, dikirim ke Supervisor
    → > 25%   → PENDING, dikirim langsung ke Branch Manager (skip Supervisor)
```

**Alasan bisnis:** Diskon besar berdampak langsung pada margin keuntungan, sehingga kewenangannya dipegang oleh posisi dengan akuntabilitas lebih tinggi.

### 4.3 Penyesuaian Stok

```
Staff Gudang/Kasir ajukan penyesuaian (dengan alasan & bukti jika ada)
    → Dikirim ke Branch Manager
    → Branch Manager review (dapat melakukan verifikasi fisik tambahan)
    → Approve → Stok sistem disesuaikan, tercatat di kartu stok sebagai "penyesuaian"
    → Reject → Stok tidak berubah, status penyesuaian ditutup sebagai ditolak
```

### 4.4 Selisih Kas Saat Tutup Shift

```
Kasir input kas fisik saat tutup shift
    → Sistem hitung selisih (kas fisik − kas seharusnya)
    → Selisih dalam batas toleransi → Shift ditutup otomatis
    → Selisih melebihi batas toleransi → PENDING ke Supervisor
        → Supervisor investigasi & approve/reject penutupan shift
        → Approve → Shift ditutup, selisih tercatat sebagai catatan resmi
        → Reject → Kasir diminta menghitung ulang kas fisik
```

### 4.5 Transaksi Piutang Melebihi Limit (Distributor)

```
Sales/Kasir buat transaksi kredit
    → Sistem cek sisa limit piutang pelanggan
    → Dalam limit → Transaksi diproses langsung
    → Melebihi limit → PENDING ke Branch Manager
        → Branch Manager review histori pelanggan
        → Approve → Transaksi diproses, limit sementara dilampaui dengan catatan approval
        → Reject → Transaksi dibatalkan, sales diarahkan menagih piutang lama terlebih dahulu
```

---

## 5. Notifikasi dalam Alur Approval

Setiap perubahan status pada alur approval memicu notifikasi (detail lihat [`notifications.md`](notifications.md)):

| Event | Penerima Notifikasi |
|---|---|
| Permintaan baru masuk (`PENDING`) | Approver yang berwenang saat ini |
| Permintaan dieskalasi (`ESCALATED`) | Approver level berikutnya |
| Permintaan disetujui (`APPROVED`) | Pengaju |
| Permintaan ditolak (`REJECTED`) | Pengaju (beserta alasan) |
| Permintaan pending terlalu lama (SLA terlampaui) | Approver saat ini + tembusan ke level di atasnya |

---

## 6. SLA Approval (Service Level Expectation)

| Jenis Aksi | Ekspektasi Waktu Respons Approver |
|---|---|
| Void transaksi | Sesegera mungkin (real-time, pelanggan menunggu) |
| Diskon manual | Sesegera mungkin (real-time, pelanggan menunggu) |
| Penyesuaian stok | Maks. 1 hari kerja |
| Selisih kas tutup shift | Sebelum shift berikutnya dimulai |
| Piutang melebihi limit | Maks. beberapa jam (tergantung kebijakan cabang) |

**Alasan bisnis:** Aksi yang melibatkan pelanggan menunggu di kasir (void, diskon) memerlukan respons instan, sedangkan aksi administratif (stok, piutang) memiliki toleransi waktu lebih longgar namun tetap terbatas agar operasional tidak terhambat.

---

## 7. Traceability

| Approval Flow | Terkait Dokumen |
|---|---|
| Aturan mendasar approval | [`business-rules.md`](business-rules.md#8-aturan-terkait-approval-berjenjang) |
| Role & kewenangan approver | [`role-permission.md`](role-permission.md) |
| Implementasi teknis | [`authorization-design.md`](../technical/authorization-design.md), [`module-design.md`](../technical/module-design.md) |
| Notifikasi terkait | [`notifications.md`](notifications.md) |

---

**Sebelumnya:** [`business-rules.md`](business-rules.md) — Business Rules
**Selanjutnya:** [`dashboard.md`](dashboard.md) — Dashboard
