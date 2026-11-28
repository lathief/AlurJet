# Review rancangan AlurJet sebagai portofolio backend middle/senior

> **Status:** hasil review dan keputusan, tanggal 2026-11-26. Dokumen ini mencatat apa yang harus diubah. **Perubahan itu belum diterapkan** pada dokumen lain maupun kode. Pelaksanaannya mengikuti [rencana di bagian akhir](#rencana-pelaksanaan). Catatan persiapan wawancara ada di [interview-notes.md](interview-notes.md).

## Tujuan dan cakupan

AlurJet dibuat sebagai portofolio untuk melamar posisi backend middle atau senior di perusahaan lokal. Review ini menilai rancangan (`SYSTEM_DESIGN.md`, `ROADMAP.md`, `AGENTS.md`, `docs/`) dari sudut pandang penilai teknis: apa yang meyakinkan, apa yang akan dipertanyakan, dan apa yang belum ada.

Saat review, repository berisi 28 file Markdown dan belum ada kode. Satu-satunya hal yang sudah dibuktikan adalah DDL v1, yang diterapkan pada MySQL 8.4.13.

## Kesimpulan

Rancangan cukup matang untuk portofolio middle dan sebagian sudah setara senior. Risiko terbesarnya bukan teknis. Nilai portofolio baru muncul kalau proyek benar-benar jadi, bisa dijalankan dengan satu perintah, dan punya bukti (tes, CI, angka). Dokumen yang tebal tanpa kode tidak menjadi bukti kemampuan.

## Kekuatan

- Masalah yang dipilih sulit dan nyata: lease dengan fencing token, dedup berbasis hash byte body, presisi angka JSON, dan idempotency outbound.
- Pertahanan SSRF dilakukan saat koneksi dibuat (`net.Dialer.Control`), bukan hanya validasi URL.
- Keputusan dicatat sebagai ADR dan trade-off ditulis jujur (desain §10, ADR-002).
- Strategi tes konkurensi memakai barrier dan row lock, bukan `sleep`.

## Celah

### Prioritas tinggi

| # | Celah | Perbaikan |
|---|---|---|
| 1 | **Bukti ada di kode, bukan dokumen.** Alur end-to-end pertama baru muncul di kartu ketujuh dan CI baru dipasang di M4. | CI dipasang di kartu pertama. Alur tipis end-to-end dibuat sebelum pengerasan jaringan dan recovery. Riwayat commit kecil dan bermakna. |
| 2 | **Tidak ada observabilitas.** Desain menyatakan "tidak perlu OpenTelemetry" dan menunda metrik. | Endpoint `/metrics` Prometheus: kedalaman antrean, hasil attempt, durasi attempt, umur execution tertua yang siap diproses. |
| 3 | **Tidak ada angka.** Klaim "antrean MySQL cukup" belum punya bukti. | Skrip beban k6 dan tabel hasil: throughput ingest, p95 waktu sampai claim, perilaku 1, 2, dan 4 worker. |
| 4 | **Narasi untuk pembaca yang sibuk tidak ada.** Hal terbaik proyek terkubur di desain yang panjang. | README pendek (masalah, solusi, bukti, demo) dan `docs/engineering-notes.md` untuk lima masalah sulit. |
| 5 | **Penempatan pasar belum jelas.** Pewawancara akan bertanya kenapa tidak memakai Hookdeck, Svix, Benthos, atau n8n. | Posisikan sebagai relay webhook andal, self-hosted. Tambahkan bagian "Kapan tidak memakai AlurJet". |

### Prioritas menengah: desain data dan perilaku

| # | Celah | Perbaikan |
|---|---|---|
| 6 | **PK UUID v4 `CHAR(36)` pada tabel antrean (ADR-004).** Insert acak memecah halaman clustered index dan setiap indeks sekunder membawa 36 byte PK. | PK `BIGINT` internal, `public_id` UUID untuk API (ADR-007). |
| 7 | **Setiap execution menyalin `flow_snapshot` ke tabel panas.** | Tabel `flow_versions` yang immutable, execution hanya menyimpan `flow_version_id`. |
| 8 | **Lock row flow pada setiap ingest** menyerialkan penerimaan per flow. | Pertahankan, ukur lewat skrip beban, dan tulis batasnya (ADR-010). |
| 9 | **Backoff tanpa jitter** memicu retry serentak setelah target pulih. | Jitter ±20% dengan sumber acak yang bisa disuntik agar tes deterministik. |
| 10 | **Fairness antar flow.** Pool worker bersama, jadi satu flow dengan target lambat dapat menyita slot. | Dicatat sebagai batasan yang diketahui (ADR-010). |
| 11 | **Tidak ada retensi, paginasi memakai offset, urutan tidak disebut.** | Job retensi sederhana, paginasi keyset, dan pernyataan bahwa urutan eksekusi tidak dijamin. |
| 12 | **HMAC webhook ditunda** (ADR-002). | Tetap pasca-MVP. Tidak dikerjakan di v0.1.0. |

### Prioritas rendah

- Fuzz test Go untuk detektor nama field JSON berulang.
- Dockerfile non-root dengan image minimal, dependabot, dan target `make demo` untuk skenario kegagalan.
- Swaggo rawan menyimpang dari kontrak. Pertimbangkan OpenAPI sebagai sumber kebenaran bila waktu ada.

## Risiko wawancara

Rancangan ini sangat rinci. Di wawancara Anda harus bisa menjelaskan dan membela setiap keputusan, misalnya kenapa guard lease memakai waktu MySQL setelah lock, kenapa `READ COMMITTED`, dan kenapa dedup membandingkan hash. Dokumen yang tebal justru menaikkan ekspektasi penilai. Pertanyaan yang mungkin muncul dan kerangka jawabannya ada di [interview-notes.md](interview-notes.md). Tulis ulang jawabannya dengan kata-kata Anda sendiri.

Berkas khas agen (`AGENTS.md`, `docs/ai/worklog.md`, `docs/ai/implementation-plan.md`) terlihat di antarmuka repository. Keputusan saat ini adalah membiarkannya di tempatnya. Pastikan Anda siap menjawab pertanyaan jujur tentang peran alat bantu AI dalam proyek ini (lihat interview-notes).

## Keputusan yang diambil

| # | Keputusan |
|---|---|
| R-1 | Target pelamar perusahaan lokal. README dua bahasa: `README.md` (Inggris, sumber) dan `README.id.md` (Indonesia). |
| R-2 | Dokumen kunci berbahasa Inggris: `SYSTEM_DESIGN.md`, catatan keputusan (ADR), dan `docs/engineering-notes.md` (baru). Dokumen lain tetap Indonesia dan tidak diterjemahkan. |
| R-3 | Struktur dokumen tidak dikonsolidasi. 28 file tetap di tempatnya dan berkas agen tetap di `docs/ai/`. |
| R-4 | Skema v2: PK `BIGINT`, `public_id` UUID, dan tabel `flow_versions` (menggantikan ADR-004). |
| R-5 | Masuk MVP: `/metrics`, skrip beban dengan tabel hasil, jitter, dan retensi sederhana. HMAC webhook tetap pasca-MVP. |
| R-6 | Item murah ikut: fuzz test, Dockerfile non-root, `make demo`, dan dependabot. |
| R-7 | Keputusan lama tetap: paket hemat (ADR-002), `headers` pada step `http` (ADR-003), toolchain (ADR-005), penyederhanaan M2 (ADR-006), module `github.com/lathief/AlurJet`. |

ADR baru yang akan ditulis: ADR-007 (skema v2), ADR-008 (operabilitas: metrics, port worker, jitter, retensi, keyset), ADR-009 (kebijakan bahasa dokumen), dan ADR-010 (batasan yang diketahui). ADR-008 sebagian menggantikan ADR-006 pada healthcheck worker, karena worker kini membuka port kecil untuk `/metrics` dan `/healthz`.

## Rancangan perubahan

### Skema v2

- `flows`: `id BIGINT`, `slug`, `name`, `current_version_id`.
- `flow_versions`: tidak pernah diubah. Menyimpan `version_no`, `definition_yaml`, `compiled_definition_json`, dan `definition_hash`. Update dengan hash sama tidak membuat versi baru.
- `executions`: PK `id BIGINT UNSIGNED` auto-increment, `public_id CHAR(36)` unik untuk API, `flow_version_id` menggantikan salinan snapshot, `replayed_from_id` BIGINT dengan `ON DELETE SET NULL`, dan `outbound_idempotency_key` yang diisi dari `public_id` saat ingest lalu disalin saat replay.
- `execution_attempts.execution_id` bertipe BIGINT dengan `ON DELETE CASCADE`.
- Ingest mengunci row flow, membaca `current_version_id`, dan menyimpannya di execution. Retry dan replay memakai versi yang sama persis.

### Operabilitas

- Port `ALURJET_METRICS_ADDR` (default `:9090`, terikat ke loopback di Compose) menyediakan `/metrics` dan `/healthz` pada API dan worker.
- Metrik tanpa label slug: `alurjet_executions{status}`, `alurjet_oldest_ready_age_seconds`, `alurjet_attempts_total{outcome}`, `alurjet_attempt_duration_seconds`, `alurjet_ingest_total{result}`, `alurjet_lease_recoveries_total`. Worker memperbarui gauge antrean setiap 10 detik.
- Subcommand `alurjet healthcheck` dipakai Compose, sehingga image tanpa shell tetap bisa dicek.
- Retensi: worker menghapus execution terminal yang lebih tua dari `ALURJET_RETENTION_DAYS` (default 14, `0` menonaktifkan) secara bertahap. Masa dedup sama dengan masa retensi.
- Paginasi memakai keyset (`limit` dan `cursor`).

### Batasan yang diketahui (ADR-010)

- Lock flow pada ingest menyerialkan penerimaan per flow.
- Pool worker bersama, jadi satu flow lambat dapat menyita slot.
- Urutan eksekusi tidak dijamin.
- Delivery at-least-once, bukan exactly-once.

### Kebijakan bahasa (ADR-009)

- Inggris: `README.md`, `SYSTEM_DESIGN.md`, `docs/ai/decisions.md`, `docs/engineering-notes.md`.
- Indonesia: `README.id.md` dan semua dokumen lain.
- Penomoran `§1–§10` dan nomor ADR dipertahankan karena dokumen Indonesia merujuknya.
- `make docs-check` memeriksa tautan, dan memeriksa bahwa kedua README memiliki jumlah heading dan blok kode yang sama.

## Rencana pelaksanaan

Estimasi total sekitar 9 minggu dan belum divalidasi. Tinjau ulang setelah Task 7.

### Fase A: dokumen (branch `docs/perbaikan-dokumen-awal`, sekitar 4–5 hari)

| Task | Isi | Bukti selesai |
|---|---|---|
| 1 | Skema v2 (ADR-007): tulis ulang DDL dan ERD, selaraskan desain, `AGENTS.md`, API, verifikasi | DDL berjalan pada `mysql:8.4` sekali pakai. Unique key, FK, `ON DELETE`, dan `CHECK` terbukti |
| 2 | Operabilitas dan batasan (ADR-008, ADR-010): variabel baru, paginasi keyset, daftar metrik | Variabel `.env.example` dan `configuration.md` cocok, tautan valid |
| 3 | Susun ulang roadmap dan kartu kerja: ALJ-01 sampai ALJ-08, lalu M3 dan M4 | Setiap kartu punya "selesai ketika" yang dapat diperiksa |
| 4 | Dokumen kunci berbahasa Inggris (ADR-009): terjemahkan `SYSTEM_DESIGN.md` dan ADR, tulis `engineering-notes.md` | 0 tautan rusak, penomoran `§` tidak berubah, Anda meninjau terjemahan |
| 5 | README dua bahasa dan `make docs-check` | Skrip lulus, dan gagal bila heading atau blok kode salah satu README diubah |

### Fase B: kode (satu branch per kartu)

| Task | Kartu | Isi | Bukti selesai |
|---|---|---|---|
| 6 | ALJ-01 | Fondasi Go, konfigurasi, `/healthz`, server metrics, CI, lint, dependabot | CI hijau, `/healthz` dan `/metrics` berhasil |
| 7 | ALJ-02 | Migrasi v2, Dockerfile non-root, Compose, mock receiver | Bootstrap dari volume kosong, startup ulang no-op, migrasi rusak menahan API dan worker |
| 8 | ALJ-03 | Parser DSL strict dan engine `validate`/`transform` | Kasus tidak valid ditolak, token angka dipertahankan |
| 9 | ALJ-04 | Manajemen flow dengan `flow_versions` | Import `201`, update membuat versi baru tanpa memengaruhi execution lama |
| 10 | ALJ-05 | Ingest webhook, dedup, kuota, fuzz test detektor field berulang | Request identik memberi ID sama, body beda memberi `409`, kuota penuh memberi `429` |
| 11 | ALJ-06 | **Alur tipis end-to-end:** worker, claim, konektor dasar, backoff dengan jitter | Webhook sampai mock berjalan penuh dan satu tes E2E di CI |
| 12 | ALJ-07 | Pengerasan outbound: dialer IP, resolusi `headers`, batas body final | Target IP privat ditolak, `Authorization` sampai ke mock |
| 13 | ALJ-08 | Recovery lease, metrik antrean, `make demo`, gate M2 | Worker dimatikan di tengah attempt lalu dipulihkan, retry dengan key sama |
| 14 | M3 | Endpoint execution (keyset) dan replay | Replay memakai versi flow lama setelah flow diubah |
| 15 | M3 | Tes reliabilitas yang ditunda (rollback claim, batas expiry, recovery serentak) dan job retensi | Tes dengan barrier dan row lock, retensi tidak menghapus `pending` atau `running` |
| 16 | M3 | Skrip beban k6 dan `docs/benchmarks.md` | Tabel hasil berisi throughput, p95 waktu sampai claim, dan efek jumlah worker |
| 17 | M4 | Rilis v0.1.0: Quickstart teruji, `LICENSE`, `CONTRIBUTING.md`, `SECURITY.md`, `docs-check` di CI | Quickstart dari lingkungan bersih dan CI hijau |

## Risiko terbuka

- Jadwal sekitar 9 minggu belum divalidasi.
- `SKIP LOCKED` di bawah konkurensi belum teruji. Baru terbukti di Task 11.
- Terjemahan Inggris perlu ditinjau oleh pemilik proyek. Kualitas bahasa memengaruhi kesan penilai.
- Dokumen lain belum diselaraskan dengan skema v2. Sampai Task 1 selesai, `database.md`, `SYSTEM_DESIGN.md`, dan `api.md` masih memakai skema v1.
