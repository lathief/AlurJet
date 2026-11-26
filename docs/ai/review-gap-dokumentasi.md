# Review celah dokumentasi awal AlurJet

> **Status:** hasil review dokumen blueprint sebelum implementasi dimulai. Semua butir di bawah berstatus **terbuka** sampai ditutup dan dicatat di [worklog](worklog.md). Keputusan yang diambil dicatat di bagian akhir dan perlu dipindahkan ke [catatan keputusan](decisions.md) sebagai ADR.

## Cakupan review

Review membaca seluruh dokumen di repository: `README.md`, `SYSTEM_DESIGN.md`, `ROADMAP.md`, `AGENTS.md`, `CHANGELOG.md`, `.env.example`, serta seluruh isi `docs/`. Repository belum memiliki kode, `go.mod`, Dockerfile, atau Compose.

Dokumen sudah kuat pada konkurensi, lease, deduplikasi, dan kebijakan jaringan. Celah terbesar ada pada skema database, kontrak API, katalog error, konfigurasi runtime, toolchain, dan fungsi dasar yang dibutuhkan integrasi nyata.

## A. Harus ditutup sebelum ALJ-01 dan ALJ-02

| # | Celah | Dampak |
|---|---|---|
| A1 | **Tidak ada DDL atau ERD.** Desain hanya menyebut nama kolom. Tipe, panjang, FK, charset, dan collation belum ditentukan. | Perbandingan `event_id` dan `replay_request_key` harus case-sensitive (§4), jadi kolom perlu collation biner. Format ID tidak seragam: `flows.id` integer (`{"id":1}` di `api.md`), `execution_id` UUID. Tipe `status`, `flow_snapshot`, dan presisi waktu belum tertulis. |
| A2 | **Versi dan toolchain belum ditetapkan.** Belum ada versi Go, module path, versi MySQL, pilihan Fiber v2 atau v3, versi golang-migrate, konfigurasi lint, atau Makefile. | Roadmap menuntut lint di CI tanpa menyebut alatnya. Hasil build bisa berbeda antar mesin. |
| A3 | **`AGENTS.md` tidak memuat konvensi koding.** Isinya kontrak perilaku. | Belum ada aturan penamaan, error wrapping, penggunaan `context`, struktur test, gaya commit, atau cara menjalankan test integrasi. |
| A4 | **Konfigurasi runtime kurang.** Tidak ada jumlah slot worker, interval dan batch recovery, timeout shutdown, pool koneksi DB, port mock-api, atau batas ukuran YAML flow. | Nilai-nilai ini akan ditentukan sembarang di kode. Daftar validasi startup dan panjang minimum secret juga belum ada. |
| A5 | **Graceful shutdown dan health belum dispesifikasikan.** Belum jelas apakah `/healthz` memeriksa DB, tidak ada healthcheck worker, dan perilaku saat SIGTERM belum ditulis. | Compose dan operator tidak bisa membedakan proses hidup dari proses sehat. |
| A6 | **Aturan parsing YAML kurang.** Belum ada aturan field tak dikenal, regex dan panjang slug, penanganan anchor/alias, atau batas ukuran body flow. | Risiko memory bomb dan flow yang diterima diam-diam padahal salah ketik. |

## B. Harus ditutup sebelum M2

| # | Celah | Dampak |
|---|---|---|
| B1 | **Target API tidak bisa diberi autentikasi.** Step `http` tidak punya header kustom. Konvensi `Content-Type` outbound juga tidak disebut. | Hampir semua API nyata memerlukan `Authorization` atau API key. Ini celah fungsional terbesar. |
| B2 | **Katalog error tidak lengkap.** `errors.md` hanya menyebut sebagian kode. Nilai `error_code` pada attempt tidak dienumerasi. | Diperlukan untuk log, detail execution, dan tes. |
| B3 | **Kontrak response API setengah jadi.** Response list/detail flow dan `PUT` belum ada. Bentuk paginasi, urutan list, kelengkapan `attempt_history`, format waktu, dan nilai `Retry-After` belum jelas. Tidak ada OpenAPI sebelum kode ada. | Tes controller tidak punya spesifikasi yang bisa diikuti. |
| B4 | **Daur hidup flow tidak lengkap.** Tidak ada `DELETE` atau disable flow. Tidak ada aturan untuk flow yang punya execution aktif. Perubahan slug belum dinyatakan terlarang. | Perilaku di tepi ditentukan oleh implementasi. |
| B5 | **Semantik `validate` dan `transform` terlalu tipis.** Belum ada aturan tipe data (string kosong, `0`, `false`). Validasi hanya top-level sedangkan transform mendukung path bersarang. Tidak ada literal konstan. | Payload yang buruk tetap mendapat `202` lalu masuk `dead_letter`, dan replay tidak membantu. Trade-off ini perlu dicatat sebagai keputusan. |
| B6 | **Tidak ada cara melihat apa yang dikirim.** Detail execution tidak menampilkan payload dan hasil transform tidak disimpan. | Operator hanya punya log untuk debugging. |
| B7 | **Perilaku multi-instance tidak dinyatakan.** `SKIP LOCKED` seharusnya aman untuk banyak worker, tetapi dokumen tidak menyatakannya. Perubahan token webhook memerlukan restart API, dan ini juga belum tertulis. | Operator tidak tahu batas skala dan prosedur rotasi. |

## C. Perlu sebelum rilis (M4)

| # | Celah |
|---|---|
| C1 | Berkas yang dirujuk belum ada: `LICENSE`, `CONTRIBUTING.md`, `SECURITY.md`, template issue, `Dockerfile`, `docker-compose.yml`, `migrations/`. |
| C2 | Tidak ada rencana backup, restore, dan retensi. `payload_json` disimpan plaintext dan jendela dedup tidak terbatas. Retensi baru muncul di backlog pasca-MVP. |
| C3 | Tidak ada model ancaman ringkas. SSRF sudah dibahas. Belum ada pembahasan brute-force token (rate limit auth gagal), ketiadaan HMAC atau timestamp pada webhook, dan rotasi admin key. |
| C4 | Tidak ada catatan pengembangan di Windows. Semua contoh memakai bash (`export`, `curl`). |
| C5 | Bahasa dokumen belum ditetapkan. Prosa memakai Bahasa Indonesia, kode dan error memakai Inggris. Tidak ada glosarium. |

## D. Risiko cakupan dan kualitas dokumen

- **Cakupan MVP lebih berat dari estimasi 6–8 minggu part-time.** Dialer yang mengunci IP, builder JSON dengan anggaran byte yang mempertahankan token angka, kuota dengan lock flow, recovery lease, dan matriks tes yang sangat rinci masing-masing setara fitur besar.
- **Spesifikasi tidak merata.** Konkurensi dan lease sangat rinci, sedangkan skema DB, error code, dan response API tidak.
- **`decisions.md` berupa tabel ringkasan,** belum berformat ADR (konteks, alternatif, konsekuensi).
- **Tanggal `Updated: 2026-11-23` di `SYSTEM_DESIGN.md`** perlu dicek kebenarannya.
- **Format curl.** Perintah curl di `quickstart.md` dan `api.md` ditulis satu baris dengan spasi ganda. Perintahnya tetap valid, jadi ini soal keterbacaan. Bagian replay di `quickstart.md` sudah memakai `\`.

## Keputusan yang diambil

| Topik | Keputusan |
|---|---|
| Arah | Tutup celah A–B di dokumen sebelum menulis kode. |
| Header target (B1) | Tambahkan `headers` pada step `http`. Nilai memakai placeholder `ALURJET_FLOW_*`. Lihat rancangan di bawah. |
| Cakupan (D) | **Paket hemat.** Tunda `ALURJET_ACTIVE_HIGH_WATERMARK` dan `503 SYSTEM_BACKPRESSURE`, serta batas byte pada transform antara. Pertahankan kuota per flow (`429`), batas payload masuk, batas body final, dan kebijakan jaringan penuh. |
| Skema (A1) | `flows.id` BIGINT auto-increment. `executions.id` UUID v4 `CHAR(36)`. Kolom kunci `ascii_bin`. Waktu `DATETIME(3)` UTC. |
| Toolchain (A2) | Go 1.23+, Fiber v2, MySQL 8.4 LTS, golang-migrate v4, golangci-lint. Test integrasi memakai Docker Compose terpisah. |

### Rancangan `headers` (B1)

```yaml
- id: forward
  type: http
  method: POST
  url: "${ALURJET_FLOW_TARGET_URL}/orders"
  headers:
    Authorization: "Bearer ${ALURJET_FLOW_TARGET_TOKEN}"
  timeout: 5s
```

- URL tetap diresolusi saat import dan disimpan di definisi terkompilasi.
- Nilai header hanya divalidasi saat import (nama berawalan `ALURJET_FLOW_` dan variabel nonkosong). Definisi terkompilasi menyimpan template, bukan nilai. Worker meresolusi dari environment saat dispatch, sehingga secret tidak masuk DB, snapshot, atau response API.
- Variabel hilang atau kosong saat dispatch menghasilkan kegagalan permanen `HEADER_RESOLVE_FAILED`. API dan worker harus memakai environment yang sama.
- Maksimum 10 header. Nama harus token HTTP valid. Nilai tidak boleh mengandung CR/LF. Header terlarang: `Host`, `Content-Length`, `Content-Type`, `Transfer-Encoding`, `Connection`, `Idempotency-Key`, `Proxy-*`.
- Nilai header tidak pernah dicatat di log atau ditampilkan di API.

### Dampak pemangkasan cakupan

- Hapus `ALURJET_ACTIVE_HIGH_WATERMARK` dan `503 SYSTEM_BACKPRESSURE` dari desain, roadmap, konfigurasi, dan referensi error.
- Ganti anggaran byte per transform dengan tiga kontrol: batas payload masuk, maksimum 64 field transform, dan pemeriksaan body final terhadap `ALURJET_MAX_OUTBOUND_BYTES` sebelum DNS atau koneksi.
- Pindahkan backpressure global dan anggaran byte per step ke [backlog pasca-MVP](../post-mvp.md).

## Status penutupan

| Butir | Status |
|---|---|
| A1–A6 | Terbuka |
| B1–B7 | Terbuka |
| C1–C5 | Terbuka |
| D | Terbuka |

Perbarui tabel ini dan catat bukti di [worklog](worklog.md) saat setiap butir ditutup.

Lihat [desain sistem](../../SYSTEM_DESIGN.md), [roadmap](../../ROADMAP.md), dan [paket kerja awal](implementation-plan.md).
