# Handoff implementasi AlurJet

Catat satu entri per pekerjaan. Gunakan ID kartu atau issue, dan sertakan commit jika tersedia. Tandai hasil sebagai `LULUS`, `GAGAL`, atau `BELUM DIJALANKAN` sesuai bukti. Sertakan alasan dan prasyarat untuk pemeriksaan yang tertunda.

## Entri awal — Paket kerja agent

- **Status:** dokumen perencanaan disiapkan. Implementasi runtime belum dimulai di snapshot ini.
- **Perubahan:** kartu tugas awal, skenario verifikasi, catatan keputusan, changelog blueprint, dan backlog setelah MVP.
- **Verifikasi dokumentasi:** pemeriksaan PowerShell lulus untuk 148 tautan relatif, struktur Markdown, dan tiga kartu tugas awal.
- **Verifikasi runtime:** belum dijalankan. Server dan suite pengujian belum tersedia.
- **Tugas terdekat:** [ALJ-01 — Fondasi API dan lingkungan demo](implementation-plan.md).

## Review lanjutan — Kontrak worker dan tes penerimaan

- **Status:** pembaruan blueprint selesai.
- **Verifikasi dokumentasi:** Dokumen M2/M3 dan kriteria finalisasi dicocokkan. Matriks tes presisi memeriksa token angka pada POST tanpa konversi float64. Pemeriksaan tautan dan keberadaan fixture cURL lulus.
- **Verifikasi runtime:** belum dijalankan karena suite dan runtime belum tersedia.
- **Tugas berikutnya:** kartu fondasi yang tercantum pada entri awal.

## Review tiga proyek — batas transform, recovery, dan autentikasi

- **Kontrak** — anggaran byte berlaku pada setiap transform dan hasil akhir. Recovery minimum pindah ke M2. Token webhook terikat slug melalui `ALURJET_WEBHOOK_TOKENS_JSON`. Blueprint, template konfigurasi, panduan integrasi, roadmap, dan skenario uji sudah diselaraskan.
- **Bukti** — pemeriksaan Python dan `bash -n` lulus untuk 148 tautan, contoh Bash/JSON, fixture event, dan binding token pada slug demo.
- **Tahap berikutnya** — ALJ-01 untuk fondasi. Tes crash worker, isolasi token, dan konsumsi memori mengikuti runtime M2.

## Review lanjutan — kapasitas antrean dan respons target

- **Kontrak** — dedup mendahului kuota ketat per flow dan ambang operasional global. Worker memakai header respons berbatas serta klasifikasi berdasarkan status. Body ditutup tanpa pembacaan aplikasi. Webhook menerapkan kuota di M2 dan replay memakainya di M3.
- **Bukti** — pemeriksaan Python dan `bash -n` mencakup 148 tautan lokal, contoh Bash/JSON, dan kecocokan tiga variabel baru antara blueprint, template, serta referensi konfigurasi.
- **Berikutnya** — ALJ-01. Uji antrean penuh, replay idempoten, dan respons target lambat setelah runtime tersedia.

> Catatan: empat entri di atas dibuat sebelum format tanggal dan ID tugas diwajibkan, sehingga tidak memuat keduanya. Angka "148 tautan" pada entri lama adalah hasil saat itu dan sudah tidak berlaku. Entri baru mengikuti format di bawah.

## 2026-11-26 — ALJ-00 Penutupan celah dokumen

- **Tujuan dan status:** selesai. Menutup celah dokumen awal sebelum kode ditulis dan menerapkan paket hemat.
- **Berkas:** branch `docs/perbaikan-dokumen-awal`. Diubah: `AGENTS.md`, `SYSTEM_DESIGN.md`, `ROADMAP.md`, `README.md`, `CHANGELOG.md`, `.env.example`, `docs/ai/*`, `docs/post-mvp.md`, `docs/reference/{api,errors,configuration}.md`, `docs/guides/{flow-configuration,sending-webhooks}.md`, `docs/getting-started/{installation,quickstart}.md`, `docs/operations/*`. Baru: `docs/reference/database.md`, `docs/getting-started/development.md`, `docs/examples/order-to-api-auth.yaml`. Dihapus: `docs/ai/review-gap-dokumentasi.md`.
- **Perintah yang dijalankan dan hasil:**
  - Pemeriksaan tautan relatif dan anchor dengan skrip Node pada seluruh `*.md`: 218 tautan, 0 rusak.
  - Pemeriksaan kecocokan variabel `.env.example` dengan `configuration.md`: semua cocok. Satu-satunya variabel yang hanya ada di dokumen adalah `ALURJET_ACTIVE_HIGH_WATERMARK`, yang sengaja disebut sebagai fitur yang ditunda.
  - DDL dari `database.md` diterapkan pada `mysql:8.4` (8.4.13) sekali pakai: tiga tabel terbentuk tanpa error. `event_id` `Order-1` dan `order-1` sama-sama diterima, duplikat `Order-1` ditolak (`ERROR 1062`, unique key `uq_exec_flow_event`), status di luar daftar ditolak (`ERROR 3819`, `ck_exec_status`), dan `SELECT ... FOR UPDATE SKIP LOCKED` dapat dijalankan pada query bentuk claim.
- **Pemeriksaan tertunda:** perilaku `SKIP LOCKED` di bawah konkurensi (dua transaksi) belum diuji, dan dokumentasi resmi MySQL 8.4 tentang locking reads tidak dapat diambil (timeout). Uji pada kartu ALJ-07. Fixture YAML baru belum diparse oleh parser nyata karena belum ada kode. Semua perintah curl dan Compose pada dokumen masih target.
- **Keputusan yang berubah:** [ADR-002 sampai ADR-005](decisions.md).
- **Risiko terbuka:** `docker-compose.yml`, `Dockerfile`, `migrations/`, `LICENSE`, `CONTRIBUTING.md`, dan `SECURITY.md` belum ada (dijadwalkan di ALJ-02 dan M4). Estimasi jadwal 6–8 minggu belum divalidasi ulang setelah pemangkasan.
- **Tugas berikutnya:** [ALJ-01](implementation-plan.md) (fondasi Go, konfigurasi, `/healthz`). Prasyarat: Go 1.23+ terpasang.

## Format entri

```text
Tanggal dan ID tugas:
Tujuan dan status:
Berkas atau commit:
Perintah yang dijalankan, hasil, dan lokasi bukti:
Pemeriksaan tertunda dan alasannya:
Keputusan yang berubah (tautan ke decisions.md):
Risiko atau blocker yang masih terbuka:
Satu tugas berikutnya dan prasyaratnya:
```

Lihat [kriteria verifikasi](verification.md), [catatan keputusan](decisions.md), dan [changelog blueprint](../../CHANGELOG.md).
