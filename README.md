# 08 — INTEGRASI & TEST (setelah semua WP selesai)

## A. Langkah integrasi

1. Simpan semua file dari AI #1–#6 sesuai path-nya di folder proyek.
2. `npm install` → pastikan tidak ada error build `better-sqlite3`.
3. `npm test` → semua test dari 6 WP harus lulus.
4. Cek cepat modul bisa di-load (tanpa menjalankan bot):
   ```bash
   BOT_TOKEN=test node --input-type=module -e "for (const f of ['config','logger','errors','utils/format','utils/html','utils/slug','utils/time','utils/proc','utils/url','db/index','db/repo','services/storage','services/queue','services/downloader','services/download-pipeline','services/media','services/cut-pipeline','web/server','bot/messages','bot/status-message','bot/middleware','bot/index']) { await import('./src/'+f+'.js'); console.log('OK', f) }"
   ```
   Error `does not provide an export named X` = ada WP yang menyimpang dari kontrak → lihat bagian C.
5. `npm start` dengan `.env` asli.

## B. Checklist test manual (urut)

| # | Tes | Hasil yang diharapkan |
|---|---|---|
| 1 | Kirim `/start` dari akun lu | Bantuan tampil |
| 2 | Kirim `/start` dari akun lain | "Akses ditolak" + User ID-nya |
| 3 | `/download <link YouTube pendek>` | Status ⏳ → ⬇️ persen → ✅ + link. Buka link di browser → video main & bisa di-seek |
| 4 | Cek resolusi | Di pesan ✅ resolusi = tertinggi yang ada (bandingkan dengan `yt-dlp -F <link>`) |
| 5 | `/download` link TikTok, Instagram Reel, Facebook video publik | Masing-masing ✅ (IG/FB bisa butuh cookies, lihat README) |
| 6 | `sqlite3 data/app.db "select id,kind,title,size_bytes,duration_sec,height,rel_path from videos;"` | Baris sesuai video yang diunduh |
| 7 | `ls storage/videos` & `ls storage/tmp` | File ada di `videos/`, `tmp/` kosong |
| 8 | `curl -sI -H "Range: bytes=0-1" https://heyziz.my.id/videos/<file>` | `HTTP/2 206`, `content-range: bytes 0-1/...` |
| 9 | `/list` | Jumlah, total ukuran/durasi, detail tiap video + tombol halaman bila > 5 |
| 10 | `/cut 10 <id>` | ✅ + link `/cuts/…`; video 10 detik; muncul di `/list` dengan ✂️ "dari #id" |
| 11 | `/cut 00:05-00:15 <id>` dan `/cut 5- <id>` | Hasil sesuai rentang |
| 12 | `/cut 999999-1000000 <id>` | ❌ rentang di luar durasi |
| 13 | `/cut abc 1`, `/cut 10 9999` | ❌ format salah / video tidak ditemukan |
| 14 | `/download https://example.com/x` | ❌ platform belum didukung |
| 15 | `/download` link video private/dihapus | ❌ "Video tidak tersedia…" + kode |
| 16 | Kirim 3 `/download` sekaligus | Maks 2 jalan, yang ke-3 "posisi #1"; `/list` tetap responsif saat download berjalan |
| 17 | Restart service saat download berjalan | Setelah nyala: bot kirim "Job #n terhenti karena server restart"; `select * from jobs` → `failed/INTERRUPTED`; `storage/tmp` bersih |
| 18 | `sudo systemctl stop videobot` | Berhenti < 30 detik, tanpa proses `yt-dlp`/`ffmpeg` yatim (`pgrep -a yt-dlp`) |

## C. Prompt untuk AI "Integrator" (opsional tapi disarankan)

Kirim: `01_KONTRAK_BERSAMA.md` + **seluruh kode** dari 6 WP (boleh dipecah per folder jika terlalu panjang) + prompt ini:

~~~
Kamu code reviewer/integrator. Terlampir kontrak proyek (01_KONTRAK_BERSAMA.md) dan kode dari 6 AI berbeda
yang tidak saling melihat. Tugasmu BUKAN menulis ulang, tapi menemukan ketidakcocokan dan memperbaikinya seminimal mungkin.

Periksa satu per satu:
1. Setiap `import` antar-file: file tujuan ada? nama export ada & bentuk (named vs default) cocok?
2. Signature tiap fungsi yang dipanggil lintas-modul vs Kontrak §8 (nama parameter objek, tipe, sinkron/async).
3. Nama env di config.js vs Kontrak §4; semua field `config.*` yang dipakai modul lain benar-benar ada.
4. Schema.sql vs Kontrak §6; nama kolom di repo.js; mapping camelCase ↔ snake_case; field Video/Job.
5. Kode error: semua `AppError('X')` memakai kode yang ada di `ErrorCode`.
6. Alur download & cut end-to-end (Kontrak §9): urutan pemanggilan, siapa yang `markJobFailed`, cleanup tmp.
7. Handler bot tidak meng-await job panjang; semua teks dinamis di-escape HTML.
8. Pelanggaran aturan: console.log, process.exit di luar index.js, dependency di luar 4 yang diizinkan, shell exec.

Output: daftar temuan (file, baris/fungsi, masalah, severity), lalu patch berupa file utuh HANYA untuk file yang perlu diubah.
~~~

## D. Kalau ada yang gagal — cara cepat mencari pelakunya

| Gejala | File pemilik |
|---|---|
| `does not provide an export named …` | WP pemilik file yang diimpor (lihat tabel Kontrak §3) |
| Error SQL / kolom tidak ada | WP2 (`schema.sql` / `repo.js`) |
| yt-dlp dipanggil dengan argumen salah | WP3 (`downloader.js` → `buildYtDlpArgs`) |
| Hasil cut tidak sesuai rentang | WP4 (`media.js` → `buildCutArgs`) atau WP1 (`utils/time.js` → `parseCutSpec`) |
| Link publik 404 | WP5 (`web/server.js`) atau nginx; cek juga WP2 `buildPublicUrl` |
| Pesan Telegram berantakan / tidak ter-escape | WP6 (`messages.js`) |
