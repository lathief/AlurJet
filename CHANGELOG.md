# Changelog AlurJet

## Unreleased — Perubahan blueprint

Entri ini mencatat perubahan dokumen dan kontrak rancangan yang tersedia di folder ini. Status rilis runtime menunggu implementasi dan verifikasi.

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
