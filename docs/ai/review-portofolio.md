# Review rancangan AlurJet sebagai portofolio backend middle/senior

> **Status:** hasil review dan keputusan, tanggal 2026-11-26. Dokumen ini mencatat apa yang harus diubah. **Perubahan itu belum diterapkan** pada dokumen desain maupun kode. Pelaksanaannya mengikuti [rencana 25 task](review-multi-peran.md#rencana-25-task). Catatan persiapan wawancara ada di [interview-notes.md](interview-notes.md).
>
> **Keputusan di dokumen ini disempurnakan oleh [review multi-peran](review-multi-peran.md). Jika bertentangan, review multi-peran yang berlaku.** Perubahan utama: tabel `flow_versions` dibatalkan, kuota per flow menjadi lunak, dan terjemahan dokumen kunci ditunda sampai v1.0.

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
| 7 | **Setiap execution menyalin `flow_snapshot` ke tabel panas.** | Diterima sebagai trade-off. Snapshot tetap disimpan di execution karena sederhana dan menjamin retry serta replay memakai definisi yang sama. Tabel `flow_versions` dibatalkan. Dampak ukuran baris diukur di benchmark. |
| 8 | **Lock row flow pada setiap ingest** menyerialkan penerimaan per flow. | Hapus lock. Kuota menjadi lunak, dedup dijamin unique key, dan kelebihan terbatas pada jumlah ingest serentak. Tulis batasnya (ADR-010). |
| 9 | **Backoff tanpa jitter** memicu retry serentak setelah target pulih. | Jitter ±20% dengan sumber acak yang bisa disuntik agar tes deterministik. |
| 10 | **Fairness antar flow.** Pool worker bersama, jadi satu flow dengan target lambat dapat menyita slot. | Dicatat sebagai batasan yang diketahui (ADR-010). |
| 11 | **Tidak ada retensi dan urutan tidak disebut.** | Job retensi sederhana dan pernyataan bahwa urutan eksekusi tidak dijamin. Paginasi tetap `page` dan `limit` di v0.1.0. Keyset hanya jika waktu ada. |
| 12 | **HMAC webhook ditunda** (ADR-002). | Dijadwalkan v0.2.0, bersama `trigger.eventId` yang dapat dikonfigurasi dan token di database. |

### Prioritas rendah

- Fuzz test Go untuk detektor nama field JSON berulang (masuk v0.1.0).
- Dockerfile non-root dengan image minimal dan target `make demo` untuk skenario kegagalan (masuk v0.1.0). Dependabot pindah ke v0.3.
- Swaggo rawan menyimpang dari kontrak. OpenAPI menjadi sumber kebenaran pada v1.0.

## Risiko wawancara

Rancangan ini sangat rinci. Di wawancara Anda harus bisa menjelaskan dan membela setiap keputusan, misalnya kenapa guard lease memakai waktu MySQL setelah lock, kenapa `READ COMMITTED`, dan kenapa dedup membandingkan hash. Dokumen yang tebal justru menaikkan ekspektasi penilai. Pertanyaan yang mungkin muncul dan kerangka jawabannya ada di [interview-notes.md](interview-notes.md). Tulis ulang jawabannya dengan kata-kata Anda sendiri.

Berkas khas agen (`AGENTS.md`, `docs/ai/worklog.md`, `docs/ai/implementation-plan.md`) terlihat di antarmuka repository. Keputusan saat ini adalah membiarkannya di tempatnya. Pastikan Anda siap menjawab pertanyaan jujur tentang peran alat bantu AI dalam proyek ini (lihat interview-notes).

## Keputusan yang diambil

| # | Keputusan |
|---|---|
| R-1 | Target pelamar perusahaan lokal. README dua bahasa: `README.md` (Inggris, sumber) dan `README.id.md` (Indonesia). |
| R-2 | Dokumen kunci (`SYSTEM_DESIGN.md`, ADR, `engineering-notes`, `threat-model`) diterjemahkan ke bahasa Inggris **setelah desain stabil, pada v1.0**. Dokumen lain tetap Indonesia dan tidak diterjemahkan. |
| R-3 | Struktur dokumen tidak dikonsolidasi. 28 file tetap di tempatnya dan berkas agen tetap di `docs/ai/`. |
| R-4 | Skema: PK `BIGINT` dengan `public_id` UUID untuk API (menggantikan ADR-004). Tabel `flow_versions` **dibatalkan**, snapshot tetap disimpan di execution. |
| R-5 | Masuk v0.1.0: `/metrics`, skrip beban dengan tabel hasil, jitter, dan retensi sederhana. HMAC webhook dijadwalkan v0.2.0. |
| R-6 | Item murah di v0.1.0: fuzz test, Dockerfile non-root, dan `make demo`. Dependabot pindah ke v0.3. |
| R-7 | Keputusan lama tetap: paket hemat (ADR-002), `headers` pada step `http` (ADR-003), toolchain (ADR-005), penyederhanaan M2 (ADR-006), module `github.com/lathief/AlurJet`. |
| R-8 | Kuota per flow lunak, tanpa lock row flow. |

ADR baru yang akan ditulis: ADR-007 (skema: PK `BIGINT` dan `public_id`), ADR-008 (operabilitas: metrics, port worker, jitter, retensi), ADR-009 (kebijakan bahasa dokumen), dan ADR-010 (batasan yang diketahui). ADR-008 sebagian menggantikan ADR-006 pada healthcheck worker, karena worker kini membuka port kecil untuk `/metrics` dan `/healthz`.

## Rancangan perubahan

### Skema

- `flows` tetap seperti sekarang.
- `executions`: PK `id BIGINT UNSIGNED` auto-increment, `public_id CHAR(36)` unik untuk API sebagai `execution_id`, `flow_snapshot` tetap disimpan di execution, `replayed_from_id` bertipe BIGINT dengan `ON DELETE SET NULL`, dan `outbound_idempotency_key` yang diisi dari `public_id` saat ingest lalu disalin saat replay.
- `execution_attempts.execution_id` bertipe BIGINT dengan `ON DELETE CASCADE`.
- Ingest membaca definisi flow dari data yang sudah commit tanpa mengunci row flow, lalu menyimpan snapshot di execution. Retry dan replay memakai snapshot itu.

### Operabilitas

- Port `ALURJET_METRICS_ADDR` (default `:9090`, terikat ke loopback di Compose) menyediakan `/metrics` dan `/healthz` pada API dan worker.
- Metrik tanpa label slug: `alurjet_executions{status}`, `alurjet_oldest_ready_age_seconds`, `alurjet_attempts_total{outcome}`, `alurjet_attempt_duration_seconds`, `alurjet_ingest_total{result}`, `alurjet_lease_recoveries_total`. Worker memperbarui gauge antrean setiap 10 detik.
- Subcommand `alurjet healthcheck` dipakai Compose, sehingga image tanpa shell tetap bisa dicek.
- Retensi: worker menghapus execution terminal yang lebih tua dari `ALURJET_RETENTION_DAYS` (default 14, `0` menonaktifkan) secara bertahap. Masa dedup sama dengan masa retensi.
- Paginasi memakai `page` dan `limit` pada v0.1.0.

### Batasan yang diketahui (calon ADR-010)

- Tidak ada lock row flow pada ingest, jadi kuota dapat terlampaui sebanyak jumlah ingest serentak.
- Pool worker bersama, jadi satu flow lambat dapat menyita slot.
- Urutan eksekusi tidak dijamin.
- Delivery at-least-once, bukan exactly-once.
- Replay dengan outbound key yang sama mungkin tidak berefek pada target yang menyimpan hasil per key.

### Kebijakan bahasa (calon ADR-009)

- Inggris: `README.md`. Dokumen kunci menyusul pada v1.0.
- Indonesia: `README.id.md` dan semua dokumen lain.
- Penomoran `§1–§10` dan nomor ADR dipertahankan karena dokumen Indonesia merujuknya.
- `make docs-check` memeriksa tautan, dan memeriksa bahwa kedua README memiliki jumlah heading dan blok kode yang sama.

## Rencana pelaksanaan

Rencana lengkap 25 task, peta rilis, dan estimasi ada di [review multi-peran](review-multi-peran.md#peta-rilis) dan [ROADMAP.md](../../ROADMAP.md#peta-rilis-sampai-v10). Ringkasnya:

| Rilis | Tema |
|---|---|
| v0.1.0 | Alur andal end-to-end |
| v0.2.0 | Dipakai dengan penyedia nyata |
| v0.3.0 | Dapat dioperasikan |
| v1.0.0 | Stabil dan terbukti |

Estimasi sekitar 16 minggu solo part-time dan belum divalidasi.

## Risiko terbuka

- Jadwal belum divalidasi.
- `SKIP LOCKED` di bawah konkurensi belum teruji. Baru terbukti di Task 8 dan 10.
- Terjemahan Inggris dokumen kunci pada v1.0 perlu ditinjau oleh pemilik proyek. Kualitas bahasa memengaruhi kesan penilai.
- Dokumen lain belum diselaraskan dengan skema baru. Sampai Task 1 selesai, `database.md`, `SYSTEM_DESIGN.md`, dan `api.md` masih memakai skema v1 dan kuota ketat.
