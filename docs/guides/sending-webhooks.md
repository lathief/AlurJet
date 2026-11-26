# Mengirim webhook

Kirim satu objek JSON ke `POST /api/v1/webhooks/{slug}` dengan `Content-Type: application/json`, token webhook khusus slug tujuan, dan `X-Event-ID`. Pilih event ID sepanjang 1–128 karakter ASCII tercetak tanpa spasi. Perbandingan ID peka huruf besar dan kecil dalam satu flow. Nama field harus unik dalam setiap objek, termasuk objek bersarang. Jika nama field berulang setelah escape JSON didecode, request menghasilkan `400 INVALID_JSON` tanpa menyimpan execution.

| Kondisi | Perilaku yang direncanakan |
|---|---|
| ID baru dan request valid | `202` setelah MySQL menyimpan execution |
| ID dan byte body sama | `202` dengan execution ID yang sama |
| ID sama, byte body berbeda | `409 EVENT_ID_CONFLICT` |
| Event baru dan kapasitas penuh | `429 FLOW_CAPACITY_EXCEEDED` dengan `Retry-After: 5` |

Jika respons hilang, kirim ulang ID dan byte body yang sama. Perubahan whitespace juga mengubah byte body. Gunakan ID baru untuk event logis baru.

Respons `202` menandakan penyimpanan berhasil. Validasi field bisnis, transformasi, dan pengiriman ke target terjadi di worker. Simpan `execution_id` dari respons untuk memeriksa hasilnya.

Lihat [panduan cepat](../getting-started/quickstart.md), [referensi API](../reference/api.md), dan [pelacakan execution](tracking-executions.md).

[Kembali ke indeks dokumentasi](../README.md)
