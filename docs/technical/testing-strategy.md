# Testing Strategy — POS

> Dokumen ini menjelaskan **strategi pengujian** di seluruh layer sistem — dari unit test business logic hingga end-to-end test lintas komponen. Dokumen ini menegakkan NFR-MT-03 pada [`non-functional-requirements.md`](../business/non-functional-requirements.md) dan secara khusus menerjemahkan skenario pada [`edge-cases.md`](../business/edge-cases.md) menjadi test case konkret.

---

## 1. Prinsip Strategi Pengujian

1. **Test Pyramid** — proporsi terbesar adalah unit test (cepat, murah, banyak), diikuti integration/feature test, dan paling sedikit end-to-end test (lambat, mahal, namun paling mendekati pengalaman pengguna nyata).
2. **Business rule kritis wajib memiliki test eksplisit** — setiap `BRULE-xxx` pada [`business-rules.md`](../business/business-rules.md) harus dapat ditelusuri ke minimal satu test case.
3. **Edge case bukan "nice to have"** — skenario pada [`edge-cases.md`](../business/edge-cases.md) yang berprioritas Kritis dan Tinggi **wajib** memiliki test case, bukan opsional.
4. **Test berjalan otomatis di setiap perubahan kode** — terintegrasi penuh dengan pipeline pada [`cicd-flow.md`](cicd-flow.md), tidak bergantung pada eksekusi manual.

---

## 2. Test Pyramid untuk Sistem Ini

```
                     ▲
                   ╱   ╲
                  ╱ E2E ╲              (Sedikit — alur kritis lintas sistem)
                 ╱───────╲
                ╱ Integr- ╲           (Sedang — interaksi Service+Repository+DB)
               ╱  ation    ╲
              ╱─────────────╲
             ╱   Unit Test   ╲      (Banyak — logic murni, Service dengan mock)
            ╱─────────────────╲
```

---

## 3. Unit Test

### 3.1 Cakupan

Menguji **Service Layer** secara terisolasi (dependency Repository di-mock), tanpa menyentuh database atau HTTP layer.

### 3.2 Contoh: Menguji Business Rule BRULE-ST-01 (Stok Tidak Boleh Negatif)

```php
// tests/Unit/Services/StockServiceTest.php
public function test_deduct_stock_throws_exception_when_insufficient(): void
{
    $mockBatchRepo = Mockery::mock(ProductBatchRepositoryInterface::class);
    $mockBatchRepo->shouldReceive('getAvailableBatchesFEFO')
        ->andReturn(collect([
            (object)['id' => 'batch-1', 'quantity_available' => 2]
        ]));

    $service = new StockService($mockBatchRepo, $this->mockStockRepo, $this->mockLedgerRepo);

    $this->expectException(InsufficientStockException::class);

    $service->deductStockForSale([
        ['product_id' => 'prod-1', 'quantity' => 5]  // diminta 5, tersedia hanya 2
    ], 'branch-1');
}
```

### 3.3 Contoh: Menguji Business Rule BRULE-AP-01 (Larangan Self-Approval)

```php
// tests/Unit/Policies/ApprovalRequestPolicyTest.php
public function test_employee_cannot_approve_own_request(): void
{
    $employee = Employee::factory()->create();
    $request = ApprovalRequest::factory()->create(['requested_by' => $employee->id]);

    $policy = new ApprovalRequestPolicy();

    $this->assertFalse($policy->decide($employee, $request));
}
```

---

## 4. Feature/Integration Test

### 4.1 Cakupan

Menguji endpoint API end-to-end **dengan database sungguhan** (test database), memastikan seluruh layer (Controller → Service → Repository → Database) bekerja bersama dengan benar.

### 4.2 Contoh: Menguji Endpoint Transaksi Penjualan (BRULE-TX-02)

```php
// tests/Feature/Sales/CreateSaleTest.php
public function test_sale_rejected_when_payment_amount_mismatch(): void
{
    $cashier = Employee::factory()->cashier()->create();
    $shift = CashierShift::factory()->open()->for($cashier)->create();
    $product = Product::factory()->create(['selling_price' => 10000]);

    $response = $this->actingAs($cashier)->postJson('/api/v1/sales', [
        'shift_id' => $shift->id,
        'items' => [['product_id' => $product->id, 'quantity' => 1]],
        'payments' => [['payment_method' => 'cash', 'amount' => 5000]],  // kurang dari total
    ]);

    $response->assertStatus(422)
        ->assertJsonPath('error_code', 'PAYMENT_MISMATCH');
}
```

### 4.3 Contoh: Menguji Data Scoping Antar Cabang (BR-10)

```php
public function test_branch_manager_cannot_access_other_branch_sales(): void
{
    $branchA = Branch::factory()->create();
    $branchB = Branch::factory()->create();
    $managerA = Employee::factory()->branchManager()->for($branchA)->create();
    $saleInBranchB = Sale::factory()->for($branchB)->create();

    $response = $this->actingAs($managerA)->getJson("/api/v1/sales/{$saleInBranchB->id}");

    $response->assertStatus(404);  // scope membuat data seolah tidak ada, bukan 403 (lihat authorization-design.md)
}
```

---

## 5. Test Konkurensi (Khusus Skenario Kritis)

### 5.1 Cakupan

Menguji skenario **race condition**, khususnya EC-SL-02 pada [`edge-cases.md`](../business/edge-cases.md) — dua transaksi bersamaan memperebutkan stok terakhir.

```php
// tests/Feature/Inventory/ConcurrentStockDeductionTest.php
public function test_concurrent_sales_do_not_oversell_last_stock_unit(): void
{
    $product = Product::factory()->create();
    ProductBatch::factory()->for($product)->create(['quantity_available' => 1]);

    // Simulasikan dua request paralel (menggunakan process/job terpisah dalam test)
    $results = $this->runConcurrently([
        fn() => $this->createSaleFor($product, 1),
        fn() => $this->createSaleFor($product, 1),
    ]);

    $successCount = collect($results)->filter(fn($r) => $r->status() === 201)->count();

    $this->assertEquals(1, $successCount, 'Hanya satu transaksi yang boleh berhasil');
}
```

**Catatan implementasi:** Test konkurensi murni memerlukan setup khusus (multiple database connection/proses paralel) yang lebih kompleks dibanding test biasa — dijalankan sebagai kategori test terpisah dalam pipeline CI, tidak digabung dengan feature test reguler agar tidak memperlambat siklus CI utama.

---

## 6. End-to-End (E2E) Test

### 6.1 Cakupan

Menguji **alur pengguna lengkap** melalui antarmuka sungguhan (browser untuk web, emulator/device untuk mobile), mencakup alur kritis dari [`use-case.md`](../business/use-case.md).

### 6.2 Alur Kritis yang Wajib Memiliki E2E Test

| Alur | Terkait Use Case |
|---|---|
| Buka shift → transaksi penjualan → tutup shift | UC-01, UC-03 |
| Void transaksi dengan approval Supervisor | UC-02, UC-04 |
| Transfer stok antar cabang lengkap | UC-08 |
| Alur open bill F&B: pesan → dapur → bayar | UC-11 |
| Validasi resep sebelum transaksi selesai (Apotek) | UC-10 |

### 6.3 Contoh (Playwright — Frontend Web)

```javascript
test('kasir dapat menyelesaikan transaksi penjualan lengkap', async ({ page }) => {
  await loginAsCashier(page);
  await page.click('[data-testid="open-shift-button"]');
  await page.fill('[data-testid="opening-balance-input"]', '500000');
  await page.click('[data-testid="confirm-open-shift"]');

  await page.fill('[data-testid="barcode-input"]', '8991234567890');
  await page.click('[data-testid="pay-button"]');
  await page.fill('[data-testid="cash-amount-input"]', '50000');
  await page.click('[data-testid="confirm-payment"]');

  await expect(page.locator('[data-testid="receipt-success"]')).toBeVisible();
});
```

---

## 7. Pengujian Khusus Mobile (Android)

| Jenis Test | Tools | Cakupan |
|---|---|---|
| Unit Test | JUnit + Mockito | `domain/usecase/`, business logic murni |
| Instrumented Test | Espresso | Interaksi UI pada Activity/Fragment |
| Sinkronisasi Offline | Test khusus dengan simulasi network terputus | Menguji `SyncService` sesuai EC-SL-01 |

### 7.1 Contoh: Menguji Sinkronisasi Offline

```java
@Test
public void testOfflineTransactionSyncsWhenConnectionRestored() {
    networkSimulator.disconnect();
    Sale sale = createTestSale();
    salesRepository.save(sale);  // tersimpan lokal dengan status pending_sync

    assertEquals(SyncStatus.PENDING, sale.getSyncStatus());

    networkSimulator.reconnect();
    syncService.triggerSync();

    assertEquals(SyncStatus.SYNCED, salesRepository.findById(sale.getId()).getSyncStatus());
}
```

---

## 8. Pemetaan Edge Case Kritis ke Test Case

Mengacu pada prioritas pada [`edge-cases.md`](../business/edge-cases.md#10-ringkasan-prioritas-penanganan-edge-case):

| Edge Case | Jenis Test | Wajib? |
|---|---|---|
| EC-SL-02 (race condition stok) | Test konkurensi | Wajib |
| EC-SL-05 (transaksi lintas hari kalender) | Feature test | Wajib |
| EC-ST-01 (opname saat transaksi berjalan) | Feature test | Wajib |
| EC-SH-01 (shift lupa ditutup) | Feature test + scheduled job test | Wajib |
| EC-SL-01 (transaksi saat offline) | E2E test (mobile) | Tinggi |
| EC-PH-02 (Apoteker tidak tersedia) | Feature test | Tinggi |

---

## 9. Code Coverage Target

| Layer | Target Coverage | Alasan |
|---|---|---|
| Service Layer | ≥ 85% | Business logic paling kritis, harus teruji ketat |
| Repository Layer | ≥ 70% | Fokus pada query kompleks/kritis, bukan CRUD sederhana |
| Controller | ≥ 60% | Lebih banyak diverifikasi via Feature Test |

**Catatan:** Coverage tinggi **bukan tujuan akhir** — tujuannya adalah kepercayaan diri bahwa business rule kritis benar-benar ditegakkan. Target di atas adalah panduan minimum, bukan pengganti penilaian kualitas test secara substansial.

---

## 10. Traceability

| Aspek Testing | Terkait Dokumen |
|---|---|
| Business rule yang diuji | [`business-rules.md`](../business/business-rules.md) |
| Skenario edge case sumber test | [`edge-cases.md`](../business/edge-cases.md) |
| Use case sumber E2E test | [`use-case.md`](../business/use-case.md) |
| Eksekusi otomatis di pipeline | [`cicd-flow.md`](cicd-flow.md) |

---

**Sebelumnya:** [`backup-strategy.md`](backup-strategy.md) — Backup Strategy

---

> 🎉 **Seluruh Technical Documentation selesai.** 17 dokumen teknis (README hingga Testing Strategy) telah tersusun lengkap, konsisten, dan saling terhubung dengan 18 dokumen bisnis. Total **35 dokumen Markdown** membentuk satu repository dokumentasi enterprise yang production-ready dan siap dijadikan acuan langsung untuk memulai implementasi kode.
