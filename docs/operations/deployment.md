# Deployment dan pembaruan

Rancangan deployment lokal menjalankan API, worker, MySQL, dan mock receiver lewat Docker Compose. Ikuti [instalasi lokal](../getting-started/installation.md) dan [panduan cepat](../getting-started/quickstart.md) setelah artefak runtime tersedia.

## Menyiapkan layanan

1. Tetapkan koneksi MySQL, kredensial, timeout worker, dan allowlist sesuai [referensi konfigurasi](../reference/configuration.md). Compose menggunakan sumber kredensial yang sama untuk MySQL, migrator, dan aplikasi.
2. Tunggu MySQL sehat dan jalankan migrasi yang disertakan rilis. Pada Compose, service `migrate` harus selesai sukses sebelum API dan worker dimulai. Jika gagal, periksa status dan log `migrate` sebelum mengulang startup.
3. Jalankan mode `alurjet api` dan `alurjet worker`. Lindungi akses API dengan HTTPS di luar demo lokal.
4. Cek `GET /healthz`, lalu kirim event uji dan verifikasi bahwa execution mencapai `succeeded`. Pemeriksaan API saja belum membuktikan worker memproses antrean.

## Pembaruan

Tinjau migrasi dan siapkan pemulihan database sebelum mengubah schema. Periksa execution yang masih tertunda dan pastikan worker versi baru dapat memproses snapshot yang sudah tersimpan. Setelah pembaruan, periksa kesehatan API, riwayat attempt, dan satu pengiriman ke target yang diizinkan.

## Backup, restore, dan pertumbuhan data

MVP tidak menghapus data lama. Tabel `executions` menyimpan `payload_json` dan `flow_snapshot` sebagai teks biasa, sehingga ukurannya tumbuh seiring jumlah event dan payload dapat memuat data pribadi. Buat backup logis sebelum pembaruan dan secara berkala, misalnya `mysqldump --single-transaction --routines alurjet > alurjet.sql`, lalu uji restore pada instance terpisah. Simpan berkas backup dengan kontrol akses yang sama seperti database. Rencana retensi ada di [backlog pasca-MVP](../post-mvp.md). Skema lengkap ada di [referensi database](../reference/database.md).

## Beberapa instance dan restart

Beberapa API dan worker dapat berjalan bersamaan. Perubahan admin key, token webhook, atau variabel `ALURJET_FLOW_*` (termasuk secret header target) hanya berlaku setelah proses direstart. API dan worker harus menerima nilai `ALURJET_FLOW_*` yang sama.

Mode `mock-api` dan pengecualian origin privat disiapkan untuk demo lokal. Lihat [pemantauan](monitoring.md) dan [keamanan operasional](security.md).

[Kembali ke indeks dokumentasi](../README.md)
