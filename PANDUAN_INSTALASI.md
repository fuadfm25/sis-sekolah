# Panduan Instalasi – SIS Sekolah (Administrasi Pembelajaran)

Panduan ini untuk komputer Windows dengan **XAMPP** atau **Laragon**. Di Linux langkahnya sama, hanya lokasi foldernya berbeda.

---

## 1. Kebutuhan

| Komponen | Versi | Catatan |
|---|---|---|
| PHP | 8.2 atau lebih baru | ekstensi `intl`, `mbstring`, `mysqli`, `zip`, `gd`, `xml` aktif |
| MySQL / MariaDB | MySQL 5.7+/8.x atau MariaDB 10.3+ | sudah termasuk di XAMPP/Laragon |
| Composer | 2.x | hanya bila folder `vendor` belum ada |
| Browser | Chrome / Edge / Firefox terbaru | bisa tanpa internet setelah aset diunduh sekali (bagian 5.1) |

**Mengaktifkan ekstensi `intl` di XAMPP:** buka `C:\xampp\php\php.ini`, cari `;extension=intl`, hapus tanda `;`, simpan, lalu restart Apache. Lakukan hal yang sama untuk `zip` dan `gd` bila masih bertanda `;`.

---

## 2. Menyalin aplikasi

1. Ekstrak `sis-sekolah-v2.zip`, misalnya ke `C:\xampp\htdocs\sis-sekolah`.
2. Siapkan folder `vendor` dengan salah satu cara:
   - salin folder `vendor` dari proyek lama Anda ke dalam `sis-sekolah`, **atau**
   - buka terminal di folder `sis-sekolah` lalu jalankan `composer install`.
3. Pastikan folder `writable` dan `public/uploads` beserta isinya bisa ditulisi (`public/uploads/logo` dipakai untuk logo sekolah). Di Windows biasanya sudah otomatis; di Linux jalankan `chmod -R 775 writable public/uploads`.

---

## 3. Cara tercepat: halaman instalasi otomatis

Setelah langkah 2, buka `http://localhost/sis-sekolah/public/` di browser. Selama aplikasi belum terpasang, Anda otomatis diarahkan ke **halaman `/install`**:

1. **Pemeriksaan server** – versi PHP, ekstensi, dan folder yang harus bisa ditulisi. Perbaiki yang bertanda merah.
2. **Database** – isi host, nama database, username, dan password MySQL. Database dibuat otomatis bila belum ada (harus kosong).
3. **Alamat & waktu** – base URL terdeteksi otomatis; pilih zona waktu (WIB/WITA/WIT).
4. **Identitas sekolah & akun admin** – nama sekolah, jenjang, username serta password admin (bukan `admin123`). Centang *data contoh* hanya untuk uji coba.

Installer menjalankan seluruh migration, mengisi data dasar, membuat token kiosk, menulis file `.env`, lalu mengunci diri (`writable/installed.lock`). Bila `.env` tidak bisa ditulis, isinya ditampilkan untuk disalin manual. Untuk memasang ulang, hapus `writable/installed.lock` dan kosongkan database.

Bila memakai installer, bagian 4 dan 5 di bawah tidak perlu dilakukan.

---

## 3a. Konfigurasi `.env` secara manual

Salin `env.contoh` menjadi `.env` (atau ubah `.env` yang sudah ada), lalu sesuaikan:

```ini
CI_ENVIRONMENT = development
app.baseURL = 'http://localhost/sis-sekolah/public/'

database.default.hostname = localhost
database.default.database = db_sis_sekolah
database.default.username = root
database.default.password =
```

- **Biarkan `app.baseURL` kosong/bertanda `#` (disarankan).** Alamat dideteksi otomatis dari alamat yang dibuka, jadi aplikasi yang sama bisa dibuka lewat `http://localhost/sis-sekolah/public/` di server dan `http://192.168.1.10/sis-sekolah/public/` di laptop guru, HP satpam, atau tablet kiosk, **tanpa `spark serve` dan tanpa mengubah `.env`**. Installer juga memakai mode otomatis ini secara bawaan.
- Isi `app.baseURL` hanya bila perlu memaksa satu alamat (mis. di balik reverse proxy/HTTPS). Bila diisi, wajib diakhiri `/` dan semua perangkat harus membuka alamat itu.
- Zona waktu aplikasi diatur ke **WIB** (`Asia/Jakarta`). Sekolah di WITA/WIT perlu mengubah `$appTimezone` di `app/Config/App.php` menjadi `Asia/Makassar` atau `Asia/Jayapura`. Status terlambat di absensi bergantung pada pengaturan ini.

---

## 4. Membuat database – pilih SALAH SATU cara

### Cara A – Import template SQL lewat phpMyAdmin (paling mudah)

1. Buka `http://localhost/phpmyadmin`.
2. Klik **New/Baru**, beri nama `db_sis_sekolah`, collation **utf8mb4_general_ci**, lalu **Create**.
3. Pilih database tersebut, buka tab **Import**, pilih file **`database/sis_sekolah_template.sql`**, lalu klik **Go/Kirim**.
4. Selesai. Isinya: seluruh tabel, role, akun admin, pengaturan sekolah, jam pelajaran contoh, daftar pembiasaan, dan jenis perilaku beserta poinnya.

### Cara B – Lewat terminal (spark)

Buat database kosong `db_sis_sekolah` terlebih dahulu, lalu jalankan di folder `sis-sekolah`:

```bash
php spark migrate
```

Satu perintah itu membuat **semua tabel sekaligus mengisi data awal** (role, akun admin `admin` / `admin123`, pengaturan sekolah, jam pelajaran, pembiasaan, jenis perilaku, jenis perangkat pembelajaran). Migration tersusun rapi per kelompok:

| Urutan | Berkas | Isi |
|---|---|---|
| 1 | `100001_PenggunaSistem` | users, roles, user_roles, pengaturan, log_aktivitas |
| 2 | `100002_DataMaster` | tahun pelajaran, jurusan, kelas, guru, siswa, mata pelajaran, jam pelajaran |
| 3 | `100003_RombelJadwal` | wali kelas, anggota kelas, guru piket, penugasan, jadwal, kalender akademik |
| 4 | `100004_Pembelajaran` | jurnal, absensi mapel, ATP, kokurikuler |
| 5 | `100005_Kesiswaan` | izin siswa & izin keluar, absensi pagi, pembiasaan, perilaku, mutasi |
| 6 | `100006_Kepegawaian` | izin & kehadiran guru |
| 7 | `100007_BimbinganKonseling` | layanan BK & peserta |
| 8 | `100008_PerangkatPembelajaran` | perangkat, riwayat, program semester, modul ajar |
| 9 | `100009_DataAwal` | menjalankan `DatabaseSeeder` (aman diulang) |
| 10 | `100010_PembiasaanMandiri` | pengaturan pembiasaan & isian siswa (pembaruan setelah susunan baru) |
| 11 | `100011_PembiasaanBerlaku` | waktu berlaku kegiatan rumah (setiap hari / hari masuk sekolah / libur) |

Data contoh untuk latihan (opsional, **jangan** di database sungguhan): `php spark db:seed DemoSeeder`.

### Cara C – Memindahkan data dari aplikasi versi lama

Database lama tidak diubah sama sekali; datanya hanya disalin.

1. Buat database baru, misalnya `db_sis_sekolah_v2`.
2. Di `.env`:
   ```ini
   database.default.database = db_sis_sekolah_v2
   database.lama.database    = db_sis_sekolah
   ```
3. Jalankan:
   ```bash
   php spark migrate
   php spark db:seed ImporDataLamaSeeder
   ```
4. Login memakai **password lama**. Setelah itu periksa: jenis kelamin dan status kepegawaian guru, NIS siswa (sementara diisi NISN), dan JP/minggu penugasan.

---

## 5. Menjalankan aplikasi

### 5.1 Tampilan tanpa internet (aset offline)
Secara bawaan tampilan (Bootstrap, ikon, TomSelect, pemindai barcode) dimuat dari CDN. Agar tetap tampil saat internet sekolah mati, unduh aset **sekali** saat server sedang online:

- menu **Sistem → Aset Offline → Unduh semua**, atau
- terminal: `php spark aset:unduh`

File disimpan di `public/assets/vendor`; setelah itu aplikasi otomatis memakai file lokal.


**Opsi 1 – Apache XAMPP (disarankan untuk dipakai sekolah):** jalankan Apache & MySQL di XAMPP Control Panel, lalu buka `http://localhost/sis-sekolah/public/`. Dari perangkat lain di jaringan yang sama cukup buka `http://IP-SERVER/sis-sekolah/public/`. Syaratnya:
- `app.baseURL` di `.env` kosong/bertanda `#` (bawaan);
- Windows Firewall mengizinkan **Apache HTTP Server** di jaringan *Private* (biasanya ditanyakan saat Apache pertama kali dijalankan; bila terlewat: *Windows Defender Firewall → Allow an app* → centang Apache HTTP Server);
- IP server sebaiknya tetap (atur IP statis atau *DHCP reservation* di router) agar alamat kiosk dan tautan yang dibagikan tidak berubah;
- cek IP server dengan perintah `ipconfig` (IPv4 Address).

**Opsi 2 – Server bawaan:** di folder `sis-sekolah` jalankan `php spark serve`, lalu buka `http://localhost:8080/`.
Supaya bisa diakses dari perangkat lain di jaringan yang sama, jalankan `php spark serve --host 0.0.0.0` dan buka `http://IP-SERVER:8080/`.

**Menghilangkan `index.php` dari alamat (opsional):** isi `app.indexPage = ''` di `.env`. Pastikan `mod_rewrite` Apache aktif; file `public/.htaccess` sudah disertakan.

---

## 6. Login pertama & pengisian data awal

1. Login dengan **admin / admin123**, lalu **langsung ganti password** lewat menu profil (pojok kanan atas).
2. Buka **Pengaturan Sekolah** dan lengkapi:
   - **Aplikasi:** nama aplikasi (tampil di menu, halaman login, dan tab browser).
   - **Identitas sekolah:** nama sekolah, jenjang, jenis nomor induk (**NPSN** untuk sekolah, **NSM** untuk madrasah) beserta nomornya, alamat, kota/kabupaten (tempat tanggal tanda tangan), telepon, dan email.
   - **Logo:** PNG/JPG/WEBP maksimal 1 MB, sebaiknya persegi berlatar transparan.
   - **Kepala sekolah:** nama lengkap dengan gelar dan NIP. NIP **boleh dikosongkan**; bila kosong, baris NIP tidak dicetak. Untuk jenjang MI/MTs/MA atau bila memakai NSM, jabatan otomatis tertulis "Kepala Madrasah".
   - **Aturan harian:** jam masuk, jam pulang, jumlah hari sekolah, dan batas poin BK.

   Identitas ini dipakai sebagai **kop** dan **tanda tangan** pada semua laporan cetak.
3. Lanjutkan sesuai urutan berikut:

| No | Menu | Isi |
|---|---|---|
| 1 | Pengaturan Akademik → Tahun Pelajaran *(kurikulum)* | buat & aktifkan, misalnya 2026/2027 Ganjil, **lengkap dengan tanggal mulai & selesai semester** |
| 1a | Pengaturan Akademik → Setting Kalender & Libur *(kurikulum)* | libur nasional, cuti bersama, libur semester/fakultatif, ujian, kegiatan sekolah (lihat bagian 8) |
| 2 | Kelas & Rombel | daftar kelas |
| 3 | Guru | centang "Buatkan akun login" |
| 4 | Siswa | Import Excel (unduh templatenya dulu); kolom *Kelas* langsung menempatkan siswa ke rombel |
| 5 | Kelas & Rombel → Rombel | tentukan wali kelas & periksa anggota |
| 6 | Mata Pelajaran, Jam Pelajaran | |
| 7 | Penugasan Guru → Setting Jadwal | jadwal yang bentrok ditolak otomatis |
| 8 | Jadwal Guru Piket | guru otomatis mendapat role *piket* |
| 9 | Pengguna & Role *(admin)* | beri role kurikulum / kesiswaan / bk / kepsek / tu |

---

## 7. Absensi kiosk dengan RFID / barcode

Halaman kiosk dipakai **tanpa login**. Siswa cukup menempelkan kartu RFID atau memindai barcode kartunya, dan hasilnya langsung tercatat di **Absensi Pagi**.

### 7.1 Mengaktifkan kiosk

1. Buka **Pengaturan Sekolah**, klik **Ganti token kiosk**. Alamat kiosk akan muncul, misalnya:
   `http://192.168.1.10/sis-sekolah/public/index.php/kiosk/9f2c…`
2. Buka alamat itu di komputer/tablet kiosk. **Rahasiakan alamat ini**: siapa pun yang memegangnya dapat mencatat kehadiran. Kalau alamatnya bocor, klik *Ganti token kiosk* lagi; alamat lama langsung tidak berlaku.
3. Kiosk bisa dimatikan sementara lewat pengaturan **Absensi kiosk aktif = Nonaktif**.

### 7.2 Perangkat

| Perangkat | Yang perlu dipastikan |
|---|---|
| **Pembaca RFID USB** (125 kHz EM4100 atau 13,56 MHz Mifare) | jenis *USB HID / keyboard emulation* dan mengirim **Enter** setelah UID. Hampir semua reader murah sudah begitu dari pabrik. |
| **Scanner barcode USB** | mode keyboard (HID) dengan akhiran **Enter/CR** (pengaturan bawaan kebanyakan scanner) |
| **Kamera** (tanpa alat) | tombol *Pindai pakai kamera* di halaman kiosk. Hanya berfungsi lewat **HTTPS** atau **localhost**, dan browser harus diizinkan memakai kamera. |

Uji perangkat dengan membuka Notepad lalu menempelkan kartu. Kalau muncul deretan angka/huruf diikuti baris baru, perangkat siap dipakai.

### 7.3 Mendaftarkan kartu siswa

- **Kartu RFID:** buka **Siswa → Edit**, klik kolom **Kode kartu RFID**, tempelkan kartu ke reader, lalu **Simpan**. Untuk banyak siswa sekaligus, isi kolom *Kode RFID* di template Excel lalu import. Satu kartu hanya boleh dipakai satu siswa.
- **Kartu barcode:** tidak perlu didaftarkan karena barcode berisi **NIS**. Cetak lewat **Siswa → Cetak Kartu**, pilih kelas, lalu **Cetak** (ukuran kartu ATM).
- Kiosk juga menerima **NISN**, dan petugas bisa mengetik NIS lewat tombol *Ketik NIS* bila kartu tertinggal.

### 7.4 Cara kiosk mencatat

| Kondisi saat kartu dipindai | Hasil |
|---|---|
| Scan pertama, jam ≤ *Jam Masuk* | **Hadir (H)**, jam datang dicatat |
| Scan pertama, jam > *Jam Masuk* | **Terlambat (T)** |
| Sebelumnya ditandai Sakit/Izin/Alpa, ternyata datang | diubah menjadi H/T, status semula disimpan di keterangan |
| Scan ulang sebelum *Jam Pulang* | hanya informasi "sudah absen" |
| Scan setelah *Jam Pulang* | **jam pulang** dicatat (sekali) |
| Kartu tidak dikenal / siswa belum punya kelas | ditolak, layar merah dengan bunyi panjang |

Siswa yang tidak scan tetap bisa diabsen manual oleh wali kelas atau guru piket di menu **Absensi Pagi**. Baris hasil kiosk ditandai ikon pemindai.

### 7.5 Tips perangkat kiosk

- Jalankan Chrome dalam mode kiosk: buat shortcut dengan target
  `"C:\Program Files\Google\Chrome\Application\chrome.exe" --kiosk "http://alamat-kiosk"`
- Matikan *sleep*/layar mati otomatis di Windows.
- Jam di server menentukan status terlambat, jadi pastikan jam & zona waktu server benar.

---

## 8. Kalender akademik & pekan efektif

Diatur oleh **kurikulum** lewat menu **Pengaturan Akademik → Setting Kalender & Libur**. Semua pengguna bisa melihatnya di menu **Kalender Akademik**.

### 8.1 Jenis agenda dan dampaknya

| Jenis | Bawaan KBM/jurnal | Bawaan absensi | Contoh |
|---|---|---|---|
| Libur Nasional, Cuti Bersama | tidak ada | tidak ada | HUT RI, Idulfitri, cuti bersama SKB |
| Libur Semester | tidak ada | tidak ada | libur akhir semester |
| Libur Fakultatif | tidak ada | tidak ada | Hari Guru, hari jadi daerah, libur keagamaan setempat |
| Ujian | tidak ada | **ada** | PTS, PAS/PAT, ujian sekolah |
| Kegiatan Sekolah | tidak ada | **ada** | class meeting, MPLS, pekan projek |

Setiap agenda punya dua centang yang bisa diubah: **"KBM tetap berjalan"** dan **"Absensi harian tetap diambil"**. Contoh: kegiatan setengah hari yang tetap ada pelajaran → centang keduanya.

### 8.2 Yang otomatis mengikuti kalender

- **Jurnal guru:** hari tanpa KBM tidak menampilkan sesi mengajar, jurnal tidak bisa diisi, dan tidak masuk pengingat "belum diisi".
- **Monitoring jurnal, keterisian jurnal guru, laporan harian piket:** hari tanpa KBM tidak dihitung sebagai sesi yang wajib diisi.
- **Absensi pagi & pembiasaan:** hari tanpa absensi tidak bisa diisi dan diberi keterangan agendanya.
- **Kiosk RFID/barcode:** menolak scan pada hari tanpa absensi, dengan pesan nama liburnya.
- **Laporan absensi kelas & rekap absensi:** kolom hari libur diarsir, nama agenda dicantumkan, dan jumlah hari efektif ditampilkan.
- **Dashboard:** menampilkan status hari ini dan agenda mendatang.
- Di luar periode tahun pelajaran (sebelum tanggal mulai atau sesudah tanggal selesai) dianggap bukan hari sekolah.

Data yang sudah tercatat sebelum suatu tanggal dijadikan libur **tidak dihapus**. Saat menyimpan agenda, aplikasi memberi tahu bila sudah ada jurnal atau absensi pada rentang tersebut.

### 8.3 Rincian pekan efektif

Cetak dari **Cetak Laporan → Rincian Pekan & Hari Efektif** atau dari tombol di halaman kalender. Isinya:

- **Per bulan:** jumlah pekan, pekan tidak efektif, pekan efektif, hari sekolah, dan hari efektif.
- **Hari efektif per hari** (Senin, Selasa, …).
- **JP efektif per mata pelajaran** untuk guru, dalam dua versi: pekan efektif × JP/pekan (cara prota konvensional), dan hitungan menurut jadwal & kalender yang sebenarnya.

Cara hitung: pekan = Senin–Minggu dan masuk ke bulan tempat hari sekolah pertamanya; pekan efektif = pekan dengan minimal satu hari KBM.

> Aplikasi tidak memuat daftar libur nasional secara otomatis. Masukkan libur sesuai SKB 3 Menteri / kalender pendidikan dinas atau Kemenag setempat untuk tahun berjalan.

---

## 9. Cetak laporan

Menu **Cetak Laporan** menampilkan laporan yang sesuai peran masing-masing pengguna. Setelah memilih filter dan menekan **Cetak**, laporan terbuka di tab baru dalam format A4 lengkap dengan kop sekolah dan kolom tanda tangan (kepala sekolah di kiri, pembuat laporan di kanan, beserta kota dan tanggal). Untuk menyimpan sebagai PDF, pilih **Save as PDF / Simpan sebagai PDF** pada dialog cetak browser.

| Laporan | Isi | Yang dapat mencetak |
|---|---|---|
| Jurnal Pembelajaran | agenda mengajar per guru: tanggal, kelas, jam, mapel, materi, status, siswa tidak hadir | guru (miliknya), kurikulum & kepsek (semua guru) |
| Rekap Kehadiran Mapel | matriks siswa × pertemuan untuk satu mapel di satu kelas | guru pengampu, wali kelas (kelasnya), kurikulum, kepsek |
| Absensi Kelas (Bulanan) | **baris nama siswa, kolom tanggal 1–akhir bulan**, jumlah H/T/S/I/A; hari libur diarsir | wali kelas (kelasnya), kesiswaan, BK, piket, kepsek |
| Jurnal Kelas | seluruh KBM di satu kelas (semua mapel & guru) dengan kolom paraf | wali kelas (kelasnya), kurikulum, kepsek |
| Catatan Perilaku Kelas | catatan per siswa beserta total poin; yang melewati batas BK ditandai | wali kelas (kelasnya), kesiswaan, BK, kepsek |
| Kartu Perilaku Siswa | riwayat satu siswa: kehadiran, poin, catatan, tindak lanjut | wali kelas (siswanya), kesiswaan, BK, kepsek; juga dari tombol di Profil Siswa |
| Rekap Pembiasaan | persentase keterlaksanaan pembiasaan per siswa | wali kelas (kelasnya), kesiswaan, BK, kepsek |
| Daftar Siswa | NIS, NISN, L/P, TTL, orang tua | wali kelas (kelasnya), TU, kesiswaan, kurikulum |
| Rekap Poin Perilaku | akumulasi poin seluruh siswa | kesiswaan, BK, kepsek (semua); wali kelas (kelasnya) |
| Rekap Kehadiran Semua Kelas | H/T/S/I/A dan % kehadiran tiap kelas per bulan | kesiswaan, BK, piket, kepsek |
| Keterisian Jurnal Guru | sesi terjadwal vs jurnal terisi per guru | kurikulum, kepsek |
| Layanan BK | daftar layanan BK; isi layanan **rahasia** hanya tercetak untuk konselornya | BK |
| Rincian Pekan & Hari Efektif | pekan/hari efektif per bulan & per hari, JP efektif per mapel | semua pengguna (JP efektif: guru untuk dirinya; kurikulum/kepsek memilih guru) |
| Laporan Harian Piket | rekap kehadiran per kelas, siswa terlambat/tidak hadir, jurnal yang belum diisi | piket, kesiswaan, kepsek |

Admin dapat mencetak semuanya. Kepala sekolah yang tidak memiliki data guru tidak mempunyai "jurnal sendiri", jadi pilih nama guru di formulir. Jumlah kolom tanggal pada absensi kelas mengikuti jumlah hari di bulan yang dipilih (28–31).

---

## 10. Sebelum dipakai sungguhan

- Ubah `CI_ENVIRONMENT = production` di `.env` supaya pesan error teknis tidak tampil. Di mode produksi, akun yang masih memakai password bawaan/mudah ditebak (`123456`, `admin123`, sama dengan username) **wajib** menggantinya sebelum bisa membuka halaman lain.
- Login dibatasi 5 kali gagal per 5 menit per akun & alamat IP.
- Semua tambah/ubah/hapus penting tercatat di **Sistem → Log Aktivitas**.
- Data yang masih dipakai (siswa yang punya absensi, guru yang punya jurnal, dst.) tidak bisa dihapus; ubah statusnya menjadi nonaktif/pindah.
- Jangan jalankan `DemoSeeder` di database sungguhan.

### 10.1 Backup
- Manual: **Sistem → Backup Database → Buat backup** (file `.sql.gz` di `writable/backups`, bisa diunduh).
- Terjadwal (disarankan harian), simpan 14 file terakhir:
  - Linux (cron): `0 18 * * * cd /var/www/sis-sekolah && php spark db:backup --simpan=14`
  - Windows (Task Scheduler): program `C:\xampp\php\php.exe`, argumen `spark db:backup --simpan=14`, *start in* folder aplikasi.
- Salin file backup ke flashdisk/Google Drive secara berkala; backup di server yang sama tidak menolong bila disk rusak.

### 10.2 Tes otomatis
Untuk pengembang: `composer test` atau `vendor/bin/phpunit` menjalankan tes kalender, pekan efektif, jadwal ber-versi, jam per hari, hak akses, upsert absensi, dan pembacaan Excel Dapodik/EMIS (memakai SQLite di memori, tidak menyentuh database sekolah; butuh ekstensi `sqlite3`).

---

## 11. Memperbarui aplikasi

1. Backup database dan folder aplikasi.
2. Timpa semua file **kecuali** `.env` dan folder `writable`. **Hapus dulu folder `app/Database/Migrations`** sebelum mengekstrak: ekstrak zip tidak menghapus berkas lama, dan berkas migration lama yang tertinggal akan bentrok dengan yang baru. Folder itu sekarang hanya berisi `2026-11-22-000000_PeriksaPemasangan.php` dan `2026-11-24-100001` … `100011`.
3. Jalankan `php spark migrate`. Tanpa terminal, jalankan file `database/upgrade_*.sql` yang menyertai pembaruan lewat phpMyAdmin. Pilih salah satu saja, jangan keduanya.
   - **Catatan susunan migration baru:** sejak versi ini migration diringkas (`2026-11-24-100001` … `100011`, didahului pemeriksaan `2026-11-22-000000`). Pemeriksaan ini menghentikan migrate dengan pesan jelas bila masih ada berkas migration lama di folder, atau bila database tujuan sudah berisi tabel lain yang bukan buatan SIS Sekolah (gunakan database kosong). Database yang sudah memakai susunan lama sampai `2026-11-24-000024` dikenali otomatis: `php spark migrate` hanya mencatat versi baru tanpa mengubah tabel/data. Database lama yang **belum** sampai 000024 akan ditolak dengan pesan — jalankan dulu `database/upgrade_v2.sql` (dan file upgrade sebelumnya bila perlu), lalu ulangi `php spark migrate`.
   - Database yang dipasang sebelum fitur kiosk → `database/upgrade_kiosk.sql`, lalu
   - database yang dipasang sebelum fitur identitas sekolah & laporan → `database/upgrade_identitas_laporan.sql`, lalu
   - database yang dipasang sebelum fitur kalender akademik → `database/upgrade_kalender.sql`, lalu
   - database versi 1 (sebelum installer, kehadiran guru, ATP, kokurikuler, reset password, mutasi, portal siswa) → `database/upgrade_v2.sql`.

---

## 11a. Fitur versi 2 secara singkat

| Fitur | Menu | Pengguna |
|---|---|---|
| Jam pelajaran khusus per hari (Jumat pendek, Senin upacara) | Pengaturan Akademik → Jam Pelajaran → tab hari | kurikulum |
| Jadwal ber-versi (perubahan di tengah semester tidak mengubah jurnal lama) | Setting Jadwal → "Berlaku mulai" | kurikulum |
| Salin penugasan, jadwal, wali kelas, piket, rombel dari semester lain | Salin dari Semester Lain | kurikulum |
| Agenda kalender khusus kelas/tingkat & hari **Masuk Pengganti** | Setting Kalender | kurikulum |
| Kehadiran guru (manual, kartu RFID guru di kiosk, otomatis dari izin) | Kehadiran Guru | piket, kurikulum (kepsek melihat) |
| Izin guru + tugas untuk kelas, disetujui kurikulum/kepsek | Izin Guru | guru, kurikulum, kepsek |
| Izin/sakit/dispensasi siswa + foto surat, otomatis mengisi absensi | Izin & Sakit Siswa | wali kelas, kesiswaan, piket |
| Ekspor Excel semua laporan; rekap S/I/A semester untuk e-Rapor | Cetak Laporan → "Unduh Excel" | sesuai hak laporan |
| Impor data dari Excel dengan template (siswa, guru, mapel, kelas, penugasan, kalender, ATP, perilaku, pembiasaan) dan unduhan Dapodik/EMIS | Impor Data Excel | sesuai bidang |
| Mutasi siswa (masuk, keluar, pindah kelas) | Kesiswaan → Mutasi Siswa | TU/admin, kesiswaan |
| Portal siswa/orang tua: jadwal, materi, kehadiran, perilaku, pembiasaan | `/portal`; kelola di Kesiswaan → Akun Portal Siswa | siswa; wali kelas mereset |
| Kenaikan kelas otomatis 1 klik + manual per siswa | Data Master → Kenaikan Kelas | TU/admin |
| ATP & modul ajar; TP dicentang di jurnal; laporan ketercapaian | ATP & Modul Ajar | guru pengampu, kurikulum |
| Kokurikuler (8 dimensi profil lulusan) otomatis masuk kalender | Kokurikuler | kurikulum; koordinator mengisi refleksi |
| Tampilan HP: tabel absensi/jurnal tampil sebagai kartu | – | semua |

**Impor data dari Excel** (menu *Impor Data Excel*, atau tombol *Impor Excel* di halaman Siswa, Guru, Mata Pelajaran, Kelas, Penugasan, Kalender, ATP, Jenis Perilaku, Pembiasaan):
- Setiap jenis data punya **template** (lembar *Data* + lembar *Petunjuk* berisi arti tiap kolom dan contohnya). Isi mulai baris 2, unggah, periksa pratinjau, lalu proses.
- Data yang sudah ada diperbarui, bukan digandakan. Bila satu baris menyebabkan galat, seluruh impor dibatalkan.
- Urutan yang disarankan untuk sekolah baru: Guru → Mata Pelajaran → Kelas (beserta wali) → Siswa → Penugasan → Agenda Kalender → ATP.
- Agenda kalender yang diimpor langsung dipakai menghitung **pekan efektif**; rincian pekan efektif bisa diunduh sebagai Excel dari menu Cetak Laporan.
- Hak impor: siswa/guru/mapel/kelas → TU/admin; penugasan & kalender → kurikulum; ATP → kurikulum; jenis perilaku & pembiasaan → kesiswaan.

**Mutasi siswa** (Kesiswaan → Mutasi Siswa; TU/admin dan kesiswaan): siswa keluar (pindah sekolah, mengundurkan diri, dikeluarkan, meninggal), siswa masuk/pindahan (data lama yang kembali diaktifkan ulang lewat NISN/NIS), dan pindah kelas. Siswa yang keluar langsung hilang dari absensi, jurnal, dan kiosk, tetapi riwayatnya tetap tersimpan. Mutasi terakhir bisa dibatalkan bila salah input. Laporannya ada di Cetak Laporan → *Mutasi Siswa*.

**Kenaikan kelas**:
1. Buat tahun pelajaran baru lalu aktifkan.
2. Siswa yang **tinggal kelas** diproses dulu lewat *Kenaikan Kelas Manual* (per kelas, per siswa).
3. Buka *Kenaikan Kelas Otomatis*: semua kelas langsung dipetakan (VII A → VIII A, 7B → 8B, X IPA 1 → XI IPA 1), kelas tujuan yang belum ada ditawarkan untuk dibuat, tingkat akhir (6/9/12 sesuai jenjang di Pengaturan Sekolah) diluluskan. Periksa, lalu klik **Proses semua kelas** sekali.
4. Siswa yang sudah punya kelas di TP baru dilewati, jadi aman diproses ulang. Untuk Ganjil → Genap, halaman ini otomatis memakai kelas yang sama (pergantian semester).

**Portal siswa** (alamat `.../portal`, tautan juga ada di halaman masuk guru):
- Siswa (bisa bersama orang tua) masuk dengan **NISN atau NIS**. Password awal = **tanggal lahir ddmmyyyy** (mis. 17052013), atau NIS bila tanggal lahir belum diisi; wajib diganti saat pertama masuk.
- Isi portal (hanya melihat): beranda ringkas, jadwal pelajaran kelas, materi & tujuan pembelajaran tiap pertemuan beserta status hadirnya, kalender kehadiran, catatan perilaku & poin, dan pembiasaan per bulan. Catatan internal guru di jurnal, tindak lanjut perilaku, dan layanan BK **tidak** ditampilkan.
- Wali kelas / kesiswaan / TU: menu **Kesiswaan → Akun Portal Siswa** untuk melihat siapa yang sudah masuk, **reset password** siswa yang lupa (kembali ke tanggal lahir), dan **cetak kartu login** per kelas.
- Portal bisa dimatikan di Sistem → Pengaturan Sekolah → Portal Siswa. Siswa yang dimutasi keluar otomatis tidak bisa masuk.

**Pengguna & reset password** (menu Sistem → Pengguna & Role, khusus admin):
- **Reset password** (ikon kunci) membuat password sementara acak, menampilkannya **sekali** dalam daftar siap cetak/Excel, dan memaksa pemilik akun membuat password baru saat login.
- Centang beberapa akun → **Reset password yang dicentang** untuk reset massal (akun Anda sendiri dikecualikan).
- **Buat akun guru** membuatkan akun untuk semua guru aktif yang belum punya akun sekaligus (username dari nama/NIP/NUPTK, role guru, langsung tertaut ke data guru).
- **Cetak password**: membuat kartu akun (nama, alamat aplikasi, username, password) siap gunting untuk semua pengguna atau per peran, bisa juga diunduh Excel. Karena password disimpan terenkripsi, password lama tidak bisa ditampilkan; yang dicetak adalah password sementara baru. Pilihan bawaan hanya mencetak akun yang **belum** mengganti password sendiri, sehingga guru yang sudah punya password pribadi tidak terganggu. Daftar tersedia 10 menit.
- **Buka kunci** bila akun terkunci 5 menit karena salah password berkali-kali; **nonaktifkan** untuk guru yang pindah (pengguna langsung keluar). Akun yang sudah mengisi jurnal/absensi tidak bisa dihapus, cukup dinonaktifkan.
- Sistem menolak menonaktifkan/menghapus/mencabut role admin terakhir, sehingga sekolah tidak terkunci dari aplikasi.
- Pengguna yang lupa password diarahkan di halaman login untuk menghubungi admin.

**Kartu RFID guru:** isi kolom *UID kartu RFID* di Data Master → Guru. Guru menempelkan kartu di kiosk yang sama dengan siswa; jam masuk guru bisa diatur terpisah lewat pengaturan `jam_masuk_guru` (bila kosong memakai jam masuk siswa).

**Impor Dapodik/EMIS:** unggah file unduhan apa adanya. Baris judul dilewati, header dua baris (Data Ayah/Ibu) digabung, NISN yang kehilangan nol di depan dipulihkan, tanggal `dd/mm/yyyy` atau "17 Mei 2013" dikenali, nama kelas "VII A", "7A", "VII-A" dianggap sama. Periksa pemetaan di halaman pratinjau sebelum memproses. Siswa dicocokkan lewat NISN (lalu NIS) sehingga impor ulang memperbarui, bukan menggandakan.

**Sinkron absensi pagi, izin, dan absensi mapel**
- Status absensi mapel diisi otomatis dari data piket/wali kelas: S/I/A di absensi pagi → sama di setiap mapel; **dispensasi** sehari → **D**; **izin keluar/pulang awal** → sesi yang dimulai pada/sesudah jam keluar berstatus sesuai jenis izin (sesi sebelumnya tetap H).
- Izin pulang awal dicatat di Kesiswaan → Izin & Sakit Siswa: pilih waktu *Pulang awal*, isi jam keluar dan penjemput. Absensi pagi siswa tetap **hadir** dengan keterangan "pulang 10:15 (izin)".
- Sinkron berjalan **dua arah waktu**: bila piket mencatat izin atau wali kelas mengoreksi absensi pagi **setelah** guru mengisi jurnal, absensi mapel yang sudah tersimpan ikut diperbarui. Menghapus izin mengembalikannya.
- Status yang **diubah sendiri oleh guru mapel** (mis. siswa terlambat datang lalu masuk kelas) ditandai "diubah guru" dan tidak pernah ditimpa sinkron.
- Di form jurnal, siswa yang terkena izin/dispensasi diberi penanda kuning, mis. "izin pulang 10.15 (dijemput Ibu)".
- Laporan Harian Guru Piket memuat bagian baru: *Siswa Izin Keluar / Pulang Awal & Dispensasi*.

**Izin keluar sekolah & pos gerbang (satpam)**
- Guru piket/kesiswaan/wali kelas menerbitkan izin di Izin & Sakit Siswa dengan pilihan waktu: *Sepanjang hari*, *Pulang awal*, atau *Keluar & kembali* (isi jam keluar dan **batas kembali**).
- Buat akun satpam di Sistem → Pengguna & Role dengan role **Satpam**. Satpam yang login langsung masuk ke **Pos Gerbang**; tampilannya ramah HP.
- Di gerbang, satpam menempelkan kartu siswa (RFID/barcode yang sama dengan kiosk) atau mengetik NIS/nama:
  - hijau **DIIZINKAN** + tombol *Catat KELUAR* bila ada izin hari ini;
  - merah **TIDAK ADA IZIN KELUAR** bila tidak ada — siswa diarahkan ke guru piket.
- Saat siswa kembali, satpam menekan **Sudah KEMBALI**. Status otomatis:
  - kembali sebelum/tepat batas → *Kembali*;
  - terlambat 1–30 menit → *Terlambat kembali*;
  - terlambat lebih dari 30 menit, atau tidak kembali sama sekali → *Membolos* (status berubah sendiri begitu lewat batas + 30 menit, walaupun satpam tidak menekan apa pun).
- Dampaknya ke absensi: absensi pagi tetap **hadir** dengan keterangan "[keluar 08:00–09:00: membolos]"; mapel selama izin = jenis izin (I/S/D); mapel yang dimulai sesudah batas kembali dan sebelum siswa kembali = **A** bila membolos; terlambat ≤ 30 menit tetap H dengan penanda untuk guru.
- Terlambat kembali dan membolos otomatis masuk **Catatan Perilaku** dengan poin dari Referensi → Jenis Perilaku ("Terlambat kembali dari izin keluar" 5 poin, "Membolos …" 15 poin). Bila status dikoreksi, catatan lama diganti; bila izin dihapus, catatan otomatis yang belum ditindaklanjuti ikut terhapus.
- Bila satpam lupa menekan tombol, guru piket/kesiswaan bisa **mengoreksi jam kembali** di Pos Gerbang.
- Pengaturan Sekolah → *Izin Keluar & Pos Gerbang*: batas menit membolos (bawaan 30), verifikasi satpam (matikan bila sekolah tidak punya satpam yang memakai aplikasi; izin lalu berlaku sejak jam yang ditulis piket), dan catatan perilaku otomatis.
- Dengan verifikasi satpam aktif, izin yang tidak pernah dicatat keluar gerbang dianggap *Tidak jadi* — siswa dianggap tetap di sekolah dan absensi mapelnya tidak diubah. Ini berlaku juga untuk izin pulang awal.

**Alur perencanaan: kalender → Promes/Prota → modul ajar**
1. **Kurikulum** mengisi tanggal mulai–selesai semester dan agenda di Kalender Akademik (libur, ujian, kegiatan tanpa KBM), serta jadwal pelajaran.
2. **Guru** membuka Pembelajaran → **Prota & Promes**, memilih mapel–tingkat yang diampu:
   - aplikasi menghitung pekan semester, pekan efektif, dan **JP tersedia tiap pekan** dari jadwal mapel itu di kelasnya (hari libur/ujian otomatis nol);
   - isi butir materi/TP dan JP — atau klik **Ambil TP dari ATP** — lalu **Sebar otomatis ke pekan efektif**. Butir disebar berurutan tanpa melebihi JP tersedia; sebaran bisa diubah manual di matriks (total per pekan diberi warna merah bila berlebih);
   - **Cetak Promes** (matriks bulan–minggu, pekan tidak efektif diarsir, keterangan agenda) dan **Cetak Prota** (gabungan semester ganjil & genap). Keduanya bisa diunduh **Word** atau **Excel**, dan diajukan langsung ke Waka Kurikulum tanpa unggah berkas.
3. Dari butir Promes, guru membuka **Modul Ajar → Buat modul**. Identitas (satuan pendidikan, penyusun, mapel, fase/kelas/semester, tahun pelajaran), topik, tujuan pembelajaran, alokasi JP, dan jumlah pertemuan terisi otomatis.
   - Guru melengkapi identifikasi (peserta didik, materi, dimensi profil lulusan), desain pembelajaran (CP, lintas disiplin, model, metode, pendekatan, kemitraan, lingkungan, pemanfaatan digital, alat & bahan, media, sumber belajar, diferensiasi), asesmen awal/proses/akhir, dan lampiran.
   - Pilih **model pembelajaran** (PBL, PjBL, Discovery, Inquiry, Cooperative, Direct Instruction, CTL) lalu klik **Isi langkah dari sintaks model**: langkah sintaks dipetakan ke tahap *memahami – mengaplikasi – merefleksi* di setiap pertemuan, siap disesuaikan.
   - **Format** mengikuti jenis satuan pendidikan (Pengaturan Sekolah → Format Modul Ajar): sekolah → *Pembelajaran Mendalam*; madrasah (MI/MTs/MA/MAK atau NSM) → *Pembelajaran Mendalam terintegrasi Kurikulum Berbasis Cinta* dengan tambahan Topik Panca Cinta dan materi insersi. Bisa diganti per modul.
   - Cetak/PDF, unduh **Word** untuk disunting, salin modul, dan ajukan semua modul mapel–tingkat itu ke Waka Kurikulum.
4. Waka Kurikulum membuka dokumen buatan aplikasi langsung dari halaman Pemeriksaan Perangkat.

**Perangkat pembelajaran (guru → kurikulum → kepala sekolah)**
- Guru membuka Pembelajaran → **Perangkat Pembelajaran**. Daftar mapel & tingkat diambil otomatis dari penugasan mengajar semester aktif; untuk tiap mapel–tingkat tersedia baris per jenis perangkat. Unggah berkas (PDF/Word/Excel/PowerPoint/gambar/ZIP) atau tempel tautan Google Drive.
- Waka Kurikulum membuka **Pemeriksaan Perangkat**: buka berkas, beri catatan, lalu **Setujui** atau **Perlu revisi** (catatan wajib). Opsi "buka pengajuan berikutnya" mempercepat pemeriksaan beruntun. Perangkat yang sudah disetujui bisa **dibuka kembali** bila perlu diganti.
- Guru melihat catatan kurikulum, mengunggah revisi (versi naik: v2, v3, …). Semua versi lama tetap tersimpan dan bisa dibuka dari riwayat.
- Kepala sekolah mengesahkan perangkat yang sudah disetujui (bisa centang banyak sekaligus).
- Tab *Kelengkapan per Guru* menampilkan jumlah wajib, terkumpul, disetujui, disahkan, dan revisi tiap guru; kurikulum dapat menetapkan **batas pengumpulan** (pengajuan setelahnya ditandai "lewat batas").
- Dashboard memberi pemberitahuan: guru (perlu revisi), kurikulum (menunggu pemeriksaan), kepala sekolah (siap disahkan).
- Cetak Laporan → *Perangkat Pembelajaran*: rekap kelengkapan semua guru (bisa Excel) atau **lembar pengesahan** per guru yang ditandatangani kepala sekolah. Guru dapat mencetak lembar miliknya sendiri.
- **Jenis perangkat** bisa diatur Waka Kurikulum (Pengaturan Akademik → Jenis Perangkat Pembelajaran): ganti nama, wajib/opsional, Tahunan/Semester, nonaktifkan. Bawaan: Analisis CP & TP, ATP, Prota (tahunan); Promes, Modul Ajar/RPP, KKTP, Instrumen Asesmen (semester, wajib); Bahan Ajar/LKPD dan Analisis Hasil Asesmen (opsional). Perangkat *Tahunan* yang sudah dikumpulkan di semester ganjil otomatis terhitung di semester genap.
- Batas unggah 10 MB per berkas, tetapi PHP bawaan XAMPP hanya 2 MB. Untuk berkas besar, ubah `php.ini`: `upload_max_filesize = 10M` dan `post_max_size = 12M`, lalu restart Apache. Batas yang berlaku ditampilkan di formulir unggah.

**Pembiasaan diisi siswa (portal) & pengaturan oleh kesiswaan**
- Referensi Kesiswaan → **Pengaturan Pembiasaan**. Untuk tiap kegiatan diatur:
  - **Diisi oleh**: *Dicatat guru* (wali kelas mencentang di halaman kelas), *Diisi siswa* (siswa mencentang di portal), atau *Guru & siswa*;
  - **Tempat**: di sekolah (selalu hanya hari masuk sekolah) atau di rumah;
  - **Kegiatan di rumah berlaku pada**: *setiap hari*, *hari masuk sekolah* (mengikuti kalender akademik — akhir pekan, libur nasional, dan libur semester tidak muncul; hari ujian tetap dihitung), atau *hari libur saja*. Contoh: bangun pagi, tidur awal, belajar di rumah → hari masuk sekolah; membersihkan rumah → setiap hari; kerja bakti → hari libur saja;
  - **Perlu verifikasi wali kelas** (laporan siswa baru dihitung setelah disetujui), **wajib keterangan** + petunjuknya (mis. "judul buku & halaman");
  - **hari** (termasuk Minggu) dan **tingkat** yang berlaku.
- Batas pengisian siswa: berapa hari ke belakang (0 = hanya hari ini). Tombol **Tambah paket 7 Kebiasaan Anak Indonesia Hebat** membuat 7 kegiatan (bangun pagi, beribadah, berolahraga, makan sehat dan bergizi, gemar belajar, bermasyarakat, tidur cepat) sebagai kegiatan rumah pada hari masuk sekolah yang diisi siswa dan diverifikasi wali kelas. *Tidur cepat* diisi pagi harinya untuk tidur semalam, sehingga tidur Minggu malam (sebelum masuk sekolah) tercatat di hari Senin.
- **Siswa** membuka Portal → Pembiasaan → *Isi pembiasaan*, memilih hari, mencentang kegiatan yang sudah dilakukan, dan menulis keterangan bila diminta. Statusnya terlihat: menunggu wali kelas / sah / ditolak. Yang sudah diputuskan guru tidak bisa diubah siswa.
- **Wali kelas** melihat kotak *Laporan siswa menunggu verifikasi* di halaman Pembiasaan Siswa → *Setujui* atau *Tolak* (bisa sekaligus). Untuk kegiatan *Guru & siswa*, laporan siswa tampil sebagai sel kuning di tabel kelas; menyimpan tabel sekaligus memverifikasinya.
- Rekap dan laporan hanya menghitung data yang sah (dicatat guru atau sudah disetujui).

**Jenis layanan BK** mengikuti Permendikbud No. 111 Tahun 2014 tentang Bimbingan dan Konseling pada Pendidikan Dasar dan Menengah:
- 4 **komponen**: Layanan Dasar, Peminatan & Perencanaan Individual, Responsif, Dukungan Sistem; 4 **bidang**: pribadi, sosial, belajar, karier.
- **Format** menurut jumlah yang dilayani (individual, kelompok, klasikal, kelas besar) dan **media** (tatap muka, daring, telepon/pesan, tertulis).
- 24 jenis layanan/kegiatan siap pilih, dikelompokkan per komponen: bimbingan klasikal/kelompok/kelas besar, orientasi, informasi, penguasaan konten, media informasi; peminatan, perencanaan individual, penempatan & penyaluran; konseling individual/kelompok, konsultasi, mediasi, advokasi, kolaborasi, konferensi kasus, pemanggilan orang tua, kunjungan rumah, alih tangan kasus; aplikasi instrumentasi (AKPD, DCM, sosiometri), himpunan data, tampilan kepustakaan, pengembangan profesi.
- Komponen dan format terisi otomatis saat memilih jenis (tetap bisa diubah). Kolom *pihak terkait* untuk konsultasi, kolaborasi, konferensi kasus, kunjungan rumah, dan alih tangan kasus.
- Dashboard BK, daftar layanan (filter komponen), dan laporan layanan BK menampilkan rekap per komponen, bidang, dan jenis. Data lama tetap terbaca dan otomatis diberi komponen.

---

## 12. Masalah umum

| Gejala | Penyebab & solusi |
|---|---|
| `Class "Locale" not found` / error intl | ekstensi `intl` belum aktif (lihat bagian 1) |
| `Unable to connect to the database` | MySQL belum jalan, atau nama database/username/password di `.env` salah |
| Halaman tanpa tampilan (polos) | tidak ada internet dan aset offline belum diunduh (bagian 5.1), atau `app.baseURL` salah |
| Selalu diarahkan ke `/install` | tabel `users` kosong/database belum terpasang; jalankan installer atau import template |
| "Terlalu banyak percobaan gagal" | tunggu 5 menit, atau admin mengatur ulang password |
| "Sesi formulir sudah kedaluwarsa" | halaman terlalu lama dibiarkan; isian dikembalikan, cukup klik simpan lagi |
| Impor Dapodik: kolom tidak dikenali | pilih pemetaan kolom secara manual di halaman pratinjau; pastikan memilih sheet/berkas unduhan asli (.xlsx) |
| Lampiran surat izin gagal diunggah | maks. 2 MB, format PDF/JPG/PNG; folder `writable/uploads` harus bisa ditulisi |
| Dilempar ke `localhost:8080` | `app.baseURL` di `.env` berisi `http://localhost:8080/`; hapus baris itu atau isi dengan alamat yang benar |
| Setelah login kembali ke halaman login | `app.baseURL` tidak sama dengan alamat yang dibuka; atau folder `writable/session` tidak bisa ditulisi |
| `The action you requested is not allowed` | sesi/CSRF kedaluwarsa: muat ulang halaman lalu kirim lagi |
| Halaman kiosk 404 | token belum dibuat/sudah diganti, atau kiosk dinonaktifkan |
| Kiosk: "Koneksi ke server terputus" | perangkat kiosk tidak bisa menjangkau server; buka kiosk lewat IP server, pastikan firewall mengizinkan Apache dan `app.baseURL` kosong (otomatis) |
| Semua siswa tercatat terlambat / jam selisih | zona waktu salah (bagian 3) atau jam server salah |
| Kamera tidak bisa dibuka di kiosk | wajib HTTPS atau localhost; izinkan akses kamera di browser |
| Reader RFID mengetik tapi tidak tercatat | reader tidak mengirim Enter; ubah setelan reader atau pakai yang mode HID |
| Logo tidak tersimpan / tidak tampil | folder `public/uploads/logo` tidak ada atau tidak bisa ditulisi; ukuran > 1 MB atau bukan PNG/JPG/WEBP |
| Jurnal/absensi tidak bisa diisi, pesan "tidak ada KBM/absensi" | tanggal tsb ditandai libur/ujian/kegiatan di kalender, atau di luar periode tahun pelajaran |
| Rincian pekan efektif tidak bisa dicetak | tanggal mulai & selesai tahun pelajaran aktif belum diisi |
| Laporan tanpa kop yang benar | identitas sekolah belum diisi di Pengaturan Sekolah |
| Hasil cetak terpotong | atur skala "Fit to page/Sesuaikan" dan pastikan ukuran kertas A4 di dialog cetak |
| Import SQL gagal `#1050 Table already exists` | import ke database yang masih berisi tabel; gunakan database kosong |
