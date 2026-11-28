# Review multi-peran AlurJet

> **Status:** hasil review tanggal 2026-11-26. Perubahan yang diusulkan di dokumen ini **belum diterapkan** pada desain (`SYSTEM_DESIGN.md`), skema (`database.md`), kontrak API (`api.md`), `AGENTS.md`, `.env.example`, maupun kode. Penerapannya dimulai dari Task 1 pada [rencana 25 task](#rencana-25-task). Dokumen ini menyempurnakan [review portofolio](review-portofolio.md). Jika keduanya bertentangan, dokumen ini yang berlaku.

## Tujuan dan sudut pandang

AlurJet dibuat sebagai portofolio untuk melamar posisi backend middle atau senior di perusahaan lokal. Review ini membaca ulang rancangan dari lima sudut pandang: senior backend engineer, senior DevOps, principal engineer, product owner, dan pengguna produk (integrator yang memasang AlurJet).

Saat review, repository berisi dokumen dan belum ada kode. Satu-satunya hal yang terbukti dijalankan adalah DDL v1 pada MySQL 8.4.13. Semua temuan lain adalah analisis rancangan dan belum diuji.

## 1. Senior backend engineer

- **Query claim kemungkinan tidak memakai indeks dengan baik.** `status IN ('pending','retry_scheduled') ... ORDER BY available_at, id LIMIT 1` pada indeks `(status, available_at, id)` kemungkinan memecah scan menjadi dua rentang dan memicu filesort. Hal ini belum diukur dengan `EXPLAIN`. Ukur pada task alur tipis (Task 8). Opsi: `UNION ALL` per status, atau satu status antrean dengan `available_at` di masa depan untuk retry.
- **Replay dengan outbound key yang sama bisa tidak berefek.** Jika target menyimpan hasil (termasuk hasil gagal) per `Idempotency-Key`, request replay dapat menerima respons lama. Ini keputusan yang disengaja, jadi trade-off-nya harus ditulis. Pertimbangkan opsi replay dengan key baru.
- **Kuota ketat memaksa lock row flow pada setiap ingest.** Dedup sudah dijamin unique key. Kuota lunak menghilangkan serialisasi per flow, dan tujuannya (melindungi database) tetap tercapai.
- **Validasi bisnis di worker membuat payload salah tetap mendapat `202`** lalu berakhir `dead_letter` yang tidak bisa diperbaiki lewat replay. Validasi `required` dapat dijalankan sinkron saat ingest dan menghasilkan `422`. Ini dicatat sebagai opsi dan **belum diputuskan**.

## 2. Senior DevOps

- **`.gitattributes` belum ada.** Git memperingatkan konversi LF ke CRLF di mesin Windows. Tambahkan `* text=auto eol=lf`.
- **Benchmark dari Docker Desktop Windows tidak representatif.** Jalankan di Linux dan catat spesifikasi mesin.
- **CI belum memuat** `govulncheck`, pemindaian image (Trivy), dan publikasi image bertag ke GHCR.
- **Metrik tanpa dashboard dan alert.** Sertakan dashboard Grafana (JSON) dan aturan alert, misalnya `alurjet_oldest_ready_age_seconds > 60` selama 5 menit.
- **Kebijakan migrasi belum ada.** Perlu ditulis: status dirty golang-migrate, rollback, dan kompatibilitas versi lama dan baru saat berjalan bersamaan (expand lalu contract).
- **Belum ada SLO.** Usulan dua SLI: p95 waktu dari ingest sampai attempt pertama, dan rasio keberhasilan pengiriman.
- **Korelasi log.** Tambahkan `request_id` agar satu request dapat dilacak sampai `execution_id`.

## 3. Principal engineer

- **Repository masih nol kode.** Risiko terbesar adalah terus menyempurnakan dokumen. Terjemahan Inggris dokumen kunci dipindah ke setelah alur tipis berjalan dan desain beku (v1.0).
- **Satu klaim inti:** "pengiriman at-least-once yang andal di atas MySQL dengan lease fencing". DSL, `headers`, dan fitur lain mendukung klaim itu. Batasi permukaan DSL karena itu biaya non-inti terbesar.
- **Tulis invarian lalu uji dengan injeksi kegagalan.**
  - Tidak ada execution yang hilang.
  - Tidak ada attempt yang difinalisasi dua kali.
  - Paling banyak satu attempt `running` per execution.
  - `attempts` tidak melebihi batas.
  - Skenario: bunuh worker acak di bawah beban, hentikan MySQL sebentar, dan ubah target antara sukses dan gagal.
- **Tabel failure mode** di engineering-notes: MySQL mati, target mati, worker crash, jam bergeser, event duplikat, payload beracun, disk penuh. Untuk tiap kasus: dampak, deteksi, pemulihan.
- **Model ancaman ringkas** (STRIDE sederhana) untuk melengkapi pembahasan SSRF.
- **Garis potong cakupan** bila waktu habis. Urutan buang: dependabot dan fuzz tambahan, keyset pagination, retensi, Swagger. Tidak boleh dibuang: alur tipis, lease dan recovery, SSRF, metrik, dan benchmark.

## 4. Product owner

- **Kontrak inbound tidak sesuai penyedia nyata.** Stripe dan GitHub tidak bisa mengirim `X-Webhook-Token` atau `X-Event-ID`. Solusi: `trigger.eventId` yang dapat dikonfigurasi (header atau JSONPath, cadangan hash body). Dijadwalkan v0.2.0.
- **Tidak ada jeda flow saat target mati dan tidak ada replay massal.** Dijadwalkan v0.2.0.
- **Tidak ada notifikasi dead-letter.** Minimal alert dari metrik (v0.3.0).
- **Pengguna tidak bisa melihat payload atau hasil transform.** Pertimbangkan endpoint admin dengan batas ukuran dan tanpa log (v0.2.0).
- **Persona dan user story belum ada.** Tulis dengan kriteria penerimaan, dan tetapkan satu metrik produk: waktu dari clone sampai pengiriman pertama berhasil di bawah 10 menit.

## 5. Pengguna produk (integrator)

- **Setiap flow baru memerlukan edit env dan restart API** karena token ada di `ALURJET_WEBHOOK_TOKENS_JSON`. Ini gesekan terbesar. Solusi v0.2.0: token dibuat saat flow dibuat, tampil sekali, dan disimpan sebagai hash.
- **Setup awal berat.** Ada lebih dari delapan rahasia dan satu JSON di env. Sediakan `make init` yang membuat `.env` dengan rahasia acak.
- **Pesan error import flow harus menunjuk lokasi.** Contoh: `steps[1].fields.amount: path "$.total" tidak valid`.
- **Tambahkan filter `GET /executions?event_id=`.** Pengguna tahu ID event mereka, bukan `execution_id`.
- **Secret target `ALURJET_FLOW_*` juga memerlukan restart** API dan worker. Tulis di panduan operasional.

## Prioritas

| Prioritas | Perubahan | Biaya | Kapan |
|---|---|---|---|
| Sebelum kode | Kontrak inbound `eventId` dapat dikonfigurasi (keputusan: ditunda ke v0.2.0, ditulis sebagai batasan) | Kecil di desain, sedang di kode | v0.2.0 |
| Sebelum kode | Token webhook di database (keputusan: ditunda ke v0.2.0) | Sedang | v0.2.0 |
| Sebelum kode | Kuota lunak tanpa lock flow | Kecil | Task 1 |
| Sebelum kode | Terjemahan Inggris dipindah setelah desain beku | Nol | v1.0 |
| Sebelum kode | `.gitattributes` | Nol | Task 1 |
| Saat membangun | `EXPLAIN` query claim, `request_id`, pesan error berlokasi, filter `event_id`, `make init` | Kecil | Task 3 sampai 8 |
| Saat membangun | Uji invarian dengan injeksi kegagalan | Sedang | v1.0 |
| Setelah jalan | Tabel failure mode, model ancaman, dashboard dan alert, SLO | Kecil sampai sedang | v0.3 dan v1.0 |

## Keputusan terkini

| Topik | Keputusan |
|---|---|
| Skema | PK `BIGINT` dengan `public_id` UUID untuk API. Tabel `flow_versions` **dibatalkan**: snapshot tetap disimpan di execution |
| Kuota per flow | Lunak, tanpa lock row flow. Dedup dijamin unique key. Kelebihan terbatas pada jumlah ingest serentak |
| Kontrak inbound v0.1.0 | Tetap `X-Event-ID` dan `X-Webhook-Token`. Ditulis sebagai batasan. Dapat dikonfigurasi di v0.2.0 |
| Token webhook v0.1.0 | Tetap di `ALURJET_WEBHOOK_TOKENS_JSON`. Pindah ke database (hash, tampil sekali) di v0.2.0 |
| Bahasa dokumen | README dua bahasa (`README.md` Inggris, `README.id.md`). Dokumen kunci (`SYSTEM_DESIGN.md`, ADR, engineering-notes, threat-model) diterjemahkan **setelah desain stabil, pada v1.0** |
| Paginasi | `page` dan `limit` di v0.1.0 |
| Masuk v0.1.0 | `/metrics`, jitter backoff, retensi sederhana, CI sejak awal, skrip beban k6, `make demo`, fuzz test detektor field JSON berulang |
| Dibuang dari v0.1.0 | Dependabot, dashboard Grafana, SLO, model ancaman terpisah (semua pindah ke v0.3 atau v1.0) |
| Keputusan lama yang tetap | ADR-002 (paket hemat), ADR-003 (`headers` target), ADR-005 (toolchain), ADR-006 (penyederhanaan M2, kecuali healthcheck worker diganti server `/metrics` dan `/healthz`), module `github.com/lathief/AlurJet` |

## Batasan yang diketahui (calon ADR-010)

- Tidak ada lock row flow pada ingest (kuota lunak). Kuota dapat terlampaui sebanyak jumlah ingest serentak.
- Pool worker bersama, sehingga satu flow dengan target lambat dapat menyita slot flow lain.
- Urutan eksekusi tidak dijamin.
- Delivery at-least-once, bukan exactly-once.
- Replay dengan outbound key yang sama mungkin tidak berefek pada target yang menyimpan hasil per key.

## Peta rilis

Peta ini adalah rencana, bukan status selesai. Roadmap utama ada di [ROADMAP.md](../../ROADMAP.md#peta-rilis-sampai-v10).

| Rilis | Tema | Kriteria selesai |
|---|---|---|
| v0.1.0 | Alur andal end-to-end | Webhook sampai mock, retry, dead-letter, replay, recovery crash, CI hijau, `make demo`, benchmark awal |
| v0.2.0 | Dipakai dengan penyedia nyata | Flow GitHub atau Stripe berjalan tanpa mengubah pengirim, token dikelola lewat API |
| v0.3.0 | Dapat dioperasikan | Dashboard, alert, SLO, kebijakan migrasi, rilis otomatis dengan image bertanda tangan |
| v1.0.0 | Stabil dan terbukti | Kontrak `/api/v1` dibekukan, uji invarian lulus, model ancaman dan review keamanan, dokumen kunci dua bahasa |

Estimasi sekitar 16 minggu solo part-time dan **belum divalidasi**: Fase A 2 hari, v0.1.0 sekitar 6 minggu, v0.2 sekitar 3 minggu, v0.3 sekitar 3 minggu, v1.0 sekitar 2 minggu. Tinjau ulang di akhir Task 4, 11, dan 16.

## Rencana 25 task

| Task | Judul | Bukti selesai |
|---|---|---|
| 1 | Selaraskan desain dengan keputusan sederhana | DDL berjalan pada `mysql:8.4` sekali pakai, 0 tautan rusak, variabel `.env.example` dan `configuration.md` cocok |
| 2 | README dua bahasa, roadmap v1.0, dan `make docs-check` | `docs-check` lulus dan gagal bila heading atau blok kode salah satu README diubah |
| 3 | Fondasi, CI, `/healthz`, `/metrics` | CI hijau, `/healthz` dan `/metrics` berhasil, konfigurasi salah keluar nonnol tanpa mencetak rahasia |
| 4 | Migrasi, Compose, Dockerfile non-root, mock receiver | Bootstrap dari volume kosong, startup ulang no-op, migrasi rusak menahan API dan worker |
| 5 | Parser DSL dan engine | Kasus tidak valid ditolak dengan lokasi error, token angka keluar identik |
| 6 | Manajemen flow | Import `201`, kode `401`, `415`, `413`, `422`, `409`, `404` teruji |
| 7 | Ingest webhook dengan fuzz test | Request identik memberi ID sama, body beda memberi `409`, kuota penuh memberi `429`, ingest serentak menghasilkan satu baris |
| 8 | Alur tipis end-to-end (worker, claim `SKIP LOCKED`, retry dengan jitter, `EXPLAIN` claim) | Dua worker tidak mengklaim execution yang sama, `500` menjadi retry lalu sukses dengan key sama, satu tes E2E di CI |
| 9 | Pengerasan outbound (SSRF, `headers`) | Target IP privat ditolak, `Authorization` sampai ke mock, nilai header tidak tercatat |
| 10 | Recovery, replay, retensi, `make demo` | Worker dimatikan di tengah attempt lalu dipulihkan, replay serentak menghasilkan satu execution, retensi tidak menghapus `pending` atau `running` |
| 11 | Benchmark k6 dan rilis v0.1.0 | `docs/benchmarks.md` berisi angka, Quickstart dari lingkungan bersih, tag `v0.1.0` |
| 12 | Kontrak inbound `trigger.eventId` dapat dikonfigurasi | ID dari header, body, dan hash teruji, dedup benar untuk ketiganya |
| 13 | Verifikasi HMAC webhook | Tanda tangan benar, salah, dan kedaluwarsa teruji dengan vektor dari penyedia |
| 14 | Token webhook di database dengan rotasi | Webhook diterima tanpa restart, tidak ada token mentah di DB atau log |
| 15 | Pause flow, replay massal, lihat payload | Target dimatikan, flow dijeda, dead-letter direplay massal sampai `succeeded` |
| 16 | Contoh integrasi GitHub nyata dan rilis v0.2.0 | Event GitHub nyata sampai ke tujuan, tag `v0.2.0` |
| 17 | Dashboard, alert, SLO | Alert terpicu saat worker dimatikan |
| 18 | Kebijakan migrasi dan kompatibilitas | Migrasi v0.1 ke v0.3 pada data berisi, dua versi berjalan terhadap skema baru |
| 19 | Rilis otomatis (goreleaser, GHCR, SBOM, cosign, govulncheck, dependabot) | `cosign verify` berhasil pada image rilis percobaan |
| 20 | `request_id`, tracing opsional, rilis v0.3.0 | Satu event dapat dilacak dari `202` sampai attempt lewat log, tag `v0.3.0` |
| 21 | Uji invarian dengan injeksi kegagalan | Lari satu jam tanpa pelanggaran invarian |
| 22 | Model ancaman, tabel failure mode, review keamanan | Tiap baris failure mode menaut ke tes atau metrik |
| 23 | Pembekuan kontrak API (OpenAPI, tes kontrak, kebijakan deprecation) | Perubahan yang memutus kompatibilitas membuat CI gagal |
| 24 | Dokumen kunci dua bahasa | `docs-check` lulus pada dokumen kunci dan Anda meninjau terjemahan |
| 25 | Interview notes dengan bukti nyata dan rilis v1.0.0 | Quickstart bersih, CI hijau, uji invarian lulus, tag `v1.0.0` |

## Risiko terbuka

- Estimasi belum divalidasi.
- Task 12 sampai 15 mengubah skema dan kontrak. Butuh ADR dan migrasi.
- Task 14 menggantikan bagian ADR-001 tentang token di konfigurasi. Butuh ADR pengganti.
- Perilaku `SKIP LOCKED` konkuren baru terbukti di Task 8 dan 10.
- Proyek ini besar. Jika waktu terbatas, v0.1.0 dan v0.2.0 sudah cukup sebagai portofolio.

Lihat [review portofolio](review-portofolio.md), [catatan wawancara](interview-notes.md), [roadmap](../../ROADMAP.md), dan [catatan keputusan](decisions.md).
