# Catatan keputusan AlurJet

Keputusan teknis lengkap berada di [desain sistem](../../SYSTEM_DESIGN.md). Catat perubahan yang memengaruhi kontrak beserta tautan ke bagian yang diperbarui.

| Keputusan | Dampak implementasi | Rujukan |
|---|---|---|
| MySQL adalah penyimpanan dan antrean | Job langsung siap setelah commit, claim dan riwayat attempt atomik | Desain §§4–5 |
| Tepat satu HTTP POST di step terakhir | Worker dapat mengulang flow dari snapshot pada setiap attempt | Desain §6 |
| Dedup webhook membandingkan byte body | ID sama dan byte berbeda menghasilkan `409`; replay memiliki key request tersendiri | Desain §5 |
| Kapasitas berlaku untuk insert execution baru | Deduplikasi yang sudah commit tetap bisa dibaca. Kuota per flow ketat dan ambang global operasional | Desain §5 |
| Connector memakai status target dan membatasi header | Body target ditutup tanpa dibaca pada MVP | Desain §5 |
| Replay mempertahankan snapshot dan outbound key asal | Perubahan flow berlaku untuk event baru, sedangkan kebijakan outbound aktif tetap diperiksa | Desain §§5, 8 |
| JSON mempertahankan token angka sejak ingest | Worker memeriksa presisi hingga serialisasi POST dan membatasi byte setiap step | Desain §§4–6, 9 |
| Token webhook terikat slug | Satu token untuk satu flow, diatur oleh operator melalui konfigurasi API | Desain §§7–8 |
| Recovery minimum tersedia sejak M2 | Restart worker memulihkan lease expired; retry memakai outbound key yang sama | Desain §5 dan roadmap |

Untuk entri baru, catat keputusan, alasan, konsekuensi untuk API/schema/tes, dan rujukan ke perubahan. Bukti eksekusi per tugas masuk ke [worklog](worklog.md).
