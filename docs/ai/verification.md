# Verifikasi implementasi AlurJet

> Perintah berikut adalah target implementasi. Jalankan setelah Go module, package, suite integrasi, dan Compose tersedia. Catat perintah yang benar-benar dijalankan di [worklog](worklog.md).

## Jalur verifikasi

| Perubahan | Pemeriksaan minimum |
|---|---|
| Go lokal | `gofmt` pada berkas yang berubah, `go vet ./...`, `go test ./... -count=1`, dan `go build ./cmd/alurjet` |
| Lint | `golangci-lint run` memakai `.golangci.yml` |
| Transaksi MySQL | `docker compose -p alurjet-test -f docker-compose.test.yml up -d --wait`, lalu `go test -tags=integration ./tests/integration/... -count=1` memakai MySQL terisolasi dan migrasi aktual. Lihat [panduan pengembangan](../getting-started/development.md#tes-integrasi). Selaraskan dengan CI saat suite tersedia |
| Worker dan concurrency | `go test -race ./... -count=1`. Pada Windows tanpa cgo, jalankan lewat kontainer `golang:1.23`. Gunakan tes MySQL dengan row lock dan waktu database untuk race expiry |
| Perubahan HTTP dan rilis | Selaraskan dan periksa Swagger; dari Compose bersih jalankan [panduan cepat](../getting-started/quickstart.md) dengan mock receiver dan konfirmasi riwayat execution |

Gate tiap milestone mengikuti [roadmap](../../ROADMAP.md).

## Matriks kegagalan

| Kasus | Cara memicu | Bukti lulus |
|---|---|---|
| Startup Compose dan migrasi | Boot dengan volume kosong dan konfigurasi MySQL bersama, lalu ulangi. Pada project test terisolasi, paksa migrasi gagal | Migrasi berhasil sebelum API/worker mulai, startup ulang menghasilkan no-op, dan error migrasi terlihat serta memblokir startup |
| Token per flow | Kirim token flow A ke flow B atau slug tanpa binding. Ulangi dengan token benar pada A | Request lintas flow memberi `401` tanpa insert. Token A tetap menerima webhook A |
| Batas field transform | Impor flow dengan 65 field pada satu `transform` | `422 INVALID_FLOW`. Flow dengan 64 field diterima |
| Batas byte outbound | Gunakan body dengan representasi UTF-8 tepat pada batas, lalu hasil transform satu byte lebih besar. Uji juga flow tanpa transform | Body tepat pada batas dikirim. Body di atas batas menjadi `dead_letter` dengan `OUTBOUND_PAYLOAD_TOO_LARGE`, tanpa DNS, koneksi, atau status HTTP target |
| DSL dan resolusi | Impor dua step HTTP atau placeholder yang tidak dapat di-resolve | `422`; flow invalid tidak disimpan |
| Parser YAML | Field tak dikenal, anchor/alias, dokumen ganda, slug tidak valid, body melebihi `ALURJET_MAX_FLOW_YAML_BYTES`, YAML rusak | `422 INVALID_FLOW`, `413`, atau `400 INVALID_YAML` sesuai kasus. Tidak ada baris flow yang tersimpan |
| Header target | Flow dengan `headers` (fixture `order-to-api-auth.yaml`). Variasi: header terlarang, lebih dari 10 header, nilai berisi CR/LF, variabel dihapus sebelum dispatch | Header sampai ke mock. Kasus tidak valid ditolak `422` saat import. Variabel hilang menghasilkan `HEADER_RESOLVE_FAILED` dan `dead_letter`. Nilai header tidak ada di DB, response API, atau log |
| Health dan shutdown | Matikan MySQL lalu panggil `/healthz`. Kirim SIGTERM ke worker saat attempt berjalan | `503` saat DB mati. Worker berhenti mengklaim, menunggu sampai `ALURJET_SHUTDOWN_TIMEOUT`, dan sisa attempt dipulihkan oleh recovery |
| Multi-worker | Jalankan dua worker pada satu antrean berisi banyak execution | Tiap execution diklaim sekali. Tidak ada attempt ganda dengan nomor sama |
| Validasi startup | Jalankan dengan secret kurang dari 32 karakter, admin key sama dengan token webhook, `attempt_timeout >= lease`, atau JSON token rusak | Proses keluar nonnol dengan pesan jelas tanpa mencetak nilai rahasia |
| Kapasitas aktif | Buat target offline. Isi flow dengan `pending`, `running`, dan `retry_scheduled` sampai `ALURJET_MAX_ACTIVE_PER_FLOW`. Uji webhook dan replay baru atau ulang, serta commit terminal yang melepas slot | Baru terkena `429` dengan `Retry-After: 5`. Replay atau webhook identik yang sudah commit tetap `202`. Event ID dengan byte berbeda tetap `409` |
| Lock kuota flow | Sisakan satu slot, lalu ajukan webhook dan replay baru bersamaan untuk flow yang sama | Tepat satu insert commit. Transaksi lain mendapat `429 FLOW_CAPACITY_EXCEEDED` |
| Respons target | Mock beri `200` dengan body besar atau lambat. Uji juga header yang melebihi batas | `200` sukses, body dikosongkan dari persistence dan log, slot kembali tersedia. Header berlebih menghasilkan error transport sebelum status disimpan |
| Ingest serentak | Ulang ID dan byte body yang sama, lalu ulang ID dengan body berbeda | ID sama untuk request identik, `409` untuk body berbeda, selalu satu row webhook |
| Nama field JSON berulang | Kirim nama sama pada objek root, objek bersarang, objek di dalam array, dan nama ekuivalen setelah escape | `400 INVALID_JSON`, tanpa row execution. Ulangi pada ID yang sudah dikenal untuk memastikan validasi body mendahului deduplikasi |
| Claim dan presisi | Kirim integer `9007199254740993` dan desimal `0.123456789012345678901`, lalu baca body POST yang diterima mock | Row bisa di-claim setelah commit. Bandingkan token angka dengan nilai literal yang diharapkan menggunakan `json.RawMessage` atau decoder `UseNumber`. Tolak angka dalam string. Jangan konversi nilai input, expected, atau aktual ke `float64` |
| Finalisasi M2 | Mock Compose mode `fail_once` memberi `500` saat jatah masih ada. Untuk execution berbeda, gunakan target HTTP khusus dalam jaringan test terisolasi yang mengembalikan `400`. Konfigurasikan origin target sesuai kebijakan outbound test | `500` menutup attempt dan menyimpan `retry_scheduled` serta `available_at` dalam satu commit. Poller menunggu due. `400` berakhir `dead_letter` tanpa retry |
| Claim rollback | Gagalkan insert attempt setelah update claim | Execution tetap dapat diklaim, tanpa kenaikan attempts atau attempt yatim |
| Finalisasi terlambat | Gunakan token yang masih sama saat lease tepat kedaluwarsa atau sudah lewat | `ErrLostLease`; execution dan attempt tidak berubah |
| Crash setelah POST pada M2 | Mock mencatat body dan key, lalu worker dihentikan sebelum commit finalisasi. Mulai worker lagi setelah expiry | Recovery menutup attempt lama dan menjadwalkan retry tanpa menaikkan counter sampai claim baru. Mock menerima body dan key yang sama |
| Crash pada attempt terakhir | Hentikan worker dan gunakan waktu MySQL untuk memicu recovery | Satu attempt ditutup sebagai `lease_expired`, execution menjadi `dead_letter`, counter tetap |
| Retry dan replay | Mock gagal sekali untuk retry; pada execution lain paksa dead-letter lalu replay dengan key request sama secara serentak | Retry sukses dengan outbound key sama; hanya satu row replay per request key; snapshot dan outbound key asal dipertahankan |
| Kebijakan outbound | Mock mengembalikan redirect atau resolver terkendali memberi IP yang dilarang | Connector menolak koneksi yang tidak sesuai kebijakan, tanpa meneruskan POST ke tujuan yang ditolak |

Untuk race, atur urutan dengan barrier dan row lock. Gunakan fixture waktu untuk batas expiry dan mock/resolver terkontrol untuk HTTP sesuai [desain §§5, 8–9](../../SYSTEM_DESIGN.md).
