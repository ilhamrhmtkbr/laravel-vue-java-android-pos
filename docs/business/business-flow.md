# Business Flow — POS

> Dokumen ini menjelaskan **alur proses bisnis nyata** yang terjadi di lapangan, baik alur umum (berlaku di semua domain) maupun alur khusus per domain bisnis (retail, F&B, apotek, distributor). Dokumen ini menjadi acuan utama untuk [`use-case.md`](use-case.md) dan [`module-design.md`](../technical/module-design.md).

---

## 1. Alur Bisnis Umum (Berlaku di Semua Domain)

### 1.1 Siklus Operasional Harian Cabang

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Buka Toko/   │──▶│ Buka Shift   │──▶│  Transaksi   │──▶│ Tutup Shift  │──▶│ Tutup Toko/  │
│ Buka Kasir   │   │  Kasir       │   │  Berjalan    │   │  (Rekonsi-   │   │ Rekap Harian │
│              │   │ (Setor Modal)│   │              │   │   liasi Kas) │   │              │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

**Penjelasan tiap tahap:**

1. **Buka Toko/Buka Kasir** — Branch Manager/Supervisor memastikan sistem dan perangkat kasir siap (printer, laci kas, scanner terhubung).
2. **Buka Shift Kasir** — Kasir login dengan akun individu, mencatat modal kas awal (opening balance). Ini menjadi baseline rekonsiliasi di akhir shift.
3. **Transaksi Berjalan** — Kasir memproses penjualan sepanjang shift. Setiap transaksi tercatat dengan kasir & shift terkait (lihat [`transactions.md`](transactions.md)).
4. **Tutup Shift** — Kasir menghitung kas fisik, sistem membandingkan dengan kas seharusnya (opening balance + total penjualan tunai − retur tunai). Selisih (jika ada) dicatat dan memerlukan approval Supervisor.
5. **Tutup Toko/Rekap Harian** — Branch Manager meninjau rekap seluruh shift hari itu, memastikan semua shift sudah ditutup sebelum hari berikutnya dimulai.

**Alasan bisnis:** Siklus shift adalah kontrol dasar terhadap risiko kecurangan dan human error dalam pengelolaan kas — inti dari operasional ritel yang menyentuh uang tunai secara langsung.

### 1.2 Alur Umum Transaksi Penjualan

```
Pilih/Scan Produk → Keranjang Belanja → (Opsional: Terapkan Promo/Member)
    → Pilih Metode Pembayaran → Konfirmasi Pembayaran → Cetak Struk → Transaksi Selesai
```

Detail skenario alternatif (pembayaran gagal, produk tidak ditemukan, dll) dijelaskan pada [`edge-cases.md`](edge-cases.md).

### 1.3 Alur Umum Manajemen Stok

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  Purchase      │───▶│  Penerimaan   │───▶│  Stok Masuk   │
│  Order (PO)    │    │  Barang       │    │  di Cabang/   │
│  ke Supplier   │    │  (Goods       │    │  Gudang       │
│                │    │  Receipt)     │    │               │
└───────────────┘    └───────────────┘    └───────┬───────┘
                                                    │
                    ┌───────────────────────────────┼───────────────────────────────┐
                    ▼                               ▼                               ▼
            ┌───────────────┐              ┌───────────────┐              ┌───────────────┐
            │  Dijual ke    │              │  Transfer ke  │              │  Stok Opname/ │
            │  Pelanggan    │              │  Cabang Lain  │              │  Penyesuaian  │
            │  (Sales)      │              │               │              │               │
            └───────────────┘              └───────────────┘              └───────────────┘
```

**Alasan bisnis:** Stok adalah aset perusahaan yang bergerak melalui banyak titik (gudang, antar cabang, ke pelanggan). Sistem harus mampu melacak pergerakan ini secara end-to-end agar nilai persediaan selalu akurat — ini adalah fondasi dari laporan keuangan retail manapun.

---

## 2. Alur Bisnis per Domain

### 2.1 Domain: Retail / Minimarket / Supermarket

**Karakteristik utama:** volume transaksi tinggi, waktu layanan singkat, banyak SKU.

```
Pelanggan datang → Ambil produk → Kasir scan barcode →
Sistem otomatis kalkulasi harga (termasuk promo aktif) →
Pelanggan bayar → Struk dicetak → Selesai
```

**Poin bisnis penting:**
- Promo harus diterapkan otomatis tanpa intervensi manual kasir untuk menjaga kecepatan layanan.
- Produk dengan banyak varian ukuran/satuan (misal galon vs botol) harus mudah dibedakan saat scan.
- Stok harus real-time karena kecepatan perputaran barang tinggi.

### 2.2 Domain: Restoran / F&B

**Karakteristik utama:** transaksi tidak selalu selesai seketika (ada jeda antara pemesanan dan pembayaran), melibatkan proses dapur.

```
Pelanggan duduk di meja → Waiter input pesanan (meja X, item, modifier) →
Pesanan diteruskan ke dapur (kitchen order) → Dapur menyiapkan pesanan →
Pesanan diantar → (Opsional: tambah pesanan lagi - open bill) →
Pelanggan minta bill → Kasir proses pembayaran (single/split bill) → Meja direset untuk pelanggan berikutnya
```

**Poin bisnis penting:**
- Satu meja dapat memiliki satu "open bill" yang bisa ditambah item berkali-kali sebelum pembayaran final.
- Modifier menu (misal "tanpa es", "pedas level 2") harus tersampaikan jelas ke dapur.
- Split bill (bagi tagihan antar pelanggan dalam satu meja) adalah kebutuhan umum di F&B.
- Status meja (kosong, terisi, menunggu pembayaran) harus terlihat jelas oleh staff.

### 2.3 Domain: Distributor / Grosir

**Karakteristik utama:** transaksi B2B, nilai transaksi besar, hubungan pelanggan jangka panjang, sering menggunakan piutang.

```
Pelanggan (toko/reseller) melakukan pemesanan (sales order) →
Sistem cek limit piutang pelanggan → Sistem hitung harga sesuai tingkat/kelas pelanggan (tiered pricing) →
Barang disiapkan & dikirim → Invoice diterbitkan →
Pembayaran diterima (tunai/tempo) → Piutang diperbarui
```

**Poin bisnis penting:**
- Harga berbeda-beda tergantung kelas/tingkatan pelanggan (misal harga grosir besar vs kecil).
- Sistem harus mencegah transaksi baru jika pelanggan telah melewati limit piutang yang disetujui, kecuali ada approval khusus.
- Term of payment (misal net 30 hari) memengaruhi kapan piutang jatuh tempo dan dilaporkan.

### 2.4 Domain: Apotek

**Karakteristik utama:** regulasi ketat, kontrol expired, sebagian produk memerlukan resep.

```
Pelanggan datang (dengan/tanpa resep) →
Jika obat keras/resep: apoteker validasi resep terlebih dahulu →
Kasir/apoteker proses transaksi → Sistem cek status batch & expired produk yang dijual →
Pembayaran → Struk (dengan informasi obat sesuai regulasi bila diperlukan)
```

**Poin bisnis penting:**
- Produk dengan kategori "obat keras" tidak boleh dijual tanpa validasi resep tercatat di sistem.
- Sistem harus memprioritaskan penjualan stok dengan tanggal kadaluarsa terdekat (FEFO — First Expired First Out).
- Notifikasi produk mendekati expired sangat kritikal untuk mencegah kerugian dan risiko regulasi.

### 2.5 Domain: UMKM

**Karakteristik utama:** skala kecil, kebutuhan sederhana namun tetap butuh jalur pertumbuhan ke fitur enterprise.

```
Alur sama seperti Retail dasar, namun:
- Struktur cabang bisa hanya 1 outlet
- Approval workflow dapat disederhanakan (opsional diaktifkan)
- Fitur lanjutan (multi-cabang, piutang kompleks) tidak wajib digunakan sejak awal
```

**Poin bisnis penting:** Arsitektur sistem tetap sama (tidak ada versi "ringan" terpisah), namun konfigurasi memungkinkan UMKM menggunakan subset fitur sesuai kebutuhan, dengan opsi mengaktifkan fitur lanjutan seiring pertumbuhan bisnis — sejalan dengan prinsip *scalable by design* pada [`README.md`](../../README.md#5-prinsip-desain-sistem).

---

## 3. Alur Approval dalam Konteks Bisnis

Alur approval detail dijelaskan penuh pada [`approval-flow.md`](approval-flow.md), namun secara umum mengikuti pola berikut di seluruh domain:

```
Aksi Sensitif Diminta (void/diskon manual/penyesuaian stok/retur)
        │
        ▼
Sistem Cek Apakah Melebihi Threshold Approval
        │
   ┌────┴────┐
   ▼         ▼
Tidak       Ya
Perlu       Perlu
Approval    Approval ──▶ Notifikasi ke Approver ──▶ Approver Setujui/Tolak ──▶ Aksi Dieksekusi/Dibatalkan
```

---

## 4. Alur Konsolidasi Data ke Head Office

```
Transaksi di Cabang (real-time) ──▶ Data tersimpan di database pusat (arsitektur terpusat, lihat system-architecture.md)
        │
        ▼
Dashboard & Laporan Head Office menampilkan data konsolidasi otomatis
        │
        ▼
Owner/Direktur mengambil keputusan (harga, promo, ekspansi, dsb.)
```

**Alasan bisnis:** Karena sistem menggunakan **arsitektur terpusat dengan resiliensi lokal** (bukan sistem terpisah per cabang yang disatukan belakangan), data konsolidasi tersedia tanpa proses batch/manual — ini adalah keunggulan inti dibanding sistem kasir konvensional yang disebutkan pada [`business-requirements.md`](business-requirements.md#1-latar-belakang-masalah-business-problem).

---

## 5. Ringkasan Perbedaan Alur Antar Domain

| Aspek | Retail | F&B | Distributor | Apotek | UMKM |
|---|---|---|---|---|---|
| Kecepatan transaksi | Sangat cepat | Bertahap (order → bayar) | Proses (order → kirim → invoice) | Cepat, dengan validasi | Cepat |
| Melibatkan piutang | Jarang | Tidak | Ya, umum | Jarang | Jarang |
| Kontrol regulasi | Standar | Standar | Standar | Ketat (resep, obat keras) | Standar |
| Struktur order | Langsung | Meja/open bill | Sales order | Langsung/resep | Langsung |
| Kompleksitas harga | Sedang (promo) | Sedang (modifier) | Tinggi (tiered pricing) | Sedang | Rendah-sedang |

---

**Sebelumnya:** [`non-functional-requirements.md`](non-functional-requirements.md) — Non-Functional Requirements
**Selanjutnya:** [`sop.md`](sop.md) — Standard Operating Procedure
