# Instalasi dan persyaratan

> Panduan ini adalah target untuk v0.1.0. Jalankan prosedur setelah aplikasi, migrasi, dan Docker Compose tersedia.

## 1. Persyaratan

- Docker + Docker Compose.
- Terminal dengan `curl`.
- Go hanya diperlukan jika menjalankan atau mengembangkan service tanpa Docker.

## 2. Setup

```bash
git clone https://github.com/lathief/AlurJet.git
cd AlurJet
cp .env.example .env
```

Edit `.env` dan ganti **`ALURJET_ADMIN_API_KEY`**, token `order-to-api` dalam **`ALURJET_WEBHOOK_TOKENS_JSON`**, **`ALURJET_MYSQL_PASSWORD`**, serta **`ALURJET_MYSQL_ROOT_PASSWORD`** sebelum startup pertama. Compose memakai `ALURJET_MYSQL_DATABASE`, `ALURJET_MYSQL_USER`, dan `ALURJET_MYSQL_PASSWORD` yang sama untuk menyiapkan database dan memberi akses ke migrator, API, dan worker. Password root diberikan hanya ke service MySQL. Compose demo akan menyertakan `mock-api` dengan `POST /orders`. Pertahankan origin demo dari `.env.example` untuk `ALURJET_FLOW_TARGET_URL`, `ALURJET_ALLOWED_ORIGINS`, dan `ALURJET_DEV_PRIVATE_ORIGIN`. Admin key dan setiap token webhook harus acak dan minimal 32 karakter. `ALURJET_FLOW_TARGET_TOKEN` hanya diperlukan untuk flow yang memakai `headers`. Port API dan mock receiver dipublikasikan hanya ke `127.0.0.1`. Pengembang yang menjalankan tes dapat melanjutkan ke [panduan pengembangan](development.md).

Setelah Docker Compose tersedia, jalankan perintah di bawah. MySQL harus sehat, lalu service `migrate` harus keluar sukses sebelum API dan worker dimulai.

```bash
docker compose up --build -d
curl --fail http://localhost:8080/healthz
```

Respons yang diharapkan adalah HTTP `200`, misalnya dengan body `{"status":"ok"}`. Jika proses tertahan, periksa `docker compose ps -a` dan `docker compose logs migrate`. Error migrasi harus terlihat dan API/worker tetap belum berjalan. Perubahan kredensial pada `.env` setelah volume dibuat harus diselaraskan dengan user yang sudah ada di MySQL.

Lanjutkan ke [panduan cepat](quickstart.md) setelah layanan API dapat diakses.

[Kembali ke indeks dokumentasi](../README.md)
