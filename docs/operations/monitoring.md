# Pemantauan dan log

Gunakan `execution_id` untuk menghubungkan respons webhook, detail execution, dan log API serta worker. Rancangan log terstruktur memakai field `event`, `execution_id`, `flow_slug`, `attempt`, `duration_ms`, dan `error_code`.

| Sinyal | Yang perlu diperiksa |
|---|---|
| Banyak execution lama pada `pending` | Worker berjalan, koneksi MySQL, dan hasil polling |
| Lonjakan `retry_scheduled` | Status HTTP target, timeout, serta riwayat attempt |
| Attempt `lease_expired` berulang | Restart worker, durasi pekerjaan, dan kegagalan recovery |
| `429` saat ingest/replay | Jumlah execution aktif per flow, umur antrean, serta kegagalan target |
| `/healthz` mengembalikan `503` | Koneksi dan kredensial MySQL, serta status kontainer database |
| Peningkatan `dead_letter` | Error permanen, konfigurasi outbound, atau jatah attempt habis |

Untuk deployment Compose yang direncanakan, gunakan `docker compose logs --tail=100 api worker`. Periksa hasil dan riwayat attempt melalui [API detail execution](../reference/api.md). Saat melaporkan masalah, sertakan ID dan timestamp serta redaksi kredensial dan data sensitif.

Lihat [pemecahan masalah](troubleshooting.md).

[Kembali ke indeks dokumentasi](../README.md)
