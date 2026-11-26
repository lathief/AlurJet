# AlurJet — Panduan agent implementasi

AlurJet v0.1.0 adalah backend integrasi dengan Go, Fiber, dan antrean MySQL. Status implementasi tiap milestone ada di [roadmap](ROADMAP.md). Baca [desain sistem](SYSTEM_DESIGN.md), [roadmap](ROADMAP.md), dan [indeks dokumentasi](docs/README.md) sebelum memilih satu milestone atau issue.

## Kontrak yang wajib dijaga

- Flow linear dengan tepat satu HTTP POST pada step terakhir. Simpan snapshot terkompilasi saat ingest dan gunakan untuk retry serta replay. Pertahankan presisi angka JSON sampai request keluar. Isi `available_at` dari MySQL untuk setiap execution baru agar bisa diklaim setelah commit.
- Kembalikan `202` setelah commit. Terapkan kuota per flow (`429`) untuk webhook dan replay baru setelah dedup. Dedup webhook berdasarkan flow dan event ID. ID sama dengan body berbeda mengembalikan `409`. Replay punya unique key request sendiri dan mempertahankan outbound key asal.
- Claim, penambahan counter, dan pembuatan attempt terjadi dalam satu transaksi. Tidak ada I/O jaringan di dalam transaksi. Finalisasi mencocokkan status, token, dan expiry memakai waktu MySQL setelah row lock. Recovery minimum wajib pada M2 dan menutup attempt expired sekali, lalu menjadwalkan ulang atau mengakhiri execution saat jatah habis.
- Ikat token webhook ke slug lewat konfigurasi API. Batasi step `transform` sampai 64 field dan verifikasi body final terhadap `ALURJET_MAX_OUTBOUND_BYTES` sebelum DNS atau koneksi outbound. Hasil yang melebihi batas berakhir `dead_letter`. Batasi header respons target dan tutup body tanpa membaca isinya pada MVP. Pemeriksaan outbound mengikuti §8 blueprint. Gunakan target mock untuk tes.
- Header target (`headers` pada step `http`) disimpan sebagai template. Resolusi nilai terjadi saat dispatch dari environment, bukan saat import. Jangan menyimpan nilai header di DB, snapshot, response API, atau log ([ADR-003](docs/ai/decisions.md)).
- Log hanya metadata yang dibutuhkan (`event`, `execution_id`, `flow_slug`, `attempt`, `duration_ms`, `error_code`). Redaksi token, nilai header, payload, DSN, dan detail sensitif lain.
- Saat bootstrap Compose, API dan worker menunggu migrasi satu kali setelah MySQL sehat. Pertahankan batas controller, usecase, engine, repository, connector, dan worker sebagaimana dirancang. Gunakan MySQL untuk antrean dan fokus pada scope setiap milestone.
- Fitur yang ditunda tidak dikerjakan di v0.1.0: ambang global `ALURJET_ACTIVE_HIGH_WATERMARK` dan `503 SYSTEM_BACKPRESSURE`, anggaran byte per step transform, retensi data, dan semua item di [backlog pasca-MVP](docs/post-mvp.md) ([ADR-002](docs/ai/decisions.md)).

## Toolchain

Go 1.23+, Fiber v2, MySQL 8.4 LTS, golang-migrate v4, `gopkg.in/yaml.v3`, `database/sql` dengan `go-sql-driver/mysql`, `log/slog`, dan golangci-lint ([ADR-005](docs/ai/decisions.md)). Jangan menambah dependency tanpa alasan tertulis di [catatan keputusan](docs/ai/decisions.md). Perintah harian ada di [panduan pengembangan](docs/getting-started/development.md).

## Konvensi koding

- Identifier, komentar kode, error code, dan log memakai bahasa Inggris. Dokumentasi memakai bahasa Indonesia.
- `context.Context` menjadi argumen pertama fungsi yang melakukan I/O. Bungkus error dengan `%w` dan bandingkan dengan `errors.Is` atau `errors.As`. Jangan `panic` di jalur request atau worker.
- Fiber hanya diimpor di `internal/controllers`. Usecase tidak mengimpor Fiber atau driver MySQL. Interface repository berada di `internal/domain`.
- Tes memakai gaya table-driven. Tes unit tidak melakukan I/O jaringan. Tes integrasi memakai build tag `integration` dan Compose terpisah.
- Nilai waktu keputusan (claim, finalisasi, recovery, `available_at`) selalu berasal dari MySQL, bukan jam aplikasi.
- Satu perubahan, satu tujuan. Pesan commit memakai awalan `docs:`, `feat:`, `fix:`, `test:`, atau `chore:`.

## Prioritas dokumen

Jika dokumen saling bertentangan, urutan acuannya adalah [desain sistem](SYSTEM_DESIGN.md), lalu [catatan keputusan](docs/ai/decisions.md), lalu referensi ([API](docs/reference/api.md), [error](docs/reference/errors.md), [database](docs/reference/database.md), [konfigurasi](docs/reference/configuration.md)), lalu panduan. Selaraskan semuanya dalam perubahan yang sama dan catat keputusannya sebagai ADR bila memengaruhi kontrak.

## Paket kerja agent

Mulai dari [kartu tugas implementasi](docs/ai/implementation-plan.md) dan [matriks verifikasi](docs/ai/verification.md). Catat hasil di [worklog](docs/ai/worklog.md) dan tautkan perubahan desain ke [catatan keputusan](docs/ai/decisions.md).

## Penyelesaian pekerjaan

1. Baca kontrak dan kode yang benar-benar ada. Jika dokumen bertentangan, selaraskan desain, referensi API/DSL, dan tes dalam perubahan yang sama.
2. Tambahkan unit test atau tes integrasi untuk perilaku yang dikerjakan sesuai roadmap. Untuk lease, gunakan fixture waktu MySQL dan sinkronisasi transaksi. Untuk HTTP, gunakan mock dan resolver terkontrol.
3. Jalankan `gofmt`, `golangci-lint`, `go test ./...`, build, dan tes integrasi MySQL yang relevan setelah runtime tersedia. Jika kontrak HTTP berubah, selaraskan Swagger dan panduan.
4. Laporkan verifikasi yang benar-benar dijalankan, hasilnya, dan kasus yang masih belum teruji. Gunakan label rancangan sampai Quickstart berjalan dari lingkungan bersih.
