# Referensi API HTTP

> **Proposed contract / belum live.** Ketika endpoint sudah diimplementasikan, dokumentasi generated Swaggo adalah referensi status terbaru. Gunakan contoh ini sebagai target API tests.

**Base URL (local):** `http://localhost:8080`  
**Management auth:** `Authorization: Bearer <ALURJET_ADMIN_API_KEY>`  
**Webhook auth:** `X-Webhook-Token: <token untuk slug tujuan>` dan `X-Event-ID: <unique-event-id>`

## Daftar endpoint

| Method | Endpoint | Auth | Status sukses |
|---|---|---|---|
| GET | `/healthz` | Tidak | 200 |
| POST | `/api/v1/flows` | Admin | 201 |
| GET | `/api/v1/flows` | Admin | 200 |
| GET | `/api/v1/flows/{slug}` | Admin | 200 |
| PUT | `/api/v1/flows/{slug}` | Admin | 200 |
| POST | `/api/v1/webhooks/{slug}` | Webhook | 202 |
| GET | `/api/v1/executions?flow={slug}&status={status}` | Admin | 200 |
| GET | `/api/v1/executions/{id}` | Admin | 200 |
| POST | `/api/v1/executions/{id}/replay` | Admin | 202 |

## Create / update flow

`POST /api/v1/flows` atau `PUT /api/v1/flows/{slug}` menggunakan `Content-Type: application/yaml` dan raw YAML body.

```bash
curl -X POST http://localhost:8080/api/v1/flows   -H "Authorization: Bearer ${ALURJET_ADMIN_API_KEY}"   -H 'Content-Type: application/yaml'   --data-binary @docs/examples/order-to-api.yaml
```

Contoh respons `201`:

```json
{"data":{"id":1,"slug":"order-to-api"}}
```

`PUT` memperbarui konfigurasi untuk event baru. Jika slug di body dan URL berbeda, balas `422 INVALID_FLOW`. `POST` dengan slug yang sudah ada mengembalikan `409 FLOW_SLUG_CONFLICT`. `PUT` untuk slug yang belum ada mengembalikan `404`.

## Webhook ingest

Isi `WEBHOOK_TOKEN` dengan token untuk slug `order-to-api`. Slug tanpa binding dan token yang salah menghasilkan `401` yang sama. Slug yang sudah terikat token valid tetapi belum diimpor menghasilkan `404`.

```bash
curl -X POST http://localhost:8080/api/v1/webhooks/order-to-api   -H "X-Webhook-Token: ${WEBHOOK_TOKEN}"   -H 'X-Event-ID: order-001'   -H 'Content-Type: application/json'   --data-binary @docs/examples/order-event.json
```

Respons `202`:

```json
{"data":{"execution_id":"<uuid>","status":"pending"}}
```

`X-Event-ID` wajib, sepanjang 1–128 karakter ASCII tercetak tanpa spasi, dan dibandingkan secara case-sensitive dalam satu flow. ID dan byte body yang sama mengembalikan `202` dengan execution ID lama dan status saat dibaca. ID sama dengan byte body berbeda, termasuk perubahan whitespace, mengembalikan `409 EVENT_ID_CONFLICT`. Body webhook harus berisi satu objek JSON UTF-8. Nama field berulang pada objek mana pun, termasuk objek bersarang atau di dalam array, menghasilkan `400 INVALID_JSON` sebelum deduplikasi dan penyimpanan execution. Nama dibandingkan setelah escape JSON didecode, secara case-sensitive. Validasi field bisnis terjadi secara asinkron setelah ingest.

Webhook baru dapat menerima `429 FLOW_CAPACITY_EXCEEDED` atau `503 SYSTEM_BACKPRESSURE` dengan `Retry-After`. Deduplikasi untuk event yang sudah commit tetap berlaku saat kapasitas penuh.

## Execution list / detail

- List: respons `data` array execution, paginasi sederhana `page` dan `limit` (default 20, maksimal 100).
- Detail: `data` berisi `execution_id`, `flow_slug`, `status`, `attempts`, `last_error`, `created_at`, `updated_at`, dan `attempt_history`.
- Jangan kembalikan payload lengkap atau token dalam response list/detail pada MVP.

Contoh detail:

```json
{
  "data": {
    "execution_id": "<uuid>",
    "flow_slug": "order-to-api",
    "status": "retry_scheduled",
    "attempts": 1,
    "last_error": "target returned 503",
    "attempt_history": [
      {"attempt": 1, "status": "failed", "http_status": 503}
    ]
  }
}
```

## Replay

Kirim `POST /api/v1/executions/{id}/replay` tanpa body. Header `Idempotency-Key` wajib, sepanjang 1–128 karakter ASCII tercetak tanpa spasi dan dibandingkan secara case-sensitive. Sumber replay harus berstatus `dead_letter`.

```json
{"data":{"execution_id":"<uuid-replay>","replayed_from_id":"<uuid-asal>","status":"pending"}}
```

Request ulang dengan key yang sama untuk execution asal yang sama mengembalikan `202` dengan ID replay yang sama dan status saat dibaca. Key baru membuat execution bila kapasitas flow asal dan ambang global mengizinkan. Replay menyalin payload dan snapshot lama serta mempertahankan outbound idempotency key asal. Outbound key berbeda dari header yang mendeduplikasi request replay. Target yang tidak mendukung atau sudah melupakan outbound key masih dapat mengulang efek bisnis.

Lihat [error HTTP](errors.md) serta [panduan cepat](../getting-started/quickstart.md). Swagger UI direncanakan di `/swagger/index.html` pada mode pengembangan.

[Kembali ke indeks dokumentasi](../README.md)
