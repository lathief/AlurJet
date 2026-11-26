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
| `metadata.slug` | Ya | ID unik, 1–64 karakter, pola `^[a-z0-9]+(-[a-z0-9]+)*$`. Tidak dapat diubah setelah dibuat |
| `spec.trigger.type` | Ya | Hanya `webhook` |
| `spec.steps` | Ya | Array 1–10 step dengan ID unik. Tepat satu `http` di posisi terakhir |
| `spec.retry.maxAttempts` | Tidak | Integer `1`–`10`. Default `3`, termasuk percobaan pertama |
| `spec.retry.initialDelay` | Tidak | Durasi `1ms`–`60s`, default `2s`. Backoff eksponensial dibatasi `60s` |

## Step yang didukung

**`validate`**: `required` berisi nama field top-level JSON yang harus ada dan tidak `null`. String kosong, `0`, dan `false` dianggap ada. Jika tidak valid, attempt gagal permanen dengan `VALIDATION_FAILED` dan execution berakhir `dead_letter`. Karena validasi berjalan di worker, payload yang salah tetap menerima `202` saat ingest. Replay tidak membantu pada kasus ini karena payload dan snapshot sama, jadi kirim event baru dengan `X-Event-ID` baru setelah data diperbaiki.

**`transform`**: `fields` membentuk objek JSON baru dengan maksimum 64 field. Gunakan path `$.field` atau `$.nested.field`. Bila path tidak ditemukan, step gagal permanen dengan `MAPPING_PATH_NOT_FOUND` (tidak diam-diam mengisi `null`). Tidak mendukung array, literal konstan, ekspresi, atau JS di v0.1.0. Pemetaan mempertahankan token angka JSON, termasuk integer besar dan desimal. Misalnya `9007199254740993` tetap dikirim sebagai angka dengan nilai yang sama. Ukuran dibatasi oleh body masuk (`ALURJET_MAX_PAYLOAD_BYTES`) dan pemeriksaan body final (`ALURJET_MAX_OUTBOUND_BYTES`).

**`http`** mengirim JSON aktif ke `url` dengan `method: POST` dan `Content-Type: application/json`. Gunakan timeout positif dengan default `5s` dan maksimum sebesar `ALURJET_WORKER_ATTEMPT_TIMEOUT`. Field `headers` bersifat opsional (lihat bagian Header target). Respons `2xx` berhasil berdasarkan status. Header respons punya batas byte dan worker menutup body tanpa membacanya. HTTP `408`, `429`, `5xx`, timeout, dan kegagalan transport sementara dapat dicoba lagi. Redirect ditolak. Body JSON hasil serialisasi di atas `ALURJET_MAX_OUTBOUND_BYTES` menghasilkan `OUTBOUND_PAYLOAD_TOO_LARGE` dan `dead_letter` tanpa koneksi outbound. Setiap retry mengulang flow dari payload asli dan memakai outbound `Idempotency-Key` yang sama. Replay juga memakai outbound key asal.

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

## Header target

Step `http` dapat mengirim header tambahan, misalnya untuk autentikasi ke API target. Contoh lengkap: [order-to-api-auth.yaml](../examples/order-to-api-auth.yaml).

```yaml
headers:
  Authorization: "Bearer ${ALURJET_FLOW_TARGET_TOKEN}"
```

- Placeholder harus berawalan `ALURJET_FLOW_` dan variabelnya nonkosong saat create/update, jika tidak flow ditolak dengan `422 INVALID_FLOW`.
- Sistem menyimpan template, bukan nilai. Worker membaca nilai dari environment saat dispatch, sehingga secret tidak masuk database, snapshot, atau response API. API dan worker harus menerima variabel yang sama. Jika variabel hilang atau kosong saat dispatch, attempt gagal permanen dengan `HEADER_RESOLVE_FAILED`.
- Maksimum 10 header. Nama harus token HTTP valid dan nilai tidak boleh memuat CR atau LF.
- Header terlarang: `Host`, `Content-Length`, `Content-Type`, `Transfer-Encoding`, `Connection`, `Idempotency-Key`, dan `Proxy-*`.
- Nilai header tidak dicatat di log dan tidak ditampilkan di API.
- Rotasi secret memerlukan restart API dan worker, tanpa impor ulang flow.

## Aturan parsing YAML

- Body maksimum `ALURJET_MAX_FLOW_YAML_BYTES` (default 64 KiB). Lebih besar menghasilkan `413 PAYLOAD_TOO_LARGE`.
- Field yang tidak dikenal ditolak dengan `422 INVALID_FLOW`. Anchor, alias, dan dokumen YAML ganda ditolak.
- YAML yang tidak dapat diparse menghasilkan `400 INVALID_YAML`.

## Batasan yang disengaja

- Linear, tidak ada `if`, `switch`, `loop`, atau parallel steps.
- Hanya webhook trigger dan HTTP POST output.
- YAML yang bisa diparse tetapi melanggar DSL menghasilkan `422 INVALID_FLOW`. Slug yang sudah terpakai pada create menghasilkan `409 FLOW_SLUG_CONFLICT`.
- Execution menyimpan definisi terkompilasi saat ingest, termasuk URL teresolusi. Retry dan replay memakai snapshot itu. Kebijakan koneksi jaringan yang berlaku tetap diperiksa saat request keluar.

Contoh siap salin: [order-to-api.yaml](../examples/order-to-api.yaml).

[Kembali ke indeks dokumentasi](../README.md)
