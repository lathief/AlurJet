# Paket kerja awal AlurJet

> Cakupan lengkap setiap milestone ada di [roadmap](../../ROADMAP.md). Cakupan MVP mengikuti paket hemat di [ADR-002](decisions.md).

Ambil satu kartu pada satu waktu. Cocokkan implementasi dengan [desain sistem](../../SYSTEM_DESIGN.md) dan [matriks verifikasi](verification.md). Catat hasil serta tugas berikutnya di [worklog](worklog.md) dan perbarui kolom status di bawah.

## Status kartu

| Kartu | Milestone | Judul | Status |
|---|---|---|---|
| ALJ-00 | Pra-M0 | Penutupan celah dokumen | Selesai. DDL terverifikasi pada MySQL 8.4.13 (lihat [worklog](worklog.md)) |
| ALJ-01 | M0 | Fondasi Go, konfigurasi, dan `/healthz` | Direncanakan |
| ALJ-02 | M0 | Migrasi, Compose, dan mock receiver | Direncanakan |
| ALJ-03 | M1 | Parser DSL dan engine `validate`/`transform` | Direncanakan |
| ALJ-04 | M1 | Manajemen flow (create, list, detail, update) | Direncanakan |
| ALJ-05 | M2 | Ingest webhook, dedup, dan kuota | Direncanakan |
| ALJ-06 | M2 | Konektor HTTP outbound dan kebijakan jaringan | Direncanakan |
| ALJ-07 | M2 | Worker: claim, finalisasi, dan retry | Direncanakan |
| ALJ-08 | M2 | Recovery lease dan gate M2 | Direncanakan |

Status yang valid: `Direncanakan`, `Dikerjakan`, `Selesai`, `Diblokir`. Ubah menjadi `Selesai` hanya setelah verifikasi kartu terkait tercatat di worklog.

## ALJ-00 — Penutupan celah dokumen

- **Cakupan:** skema database ([database.md](../reference/database.md)), katalog error dan kontrak API ([errors.md](../reference/errors.md), [api.md](../reference/api.md)), spesifikasi DSL termasuk `headers`, konfigurasi runtime, konvensi di [AGENTS.md](../../AGENTS.md), [panduan pengembangan](../getting-started/development.md), dan ADR-002 sampai ADR-005.
- **Selesai ketika:** `SYSTEM_DESIGN.md`, `ROADMAP.md`, `.env.example`, dan referensi tidak saling bertentangan. DDL berjalan pada MySQL 8.4 sekali pakai dan perbandingan `event_id` terbukti case-sensitive.

## ALJ-01 — Fondasi Go, konfigurasi, dan `/healthz`

- **Prasyarat:** ALJ-00, Go 1.23+.
- **Cakupan:** modul Go `github.com/lathief/AlurJet` (Go 1.23.1 terpasang di mesin pengembangan), `cmd/alurjet` dengan subcommand `api` dan `worker`, `internal/config` dengan validasi startup (nilai positif, `0 < attempt_timeout < lease`, secret minimal 32 karakter, token unik, admin key berbeda dari token webhook), `slog` JSON, Fiber v2 dengan error envelope, `GET /healthz` yang memeriksa DB, manual DI, dan graceful shutdown. Tambahkan `Makefile` dan `.golangci.yml`.
- **Selesai ketika:** `gofmt`, `golangci-lint`, dan `go test ./...` lulus. Konfigurasi tidak valid keluar nonnol tanpa mencetak nilai rahasia. Ada satu contoh unit test usecase.
- **Tugas terpisah:** migrasi, Compose, import flow, ingest, dan worker.

## ALJ-02 — Migrasi, Compose, dan mock receiver

- **Prasyarat:** ALJ-01.
- **Cakupan:** `migrations/` dari [database.md](../reference/database.md), `Dockerfile`, `docker-compose.yml` (MySQL dengan healthcheck, `migrate` sekali jalan, `api`, `worker`, `mock-api`), `docker-compose.test.yml`, dan mock `POST /orders` dengan mode `success`, `fail_once` (per outbound key), `always_fail`, serta `GET /_received` yang hanya dipublikasikan ke loopback.
- **Selesai ketika:** startup dari volume kosong berhasil, startup ulang membuat migrasi no-op, migrasi yang dirusak pada project terisolasi menahan API/worker dan terlihat di status/log tanpa mencetak password, dan tiga mode mock teruji.

## ALJ-03 — Parser DSL dan engine `validate`/`transform`

- **Prasyarat:** ALJ-01.
- **Cakupan:** parser YAML strict (`KnownFields(true)`, tolak anchor/alias, batas ukuran), validasi aturan [flow-configuration.md](../guides/flow-configuration.md), resolusi URL dan validasi template `headers`, serialisasi definisi terkompilasi, serta runner `validate` dan `transform` yang mempertahankan token angka.
- **Selesai ketika:** fixture [order-to-api.yaml](../examples/order-to-api.yaml) dan [order-to-api-auth.yaml](../examples/order-to-api-auth.yaml) terkompilasi. Kasus tidak valid (dua step HTTP, ID duplikat, placeholder tidak didukung, header terlarang, 65 field, field tak dikenal, alias) ditolak. `9007199254740993` dan `0.123456789012345678901` keluar identik dari transform.

## ALJ-04 — Manajemen flow

- **Prasyarat:** ALJ-02 dan ALJ-03.
- **Cakupan:** repository, usecase, dan controller untuk `POST/GET/PUT /api/v1/flows` dengan auth admin, sesuai [api.md](../reference/api.md).
- **Selesai ketika:** fixture lolos dengan `201`. Konfigurasi tidak valid atau dua step HTTP menghasilkan `422`. Slug duplikat menghasilkan `409`. Token keliru menghasilkan `401`. Definisi terkompilasi menyimpan URL teresolusi dan template header. `PUT` memperbarui flow tanpa mengubah execution yang ada. Gate M1 di roadmap terpenuhi.

## ALJ-05 — Ingest webhook, dedup, dan kuota

- **Prasyarat:** ALJ-04.
- **Cakupan:** middleware token terikat slug, validasi urutan sesuai [errors.md](../reference/errors.md#urutan-validasi-webhook), detektor nama field berulang, hash dari byte asli, transaksi `READ COMMITTED` dengan lock flow, dedup, kuota `ALURJET_MAX_ACTIVE_PER_FLOW`, dan insert `pending` dengan `available_at` dari MySQL.
- **Selesai ketika:** request identik mengembalikan ID yang sama. ID sama dengan byte body berbeda menghasilkan `409`. JSON dengan nama field berulang menghasilkan `400` tanpa insert. Kuota penuh menghasilkan `429` sementara dedup tetap `202`. Ingest serentak menghasilkan satu baris. Execution baru dapat diklaim pada polling pertama setelah commit.

## ALJ-06 — Konektor HTTP outbound

- **Prasyarat:** ALJ-03.
- **Cakupan:** validasi origin, dialer yang memeriksa IP dengan daftar CIDR eksplisit, penolakan redirect dan proxy, batas header respons, body ditutup tanpa dibaca, resolusi `headers` saat dispatch, `Idempotency-Key`, klasifikasi hasil, dan pemeriksaan body final sebelum DNS.
- **Selesai ketika:** tes dengan resolver dan dialer terkontrol lulus untuk origin ditolak, redirect, alamat terlarang dari DNS (termasuk jawaban campuran) tidak pernah dihubungi, IP koneksi sama dengan IP tervalidasi, pengecualian dev terbatas pada satu origin, header terkirim dan tidak tercatat di log, serta batas byte tepat dan lebih satu byte.

## ALJ-07 — Worker: claim, finalisasi, dan retry

- **Prasyarat:** ALJ-05 dan ALJ-06.
- **Cakupan:** polling dengan `ALURJET_WORKER_CONCURRENCY`, claim `SKIP LOCKED`, attempt dalam transaksi yang sama, `ProcessExecution`, finalisasi dengan guard status, token, dan expiry, klasifikasi sukses, retryable, dan permanen, serta backoff.
- **Selesai ketika:** setelah `202`, execution diklaim pada polling berikutnya. Dua worker tidak mengklaim execution yang sama, dan finalisasi setelah lease kedaluwarsa ditolak dengan `ErrLostLease`. Rollback claim dan batas expiry tepat (`lease_until == db_now`) diuji di M3 ([ADR-006](decisions.md)). `500` dari mock `fail_once` menghasilkan `retry_scheduled` dengan `available_at` masa depan lalu sukses dengan key yang sama. `400` dari fixture khusus tes menjadi `dead_letter`. Body di atas batas menjadi `dead_letter` tanpa koneksi.

## ALJ-08 — Recovery lease dan gate M2

- **Prasyarat:** ALJ-07.
- **Cakupan:** recovery saat startup dan berkala, mode proses `api` dan `worker`, Quickstart M2, dan pembaruan roadmap sesuai bukti.
- **Selesai ketika:** satu skenario crash setelah mock menerima POST dan sebelum finalisasi menghasilkan attempt lama ditutup sekali lalu retry dengan body dan key yang sama. Recovery pada attempt terakhir dan recovery serentak diuji di M3. `go test ./... -race` dan tes integrasi lulus. Quickstart bagian 1–3 berjalan dari volume kosong.
- **Tugas terpisah:** verifikasi retry berulang, history endpoint, dan replay di M3 (lihat bawah).

## Setelah kartu awal

Pecah M3 menjadi verifikasi retry berulang, uji concurrency recovery M2, serta endpoint execution (list, detail) dan replay. Tuntaskan dokumentasi dan demo yang dapat diulang di M4. Perubahan di luar v0.1.0 ada di [backlog setelah MVP](../post-mvp.md).
