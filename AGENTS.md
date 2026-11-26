# AlurJet — Panduan agent implementasi

AlurJet v0.1.0 adalah backend integrasi dengan Go, Fiber, dan antrean MySQL. Folder ini berisi blueprint. Baca [desain sistem](SYSTEM_DESIGN.md), [roadmap](ROADMAP.md), dan [indeks dokumentasi](docs/README.md) sebelum memilih satu milestone atau issue.

## Kontrak yang wajib dijaga

- Flow linear dengan tepat satu HTTP POST pada step terakhir. Simpan snapshot terkompilasi saat ingest dan gunakan untuk retry serta replay. Pertahankan presisi angka JSON sampai request keluar. Isi `available_at` dari MySQL untuk setiap execution baru agar bisa diklaim setelah commit.
- Kembalikan `202` setelah commit. Terapkan kapasitas MySQL untuk webhook dan replay baru setelah dedup. Dedup webhook berdasarkan flow dan event ID. ID sama dengan body berbeda mengembalikan `409`. Replay punya unique key request sendiri dan mempertahankan outbound key asal.
- Claim, penambahan counter, dan pembuatan attempt terjadi dalam satu transaksi. Tidak ada I/O jaringan di dalam transaksi. Finalisasi mencocokkan status, token, dan expiry memakai waktu MySQL setelah row lock. Recovery minimum wajib pada M2 dan menutup attempt expired sekali, lalu menjadwalkan ulang atau mengakhiri execution saat jatah habis.
- Ikat token webhook ke slug lewat konfigurasi API. Periksa anggaran byte sebelum menambahkan nilai pada tiap transform dan verifikasi body final sebelum koneksi outbound. Hasil yang melebihi batas berakhir `dead_letter`. Batasi header respons target dan tutup body tanpa membaca isinya pada MVP. Pemeriksaan outbound mengikuti §8 blueprint. Gunakan target mock untuk tes. Log hanya metadata yang dibutuhkan dan redaksi token, payload, serta detail sensitif.
- Saat bootstrap Compose, API dan worker menunggu migrasi satu kali setelah MySQL sehat. Pertahankan batas controller, usecase, engine, repository, connector, dan worker sebagaimana dirancang. Gunakan MySQL untuk antrean dan fokus pada scope setiap milestone.

## Paket kerja agent

Mulai dari [kartu tugas implementasi](docs/ai/implementation-plan.md) dan [matriks verifikasi](docs/ai/verification.md). Catat hasil di [worklog](docs/ai/worklog.md) dan tautkan perubahan desain ke [catatan keputusan](docs/ai/decisions.md).

## Penyelesaian pekerjaan

1. Baca kontrak dan kode yang benar-benar ada. Jika dokumen bertentangan, selaraskan desain, referensi API/DSL, dan tes dalam perubahan yang sama.
2. Tambahkan unit test atau tes integrasi untuk perilaku yang dikerjakan sesuai roadmap. Untuk lease, gunakan fixture waktu MySQL dan sinkronisasi transaksi. Untuk HTTP, gunakan mock dan resolver terkontrol.
3. Jalankan `gofmt`, `go test ./...`, build, dan tes integrasi MySQL yang relevan setelah runtime tersedia. Jika kontrak HTTP berubah, selaraskan Swagger dan panduan.
4. Laporkan verifikasi yang benar-benar dijalankan, hasilnya, dan kasus yang masih belum teruji. Gunakan label rancangan sampai Quickstart berjalan dari lingkungan bersih.
