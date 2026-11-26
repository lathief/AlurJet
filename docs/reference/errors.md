# Error HTTP dan retry

> Referensi ini mendokumentasikan kontrak yang direncanakan.

## Format respons error

```json
{"error":{"code":"INVALID_FLOW","message":"Unknown step type"}}
```

`message` tidak memuat credential, nilai header, payload, atau stack trace.

## Kode error HTTP

| HTTP | Kode | Kapan terjadi |
|---|---|---|
| 400 | `INVALID_JSON` | Body webhook bukan satu objek JSON UTF-8, atau nama field berulang pada objek mana pun |
| 400 | `INVALID_YAML` | Body flow tidak dapat diparse sebagai YAML |
| 400 | `INVALID_EVENT_ID` | `X-Event-ID` hilang atau bukan 1–128 karakter ASCII tercetak tanpa spasi |
| 400 | `INVALID_IDEMPOTENCY_KEY` | `Idempotency-Key` replay hilang atau formatnya sama seperti aturan event ID |
| 401 | `UNAUTHORIZED` | Admin key atau webhook token salah, atau slug webhook tanpa binding token. Pesan sama untuk semua sebab |
| 404 | `FLOW_NOT_FOUND` | Slug flow tidak ada (setelah token webhook valid, atau pada `GET`/`PUT` flow) |
| 404 | `EXECUTION_NOT_FOUND` | ID execution tidak ada |
| 409 | `FLOW_SLUG_CONFLICT` | `POST` flow dengan slug yang sudah ada |
| 409 | `EVENT_ID_CONFLICT` | Event ID sama dengan byte body berbeda pada flow yang sama |
| 409 | `REPLAY_NOT_ALLOWED` | Execution sumber replay belum berstatus `dead_letter` |
| 413 | `PAYLOAD_TOO_LARGE` | Body melebihi `ALURJET_MAX_PAYLOAD_BYTES` (webhook) atau `ALURJET_MAX_FLOW_YAML_BYTES` (flow) |
| 415 | `UNSUPPORTED_MEDIA_TYPE` | `Content-Type` bukan `application/json` (webhook) atau `application/yaml` (flow) |
| 422 | `INVALID_FLOW` | YAML valid tetapi melanggar DSL, atau slug di body berbeda dari URL pada `PUT` |
| 429 | `FLOW_CAPACITY_EXCEEDED` | Kuota execution aktif flow tercapai. Respons memuat `Retry-After: 5` |
| 500 | `INTERNAL_ERROR` | Kesalahan server yang tidak terduga |
| 503 | `INTERNAL_ERROR` | Hanya pada `GET /healthz` saat database tidak dapat dijangkau |

Ambang global antrean dan `503 SYSTEM_BACKPRESSURE` ditunda ke [backlog pasca-MVP](../post-mvp.md).

## Urutan validasi webhook

1. Token webhook terhadap slug (`401`).
2. `Content-Type` (`415`).
3. Ukuran body (`413`).
4. `X-Event-ID` (`400 INVALID_EVENT_ID`).
5. Struktur JSON dan nama field berulang (`400 INVALID_JSON`).
6. Flow ada (`404`).
7. Deduplikasi (`202` atau `409`).
8. Kuota per flow (`429`).

## Mengulang request dari client

- Jika pengiriman webhook timeout atau respons hilang, ulangi ID dan byte body yang sama. Untuk request replay, gunakan `Idempotency-Key` yang sama.
- Jika kapasitas menolak request baru, hormati `Retry-After` dan gunakan ID/body atau key replay yang sama saat mengulang.
- Jika menerima `409 EVENT_ID_CONFLICT`, periksa ID dan byte body sebelum mengirim ulang.
- Setelah `202 Accepted`, periksa status execution. Retry ke target dikelola oleh worker sesuai kebijakan flow.

## Kode error pada attempt

Kode ini muncul di `attempt_history[].error_code`, `last_error`, dan log. Kolom "Klasifikasi" menentukan apakah attempt dicoba lagi.

| Kode | Klasifikasi | Arti |
|---|---|---|
| `VALIDATION_FAILED` | Permanen | Field `required` tidak ada atau bernilai `null` |
| `MAPPING_PATH_NOT_FOUND` | Permanen | Path `$.…` pada `transform` tidak ditemukan |
| `OUTBOUND_PAYLOAD_TOO_LARGE` | Permanen | Body JSON final melebihi `ALURJET_MAX_OUTBOUND_BYTES`. Tanpa DNS atau koneksi |
| `HEADER_RESOLVE_FAILED` | Permanen | Variabel `ALURJET_FLOW_*` pada `headers` hilang atau kosong saat dispatch |
| `ORIGIN_NOT_ALLOWED` | Permanen | Origin target tidak ada di `ALURJET_ALLOWED_ORIGINS` pada kebijakan aktif |
| `IP_NOT_ALLOWED` | Permanen | IP hasil resolusi ditolak, atau DNS mengembalikan kombinasi diizinkan dan dilarang |
| `REDIRECT_REJECTED` | Permanen | Target membalas redirect. Redirect tidak diikuti |
| `TLS_ERROR` | Permanen | Verifikasi TLS gagal |
| `TARGET_STATUS_REJECTED` | Permanen | Status HTTP di luar `2xx`, `408`, `429`, dan `5xx` |
| `TARGET_STATUS_RETRYABLE` | Retryable | Status `408`, `429`, atau `5xx` |
| `TARGET_TIMEOUT` | Retryable | Timeout step atau anggaran attempt habis |
| `TRANSPORT_ERROR` | Retryable | Kegagalan koneksi sementara. Termasuk header respons melebihi batas bila diklasifikasikan sebagai transport |
| `LEASE_EXPIRED` | Retryable | Worker berhenti sebelum finalisasi. Recovery menutup attempt dengan status `lease_expired` |

Retryable dengan jatah habis berakhir sebagai `dead_letter`. Error transport yang tidak dapat diklasifikasikan sebagai sementara bersifat permanen. Status `dead_letter` tanpa `http_status` berarti attempt gagal sebelum respons target diterima.

Lihat [API HTTP](api.md) dan [pelacakan execution](../guides/tracking-executions.md).

[Kembali ke indeks dokumentasi](../README.md)
