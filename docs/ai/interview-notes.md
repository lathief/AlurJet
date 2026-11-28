# Catatan persiapan wawancara AlurJet

> **Status: draf kerangka.** Dokumen ini berisi pertanyaan yang mungkin muncul dan poin jawaban yang diturunkan dari rancangan. **Tulis ulang setiap jawaban dengan kata-kata Anda sendiri.** Jawaban yang Anda hafal dari dokumen akan terdengar kosong saat pewawancara bertanya satu tingkat lebih dalam. Perbarui kolom "Bukti" setelah implementasi dan tes ada.

## Cara memakai dokumen ini

1. Baca poin jawaban, lalu tutup dokumen dan jelaskan keras-keras dengan kata sendiri.
2. Setelah kode ada, buka bagian kode yang relevan dan pastikan Anda bisa menunjukkannya.
3. Ganti kolom "Bukti" dari `Rancangan` menjadi tautan ke tes, ADR, atau hasil benchmark.
4. Jangan mengklaim sesuatu yang berstatus `Rancangan` sebagai sudah berjalan.

Legenda bukti: **Terbukti** (sudah dijalankan dan tercatat di [worklog](worklog.md)), **Rancangan** (ada di dokumen, belum diimplementasi atau diuji).

Catatan: sampai Task 1 di [review multi-peran](review-multi-peran.md#rencana-25-task) selesai, dokumen desain masih memakai skema v1 (`executions.id` UUID) dan kuota ketat dengan lock row flow. Yang direncanakan adalah PK `BIGINT` dengan `public_id` UUID dan kuota lunak. Tabel `flow_versions` dibatalkan dan snapshot tetap disimpan di execution. Semua itu belum tertulis di desain.

## Pitch

### 30 detik

> AlurJet adalah relay webhook self-hosted. Event masuk lewat webhook, divalidasi dan dipetakan, lalu dikirim ke API tujuan oleh worker dengan retry, dead-letter, dan replay. Fokusnya keandalan: antrean di MySQL, deduplikasi, lease dengan fencing, dan idempotency pada request keluar.

### 2 menit (kerangka)

1. **Masalah:** kirim event ke API pihak ketiga tanpa kehilangan event, tanpa kirim ganda yang berbahaya, dan dengan jejak yang bisa diperiksa.
2. **Pendekatan:** terima dan simpan dulu (`202` setelah commit), proses di background, semua status ada di MySQL.
3. **Tiga hal tersulit:** worker yang mati di tengah kerja, request yang sama datang dua kali, dan target yang tidak bisa dipercaya (SSRF).
4. **Bukti:** tes konkurensi, skenario crash worker, dan angka benchmark (isi setelah ada).
5. **Batas yang jujur:** at-least-once, bukan exactly-once. Bukan pengganti broker pada throughput tinggi.

Tulis versi Anda sendiri di sini:

```text
(isi dengan kata-kata Anda)
```

## Pertanyaan dan kerangka jawaban

### A. Arsitektur dan pilihan

**A1. Kenapa antrean di MySQL, bukan Kafka, RabbitMQ, atau Redis?**
- Satu komponen persisten untuk flow, execution, riwayat, dan antrean. Status dan antrean berubah dalam transaksi yang sama, jadi tidak ada masalah dual-write.
- Cukup untuk beban demo dan menengah. Bukan pengganti broker pada throughput tinggi.
- Kapan beralih: ketika polling atau kontensi MySQL terbukti menjadi bottleneck lewat pengukuran. Rencana peralihan ada di [backlog pasca-MVP](../post-mvp.md).
- Rujukan: [ADR-001](decisions.md), desain §10. Bukti: **Rancangan** (angka kinerja menunggu skrip beban).

**A2. Kenapa Clean Architecture (controller, usecase, repository)? Bukankah berlebihan?**
- Batasnya pragmatis: Fiber hanya ada di controller, usecase tidak mengimpor driver MySQL, interface repository ada di `domain`.
- Manfaatnya konkret: usecase `ProcessExecution` bisa diuji dengan repository palsu tanpa database dan tanpa HTTP.
- Siap mengakui bahwa untuk proyek sekecil ini sebagian pemisahan adalah pilihan gaya, bukan kebutuhan.

**A3. Kenapa `202` setelah commit, bukan menunggu pengiriman ke target?**
- Pengirim webhook biasanya punya timeout pendek dan tidak boleh bergantung pada kesehatan target.
- Dengan `202` setelah commit, event sudah tahan lama (durable) sebelum kita membalas.
- Konsekuensi: validasi bisnis berjalan di worker, jadi payload yang salah tetap mendapat `202` lalu berakhir `dead_letter`. Ini trade-off yang disengaja dan tercatat di panduan flow.
- Pertanyaan lanjutan yang mungkin muncul: "mengapa validasi bisnis di worker?" Jawaban jujur: agar jalur ingest tetap tipis dan tidak bergantung pada logika flow. Kelemahannya, pengirim tidak mendapat umpan balik cepat dan replay tidak membantu karena payload sama.
- Opsi yang dipertimbangkan: validasi `required` dijalankan sinkron saat ingest dan menghasilkan `422`. Validasi itu murni dan murah. **Belum diputuskan**. Siapkan alasan untuk kedua sisi.

**A4. Kenapa flow linear tanpa percabangan?**
- Setiap attempt mengulang flow dari payload awal, jadi retry mudah dan deterministik. Satu HTTP POST di akhir menjaga semantik idempotency tetap sederhana.
- Percabangan dan scripting sengaja di luar MVP agar cakupan terkendali.

### B. Konkurensi dan keandalan

**B1. Bagaimana dua worker tidak mengambil pekerjaan yang sama?**
- Claim memakai `SELECT ... FOR UPDATE SKIP LOCKED`. Baris yang sedang dikunci worker lain dilewati, bukan ditunggu.
- Dalam satu transaksi: ubah status ke `running`, naikkan `attempts`, buat token lease baru, dan buat attempt. Commit dulu, baru kerjakan I/O.
- Tidak ada I/O jaringan di dalam transaksi.
- Query claim memakai `status IN (...) ORDER BY available_at, id`, yang kemungkinan memicu filesort pada indeks `(status, available_at, id)`. Belum diukur. Rencananya `EXPLAIN` di Task 8, dengan opsi `UNION ALL` per status.
- Rujukan: desain §5 "Claim, finalisasi, dan recovery". Bukti: **Rancangan** (tes dua worker di Task 8). Perilaku `SKIP LOCKED` konkuren belum diuji.

**B2. Apa itu lease dan fencing token? Kenapa perlu?**
- Lease: worker "menyewa" execution sampai `lease_until`. Jika worker mati, sewa kedaluwarsa dan recovery mengambil alih.
- Fencing token: setiap claim menghasilkan token baru. Finalisasi hanya sah jika status masih `running`, token cocok, dan lease belum lewat. Worker lama yang "bangun" setelah diambil alih tidak bisa menimpa hasil worker baru.
- Contoh yang bisa Anda ceritakan: worker A macet 90 detik, lease 60 detik. Recovery menyerahkan ke worker B. Saat A pulih dan mencoba finalisasi, guard menolaknya (`ErrLostLease`).

**B3. Kenapa waktu lease memakai jam MySQL, bukan jam aplikasi?**
- Jam antar mesin bisa berbeda. Dengan satu sumber waktu (MySQL), keputusan "lease sudah habis atau belum" konsisten untuk semua worker.
- Waktu keputusan diambil setelah row lock, sehingga tidak ada celah antara pengecekan dan penulisan.
- Pertanyaan lanjutan yang mungkin: bagaimana kalau jam MySQL melompat? Jawab jujur sebagai risiko yang diterima di MVP.

**B4. Apa yang terjadi jika worker mati setelah target menerima POST, tetapi sebelum hasil disimpan?**
- Execution tetap `running` sampai lease habis. Recovery menutup attempt lama sebagai `lease_expired` lalu menjadwalkan ulang.
- Retry mengirim POST lagi dengan `Idempotency-Key` yang sama. Apakah efek ganda dicegah bergantung pada target yang menghormati key itu.
- Karena itu sistem ini at-least-once, dan dokumen tidak mengklaim exactly-once.

**B5. Kenapa `READ COMMITTED`, dan bagaimana kuota dijaga tanpa lock row flow?**
- Cocok dengan pola "baca dedup dari data committed, lalu insert". Mengurangi gap lock dibanding `REPEATABLE READ`.
- Dedup dijamin unique key `(flow_id, event_id)`, jadi request serentak yang sama tidak menghasilkan dua baris walaupun tidak ada lock row flow.
- Kuota per flow lunak: hitung execution aktif lewat probe berindeks, lalu insert. Kelebihan terbatas pada jumlah ingest serentak. Tujuannya melindungi database, bukan menjamin angka persis.
- Trade-off yang harus bisa Anda jelaskan: kuota ketat memerlukan lock row flow dan menyerialkan ingest per flow. Kuota lunak menukar ketepatan angka dengan konkurensi.
- Claim dan finalisasi hanya mengunci execution, sehingga tidak ada urutan lock bertingkat yang berisiko deadlock.
- Bukti: **Rancangan** (kuota lunak belum tertulis di desain dan belum diuji).

**B6. Bagaimana retry dan backoff bekerja?**
- `initialDelay × 2^(n-1)` dengan batas 60 detik, default 3 attempt. Jitter ±20% (direncanakan) agar retry tidak serentak setelah target pulih.
- Hanya timeout, `408`, `429`, `5xx`, dan error transport sementara yang dicoba lagi. Error validasi, redirect, TLS, dan `4xx` lain langsung `dead_letter`.
- Bukti jitter: **Rancangan**.

**B7. Replay memakai outbound key yang sama. Apakah replay pasti berefek di target?**
- Tidak pasti. Jika target menyimpan hasil per `Idempotency-Key`, termasuk hasil gagal, request replay dapat menerima respons lama tanpa memproses ulang.
- Alasan mempertahankan key asal: mencegah efek ganda bila attempt awal sebenarnya sudah diproses target.
- Trade-off: aman dari efek ganda, tetapi replay bisa tidak berefek. Opsi yang dipertimbangkan adalah replay dengan key baru. **Belum diputuskan**. Tulis keputusan akhir Anda dan alasannya.
- Bukti: **Rancangan**.

**B8. Bagaimana Anda membuktikan sistem tidak kehilangan atau menggandakan pekerjaan?**
- Tulis invarian: tidak ada execution hilang, tidak ada attempt difinalisasi dua kali, paling banyak satu attempt `running` per execution, `attempts` tidak melebihi batas.
- Uji dengan injeksi kegagalan di bawah beban: bunuh worker acak, hentikan MySQL sebentar, ubah target antara sukses dan gagal. Periksa invarian setelah tiap skenario, dengan seed acak yang dicatat.
- Dijadwalkan di Task 21 (v1.0). Sebelum itu, hanya tes crash tunggal di Task 10.
- Bukti: **Rancangan**.

### C. Data dan skema

**C1. Kenapa PK `BIGINT` dan `public_id` UUID terpisah?**
- Awalnya PK UUID v4 `CHAR(36)`. Masalahnya: insert acak memecah halaman clustered index dan setiap indeks sekunder membawa 36 byte PK.
- Rencananya PK `BIGINT` urut untuk performa, `public_id` UUID untuk API supaya ID tidak bisa ditebak dan tidak membocorkan jumlah data.
- Jujur: ini keputusan yang berubah setelah review. Itu sehat, dan ADR-007 akan mencatat alasannya.
- Bukti: **Rancangan** (DDL v1 terbukti, skema baru belum).

**C2. Kenapa snapshot flow disimpan di setiap execution, bukan di tabel versi terpisah?**
- Tabel `flow_versions` sempat dipertimbangkan lalu dibatalkan demi kesederhanaan.
- Snapshot di execution menjamin retry dan replay memakai definisi yang sama persis walaupun flow diubah kemudian, tanpa join tambahan.
- Konsekuensi: baris `executions` lebih besar dan definisi yang sama tersimpan berulang. Dampaknya diukur di benchmark dan ditulis sebagai batasan. Jika terbukti menjadi masalah, pemisahan ke tabel versi adalah langkah evolusi yang jelas.
- Bukti: **Rancangan**.

**C3. Bagaimana deduplikasi webhook bekerja? Kenapa membandingkan hash body?**
- Kunci dedup: `(flow_id, event_id)`. Pengirim yang mengulang request (karena respons hilang) mendapat `202` dengan execution yang sama.
- Jika `event_id` sama tetapi isi berbeda, itu bug atau penyalahgunaan di sisi pengirim, jadi `409 EVENT_ID_CONFLICT`.
- Hash dihitung dari byte asli tanpa serialisasi ulang. Spasi pun dianggap beda. Pengirim harus mengulang byte yang sama.
- Unique key adalah pengaman untuk request serentak.

**C4. Kenapa payload disimpan sebagai `LONGTEXT`, bukan `JSON`?**
- Tipe `JSON` MySQL menormalisasi angka dan urutan key. Kita perlu menjaga token angka asli (misalnya `9007199254740993`) sampai request keluar.
- MySQL tidak perlu mengkueri isi payload, jadi `LONGTEXT` cukup.

**C5. Bagaimana presisi angka JSON dijaga?**
- Di Go, memakai `json.RawMessage` atau `json.Number` dan menghindari konversi ke `float64`.
- Tes: integer `9007199254740993` dan desimal `0.123456789012345678901` harus keluar sebagai token yang sama di body POST.
- Bukti: **Rancangan**.

**C6. Bagaimana ukuran tabel dikendalikan?**
- Job retensi menghapus execution terminal yang lebih tua dari batas (default 14 hari) secara bertahap. Masa dedup sama dengan masa retensi.
- Tidak menghapus `pending` atau `running`.
- Bukti: **Rancangan**.

### D. Keamanan

**D1. Bagaimana mencegah SSRF?**
- Origin tujuan harus ada di allowlist (skema, host, port).
- Pemeriksaan IP dilakukan di `net.Dialer.Control`, yaitu setelah DNS dan sebelum koneksi. Dengan begitu DNS rebinding tidak menembus, karena yang diperiksa adalah alamat yang benar-benar akan dihubungi.
- Daftar CIDR eksplisit (private, loopback, link-local, CGNAT, multicast, reserved). Dokumentasi Go sendiri menyatakan `IP.IsPrivate` tidak boleh dipakai sendirian untuk kontrol akses.
- Redirect dan proxy dari environment dimatikan.
- Bukti: **Rancangan**.

**D2. Bagaimana rahasia untuk header target (misalnya `Authorization`) ditangani?**
- Database hanya menyimpan template (`Bearer ${ALURJET_FLOW_TARGET_TOKEN}`). Nilai dibaca dari environment worker saat dispatch.
- Konsekuensinya: secret tidak masuk database, snapshot, response API, atau log. API dan worker harus memakai environment yang sama.
- Rujukan: [ADR-003](decisions.md).

**D3. Apa kelemahan autentikasinya?**
- Satu admin key dan satu token statis per flow di `ALURJET_WEBHOOK_TOKENS_JSON`. Belum ada rate limit untuk percobaan gagal, belum ada HMAC atau timestamp pada webhook, rotasi memerlukan restart.
- Kontrak inbound v0.1.0 juga mengharuskan `X-Event-ID` dan `X-Webhook-Token`. Penyedia seperti Stripe dan GitHub tidak bisa mengirim header itu.
- Itu keputusan v0.1.0 yang tercatat sebagai batasan. Dijadwalkan v0.2.0: `trigger.eventId` yang dapat dikonfigurasi (header atau JSONPath, cadangan hash body), verifikasi HMAC, dan token di database (hash, tampil sekali, ada rotasi).
- Siap menjelaskan cara Stripe atau GitHub menandatangani webhook, dan kenapa timestamp dalam tanda tangan mencegah replay.

**D4. Bagaimana token dibandingkan?**
- Perbandingan waktu konstan (`subtle.ConstantTimeCompare`) pada token yang terikat ke slug. Token flow A tidak berlaku di flow B, dan jawaban `401` sama untuk semua sebab agar tidak membocorkan informasi.

### E. Operabilitas

**E1. Bagaimana Anda tahu sistem sedang bermasalah?**
- `/metrics` Prometheus: jumlah execution per status, umur execution tertua yang siap diproses, hasil dan durasi attempt, jumlah recovery lease.
- Log JSON dengan `execution_id`, `flow_slug`, `attempt`, `error_code`. Token, header target, dan payload tidak dicatat.
- Sinyal klasik: umur execution tertua naik berarti worker mati atau target lambat.
- Bukti: **Rancangan**.

**E2. Bagaimana graceful shutdown?**
- SIGTERM menghentikan polling dan claim baru, menunggu attempt aktif sampai `ALURJET_SHUTDOWN_TIMEOUT`, lalu keluar. Attempt yang tidak selesai dipulihkan oleh recovery lease.

**E3. Bisakah dijalankan lebih dari satu instance?**
- Ya untuk API dan worker, karena claim memakai `SKIP LOCKED`. Konfigurasi dibaca saat startup, jadi perubahan token memerlukan restart.

**E4. Apa saja yang bisa gagal, dan bagaimana sistem bereaksi?**
- Siapkan tabel failure mode: MySQL mati, target mati, worker crash, jam bergeser, event duplikat, payload beracun, disk penuh. Per kasus: dampak, deteksi (metrik atau log), dan pemulihan.
- Contoh jawaban ringkas: MySQL mati membuat `/healthz` mengembalikan `503` dan ingest gagal, tetapi tidak ada data yang hilang karena `202` hanya dikirim setelah commit.
- Dijadwalkan di `engineering-notes` (Task 22, v1.0). Bukti: **Rancangan**.

### F. Skala dan trade-off

**F1. Apa batas sistem ini?**
- Kuota per flow lunak, jadi bisa terlampaui sebanyak jumlah ingest serentak. Sebagai gantinya ingest tidak diserialkan per flow.
- Pool worker bersama, jadi satu flow lambat dapat menyita slot flow lain.
- Urutan eksekusi tidak dijamin.
- Angka nyata: **isi setelah skrip beban dijalankan** (lihat `docs/benchmarks.md`). Jangan menyebut angka sebelum diukur.

**F2. Bagaimana Anda menskalakannya 10x?**
- Ukur dulu. Kandidat: batch claim, indeks atau partisi tabel antrean, memisahkan payload dari tabel panas, polling adaptif dengan jitter.
- Jika kontensi MySQL tetap menjadi bottleneck: pindah ke broker dengan rencana migrasi yang menjaga dedup dan execution lama.

**F3. Apa yang akan Anda ubah jika mengulang?**
- Siapkan dua atau tiga jawaban jujur, misalnya: skema v1 dengan UUID sebagai PK (diubah), lock row flow untuk kuota ketat (diganti kuota lunak), menunda observabilitas terlalu lama, atau terlalu banyak dokumen sebelum ada kode.

### G. Proses dan alat bantu

**G1. Seberapa besar peran AI dalam proyek ini?**
- Jawab jujur. Repository memuat `AGENTS.md` dan `docs/ai/`, jadi penilai dapat melihat bahwa agen dipakai.
- Yang perlu Anda tunjukkan: keputusan desain (misalnya skema PK `BIGINT`, kuota lunak, cakupan MVP, header target) lahir dari review dan pilihan Anda, dan Anda memverifikasi hasilnya (DDL dijalankan di MySQL, tes dijalankan, tautan diperiksa).
- Siap menjelaskan satu keputusan yang Anda ubah setelah mempertimbangkan saran, dan satu hal yang Anda tolak.
- Jangan mengklaim bagian yang tidak Anda pahami.

**G2. Bagaimana Anda memverifikasi bahwa kode benar?**
- Tes unit table-driven, tes integrasi MySQL di Compose terpisah, fuzz test untuk detektor field JSON berulang, `go test -race`, dan CI.
- Tes race lease memakai barrier dan row lock, bukan `sleep`.
- Bukti: **Rancangan** sampai CI ada.

## Pertanyaan balik yang bisa Anda ajukan

- Bagaimana tim Anda menangani webhook atau event yang gagal terkirim saat ini?
- Apakah ada kebutuhan idempotency pada integrasi pihak ketiga?
- Seperti apa praktik observabilitas dan on-call di tim ini?

## Daftar periksa sebelum wawancara

- [ ] Saya bisa menggambar alur webhook sampai target di papan tulis tanpa melihat dokumen.
- [ ] Saya bisa menjelaskan lease, fencing token, dan alasan memakai waktu MySQL.
- [ ] Saya bisa menjelaskan dedup `(flow_id, event_id)` dan perbedaan `202` ulang dengan `409`.
- [ ] Saya bisa menjelaskan SSRF di `Dialer.Control` dan kenapa `IsPrivate` saja tidak cukup.
- [ ] Saya punya angka benchmark nyata dan tahu batas sistemnya.
- [ ] Saya bisa menunjukkan satu tes konkurensi dan satu skenario crash worker yang berjalan.
- [ ] Saya sudah menulis ulang semua jawaban di atas dengan kata-kata sendiri.
- [ ] Saya bisa menjelaskan invarian sistem dan cara mengujinya dengan injeksi kegagalan.
- [ ] Saya bisa menjelaskan trade-off kuota lunak dan replay dengan key yang sama.
- [ ] Saya punya jawaban jujur tentang peran alat bantu AI.

Lihat [review multi-peran](review-multi-peran.md), [review portofolio](review-portofolio.md), [catatan keputusan](decisions.md), [desain sistem](../../SYSTEM_DESIGN.md), dan [verifikasi](verification.md).
