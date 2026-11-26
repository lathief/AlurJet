# Konfigurasi flow YAML

> **Target DSL v0.1.0** — spesifikasi milik AlurJet, bukan standar umum. Contoh belum menunjukkan hasil eksekusi aplikasi.

## Contoh lengkap

```yaml
apiVersion: alurjet/v1
kind: Flow
metadata:
  name: Order to API
  slug: order-to-api
spec:
  trigger:
    type: webhook
  steps:
    - id: validate
      type: validate
      required: [order_id, total]
    - id: mapping
      type: transform
      fields:
        orderNumber: "$.order_id"
        amount: "$.total"
    - id: forward
      type: http
      method: POST
      url: "${ALURJET_FLOW_TARGET_URL}/orders"
      timeout: 5s
  retry:
    maxAttempts: 3
    initialDelay: 2s
```

### Root properties

| Field | Wajib | Penjelasan |
|---|---|---|
| `apiVersion` | Ya | Selalu `alurjet/v1` untuk MVP |
| `kind` | Ya | Selalu `Flow` |
| `metadata.name` | Ya | Nama yang ditampilkan |
| `metadata.slug` | Ya | ID manusiawi unik (huruf kecil/angka/tanda minus) |
| `spec.trigger.type` | Ya | Hanya `webhook` |
| `spec.steps` | Ya | Array 1–10 step dengan ID unik. Tepat satu `http` di posisi terakhir |
| `spec.retry.maxAttempts` | Tidak | Integer `1`–`10`. Default `3`, termasuk percobaan pertama |
| `spec.retry.initialDelay` | Tidak | Durasi `1ms`–`60s`, default `2s`. Backoff eksponensial dibatasi `60s` |

## Step yang didukung

**`validate`**: `required` berisi nama field top-level JSON yang harus ada dan tidak `null`. Jika tidak valid, execution berakhir sebagai `dead_letter`.

**`transform`**: `fields` membentuk objek JSON baru. Gunakan path `$.field` atau `$.nested.field`. Bila path tidak ditemukan, step gagal (tidak diam-diam mengisi `null`). Tidak mendukung array/ekspresi/JS di v0.1.0. Tiap step menolak output yang melampaui `ALURJET_MAX_OUTBOUND_BYTES` sebelum menyalin fragmen yang akan melewati batas. Pemetaan mempertahankan token angka JSON, termasuk integer besar dan desimal. Misalnya `9007199254740993` tetap dikirim sebagai angka dengan nilai yang sama.

**`http`** mengirim JSON aktif ke `url` dengan `method: POST`. Gunakan timeout positif dengan default `5s` dan maksimum sebesar `ALURJET_WORKER_ATTEMPT_TIMEOUT`. Respons `2xx` berhasil berdasarkan status. Header respons punya batas byte dan worker menutup body tanpa membacanya. HTTP `408`, `429`, `5xx`, timeout, dan kegagalan transport sementara dapat dicoba lagi. Redirect ditolak. Body JSON hasil serialisasi di atas `ALURJET_MAX_OUTBOUND_BYTES` menghasilkan `OUTBOUND_PAYLOAD_TOO_LARGE` dan `dead_letter` tanpa koneksi outbound. Setiap retry mengulang flow dari payload asli dan memakai outbound `Idempotency-Key` yang sama. Replay juga memakai outbound key asal.

## Contoh input dan output

Input event:

```json
{"order_id":"ORD-001","total":150000}
```

Setelah transform:

```json
{"orderNumber":"ORD-001","amount":150000}
```

## Environment variable di URL

Server me-resolve placeholder `${ALURJET_FLOW_TARGET_URL}` saat create/update. Hanya nama berawalan `ALURJET_FLOW_` dengan nilai nonkosong yang diterima. URL akhir harus cocok dengan origin pada `ALURJET_ALLOWED_ORIGINS`, termasuk skema dan port. HTTPS berlaku untuk target biasa, dengan pengecualian origin demo di konfigurasi Compose. Impor ulang flow setelah mengubah variabel. Jangan menaruh secret pada URL atau mencetaknya di log.

## Batasan yang disengaja

- Linear, tidak ada `if`, `switch`, `loop`, atau parallel steps.
- Hanya webhook trigger dan HTTP POST output.
- YAML yang bisa diparse tetapi melanggar DSL menghasilkan `422 INVALID_FLOW`. Slug yang sudah terpakai pada create menghasilkan `409 FLOW_SLUG_CONFLICT`.
- Execution menyimpan definisi terkompilasi saat ingest, termasuk URL teresolusi. Retry dan replay memakai snapshot itu. Kebijakan koneksi jaringan yang berlaku tetap diperiksa saat request keluar.

Contoh siap salin: [order-to-api.yaml](../examples/order-to-api.yaml).

[Kembali ke indeks dokumentasi](../README.md)
