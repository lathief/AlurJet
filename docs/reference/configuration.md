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
| `ALURJET_MAX_OUTBOUND_BYTES` | `1048576` | Batas byte output setiap transform dan body final. Harus positif |
| `ALURJET_MAX_RESPONSE_HEADER_BYTES` | `65536` | Batas byte header respons HTTP target |
| `ALURJET_MAX_ACTIVE_PER_FLOW` | `250` | Kuota per flow untuk pending, running, dan retry_scheduled |
| `ALURJET_ACTIVE_HIGH_WATERMARK` | `2000` | Ambang operasional global untuk submit dan replay baru |
| `ALURJET_LOG_LEVEL` | `info` | Level log aplikasi |

Batas kapasitas dan header respons harus positif. Nilai kapasitas di atas adalah titik awal demo. Anggaran attempt harus positif dan lebih pendek dari durasi lease. Konfigurasi timeout HTTP pada flow dibatasi oleh anggaran attempt. Jika variabel URL berubah, impor ulang atau perbarui flow untuk menerapkannya ke event baru. Snapshot lama tetap memakai URL yang sudah tersimpan.

Lihat [konfigurasi YAML](../guides/flow-configuration.md), [keamanan](../operations/security.md), dan [deployment](../operations/deployment.md).

[Kembali ke indeks dokumentasi](../README.md)
