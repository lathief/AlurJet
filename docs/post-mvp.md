# Backlog AlurJet setelah MVP

> Semua item masih berupa kandidat. Mulai setelah v0.1.0 memenuhi [roadmap M0–M4](../ROADMAP.md) dan hasil verifikasinya tercatat.

## Kapan backlog bisa dimulai

- Panduan cepat berjalan dari lingkungan bersih, termasuk sukses, transient failure, dead-letter, dan replay. Tes MySQL untuk dedup serta recovery lease lulus.
- Catat baseline p95 waktu hingga claim, throughput, penggunaan database, dan waktu pemulihan pada profil beban yang bisa diulang. Tetapkan target sesuai beban dan verifikasi lagi setelah perubahan.
- Temuan keamanan, kehilangan execution, atau perubahan snapshot yang tidak semestinya masuk kembali ke backlog MVP. Catat bukti, batas uji, dan blocker di [worklog](ai/worklog.md).

## Ditunda dari MVP

Kedua butir ini bagian dari blueprint awal dan dipindahkan oleh [ADR-002](ai/decisions.md) untuk menjaga jadwal v0.1.0.

| Kandidat | Ambil saat | Kriteria selesai |
|---|---|---|
| Ambang global antrean (`ALURJET_ACTIVE_HIGH_WATERMARK`) dengan `503 SYSTEM_BACKPRESSURE` | Banyak flow aktif sekaligus dan kuota per flow tidak cukup melindungi database | Insert baru ditolak dengan `503` dan `Retry-After` saat ambang tercapai. Dedup yang sudah commit tetap `202`. Query plan dan lock wait diukur sebelum menambah counter |
| Anggaran byte per step `transform` | Flow dengan banyak step transform membesarkan data antara | Setiap step menolak output di atas batas sebelum menyalin fragmen. Uji field berulang, UTF-8/escaping, dan step yang mengecilkan objek |
| Rate limit autentikasi gagal, HMAC atau timestamp webhook | Endpoint terbuka ke jaringan publik tanpa reverse proxy | Percobaan token salah dibatasi dan replay request lama ditolak berdasarkan tanda tangan |
| Retensi dan masa dedup (lihat P1 di bawah) | Storage mulai membesar | Lihat kandidat retensi |

## P1 — Operasional dan pengendalian data

| Urutan | Kandidat | Ambil saat | Kriteria selesai |
|---|---|---|---|
| 1 | Metrik antrean dan ringkasan operasional | Log dan history belum cukup untuk mengukur backlog atau memisahkan kegagalan target dari kegagalan worker | Waktu tunggu, attempt, lease expiry, dan jumlah status dapat diukur dengan label terbatas. Demo gangguan menunjukkan sinyal yang sesuai. |
| 2 | Retensi execution dan attempt | Payload atau riwayat mulai membebani storage atau backup | Bersihkan data terminal secara bertahap. Tetapkan masa dedup dan hubungan replay dengan data yang dipertahankan, lalu uji bahwa pending dan pemulihan tidak terhapus. |
| 3 | Kredensial per integrasi | Operator memerlukan rotasi dan pencabutan token pada satu integrasi | Rotasi dapat dilakukan tanpa mengganggu flow lain. Kredensial mentah tidak dipersist atau dicatat, dan akses lintas integrasi diuji. |

## P2 — Perluasan dengan kebutuhan yang terbukti

| Kandidat | Ambil saat | Kriteria selesai |
|---|---|---|
| Satu trigger terjadwal | Ada flow periodik yang saat ini bergantung pada pengirim webhook eksternal | Jadwal dan dedup berjalan konsisten saat restart. Snapshot, worker, dan recovery yang sudah ada digunakan kembali. |
| Satu jenis integrasi outbound tambahan | Ada target konkret yang tidak bisa ditangani oleh HTTP POST | Kontrak timeout, retry, idempotensi, dan pembatasan tujuan sudah diuji untuk integrasi tersebut. |
| Evaluasi Redis atau NATS | Pengukuran menunjukkan polling atau kontensi MySQL tetap menjadi bottleneck setelah tuning | Buat ADR dan uji beban yang membandingkan throughput, waktu tunggu, serta biaya operasi. Rencana peralihan menjaga execution lama dan dedup. |

Ambil satu perluasan dalam satu waktu. Setiap usulan perlu profil beban atau kebutuhan integrasi, tes penerimaan, dan rencana kompatibilitas dengan flow yang sudah disimpan.
