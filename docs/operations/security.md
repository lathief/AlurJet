# Keamanan operasional

- Gunakan satu rahasia admin dan rahasia webhook berbeda untuk tiap slug. Simpan peta token di proses API. YAML flow dan snapshot hanya membawa konfigurasi flow. Hindari mencatat nilai token, DSN, payload mentah, dan URL yang memuat data sensitif.
- Gunakan HTTPS di luar demo lokal. Batasi akses ke Swagger UI pengembangan dan port database.
- Izinkan hanya origin tujuan yang diperlukan. Untuk target biasa, gunakan HTTPS dan kosongkan `ALURJET_DEV_PRIVATE_ORIGIN`.
- Pengecualian satu origin privat berlaku untuk demo Compose yang dikonfigurasi. Pastikan origin demo yang sama masuk allowlist.
- Desain connector memeriksa alamat IP saat membuka koneksi, menolak redirect, dan mengecek ulang kebijakan aktif untuk retry serta replay. Setelah perubahan kebijakan, periksa execution lama yang masih merujuk snapshot sebelumnya.

- Header target (misalnya `Authorization`) dibaca dari environment worker saat dispatch dan tidak disimpan di database, snapshot, atau response API. Jangan tulis secret langsung di YAML. Gunakan placeholder `${ALURJET_FLOW_*}`.
- MVP belum membatasi percobaan autentikasi yang gagal dan webhook belum memakai HMAC atau timestamp. Pasang rate limit di reverse proxy dan gunakan secret acak minimal 32 karakter. Rotasi admin key dan token dilakukan dengan mengganti nilai lalu merestart proses.
- Payload disimpan tanpa enkripsi di MySQL. Batasi akses ke database dan backup, dan hindari mengirim data sensitif sebelum retensi tersedia.

Lihat [autentikasi](../guides/authentication.md), [referensi konfigurasi](../reference/configuration.md), dan [kebijakan outbound pada desain sistem](../../SYSTEM_DESIGN.md).

[Kembali ke indeks dokumentasi](../README.md)
