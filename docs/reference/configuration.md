# Referensi konfigurasi lingkungan

Tabel ini mengikuti nilai contoh di [.env.example](../../.env.example). Nilai contoh digunakan untuk demo lokal dan perlu diganti sesuai lingkungan.

| Variabel | Nilai contoh | Kegunaan |
|---|---|---|
| `ALURJET_HTTP_ADDR` | `:8080` | Alamat listener API |
| `ALURJET_MYSQL_HOST`, `ALURJET_MYSQL_PORT` | `mysql`, `3306` | Host dan port MySQL untuk API, worker, serta migrator |
| `ALURJET_MYSQL_DATABASE`, `ALURJET_MYSQL_USER` | `alurjet`, `alurjet` | Database dan user bersama. Compose memakai nilai yang sama untuk menyiapkan MySQL |
| `ALURJET_MYSQL_PASSWORD` | Placeholder | Password user aplikasi dan migrator. Ganti sebelum volume demo diinisialisasi |
| `ALURJET_MYSQL_ROOT_PASSWORD` | Placeholder | Password root yang terpisah, hanya untuk inisialisasi service MySQL pada Compose demo |
| `ALURJET_ADMIN_API_KEY` | Placeholder | Rahasia untuk management API |
| `ALURJET_WEBHOOK_TOKENS_JSON` | `{"order-to-api":"<token>"}` | Peta slug ke token unik untuk proses API. Siapkan token untuk setiap flow yang menerima webhook |
| `ALURJET_FLOW_TARGET_URL` | `http://mock-api:8081` | Origin yang dipakai oleh flow contoh |
| `ALURJET_ALLOWED_ORIGINS` | `http://mock-api:8081` | Allowlist origin berdasarkan skema, hostname, dan port efektif |
| `ALURJET_DEV_PRIVATE_ORIGIN` | `http://mock-api:8081` | Pengecualian satu origin untuk demo privat. Kosongkan di luar demo |
| `ALURJET_MOCK_MODE` | `success` | Mode mock receiver. Pilihan lain adalah `fail_once` dan `always_fail` |
| `ALURJET_WORKER_POLL_INTERVAL` | `2s` | Interval polling worker |
| `ALURJET_WORKER_ATTEMPT_TIMEOUT` | `15s` | Anggaran waktu satu attempt |
| `ALURJET_WORKER_LEASE_DURATION` | `60s` | Durasi lease execution |
| `ALURJET_MAX_PAYLOAD_BYTES` | `1048576` | Batas ukuran body payload masuk |
| `ALURJET_MAX_OUTBOUND_BYTES` | `1048576` | Batas byte body JSON final sebelum DNS dan koneksi outbound. Harus positif |
| `ALURJET_MAX_RESPONSE_HEADER_BYTES` | `65536` | Batas byte header respons HTTP target |
| `ALURJET_MAX_ACTIVE_PER_FLOW` | `250` | Kuota per flow untuk pending, running, dan retry_scheduled |
| `ALURJET_MAX_FLOW_YAML_BYTES` | `65536` | Batas ukuran body YAML pada create/update flow |
| `ALURJET_WORKER_CONCURRENCY` | `4` | Jumlah attempt paralel per proses worker |
| `ALURJET_WORKER_RECOVERY_INTERVAL` | `15s` | Jeda antar pemindaian lease kedaluwarsa |
| `ALURJET_WORKER_RECOVERY_BATCH` | `50` | Jumlah maksimum execution yang dipulihkan per pemindaian |
| `ALURJET_MYSQL_MAX_OPEN_CONNS` | `20` | Batas koneksi terbuka ke MySQL per proses |
| `ALURJET_SHUTDOWN_TIMEOUT` | `20s` | Waktu tunggu attempt dan request berjalan saat SIGTERM |
| `ALURJET_MOCK_PORT` | `8081` | Port mock receiver di jaringan Compose |
| `ALURJET_FLOW_*` (mis. `ALURJET_FLOW_TARGET_TOKEN`) | Placeholder | Variabel yang dirujuk flow. Dipakai untuk URL (saat import) dan `headers` (saat dispatch). API dan worker harus menerima nilai yang sama |
| `ALURJET_LOG_LEVEL` | `info` | Level log aplikasi |

Batas ukuran dan kapasitas harus positif. Nilai kapasitas di atas adalah titik awal demo. `ALURJET_ADMIN_API_KEY` dan setiap token webhook harus acak dan minimal 32 karakter, dan startup gagal bila tidak terpenuhi. Anggaran attempt harus positif dan lebih pendek dari durasi lease. Ambang global antrean (`ALURJET_ACTIVE_HIGH_WATERMARK`) ditunda ke [backlog pasca-MVP](../post-mvp.md).

Semua variabel dibaca saat startup. Perubahan nilai, termasuk rotasi token dan `ALURJET_FLOW_*`, memerlukan restart proses. Konfigurasi timeout HTTP pada flow dibatasi oleh anggaran attempt. Jika variabel URL berubah, impor ulang atau perbarui flow untuk menerapkannya ke event baru. Snapshot lama tetap memakai URL yang sudah tersimpan.

Lihat [konfigurasi YAML](../guides/flow-configuration.md), [keamanan](../operations/security.md), dan [deployment](../operations/deployment.md).

[Kembali ke indeks dokumentasi](../README.md)
