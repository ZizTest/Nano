# 00 — README (untuk lu, BUKAN untuk dikirim ke AI coder)

Paket spec: **Telegram bot downloader (YT / FB / IG / TikTok) + list + cut + web server file di port 54321**.

## 1. Isi paket

| File | Kirim ke | Isi |
|---|---|---|
| `01_KONTRAK_BERSAMA.md` | **SEMUA AI** | Arsitektur, env, schema DB, kode error, signature semua modul |
| `02_WP1_core.md` | AI #1 | config, logger, errors, utils, proc, package.json |
| `03_WP2_data_storage.md` | AI #2 | SQLite (better-sqlite3), storage, antrian job |
| `04_WP3_downloader.md` | AI #3 | yt-dlp, resolusi tertinggi, pipeline download |
| `05_WP4_media_cut.md` | AI #4 | ffprobe, ffmpeg, pipeline `/cut` |
| `06_WP5_web_bootstrap.md` | AI #5 | web server :54321, `index.js`, systemd, contoh nginx |
| `07_WP6_telegram_bot.md` | AI #6 | semua perintah bot, pesan, tombol |
| `08_INTEGRASI_DAN_TEST.md` | lu / AI #7 | checklist test + prompt "integrator" |

Enam WP **dikerjakan paralel** — nggak ada yang nunggu yang lain, karena semua sudah terikat ke kontrak.

## 2. Alur kerja

1. Tiap AI (#1–#6) dikasih **2 file**: `01_KONTRAK_BERSAMA.md` + file WP-nya. Pakai prompt di bawah.
2. Tiap AI membalas file-file kode dengan header `### path/file`. Simpan **persis di path itu** di dalam folder proyek (mis. `/opt/videobot/`).
3. Semua WP selesai → kirim semua kode ke 1 AI "integrator" (prompt ada di file 08) untuk cek ketidakcocokan import/export. Terapkan perbaikannya.
4. Setup server (§4), `npm install`, isi `.env`, jalankan, lalu ikuti checklist di file 08.

### Prompt untuk tiap AI coder (copy-paste)

~~~
Kamu senior Node.js engineer. Aku lampirkan 2 dokumen:
1) 01_KONTRAK_BERSAMA.md — kontrak untuk seluruh proyek (WAJIB dipatuhi persis).
2) <NAMA FILE WP> — tugas khususmu.

Kerjakan HANYA file yang tercantum di tabel "File yang harus kamu hasilkan" pada dokumen WP.
Aturan: nama export, signature, path, nama env, dan kode error harus sama persis dengan kontrak.
Tidak boleh ada placeholder / TODO / "..." — setiap file harus utuh dan langsung jalan.
Format jawaban: tiap file diawali baris `### path/ke/file` lalu satu code block berisi isi file utuh.
Tutup dengan: daftar file, asumsi yang kamu ambil, dan cara menjalankan test.
~~~

> Kalau AI-nya punya sandbox/code execution, tambahkan: *"Jalankan `node --test` untuk test milikmu sebelum menjawab."*

## 3. Struktur akhir di server

```
/opt/videobot/
├── package.json  .env  .env.example  .gitignore
├── src/ …            (dari 6 AI)
├── deploy/ …
├── tests/ …
├── data/             (otomatis: app.db)
└── storage/          (otomatis: videos/ cuts/ tmp/)
```

## 4. Setup server (Ubuntu/Debian)

```bash
# 1) Paket dasar
sudo apt update && sudo apt install -y ffmpeg curl build-essential python3 sqlite3

# 2) yt-dlp (binary resmi, selalu update)
sudo curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
sudo chmod a+rx /usr/local/bin/yt-dlp
yt-dlp --version

# 3) Node.js >= 20 (disarankan LTS terbaru)
node -v

# 4) Proyek
cd /opt/videobot
npm install
cp .env.example .env
nano .env        # isi BOT_TOKEN, ALLOWED_USER_IDS, dst.
npm test         # opsional
npm start
```

**.env minimal**
```
BOT_TOKEN=xxxxxxxx:yyyyyyyy
ALLOWED_USER_IDS=123456789
PUBLIC_BASE_URL=https://heyziz.my.id
PORT=54321
```
- Token bot boleh juga diambil dari env sistem; nama yang dikenali: `BOT_TOKEN`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_TOKEN`.
- Belum tahu user ID lu? Jalankan dulu dengan `ALLOWED_USER_IDS` kosong → kirim apa saja ke bot → bot balas "Akses ditolak, User ID kamu: …". Isi ID itu ke `.env`, restart.

**Jalan 24/7 (systemd)**
```bash
sudo cp deploy/videobot.service /etc/systemd/system/   # edit path/user dulu
sudo systemctl daemon-reload
sudo systemctl enable --now videobot
journalctl -u videobot -f
```

**Rutin penting:** update yt-dlp tiap 1–2 minggu (platform sering ganti): `sudo yt-dlp -U`.

## 5. Keputusan desain yang gua ambil (bisa lu ubah di kontrak sebelum dikirim)

| Topik | Keputusan |
|---|---|
| Stack | Node.js + JavaScript (tanpa build) — karena lu nyebut `better-sqlite3`; bot pakai grammY (long polling) |
| URL publik | `https://heyziz.my.id/videos/<file>` (download) dan `/cuts/<file>` (hasil cut). Nama file: `judul-slug-<8 hex acak>.mp4` supaya link tidak bisa ditebak |
| `/cut` | `/cut <durasi> <id>`: `30` = 30 detik pertama · `00:10-00:45` = dari detik 10 ke 45 · `10-` = dari detik 10 sampai akhir. Hasil = **video baru** (original aman), muncul di `/list` dengan ikon ✂️ |
| Mode cut | Default `reencode` (akurat per-frame, hasil H.264/MP4 pasti bisa dipreview). Mau cepat tapi kurang presisi: `CUT_MODE=copy` |
| Resolusi | `yt-dlp -S "res,fps,..."` → resolusi tertinggi prioritas mutlak, codec cuma tie-breaker |
| Keamanan | Whitelist `ALLOWED_USER_IDS` (kalau tidak, siapa pun yang tahu username bot bisa pakai server lu) |
| Antrian | Download maks 2 paralel, cut 1 paralel (ubah di env) |
| Gagal | Pesan ❌ berisi alasan (bahasa Indonesia) + kode error; semua job (termasuk gagal) tercatat di tabel `jobs` |

## 6. Troubleshooting cepat

| Gejala | Penyebab & solusi |
|---|---|
| Instagram / Facebook gagal "login required" | Platform minta login. Export cookies browser (format Netscape `cookies.txt`) → set `YTDLP_COOKIES_FILE=/path/cookies.txt` |
| YouTube "Sign in to confirm you're not a bot" | IP server (datacenter) kena anti-bot. Pakai cookies yang sama seperti di atas |
| YouTube error soal JS challenge / "n challenge" / format tidak tersedia | Update yt-dlp dulu (`yt-dlp -U`). YouTube sekarang butuh JS runtime di server; ikuti panduan EJS di dokumentasi yt-dlp (biasanya cukup pasang `deno`), atau set `YTDLP_EXTRA_ARGS=--js-runtimes node`. Cek opsi yang valid dengan `yt-dlp --help` |
| Video 4K tidak bisa diputar di HP/Safari | Resolusi tertinggi YouTube hanya tersedia di VP9/AV1 — normal. Chrome/Firefox desktop aman. Hasil `/cut` selalu H.264 sehingga kompatibel |
| Link 404 dari domain, padahal file ada | nginx belum meneruskan `/videos/` dan `/cuts/` ke `127.0.0.1:54321` — cocokkan dengan `deploy/nginx.example.conf` |
| `EADDRINUSE` | Port 54321 sudah dipakai proses lain: `ss -ltnp | grep 54321` |
| Bot diam | `journalctl -u videobot -f`; cek token & `ALLOWED_USER_IDS` |
| Disk penuh | Job ditolak otomatis (`MIN_FREE_DISK_MB`). Hapus video lama manual di `storage/` (hapus filenya, lalu baris di tabel `videos` bila perlu) |

> Catatan: fitur hapus video via bot (`/delete`) sengaja **belum** dimasukkan karena di luar permintaan awal. Mudah ditambah belakangan sebagai WP7.
