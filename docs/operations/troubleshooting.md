# Pemecahan masalah

> Panduan diagnosis target MVP. Sesuaikan command log/service dengan nama service di `docker-compose.yml` aktual saat implementasi.

## Diagnosa cepat

```bash
# Cek HTTP API
curl -i http://localhost:8080/healthz

# Cek container dan logs (jika memakai Docker Compose)
docker compose ps
docker compose logs --tail=100 api
docker compose logs --tail=100 worker
```

| Masalah | Penyebab yang mungkin | Tindakan |
|---|---|---|
| `401 Unauthorized` saat API flow | Admin key tidak cocok | Cocokkan `Authorization: Bearer ...` dengan `ALURJET_ADMIN_API_KEY` |
| `401 Unauthorized` saat webhook | Token webhook salah | Periksa `X-Webhook-Token` dan entri slug tujuan pada `ALURJET_WEBHOOK_TOKENS_JSON` |
| `400 Bad Request` | JSON invalid atau `X-Event-ID` tidak ada | Validasi JSON serta header |
| `409 EVENT_ID_CONFLICT` | ID pernah dipakai dengan byte body berbeda | Ulangi body asli atau gunakan ID baru untuk event logis baru |
| `422 INVALID_FLOW` | Step/field YAML tidak didukung | Lihat [konfigurasi flow](../guides/flow-configuration.md) |
| API atau worker belum mulai setelah Compose boot | Migrasi gagal atau MySQL belum sehat | Periksa `docker compose ps -a` dan `docker compose logs migrate`. Cocokkan database, user, serta password pada server dan `.env` |
| `OUTBOUND_PAYLOAD_TOO_LARGE` pada attempt | Output transform atau body JSON final melebihi `ALURJET_MAX_OUTBOUND_BYTES` | Ringkas field yang dipetakan. Execution `dead_letter` tanpa request ke target |
| Webhook `202` tapi status terus `pending` | Worker mati atau koneksi MySQL gagal | `docker compose ps` dan log worker |
| Status `retry_scheduled` | Target timeout, 408, 429, 5xx, atau lease attempt habis | Cek target API, URL, DNS, dan log dengan `execution_id` |
| Status `dead_letter` | Validation error atau retry habis | Periksa riwayat attempt. Replay setelah target pulih, atau kirim event baru jika perlu memperbaiki data/flow |
| Flow benar tapi target tidak dihubungi | Host tidak diizinkan / resolusi URL salah | Cek origin snapshot, `ALURJET_ALLOWED_ORIGINS`, dan pengecualian `ALURJET_DEV_PRIVATE_ORIGIN` di profil demo |
| Event dikirim ulang tetapi tidak ada execution baru | `X-Event-ID` sama untuk flow yang sama | Gunakan ID baru hanya untuk event baru, bukan retry event lama |
| Worker crash saat berjalan | Lease belum expired | Periksa recovery. Attempt lama harus menjadi `lease_expired`; jika itu attempt terakhir, execution harus menjadi `dead_letter` |

## Memahami error tanpa membocorkan rahasia

Gunakan `execution_id` untuk mencari log dari request inbound hingga pengiriman outbound. Field penting: `event`, `flow_slug`, `attempt`, `duration_ms`, `error_code`. Jangan tempelkan raw API key, webhook token, header Authorization, atau payload personal di issue publik.

## Jika belum terselesaikan

Saat melaporkan issue, sertakan versi AlurJet, cara menjalankan (Docker/local), YAML minimal tanpa secret, langkah reproduksi, expected/actual result, dan log yang sudah disanitasi.

[Kembali ke indeks dokumentasi](../README.md)
