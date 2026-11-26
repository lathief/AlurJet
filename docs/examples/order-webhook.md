# Contoh integrasi webhook pesanan

Contoh ini memakai [flow pesanan](order-to-api.yaml) dan [event JSON](order-event.json) di folder `docs/examples/`. Ikuti [panduan cepat](../getting-started/quickstart.md) untuk mengimpor flow dan mengirim event.

Flow `order-to-api` mewajibkan `order_id` dan `total`, lalu mengubah nama field sebelum mengirimnya.

| Field masuk | Field keluar | Contoh nilai |
|---|---|---|
| `order_id` | `orderNumber` | `"ORD-001"` |
| `total` | `amount` | `150000` |

Target menerima POST ke `/orders` dengan outbound `Idempotency-Key` yang sama sepanjang retry dan replay. Untuk demo lokal, target yang direncanakan adalah mock receiver Compose. Periksa status HTTP target pada riwayat attempt dan status akhir AlurJet melalui [detail execution](../reference/api.md).

Lihat [konfigurasi flow YAML](../guides/flow-configuration.md) untuk aturan pemetaan.

[Kembali ke indeks dokumentasi](../README.md)
