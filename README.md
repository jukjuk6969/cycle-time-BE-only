# Cycle Time Monitor — Backend

REST API dan layanan real-time untuk **Cycle Time Monitor**. Backend ini menerima data cycle time dan status stasiun, menyimpan serta menyajikan data melalui PostgreSQL, dan mengirim pembaruan ke dashboard melalui Socket.IO. Backend juga dapat meneruskan live stream dan video dari komputer stasiun (FD).

## Teknologi

- Node.js dan Express
- PostgreSQL (`pg`)
- Socket.IO untuk pembaruan real-time
- TCP/UDP untuk heartbeat stasiun

## Persyaratan

- Node.js dan npm
- PostgreSQL yang dapat dijangkau dari server
- Database dengan tabel dan kolom yang digunakan aplikasi. Skema database/migrasi tidak disediakan di repository ini.
- Frontend dashboard dan/atau komputer stasiun yang dikonfigurasi untuk menggunakan backend ini

## Menjalankan secara lokal

```bash
npm install
```

Buat file `.env` di direktori proyek. Contoh:

```dotenv
# HTTP API dan Socket.IO
PORT=3000
HB_PORT=5002

# PostgreSQL — isi sesuai lingkungan Anda
DBSERVER=localhost
DBPORT=5432
DBNAME=CTMonitor
DBUSER=postgres
DBPASSWORD=change-me
DB_POOLMAX=30
DBENCRYPT=false

# Daftar origin dashboard, pisahkan dengan koma
CORS_ORIGINS=http://localhost:5173

# Ganti dengan rahasia acak yang panjang dan simpan di luar kontrol versi
AUTH_SECRET=replace-with-a-long-random-secret
AUTH_MODE=enforce
AUTH_SESSION_HOURS=12
AUTH_COOKIE_SECURE=false

# Opsional
LOG_VERBOSE=false
```

Jangan commit `.env` atau rahasia produksi ke repository. Gunakan `AUTH_COOKIE_SECURE=true` jika akses browser menggunakan HTTPS. Atur `CORS_ORIGINS` ke origin frontend yang benar; daftar kosong tidak mengizinkan origin browser.

Pasang dependensi, lalu jalankan:

```bash
npm start
```

Untuk mode pengembangan dengan restart otomatis:

```bash
npm run dev
```

Secara default HTTP API dan Socket.IO menggunakan port `3000`, sedangkan heartbeat TCP dan UDP menggunakan port `5002`. Konfigurasikan firewall agar port yang diperlukan dapat dijangkau oleh dashboard dan stasiun.

Alternatif konfigurasi port: `PORT_FAMILY=9000` menggunakan HTTP `9000` dan heartbeat `9002` (offset +2). `PORT` dan `HB_PORT` yang diatur secara eksplisit akan mengesampingkan nilai tersebut.

## Konfigurasi autentikasi

Autentikasi API diatur di `auth.js`. Mode default adalah `enforce`; mode `report` hanya mencatat tindakan yang seharusnya ditolak, sedangkan `off` menonaktifkan pemeriksaan. Sesi berlaku 12 jam secara default.

Akun dan role didefinisikan di dalam `auth.js` (`admin` dan `viewer`). Tinjau dan ubah konfigurasi akun/hash sebelum deployment. Buat hash password dengan:

```bash
node auth.js hash "password-baru"
```

Jika `AUTH_SECRET` tidak disetel atau kurang dari 16 karakter, server membuat secret acak saat mulai. Akibatnya, sesi yang ada tidak berlaku lagi setelah server restart.

## API utama

Semua route API menggunakan prefix `/api`.

| Method | Route | Kegunaan |
| --- | --- | --- |
| `GET` | `/api/health` | Memeriksa koneksi database |
| `POST` | `/api/auth/login` | Login dan membuat sesi |
| `POST` | `/api/auth/logout` | Mengakhiri sesi |
| `GET` | `/api/auth/me` | Memeriksa sesi aktif |
| `POST` | `/api/cycle` | Menerima data cycle time dari stasiun |
| `POST` | `/api/heartbeat` | Menerima status heartbeat |
| `GET` | `/api/summary` | Mendapatkan ringkasan stasiun |
| `GET` | `/api/stations` | Mendapatkan daftar stasiun |
| `GET` | `/api/station/:line/:station/history` | Mendapatkan riwayat cycle |
| `GET` | `/api/station/:line/:station/stats` | Mendapatkan statistik stasiun |
| `GET` | `/api/station/:line/:station/hourly` | Mendapatkan statistik per bucket waktu |
| `GET` | `/api/videos` | Mendapatkan daftar video dengan filter dan pagination |
| `GET` | `/api/videos/export` | Mengekspor metadata video sebagai CSV |

Endpoint pengelolaan stasiun, video, dan proxy ke FD juga tersedia di `server.js`. Akses tulis umumnya memerlukan role admin. Endpoint yang dikecualikan untuk komunikasi stasiun dijabarkan di `auth.js`.

### Socket.IO

Dashboard menerima event antara lain `cycle_complete`, `heartbeat`, `heartbeat_batch`, `summary_update`, dan `fd_status`. Klien dapat mengirim event `subscribe` dengan daftar nomor line untuk membatasi pembaruan ke line tertentu.

## Struktur proyek

- `server.js` — Express API, Socket.IO, heartbeat, dan proxy stream/video
- `auth.js` — login, sesi, dan pemeriksaan role
- `db.js` — koneksi PostgreSQL serta query data
- `engine.js` — ringkasan stasiun, penghitungan cycle, dan antrean penyimpanan
- `package.json` — skrip dan dependensi Node.js

## Catatan operasional

- Pastikan konfigurasi PostgreSQL dan skema database siap sebelum menjalankan server.
- Gunakan secret autentikasi dan kredensial database yang kuat; jangan gunakan nilai contoh atau default untuk produksi.
- Sesuaikan `CORS_ORIGINS` dengan alamat dashboard yang sebenarnya.
- Set `LOG_VERBOSE=true` untuk mengaktifkan log diagnostik yang lebih rinci.
