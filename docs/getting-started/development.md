# Panduan pengembangan

> Panduan ini adalah target untuk repository yang sudah memiliki `go.mod`, `Makefile`, dan Compose. Perintah belum dijalankan terhadap runtime. Catat hasil verifikasi di [worklog](../ai/worklog.md).

## Persyaratan

| Alat | Versi |
|---|---|
| Go | 1.23 atau lebih baru |
| Docker dan Docker Compose | v2 |
| golangci-lint | versi yang dipin di `Makefile` |
| GNU Make | opsional, perintah setara tersedia di bawah |

## Perintah harian

| Target | Fungsi |
|---|---|
| `make fmt` | `gofmt` pada seluruh modul |
| `make lint` | `golangci-lint run` memakai `.golangci.yml` (`govet`, `errcheck`, `staticcheck`, `gosec`) |
| `make test` | `go test ./... -count=1` (tanpa database) |
| `make test-integration` | Menjalankan tes MySQL, lihat bagian berikut |
| `make up` / `make down` | `docker compose up --build -d` dan `docker compose down` |

## Tes integrasi

Tes integrasi memakai build tag `integration` dan Compose terpisah agar tidak menyentuh data demo.

```bash
docker compose -p alurjet-test -f docker-compose.test.yml up -d --wait
go test -tags=integration ./tests/integration/... -count=1
docker compose -p alurjet-test -f docker-compose.test.yml down -v
```

- `docker-compose.test.yml` menjalankan MySQL 8.4 sekali pakai pada port loopback yang dipetakan acak. Alamatnya dibaca oleh tes dari `ALURJET_TEST_MYSQL_DSN`, yang dicetak oleh `make test-integration`.
- Tes memakai migrasi aktual dari `migrations/`. Setiap paket tes membuat database sendiri.
- Hindari `sleep` untuk mengatur urutan. Gunakan barrier dan row lock seperti dijelaskan di [matriks verifikasi](../ai/verification.md).
- `go test -race` memerlukan cgo. Jika compiler C tidak tersedia, jalankan lewat kontainer: `docker run --rm -v "$PWD":/src -w /src golang:1.23 go test -race ./... -count=1`.

## Windows

Mesin pengembangan Windows dapat memakai salah satu cara berikut.

1. **Git Bash atau WSL (disarankan).** Contoh `curl` dan `export` di dokumentasi dapat disalin apa adanya.
2. **PowerShell.** Gunakan `curl.exe` (bukan alias `curl`), `$env:NAMA='nilai'` pengganti `export`, dan backtick `` ` `` pengganti `\` di akhir baris.
3. Simpan berkas `.sh`, `.yaml`, dan `.sql` dengan akhir baris LF. Atur `git config core.autocrlf input` agar migrasi dan skrip tidak rusak.

## Konvensi singkat

Aturan lengkap ada di [AGENTS.md](../../AGENTS.md). Ringkasnya:

- Kode, identifier, error code, dan log memakai bahasa Inggris. Dokumentasi memakai bahasa Indonesia.
- `context.Context` menjadi argumen pertama fungsi yang melakukan I/O. Bungkus error dengan `%w`. Jangan `panic` di jalur request.
- Tes bergaya table-driven. Tidak ada I/O jaringan nyata pada tes unit.
- Jangan mencatat token, nilai header, atau payload mentah.

## Proses runtime

- **Graceful shutdown:** SIGTERM menghentikan polling baru, menunggu attempt aktif sampai `ALURJET_SHUTDOWN_TIMEOUT`, lalu keluar. Attempt yang tidak selesai dipulihkan oleh recovery lease.
- **Multi-instance:** beberapa API dan worker aman dijalankan bersamaan karena claim memakai `SKIP LOCKED`. Admin key, token webhook, dan variabel `ALURJET_FLOW_*` dibaca saat startup, jadi perubahan memerlukan restart. API dan worker harus menerima nilai `ALURJET_FLOW_*` yang sama.
- **Health:** `GET /healthz` memeriksa database. Worker tidak memiliki healthcheck pada MVP. Periksa `docker compose logs worker` dan execution yang tertahan di `pending`.

[Kembali ke indeks dokumentasi](../README.md)
