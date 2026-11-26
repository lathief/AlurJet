# Panduan cepat menjalankan flow pertama

> Contoh ini adalah target acceptance test v0.1.0.

Selesaikan [instalasi lokal](installation.md) dan jalankan perintah berikut dari root proyek.

## 1. Import flow YAML

Flow contoh: [`docs/examples/order-to-api.yaml`](../examples/order-to-api.yaml).

```bash
export ALURJET_ADMIN_API_KEY='isi-sama-dengan-env'

curl -i -X POST http://localhost:8080/api/v1/flows   -H "Authorization: Bearer ${ALURJET_ADMIN_API_KEY}"   -H 'Content-Type: application/yaml'   --data-binary @docs/examples/order-to-api.yaml
```

Respons target: `201 Created` berisi `data.slug = "order-to-api"`.

## 2. Kirim webhook

```bash
export WEBHOOK_TOKEN='isi-token-order-to-api-dari-konfigurasi-server'

curl -i -X POST http://localhost:8080/api/v1/webhooks/order-to-api   -H "X-Webhook-Token: ${WEBHOOK_TOKEN}"   -H 'X-Event-ID: demo-order-001'   -H 'Content-Type: application/json'   --data-binary @docs/examples/order-event.json
```

Respons target `202 Accepted`:

```json
{
  "data": {
    "execution_id": "<generated-uuid>",
    "status": "pending"
  }
}
```

`202` berarti diterima dan sudah **disimpan**, bukan berarti outbound HTTP selesai.

## 3. Periksa execution

```bash
curl -H "Authorization: Bearer ${ALURJET_ADMIN_API_KEY}"   'http://localhost:8080/api/v1/executions?flow=order-to-api'

curl -H "Authorization: Bearer ${ALURJET_ADMIN_API_KEY}"   'http://localhost:8080/api/v1/executions/<execution_id>'
```

Status akan berubah dari `pending` menjadi `running`, lalu `succeeded`, `retry_scheduled`, atau `dead_letter`.

## 4. Coba error dan retry

Setelah Compose dan mock receiver tersedia, jalankan target dalam mode gagal sekali.

```bash
ALURJET_MOCK_MODE=fail_once docker compose up -d mock-api
```

Kirim ulang perintah webhook dengan **ID event baru** (misalnya `demo-order-002`). Untuk outbound key baru, mock mengembalikan `500` pada POST pertama dan `200` pada berikutnya. Periksa riwayat attempt dan status akhir `succeeded`.

Untuk membuat dead-letter, jalankan `ALURJET_MOCK_MODE=always_fail docker compose up -d mock-api`, lalu kirim event lain dengan ID baru (misalnya `demo-order-003`). Tunggu hingga jatah attempt habis. Sebelum replay, pulihkan target dengan `ALURJET_MOCK_MODE=success docker compose up -d mock-api`.

Isi `EXECUTION_ID` dengan ID execution `dead_letter`, lalu kirim request replay:

```bash
curl -i -X POST "http://localhost:8080/api/v1/executions/${EXECUTION_ID}/replay" \
  -H "Authorization: Bearer ${ALURJET_ADMIN_API_KEY}" \
  -H 'Idempotency-Key: demo-replay-001'
```

Respons `202` berisi ID baru dan `replayed_from_id`. Gunakan key request yang sama bila respons replay hilang. Execution replay meneruskan outbound key dari execution asal.

## 5. Stop environment

```bash
docker compose down
```

Gunakan `docker compose down -v` **hanya jika memang ingin menghapus volume database lokal**.

Lihat [pelacakan execution](../guides/tracking-executions.md) dan [konfigurasi flow](../guides/flow-configuration.md).

[Kembali ke indeks dokumentasi](../README.md)
