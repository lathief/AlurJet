# Keamanan operasional

- Gunakan satu rahasia admin dan rahasia webhook berbeda untuk tiap slug. Simpan peta token di proses API. YAML flow dan snapshot hanya membawa konfigurasi flow. Hindari mencatat nilai token, DSN, payload mentah, dan URL yang memuat data sensitif.
- Gunakan HTTPS di luar demo lokal. Batasi akses ke Swagger UI pengembangan dan port database.
- Izinkan hanya origin tujuan yang diperlukan. Untuk target biasa, gunakan HTTPS dan kosongkan `ALURJET_DEV_PRIVATE_ORIGIN`.
- Pengecualian satu origin privat berlaku untuk demo Compose yang dikonfigurasi. Pastikan origin demo yang sama masuk allowlist.
- Desain connector memeriksa alamat IP saat membuka koneksi, menolak redirect, dan mengecek ulang kebijakan aktif untuk retry serta replay. Setelah perubahan kebijakan, periksa execution lama yang masih merujuk snapshot sebelumnya.

Lihat [autentikasi](../guides/authentication.md), [referensi konfigurasi](../reference/configuration.md), dan [kebijakan outbound pada desain sistem](../../SYSTEM_DESIGN.md).

[Kembali ke indeks dokumentasi](../README.md)
