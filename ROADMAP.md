# AlurJet — Roadmap (Mid-Level Backend Scope)

> **Target:** v0.1.0 sebagai proyek open-source/portfolio yang dapat dijalankan dan diuji. Estimasi 6–8 minggu untuk solo developer part-time; fleksibel menurut waktu yang tersedia.
> **Status:** implementation plan, bukan checklist fitur yang sudah selesai.

## Fokus v0.1.0

**Satu demo yang benar-benar berjalan**: webhook -> validate -> transform -> outbound HTTP -> retry jika gagal -> execution history. Tidak mengejar visual UI, microservices, atau puluhan connector.

| Fase | Estimasi | Hasil yang dapat didemokan |
|---|---|---|
| M0 — Foundation | Minggu 1 | Go/Fiber, MySQL, migration, Docker Compose, struktur layers |
| M1 — Flow Management | Minggu 2 | YAML parser + API import/list/detail/update flow |
| M2 — Webhook & Worker | Minggu 3–5 | 202 setelah persist, background worker, HTTP connector, recovery minimum ([ADR-006](docs/ai/decisions.md)) |
| M3 — Reliability | Minggu 6–7 | Tes lanjutan retry, recovery, dan race lease; status/history; DLQ/replay |
| M4 — Docs, Tests & Release | Minggu 8 | Docs, CI, Swagger, v0.1.0 |

Estimasi ini belum divalidasi. Tinjau ulang setelah ALJ-02 selesai dengan kecepatan kerja yang nyata.

## Peta rilis sampai v1.0

> **Rencana, bukan status selesai.** Peta ini berasal dari [review multi-peran](docs/ai/review-multi-peran.md), yang juga memuat rencana 25 task dan alasan tiap keputusan. Milestone M0 sampai M4 di bawah adalah isi rilis v0.1.0. Perubahan desain dan skema yang diusulkan belum diterapkan.

| Rilis | Tema | Kriteria selesai |
|---|---|---|
| v0.1.0 | Alur andal end-to-end | Webhook sampai mock, retry, dead-letter, replay, recovery crash, CI hijau, `make demo`, benchmark awal |
| v0.2.0 | Dipakai dengan penyedia nyata | Flow GitHub atau Stripe berjalan tanpa mengubah pengirim, token dikelola lewat API |
| v0.3.0 | Dapat dioperasikan | Dashboard, alert, SLO, kebijakan migrasi, rilis otomatis dengan image bertanda tangan |
| v1.0.0 | Stabil dan terbukti | Kontrak `/api/v1` dibekukan, uji invarian lulus, model ancaman dan review keamanan, dokumen kunci dua bahasa |

Estimasi total sekitar 16 minggu solo part-time dan belum divalidasi: Fase A 2 hari, v0.1.0 sekitar 6 minggu, v0.2 sekitar 3 minggu, v0.3 sekitar 3 minggu, v1.0 sekitar 2 minggu. Tinjau ulang di akhir Task 4, 11, dan 16. Jika waktu terbatas, v0.1.0 dan v0.2.0 sudah cukup sebagai portofolio.

## M0 — Foundation

- [ ] `cmd/alurjet`, `internal/controllers`, `internal/usecases`, `internal/repositories/mysql`, `internal/domain`.
- [ ] Setup Fiber dan Compose dengan sumber konfigurasi MySQL yang sama untuk DB, migrator, API, dan worker. Jalankan service `migrate` satu kali setelah MySQL sehat. API/worker menunggu migrasi sukses.
- [ ] Siapkan mock receiver `POST /orders` di jaringan Compose dengan mode `success`, `fail_once` (500 pertama per outbound key, lalu 200), dan `always_fail`. Mode diatur lewat `ALURJET_MOCK_MODE`. Simpan counter mode `fail_once` selama proses mock hidup.
- [ ] `GET /healthz`, JSON error envelope, basic request logging via `slog`.
- [ ] Manual dependency injection dan 1 contoh unit test usecase.
- [ ] Siapkan `README.md`, `.env.example`, dan dokumentasi lokal.
- [ ] Siapkan `Makefile`, `.golangci.yml`, dan `docker-compose.test.yml` sesuai [panduan pengembangan](docs/getting-started/development.md). Validasi konfigurasi saat startup (nilai positif, `0 < attempt_timeout < lease`, secret minimal 32 karakter).
- [ ] `/healthz` memeriksa DB dan graceful shutdown. Worker tidak punya healthcheck Compose pada MVP ([ADR-006](docs/ai/decisions.md)).

**Done when:** startup dari volume kosong berhasil, startup ulang membuat migrasi no-op, migrasi yang gagal menahan API/worker dan terlihat pada log, serta `go test ./...` lulus. Lihat [kartu tugas awal](docs/ai/implementation-plan.md) untuk langkah implementasi.

## M1 — Flow Management

- [ ] Parser YAML strict (field tak dikenal ditolak, anchor/alias ditolak, batas `ALURJET_MAX_FLOW_YAML_BYTES`) dan validasi `apiVersion`, metadata, slug, trigger, ID step unik, serta tepat satu `http` terakhir. Resolve URL, validasi template `headers`, dan simpan definisi terkompilasi saat impor/update.
- [ ] Engine step `validate` dan `transform`, beserta unit test.
- [ ] Tabel `flows`, create/list/detail/update dari endpoint API.
- [ ] Admin API key dari environment untuk management endpoints.
- [ ] Dokumentasi `docs/guides/flow-configuration.md` dan kontrak Swagger sesuai kode.

**Done when:** user dapat mengimpor fixture YAML (termasuk yang memakai `headers`), membuat daftar flow, mengambil detail, dan memperbarui flow lewat API. Flow tidak valid menghasilkan `422` dan tidak disimpan. Unit test engine `validate` dan `transform` lulus.

## M2 — Webhook & Worker

- [ ] Tabel `executions` dan `execution_attempts`, indeks polling, serta unique key untuk dedup dan nomor attempt.
- [ ] `POST /api/v1/webhooks/{slug}` dengan token terikat slug dan `X-Event-ID`.
- [ ] Terapkan kuota aktif per flow sebelum insert execution baru (`429`; ambang global ditunda, lihat [ADR-002](docs/ai/decisions.md)). Uji target offline dan dedup saat penuh. Simpan execution dengan `available_at` dari MySQL dan flow snapshot terkompilasi sebelum membalas `202`. Uji bahwa execution baru langsung siap diklaim, serta uji ingest serentak dan konflik ID dengan body berbeda.
- [ ] Claim dengan `FOR UPDATE SKIP LOCKED`, tambah attempt dalam transaksi, lalu finalisasi dengan status, token, dan expiry setelah lock. Uji rollback claim dan penolakan hasil terlambat.
- [ ] Linear runner untuk `validate -> transform -> http` yang menjaga presisi angka JSON, batas 64 field transform dan batas body final, anggaran attempt, origin allowlist, validasi IP saat dial, penolakan redirect, dan batas header respons target. Body respons ditutup tanpa dibaca. Uji bahwa hasil melebihi batas berakhir `dead_letter` tanpa koneksi keluar. Pemeriksaan IP dilakukan pada setiap alamat yang benar-benar dihubungi (`net.Dialer.Control`), tanpa penolakan DNS campuran terpisah.
- [ ] Header target: resolusi template `headers` dari environment saat dispatch. Uji header terkirim ke mock, `HEADER_RESOLVE_FAILED` saat variabel hilang, dan nilai tidak muncul di log, DB, atau response.
- [ ] Implementasikan klasifikasi dan finalisasi sukses, retryable, serta permanen di M2. Simpan status retry dan `available_at` bersama update attempt. Uji satu `500` dari mock Compose `fail_once` yang bisa dijadwalkan ulang, serta `400` dari target yang dibuat khusus oleh suite integrasi yang menjadi `dead_letter`.
- [ ] Jalankan recovery minimum saat startup worker dan berkala. Uji crash setelah mock menerima POST. Attempt pengganti mempertahankan outbound key. Jika jatah habis, tutup sebagai `dead_letter`.
- [ ] Pisahkan proses dengan mode `alurjet api` dan `alurjet worker`.
- [ ] Tambahkan contoh request/response M2 ke Getting Started (import flow, webhook, dan hasil pada mock lewat `GET /_received`). Langkah list/detail execution dan replay baru tersedia di M3.

**Done when:** event masuk, mock menerima payload hasil transformasi, dan finalisasi menyimpan hasil yang tepat untuk sukses, gagal sementara, atau gagal permanen. Worker yang restart memulihkan attempt dengan lease kedaluwarsa.

## M3 — Reliability & Operations

- [ ] Verifikasi retry berulang untuk timeout, HTTP 408/429/5xx, dan kegagalan transport sementara. Uji batas attempts dan backoff, serta `dead_letter` ketika retry habis.
- [ ] Perluas tes recovery M2 untuk worker recovery serentak, lock wait melewati expiry, crash pada attempt terakhir, rollback claim, dan batas expiry tepat (`lease_until == db_now`). Tes M2 cukup membuktikan satu claim per execution, penolakan finalisasi setelah expiry, dan satu skenario crash lalu recovery.
- [ ] Endpoint list/detail dan replay dari dead-letter. Uji dedup request replay serentak, readiness pada polling pertama setelah commit, serta snapshot dan outbound key yang tetap sama. Uji bahwa replay baru memakai kebijakan kapasitas flow dan pengulangan replay idempoten tetap menemukan hasil lama saat antrean penuh.
- [ ] Log `execution_id`, `attempt`, dan `error_code`. Ukur umur execution siap proses, query plan claim, dan lock wait ketika ingest dan worker berjalan serentak sebelum menyesuaikan indeks, polling, atau batch recovery.
- [ ] Integration test: 500 → 200, selalu 500, crash pada attempt terakhir, lock wait lewat expiry, dan recovery serentak.
- [ ] Verifikasi perintah mock receiver, status, dan riwayat attempt dalam Quickstart.

**Done when:** failure bisa dilacak, retry bekerja, dan execution gagal bisa di-replay.

## M4 — Docs, Tests, and Release

- [ ] Pastikan semua contoh di `docs/getting-started/quickstart.md` bisa dijalankan **persis** seperti tertulis.
- [ ] Selaraskan `docs/reference/api.md` dan `docs/reference/errors.md` dengan Swagger.
- [ ] Verifikasi `docs/guides/tracking-executions.md` + `docs/operations/troubleshooting.md` dengan skenario nyata.
- [ ] Tambahkan unit tests/controller tests, integration MySQL tests, E2E happy path.
- [ ] Setup GitHub Actions minimal: format, tests, build.
- [ ] Tambahkan `CONTRIBUTING.md`, `SECURITY.md`, Apache-2.0 `LICENSE`, issue template sederhana.
- [ ] Tulis catatan backup dan restore MySQL serta peringatan pertumbuhan data di `docs/operations/deployment.md`.
- [ ] Demo video/GIF opsional; tag **v0.1.0** setelah kriteria terpenuhi.
- [ ] Hapus label “blueprint/belum siap” pada petunjuk yang benar-benar sudah lulus uji.

**Done when:** orang baru bisa clone, configure, run, kirim contoh webhook, dan mengetahui statusnya tanpa bertanya ke pembuat.

## Next backlog (hanya jika MVP selesai)

Item v0.2.0 sampai v1.0 ada di [peta rilis di atas](#peta-rilis-sampai-v10).

Lihat [backlog setelah MVP](docs/post-mvp.md) untuk kandidat yang diurutkan menurut kebutuhan operasional, hasil ukur, dan kriteria selesai.

**Jangan di v0.1.0:** RBAC, multi-tenancy, Kubernetes, visual workflow editor, CEL/scripting, distributed tracing, secret manager buatan sendiri, banyak service.

## Definition of Done untuk setiap issue

- [ ] Perubahan berada di layer yang tepat (controller/usecase/repository).
- [ ] Ada test untuk success dan satu failure case penting.
- [ ] Error dan log cukup membantu debugging, tanpa data rahasia.
- [ ] Endpoint baru terdokumentasi di Swagger dan `docs/reference/api.md`.
- [ ] Perubahan behaviour pengguna ditulis di docs dan contoh bila perlu.
- [ ] `gofmt`, `go test ./...`, dan build berhasil. Perubahan transaksi worker, ingest, atau replay memerlukan tes integrasi MySQL pada milestone yang memperkenalkannya.
