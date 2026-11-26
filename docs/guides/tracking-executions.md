# Melacak execution dan riwayat attempt

> Dokumen ini menerangkan **perilaku target MVP**, bukan fitur yang telah diverifikasi berjalan.

## Alur penggunaan

1. **Buat flow** dalam YAML: tentukan nama/slug, webhook trigger, dan step berurutan.
2. **Import flow** lewat management API dengan admin key.
3. **Kirim event** sebagai JSON ke endpoint webhook yang sesuai slug.
4. **Periksa execution** menggunakan ID dari respons `202`.
5. Jika target bermasalah, periksa riwayat percobaan dan perbaiki target sebelum replay dari `dead_letter`. Jika data atau flow perlu diubah, kirim event baru dengan ID baru.

## Istilah utama

| Istilah | Maksud |
|---|---|
| Flow | Resep integrasi yang didefinisikan dalam YAML |
| Trigger | Sumber proses; di MVP hanya webhook |
| Step | Aksi berurutan: validate, transform, HTTP |
| Execution | Satu event yang sedang/selesai diproses |
| Attempt | Satu percobaan menjalankan execution |
| Retry | Penjadwalan ulang ketika gagal sementara |
| Dead-letter | Status akhir ketika event tidak bisa selesai otomatis |
| Replay | Membuat execution baru dari payload dan snapshot lama jika kapasitas mengizinkan. Outbound idempotency key tetap sama |

## Status execution

| Status | Arti | Tindakan pengguna |
|---|---|---|
| `pending` | Event sudah tersimpan, belum diambil worker | Tunggu worker; cek service bila terlalu lama |
| `running` | Worker sedang memproses | Lihat log menggunakan execution_id |
| `retry_scheduled` | Terjadi error sementara, percobaan berikutnya terjadwal | Periksa response/error target |
| `succeeded` | Semua step selesai | Tidak ada |
| `dead_letter` | Gagal permanen atau retry habis | Perbaiki penyebab, kemudian replay manual |

## Kapan retry berjalan?

Default rancangan memberi **3 attempts**, dengan jeda 2 detik lalu 4 detik. Timeout, HTTP `408`, `429`, `5xx`, dan kegagalan transport sementara dapat dicoba lagi. Error validasi, redirect, dan sebagian besar `4xx` langsung menuju `dead_letter`. Jika worker mati, recovery menutup attempt sebagai `lease_expired`. Jatah yang tersisa menentukan apakah execution dijadwalkan ulang atau berakhir `dead_letter`.

Perhatian: **at-least-once** berarti suatu HTTP request bisa terkirim dua kali dalam skenario crash/network. Target API yang mengubah data sebaiknya menghormati `Idempotency-Key` dari AlurJet. Jaminan terhadap efek ganda bergantung pada berapa lama target menyimpan key.

## Perubahan flow

Mengedit flow memengaruhi **event baru**. Execution yang sudah masuk menyimpan snapshot terkompilasi, termasuk URL yang sudah teresolusi. Retry dan replay menggunakan snapshot asal. Jika nilai variabel URL berubah, impor ulang flow agar event baru memakai nilai tersebut.

## Praktik terbaik

- Pakai slug yang jelas seperti `order-to-api`.
- Berikan `X-Event-ID` unik per event logis. Untuk mengulang request, pakai ID dan byte body yang sama. Konflik payload menghasilkan `409`.
- Gunakan target API yang mendukung idempotency.
- Mulai dengan flow kecil (2–3 steps) dan periksa hasilnya.
- Jangan memasukkan token, password, atau data pribadi sensitif ke log/payload contoh.
- Lihat [pemecahan masalah](../operations/troubleshooting.md) jika eksekusi tertahan.

[Kembali ke indeks dokumentasi](../README.md)
