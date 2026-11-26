# Autentikasi

Operator mengelola satu admin key dan satu token berbeda untuk setiap flow yang menerima webhook.

| Akses | Header | Variabel server |
|---|---|---|
| Membuat dan membaca flow, memeriksa execution, serta replay | `Authorization: Bearer <token>` | `ALURJET_ADMIN_API_KEY` |
| Mengirim webhook | `X-Webhook-Token: <token>` | `ALURJET_WEBHOOK_TOKENS_JSON` |

Webhook juga mewajibkan `X-Event-ID` untuk deduplikasi. API mencocokkan header webhook dengan nilai untuk slug di URL. Flow tanpa binding atau token salah mengembalikan `401` yang sama. Endpoint `/healthz` tidak memerlukan token.

Simpan admin key dan token tiap flow sebagai rahasia berbeda dan gunakan HTTPS di luar demo lokal. Jika menerima `401`, cocokkan jenis header, nilai pada sumber rahasia, dan konfigurasi proses API. Saat mengganti token melalui konfigurasi server, koordinasikan pembaruan dengan pengirim yang menggunakannya.

Lihat [API HTTP](../reference/api.md), [mengirim webhook](sending-webhooks.md), dan [keamanan operasional](../operations/security.md).

[Kembali ke indeks dokumentasi](../README.md)
