# Dokumentasi AlurJet

AlurJet menerima webhook, memvalidasi dan memetakan JSON, lalu mengirim hasilnya ke API tujuan melalui worker.

> **Status:** dokumentasi ini menjelaskan rancangan v0.1.0 berdasarkan [desain sistem](../SYSTEM_DESIGN.md). Perintah, respons API, dan contoh konfigurasi masih perlu diverifikasi terhadap runtime.

## Mulai menggunakan AlurJet

1. [Instalasi dan persyaratan](getting-started/installation.md) untuk menyiapkan lingkungan lokal. Pengembang juga membaca [panduan pengembangan](getting-started/development.md).
2. [Autentikasi](guides/authentication.md) untuk memilih kredensial sesuai endpoint.
3. [Panduan cepat](getting-started/quickstart.md) untuk mengimpor flow, mengirim event, dan melihat hasilnya.

## Integrasi API

- [Konfigurasi flow YAML](guides/flow-configuration.md) untuk menulis step validasi, pemetaan, dan HTTP.
- [Mengirim webhook](guides/sending-webhooks.md) untuk memahami event ID dan deduplikasi.
- [Melacak execution](guides/tracking-executions.md) untuk membaca status, riwayat attempt, dan aturan replay.
- [Contoh integrasi pesanan](examples/order-webhook.md) untuk membandingkan payload masuk dengan outbound JSON.

## Referensi

- [API HTTP](reference/api.md)
- [Skema database](reference/database.md)
- [Konfigurasi lingkungan](reference/configuration.md)
- [Error HTTP dan retry](reference/errors.md)

## Operasional

- [Deployment dan pembaruan](operations/deployment.md)
- [Pemantauan dan log](operations/monitoring.md)
- [Pemecahan masalah](operations/troubleshooting.md)
- [Keamanan operasional](operations/security.md)
- [Pertanyaan umum](faq.md)

## Review dan perencanaan

- [Review multi-peran](ai/review-multi-peran.md): temuan lima sudut pandang, keputusan terkini, peta rilis sampai v1.0, dan rencana 25 task. Berstatus rencana, belum diterapkan.
- [Review portofolio](ai/review-portofolio.md): review awal untuk portofolio backend middle/senior. Disempurnakan oleh review multi-peran.
- [Catatan wawancara](ai/interview-notes.md): kerangka pertanyaan dan jawaban yang perlu ditulis ulang dengan kata sendiri.

## Pengembangan

- [Paket kerja awal](ai/implementation-plan.md), [matriks verifikasi](ai/verification.md), dan [handoff](ai/worklog.md).
- [Catatan keputusan](ai/decisions.md), [changelog](../CHANGELOG.md), dan [backlog setelah MVP](post-mvp.md).

## Status dan konvensi

- Gunakan `localhost:8080` untuk demo lokal dan HTTPS untuk akses lewat jaringan.
- Jalankan contoh terminal dari root proyek. Isi placeholder dan variabel menggunakan nilai lingkungan Anda.
- `202 Accepted` menunjukkan execution sudah tersimpan. Periksa detail execution untuk melihat hasil pengiriman.
- Swagger UI direncanakan untuk mode pengembangan di `/swagger/index.html`.

## Pemeliharaan dokumentasi

| Perubahan | Dokumen terkait |
|---|---|
| Endpoint, autentikasi, atau status HTTP | [API HTTP](reference/api.md), [autentikasi](guides/authentication.md), [error HTTP](reference/errors.md), serta anotasi Swaggo |
| DSL, payload, atau snapshot | [Konfigurasi flow](guides/flow-configuration.md), [webhook](guides/sending-webhooks.md), dan [contoh integrasi](examples/order-webhook.md) |
| Retry, replay, atau recovery | [Pelacakan execution](guides/tracking-executions.md), [pemantauan](operations/monitoring.md), dan [pemecahan masalah](operations/troubleshooting.md) |
| Variabel lingkungan atau deployment | [Konfigurasi lingkungan](reference/configuration.md), [deployment](operations/deployment.md), dan [keamanan](operations/security.md) |
| Skema tabel atau query antrean | [Skema database](reference/database.md) dan migrasi |
| Keputusan yang memengaruhi kontrak | [Catatan keputusan (ADR)](ai/decisions.md) |

Lihat [roadmap](../ROADMAP.md) untuk kriteria penerimaan dan [AGENTS.md](../AGENTS.md) untuk panduan implementasi. Sebelum rilis, jalankan Panduan Cepat dari lingkungan bersih dan periksa contoh sukses, retry, dead-letter, serta replay.
