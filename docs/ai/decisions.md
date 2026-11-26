# Catatan keputusan AlurJet (ADR)

Keputusan teknis lengkap berada di [desain sistem](../../SYSTEM_DESIGN.md). File ini mencatat alasan, alternatif, dan konsekuensi. Tambahkan ADR baru di bagian bawah dengan nomor berikutnya. Jangan menulis ulang ADR lama. Jika keputusan berubah, tambahkan ADR baru dan ubah status yang lama menjadi `Digantikan oleh ADR-NNN`.

Format: **Status**, **Tanggal**, **Konteks**, **Keputusan**, **Alternatif**, **Konsekuensi**. Bukti eksekusi per tugas masuk ke [worklog](worklog.md).

## Ringkasan

| ADR | Keputusan | Status |
|---|---|---|
| 001 | Kontrak inti blueprint (antrean MySQL, satu HTTP POST terakhir, dedup byte body, replay, token per slug, recovery M2) | Diterima |
| 002 | Paket hemat: tunda ambang global dan anggaran byte per transform | Diterima |
| 003 | Header target lewat template dengan resolusi saat dispatch | Diterima |
| 004 | Skema ID dan collation | Diterima |
| 005 | Toolchain dan versi | Diterima |

## ADR-001 — Kontrak inti blueprint

- **Status:** Diterima. **Tanggal:** 2026-11-23 (dicatat ulang 2026-11-26).
- **Konteks:** MVP harus menunjukkan alur webhook sampai HTTP target yang dapat dilacak, dengan perilaku konkurensi yang benar.
- **Keputusan:**

| Keputusan | Dampak implementasi | Rujukan |
|---|---|---|
| MySQL adalah penyimpanan dan antrean | Job langsung siap setelah commit, claim dan riwayat attempt atomik | Desain §§4–5 |
| Tepat satu HTTP POST di step terakhir | Worker dapat mengulang flow dari snapshot pada setiap attempt | Desain §6 |
| Dedup webhook membandingkan byte body | ID sama dan byte berbeda menghasilkan `409`; replay memiliki key request tersendiri | Desain §5 |
| Kapasitas berlaku untuk insert execution baru | Deduplikasi yang sudah commit tetap bisa dibaca. Kuota per flow ketat | Desain §5 |
| Connector memakai status target dan membatasi header | Body target ditutup tanpa dibaca pada MVP | Desain §5 |
| Replay mempertahankan snapshot dan outbound key asal | Perubahan flow berlaku untuk event baru, sedangkan kebijakan outbound aktif tetap diperiksa | Desain §§5, 8 |
| JSON mempertahankan token angka sejak ingest | Worker memeriksa presisi hingga serialisasi POST | Desain §§4–6, 9 |
| Token webhook terikat slug | Satu token untuk satu flow, diatur oleh operator melalui konfigurasi API | Desain §§7–8 |
| Recovery minimum tersedia sejak M2 | Restart worker memulihkan lease expired; retry memakai outbound key yang sama | Desain §5 dan roadmap |

- **Alternatif:** broker pesan (Redis/NATS) ditolak karena menambah komponen operasional tanpa kebutuhan terukur.
- **Konsekuensi:** throughput dibatasi polling MySQL. Evaluasi broker ada di [backlog pasca-MVP](../post-mvp.md).

## ADR-002 — Paket hemat: pemangkasan cakupan

- **Status:** Diterima. **Tanggal:** 2026-11-26.
- **Konteks:** Estimasi 6–8 minggu part-time terlalu optimistis untuk cakupan awal. Ambang global dan anggaran byte per transform paling mahal dibanding manfaat demonya.
- **Keputusan:**
  - Tunda `ALURJET_ACTIVE_HIGH_WATERMARK` dan `503 SYSTEM_BACKPRESSURE`. Kuota per flow (`429 FLOW_CAPACITY_EXCEEDED`) tetap ada.
  - Ganti anggaran byte per transform dengan tiga kontrol: batas payload masuk, maksimum 64 field transform, dan pemeriksaan body final terhadap `ALURJET_MAX_OUTBOUND_BYTES` sebelum DNS.
  - Pertahankan kebijakan jaringan penuh (origin, validasi IP saat dial, tanpa redirect).
- **Alternatif:** mempertahankan semua (ditolak: risiko jadwal), menunda juga replay idempoten (ditolak: replay sudah bagian inti demo).
- **Konsekuensi:** satu error code dan satu variabel lingkungan hilang. Beban global antrean tidak dibatasi, jadi operator bergantung pada kuota per flow dan jumlah flow. Item yang ditunda dicatat di backlog pasca-MVP.

## ADR-003 — Header target lewat template, resolusi saat dispatch

- **Status:** Diterima. **Tanggal:** 2026-11-26.
- **Konteks:** Hampir semua API target memerlukan `Authorization` atau API key. Step `http` awal tidak punya header kustom.
- **Keputusan:** Step `http` menerima `headers`. Nilai memakai placeholder `${ALURJET_FLOW_*}`. Saat create/update, sistem hanya memvalidasi nama placeholder dan variabel nonkosong, lalu menyimpan template. Worker meresolusi dari environment saat dispatch. Batas dan daftar header terlarang ada di [desain §6](../../SYSTEM_DESIGN.md).
- **Alternatif:** menyimpan nilai terkompilasi di snapshot (ditolak: secret masuk DB dan response). Tidak mendukung header (ditolak: demo tidak realistis).
- **Konsekuensi:** API dan worker harus memakai environment yang sama. Rotasi secret tidak memerlukan impor ulang, tetapi memerlukan restart proses. Variabel hilang menghasilkan `HEADER_RESOLVE_FAILED` (permanen). Nilai header tidak pernah dicatat di log atau response.

## ADR-004 — Skema ID dan collation

- **Status:** Diterima. **Tanggal:** 2026-11-26.
- **Konteks:** `event_id` dan `replay_request_key` harus dibandingkan case-sensitive. Dokumen memakai ID integer untuk flow dan UUID untuk execution tanpa tipe kolom.
- **Keputusan:** `flows.id` `BIGINT UNSIGNED` auto-increment. `executions.id` UUID v4 `CHAR(36)`. Kolom kunci `ascii_bin`. Waktu `DATETIME(3)` UTC. Skema lengkap di [referensi database](../reference/database.md).
- **Alternatif:** UUID v7 `BINARY(16)` (lebih efisien untuk indeks, lebih sulit dibaca saat debugging).
- **Konsekuensi:** ID mudah dibaca di log dan curl. Indeks UUID acak sedikit kurang efisien, dapat dievaluasi lagi setelah pengukuran.

## ADR-005 — Toolchain dan versi

- **Status:** Diterima. **Tanggal:** 2026-11-26.
- **Konteks:** Dokumen awal tidak menetapkan versi sehingga build dan perilaku dapat berbeda antar mesin.
- **Keputusan:** Go 1.23+, Fiber v2, MySQL 8.4 LTS, golang-migrate v4, `yaml.v3`, golangci-lint. Tes integrasi memakai Docker Compose terpisah (`docker-compose.test.yml`), tanpa testcontainers.
- **Alternatif:** Fiber v3 (ditolak: baru dan API masih berubah), testcontainers-go (ditolak: dependency tambahan).
- **Konsekuensi:** Docker wajib untuk tes integrasi. Pengembangan di Windows dijelaskan di [panduan pengembangan](../getting-started/development.md).
