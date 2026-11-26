# Paket kerja awal AlurJet

> Status semua kartu: **direncanakan**. Cakupan lengkap setiap milestone tetap ada di [roadmap](../../ROADMAP.md).

Ambil satu kartu pada satu waktu. Cocokkan implementasi dengan [desain sistem](../../SYSTEM_DESIGN.md) dan [matriks verifikasi](verification.md). Catat hasil serta tugas berikutnya di [worklog](worklog.md).

## ALJ-01 — Fondasi API dan lingkungan demo

- **Prasyarat:** Go, MySQL, dan Docker Compose untuk jalur lokal yang dipilih.
- **Cakupan:** modul Go, bootstrap Fiber dengan wiring manual, konfigurasi, log JSON, `GET /healthz`, dan bootstrap Compose M0 dengan satu sumber konfigurasi MySQL serta service `migrate`. Siapkan mock receiver `POST /orders` dengan mode di roadmap.
- **Selesai ketika:** build/unit test lulus, API bisa diakses setelah startup lokal, milestone lulus dari volume kosong dan startup ulang; migrasi gagal menahan API/worker dan terlihat di status/log; mock memiliki perilaku `success`, `fail_once` per key, serta `always_fail` yang teruji.
- **Tugas terpisah:** import flow, ingest, dan worker.

## ALJ-02 — Import dan simpan satu flow YAML

- **Prasyarat:** ALJ-01.
- **Cakupan:** parser DSL, resolusi URL saat import, penyimpanan YAML dan definisi terkompilasi, serta `POST /api/v1/flows` dengan auth admin. Gunakan [fixture YAML kanonis](../examples/order-to-api.yaml) dan ikuti desain §§4, 6–8.
- **Selesai ketika:** fixture lolos dengan satu HTTP POST di step terakhir; konfigurasi invalid atau dua step HTTP menghasilkan `422`; slug duplikat menghasilkan `409`; token yang keliru menghasilkan `401`; definisi terkompilasi menyimpan URL teresolusi.
- **Tugas berikutnya dalam M1:** tuntaskan list, detail, dan update serta unit test step validasi/transformasi yang diwajibkan roadmap sebelum M1 ditutup.

## ALJ-03 — Demo webhook sampai mock target

- **Prasyarat:** ALJ-02 dan gate M1 pada roadmap.
- **Cakupan:** kerjakan ingest, dedup, dan kapasitas MySQL lebih dulu, lalu claim/finalisasi, recovery lease minimum, runner dan HTTP connector. Scope token ke slug, batasi output tiap step dan header respons target, lalu tutup body tanpa membacanya. Finalisasi M2 mencakup klasifikasi hasil, backoff, dan penulisan jadwal retry secara atomik. Gunakan [event pesanan](../examples/order-event.json) dan target mock lokal. Ikuti desain §§4–8.
- **Selesai ketika:** setelah `202`, worker bisa claim execution pada polling berikutnya; request identik mengembalikan ID yang sama; ID sama dengan byte body berbeda menghasilkan `409`; JSON dengan nama field berulang menghasilkan `400` tanpa insert; mock menerima hasil pemetaan termasuk angka `9007199254740993` tanpa perubahan; claim ganda dan finalisasi setelah expiry ditolak; respons `500` dengan jatah tersisa menutup attempt dan menyimpan `retry_scheduled` beserta `available_at` di masa depan; `400` dari fixture HTTP khusus tes integrasi menjadi `dead_letter`; ukuran JSON di atas batas juga menjadi `dead_letter` tanpa koneksi ke target.
- **Tugas terpisah:** verifikasi retry berulang, exhaustion, recovery serentak, history endpoint, dan replay di M3. Periksa hasil demo M2 dari mock dan database/tes integrasi sampai endpoint pemeriksaan hasil tersedia.

## Setelah tiga kartu awal

Pecah M3 menjadi verifikasi retry berulang, uji concurrency recovery M2, serta history dan replay. Tuntaskan dokumentasi dan demo yang dapat diulang di M4. Perubahan di luar v0.1.0 ada di [backlog setelah MVP](../post-mvp.md).
