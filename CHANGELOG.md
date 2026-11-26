# Changelog AlurJet

## Unreleased — Perubahan blueprint

Entri ini mencatat perubahan dokumen dan kontrak rancangan yang tersedia di folder ini. Status rilis runtime menunggu implementasi dan verifikasi.

### 2026-11-26 — Penutupan celah dokumen awal

- Menerapkan paket hemat ([ADR-002](docs/ai/decisions.md)): `ALURJET_ACTIVE_HIGH_WATERMARK`, `503 SYSTEM_BACKPRESSURE`, dan anggaran byte per transform ditunda ke [backlog pasca-MVP](docs/post-mvp.md). Batas ukuran kini berupa payload masuk, maksimum 64 field transform, dan body final. Entri di bawah yang menyebut ambang global atau anggaran per transform sudah digantikan.
- Menambahkan `headers` pada step `http` dengan resolusi saat dispatch ([ADR-003](docs/ai/decisions.md)) dan fixture [order-to-api-auth.yaml](docs/examples/order-to-api-auth.yaml).
- Menambahkan [referensi database](docs/reference/database.md) dengan DDL dan ERD, katalog error HTTP dan error attempt lengkap, kontrak response flow, paginasi, dan format waktu.
- Menetapkan toolchain dan konvensi koding ([ADR-005](docs/ai/decisions.md)), [panduan pengembangan](docs/getting-started/development.md) termasuk Windows, serta variabel konfigurasi baru (concurrency worker, recovery, shutdown, pool koneksi, batas YAML).
- Mendokumentasikan `/healthz` yang memeriksa DB, graceful shutdown, dukungan multi-instance, rate limit dan rotasi token, serta backup dan pertumbuhan data.
- Memecah kartu kerja agent menjadi ALJ-00 sampai ALJ-08, memisahkan Quickstart M2 dari M3, dan mengubah catatan keputusan menjadi format ADR.

- Menyederhanakan M2 ([ADR-006](docs/ai/decisions.md)): tanpa penolakan DNS campuran terpisah dan tanpa healthcheck worker, tes M2 dibatasi pada bukti inti dan sisanya pindah ke M3. Jadwal menjadi M2 minggu 3–5, M3 minggu 6–7, M4 minggu 8. Module path ditetapkan `github.com/lathief/AlurJet`.

### Fondasi dan kontrak rancangan

- Memperjelas antrean MySQL, transaksi claim dan attempt, pemeriksaan lease setelah lock dengan waktu MySQL, recovery pada attempt terakhir, dan klasifikasi retry HTTP termasuk `408`.
- Membatasi flow pada satu HTTP POST terakhir. Menetapkan deduplikasi webhook berdasarkan ID dan byte body, snapshot terkompilasi, serta replay dengan key request tersendiri dan outbound key yang diwarisi.
- Merinci kebijakan origin dan IP tujuan, penolakan redirect, pengecualian mock lokal, dan skenario demo retry serta replay.

### Dokumentasi dan verifikasi blueprint

- Menata dokumentasi menjadi panduan instalasi dan quickstart, panduan integrasi, referensi, operasional, contoh, serta FAQ.
- Menyelaraskan path YAML dan JSON di `docs/examples/`, tautan, dan contoh cURL.
- Menetapkan `available_at` dari MySQL untuk webhook baru dan replay, serta kontrak agar angka JSON berpresisi tinggi tetap utuh hingga outbound POST.

- Menyelaraskan M2 dengan finalisasi retry yang atomik dan recovery lease minimum. Verifikasi retry berulang, recovery serentak, serta replay ada di M3.
- Mengikat token webhook ke slug dan membatasi byte keluaran setiap transform sebelum alokasi serta pengiriman.
- Menambahkan kuota execution per flow, ambang global, dan batas header respons HTTP target.
- Memperjelas bahwa tes presisi membandingkan token angka body POST tanpa konversi `float64`.

- Menetapkan penolakan nama field JSON berulang sebelum deduplikasi dan penyimpanan.
- Menentukan fixture HTTP integrasi khusus untuk respons `400`, dengan mode mock Compose tetap untuk demo `200` dan `500`.

- Menetapkan bootstrap Compose dengan satu sumber konfigurasi MySQL dan startup API/worker setelah migrasi sukses.
- Menambahkan batas byte JSON outbound dan kegagalan permanen sebelum koneksi untuk hasil yang melebihinya.

### Penyiapan kerja agent

- Menambahkan [kartu tugas awal](docs/ai/implementation-plan.md), [matriks verifikasi](docs/ai/verification.md), [catatan keputusan](docs/ai/decisions.md), dan [format handoff](docs/ai/worklog.md).
- Menambahkan [backlog setelah MVP](docs/post-mvp.md) dengan prasyarat, pemicu, dan kriteria selesai per kandidat.

Lihat [desain sistem](SYSTEM_DESIGN.md) untuk kontrak lengkap dan [roadmap](ROADMAP.md) untuk status tahapan implementasi.
