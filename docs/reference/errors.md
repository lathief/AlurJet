# Error HTTP dan retry

> Referensi ini mendokumentasikan kontrak yang direncanakan.

## Format respons error

```json
{"error":{"code":"INVALID_FLOW","message":"Unknown step type"}}
```

| HTTP | Contoh alasan |
|---|---|
| 400 | JSON/YAML tidak bisa di-parse, field JSON berulang (`INVALID_JSON`), atau header wajib hilang |
| 401 | Admin token atau webhook token salah |
| 404 | Flow/execution tidak ditemukan |
| 409 | `FLOW_SLUG_CONFLICT`, `EVENT_ID_CONFLICT`, atau execution belum berstatus `dead_letter` |
| 422 | Flow valid secara YAML tapi tidak sesuai aturan DSL |
| 413 | Payload melebihi batas server |
| 415 | Content type tidak didukung |
| 429 | `FLOW_CAPACITY_EXCEEDED` untuk execution baru pada flow yang mencapai kuota |
| 500 | Kesalahan server yang tidak terduga |
| 503 | `SYSTEM_BACKPRESSURE` untuk execution baru ketika ambang global tercapai |

## Mengulang request dari client

- Jika pengiriman webhook timeout atau respons hilang, ulangi ID dan byte body yang sama. Untuk request replay, gunakan `Idempotency-Key` yang sama.
- Jika kapasitas menolak request baru, hormati `Retry-After` dan gunakan ID/body atau key replay yang sama saat mengulang.
- Jika menerima `409 EVENT_ID_CONFLICT`, periksa ID dan byte body sebelum mengirim ulang.
- Setelah `202 Accepted`, periksa status execution. Retry ke target dikelola oleh worker sesuai kebijakan flow.

## Error pada attempt

`OUTBOUND_PAYLOAD_TOO_LARGE` adalah kegagalan permanen saat hasil salah satu transform atau body JSON siap kirim melebihi `ALURJET_MAX_OUTBOUND_BYTES`. Periksa hasilnya melalui detail execution. Worker menyimpan status `dead_letter` tanpa mencoba HTTP ke target.

Lihat [API HTTP](api.md) dan [pelacakan execution](../guides/tracking-executions.md).

[Kembali ke indeks dokumentasi](../README.md)
