# SIS Sekolah – Administrasi Pembelajaran (CodeIgniter 4)

Aplikasi administrasi pembelajaran: setting jadwal, jurnal mengajar + absensi mapel,
absensi pagi, pembiasaan siswa, catatan perilaku (poin), dan layanan Guru BK.

## Yang baru di versi 2

Halaman instalasi otomatis (`/install`), aset tampilan offline, backup & log aktivitas, batas percobaan login, jam pelajaran per hari, jadwal ber-versi, salin semester, kalender per kelas & masuk pengganti, kehadiran & izin guru (RFID guru di kiosk), izin siswa dengan lampiran, ekspor Excel + rekap e-Rapor, impor Dapodik/EMIS, ATP & modul ajar, kokurikuler, tampilan HP, dan tes PHPUnit. Rincian di `PANDUAN_INSTALASI.md` bagian 11a.

## Tampilan

Tema sendiri di atas Bootstrap 5.3 (`public/assets/css/sis.css`): biru dongker dengan aksen emas, huruf *Plus Jakarta Sans*, menu samping berkelompok (dropdown) yang otomatis membuka grup halaman aktif, dan logo aplikasi `public/assets/img/logo-sis.svg`. Logo sekolah dari Pengaturan Sekolah tampil di menu, halaman masuk, dashboard, kiosk, dan kop laporan. Warna bisa diubah lewat variabel di bagian `:root` file CSS tersebut.

## Instalasi

Panduan lengkap (XAMPP/Laragon, konfigurasi, kiosk RFID/barcode, troubleshooting): **[PANDUAN_INSTALASI.md](PANDUAN_INSTALASI.md)**

Ringkas:
1. Salin folder `vendor` lama atau `composer install`; salin `env.contoh` → `.env` dan sesuaikan.
2. Buat database kosong, lalu **salah satu**:
   - phpMyAdmin → Import `database/sis_sekolah_template.sql`, atau
   - `php spark migrate` (sekali jalan: semua tabel + data awal)
3. Login **admin / admin123** → ganti password.

File SQL di folder `database/`:

| File | Kegunaan |
|---|---|
| `sis_sekolah_template.sql` | instalasi baru: struktur lengkap + data awal (setara migrate + DatabaseSeeder) |
| `upgrade_kiosk.sql` | menambah fitur kiosk ke database v2 yang sudah terpasang (setara migration 000008) |
| `upgrade_identitas_laporan.sql` | menambah pengaturan identitas sekolah untuk kop laporan (setara migration 000009) |
| `upgrade_kalender.sql` | menambah tabel kalender akademik (setara migration 000010) |

## Kalender akademik

Kurikulum mengatur tahun pelajaran (dengan periode semester) dan agenda: libur nasional, cuti bersama, libur semester,
libur fakultatif, ujian, kegiatan sekolah. Tiap agenda menentukan apakah KBM/jurnal dan absensi tetap berjalan.
Kalender otomatis dipakai oleh jurnal guru, monitoring, absensi pagi, pembiasaan, kiosk, dashboard, dan laporan;
plus laporan rincian pekan/hari efektif & JP efektif per mapel. Logika di `app/Libraries/Kalender.php`.

## Laporan cetak

Menu **Cetak Laporan** (A4, kop sekolah, tanda tangan kepala sekolah & pembuat). Laporan yang muncul menyesuaikan peran:
guru → jurnal pembelajaran & rekap kehadiran mapel; wali kelas → absensi kelas bulanan (matriks tanggal × siswa), jurnal kelas,
catatan perilaku kelas/siswa, pembiasaan, daftar siswa; kesiswaan/BK/kepsek → rekap kehadiran semua kelas & rekap poin;
kurikulum → keterisian jurnal guru; BK → layanan BK; piket → laporan harian piket. Rincian di PANDUAN_INSTALASI.md bagian 9.

## Absensi kiosk (RFID / barcode)

Halaman tanpa login di `/kiosk/{token}` (alamatnya ada di Pengaturan Sekolah). Kode yang diterima: UID kartu RFID
(`siswa.rfid`), NIS, atau NISN. Scan pertama = datang (H/T menurut jam masuk), scan setelah jam pulang = pulang.
Kartu barcode NIS dicetak dari menu Siswa → Cetak Kartu. Detail di PANDUAN_INSTALASI.md bagian 7.

## Memindahkan data dari database lama

Database lama **tidak diubah**; data disalin ke database baru.

1. Buat database kosong baru, mis. `db_sis_sekolah_v2`.
2. Di `.env`:
   ```
   database.default.database = db_sis_sekolah_v2
   database.lama.database    = db_sis_sekolah
   ```
3. Jalankan:
   ```bash
   php spark migrate
   php spark db:seed ImporDataLamaSeeder
   ```
4. Login memakai **password lama** (termasuk admin – password `admin123` dari seeder ditimpa password lama).

Yang dipindahkan: akun & role (waka_kurikulum → kurikulum, waka_kesiswaan → kesiswaan), tahun pelajaran,
jurusan, kelas + wali kelas, mapel, jam pelajaran, siswa + rombel (nomor absen urut nama), penugasan guru,
jadwal, jurnal & presensi (bila ada). Akun ber-role guru otomatis dibuatkan data guru (NIP diambil dari username 18 digit).

Setelah impor, periksa: jenis kelamin & status kepegawaian guru (default L/Honorer), NIS siswa (sementara = NISN),
dan JP/minggu penugasan (default 2). Akun login siswa tidak dipindahkan.

## Urutan pengisian data awal

1. **Pengaturan Sekolah** – nama sekolah, jam masuk, 5/6 hari sekolah, batas poin BK.
2. **Tahun Pelajaran** (menu kurikulum) – buat & aktifkan dengan tanggal mulai/selesai semester, lalu isi **Kalender Akademik** (libur, ujian, kegiatan).
3. **Kelas**, **Guru** (centang "buatkan akun login"), **Siswa** (bisa import Excel – kolom *Kelas* langsung menempatkan siswa ke rombel).
4. **Kelas & Rombel** – tentukan wali kelas dan anggota kelas.
5. **Mata Pelajaran**, **Jam Pelajaran**.
6. **Penugasan Guru** (guru – mapel – kelas – JP/minggu), lalu **Setting Jadwal** per kelas (klik slot kosong; bentrok kelas & guru ditolak otomatis).
7. **Jadwal Guru Piket**, **Daftar Pembiasaan**, **Jenis Perilaku & Poin**.

## Alur harian

| Siapa | Kegiatan | Menu |
|---|---|---|
| Wali kelas / guru piket | Absensi pagi (H/T/S/I/A) | Absensi Pagi |
| Guru mapel | Isi jurnal + absensi siswa per sesi. Status S/I/A dari absensi pagi terisi otomatis. Jam berurutan (mis. jam 1–2) = 1 jurnal | Jurnal & Absensi Mapel |
| Wali kelas / piket | Checklist pembiasaan | Pembiasaan Siswa |
| Semua guru | Catat perilaku positif/negatif, bisa langsung *rujuk ke BK* | Catatan Perilaku |
| Guru | Prota & Promes otomatis dari kalender akademik + jadwal; modul ajar (Pembelajaran Mendalam / + KBC untuk madrasah) dari butir Promes, cetak/Word | Prota & Promes, Modul Ajar |
| Guru → Kurikulum → Kepsek | Unggah perangkat pembelajaran per mapel & tingkat → diperiksa (setujui/revisi) → disahkan; rekap kelengkapan & lembar pengesahan | Perangkat Pembelajaran |
| Guru BK | Terima rujukan → catat 24 jenis layanan BK per komponen Permendikbud 111/2014 (dasar, peminatan & perencanaan individual, responsif, dukungan sistem), rahasia per konselor | Bimbingan Konseling |
| Satpam | Cek izin keluar siswa di gerbang (scan kartu/NIS/nama), catat keluar & kembali; terlambat kembali > 30 menit otomatis membolos | Pos Gerbang |
| Kurikulum / kepsek / piket | Pantau keterisian jurnal; piket/kurikulum bisa mengisi sebagai guru pengganti | Monitoring Jurnal |

## Hak akses

Seluruh aturan ada di satu file: `app/Libraries/Akses.php` (dipakai oleh controller, menu, dan tombol).
Role & status akun dibaca ulang setiap request, jadi perubahan role/penonaktifan langsung berlaku.

| Fitur | admin | kurikulum | kesiswaan | bk | piket | wali kelas | guru | kepsek | tu |
|---|---|---|---|---|---|---|---|---|---|
| Data master (siswa, guru, kelas, mapel, kenaikan) | ✔ | | | | | | | | ✔ |
| Pengguna & pengaturan sekolah (identitas, logo, kepala sekolah) | ✔ | | | | | | | | |
| Tahun pelajaran, kalender akademik, jam pelajaran, penugasan, jadwal, jadwal piket | ✔ | ✔ | | | | | | | |
| Lihat kalender akademik & rincian pekan efektif | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Lihat jadwal | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Isi jurnal & absensi mapel sendiri | ✔ | | | | | | ✔ | | |
| Isi jurnal sebagai guru pengganti | ✔ | ✔ semua tanggal | | | ✔ hari piketnya | | | | |
| Monitoring jurnal | ✔ | ✔ | | | ✔ | | | lihat | |
| Riwayat & rekap absensi mapel semua guru | ✔ | ✔ | | | | kelasnya | miliknya | lihat | |
| Absensi pagi – isi | ✔ | | semua kelas | | semua kelas, hari piketnya | kelasnya | | | |
| Absensi pagi – lihat/rekap | ✔ | | ✔ | ✔ | ✔ | kelasnya | | ✔ | |
| Pembiasaan – isi | ✔ | | semua kelas | | | kelasnya | | | |
| Pembiasaan – lihat/rekap | ✔ | | ✔ | ✔ | | kelasnya | | ✔ | |
| Daftar pembiasaan & jenis perilaku/poin | ✔ | | ✔ | | | | | | |
| Catat perilaku | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Lihat catatan perilaku | ✔ | miliknya | semua | semua | miliknya | kelasnya + miliknya | miliknya | semua | miliknya |
| Ubah status / tindak lanjut | ✔ | miliknya | semua | semua | miliknya | kelasnya + miliknya | miliknya | | miliknya |
| Edit / hapus catatan | ✔ | miliknya | semua | miliknya | miliknya | miliknya | miliknya | | miliknya |
| Rekap poin | ✔ | | ✔ | ✔ | | kelasnya | | ✔ | |
| Profil siswa | ✔ penuh | dasar | penuh | penuh | dasar | penuh (kelasnya) | dasar (kelas yang diajar) | penuh | dasar |
| Modul BK | ✔ | | | ✔ | | | | | |
| Buka layanan BK rahasia | ✔ | | | konselornya | | | | | |
| Ubah / hapus layanan BK | ✔ | | | konselornya | | | | | |

*Profil dasar* = identitas tanpa alamat/kontak orang tua, kehadiran, dan catatan perilaku buatannya sendiri.
*Wali kelas* bukan role: ditentukan di menu Kelas & Rombel. *Piket*: role diberikan otomatis saat guru dijadwalkan
di menu Jadwal Guru Piket dan dicabut bila jadwalnya dihapus semua; akses "semua kelas" hanya berlaku pada hari piketnya.
Satu akun boleh memiliki beberapa role; hak aksesnya adalah gabungan.

## Struktur

```
app/Controllers/   Auth, Dashboard, Profil, KioskController (absensi tanpa login), LaporanController (cetak)
  Master/          TahunPelajaran, Jurusan, Kelas, Guru, Siswa, Mapel, KenaikanKelas, Pengguna, Pengaturan
  Akademik/        JamPelajaran, Penugasan, Jadwal, Piket, Jurnal
  Kesiswaan/       PresensiHarian, Pembiasaan, Perilaku, JenisPerilaku, ProfilSiswa
  Bk/              BkController
app/Models/        24 model
app/Database/Migrations/  10 file migrasi
database/          template & upgrade SQL
app/Filters/       AuthFilter, RoleFilter (route filter 'role:a,b')
app/Libraries/Akses.php   aturan hak akses terpusat
app/Libraries/Kalender.php  status hari (sekolah/KBM/absensi) & pekan efektif
app/Helpers/sis_helper.php
app/Views/         layouts, partials, master, akademik, kesiswaan, bk
```
