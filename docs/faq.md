# Pertanyaan umum

## Apakah panduan ini sudah bisa dijalankan?

Dokumen ini menjelaskan rancangan v0.1.0. Perintah akan dapat diuji setelah server Go, migrasi, worker, dan Docker Compose tersedia. Lihat [roadmap](../ROADMAP.md).

## Apa arti `202 Accepted`?

MySQL sudah menyimpan execution. Untuk mengetahui apakah POST ke target berhasil, ambil detail execution dengan ID dari respons. Lihat [pelacakan execution](guides/tracking-executions.md).

## Mengapa ID event yang sama menghasilkan `409`?

AlurJet membandingkan byte body untuk event ID yang sama dalam satu flow. Kirim ulang body asli jika sedang mengulang request lama. Pakai ID baru untuk event logis baru. Lihat [mengirim webhook](guides/sending-webhooks.md).

## Apa yang terjadi jika flow diubah sebelum replay?

Replay menggunakan payload, snapshot flow, dan outbound idempotency key dari execution asal. Perubahan flow berlaku untuk event baru. Lihat [API replay](reference/api.md#replay).

## Apakah target bisa menerima POST lebih dari sekali?

Ya. Timeout atau kegagalan worker dapat memicu pengiriman ulang. Target perlu menghormati outbound `Idempotency-Key` selama periode retry dan replay. Lihat [pelacakan execution](guides/tracking-executions.md).

[Kembali ke indeks dokumentasi](README.md)
