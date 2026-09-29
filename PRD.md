# PRD — Sistem Manajemen Tiket IT Rumah Sakit

**Versi:** 1.0
**Tanggal:** 25 September 2026
**Disusun oleh:** NiceKleng

---

## 1. Latar Belakang

Sebagai programmer yang me-maintain SIMRS dan aplikasi lain di rumah sakit, pekerjaan sering menjadi berantakan karena:
- Aduan, permintaan penyesuaian alur, laporan bug, dan permintaan aplikasi baru masuk secara tidak terstruktur (lisan, WA pribadi, dsb).
- Tidak ada pencatatan progres yang bisa dipantau pihak lain.
- Atasan tidak memiliki visibilitas terhadap beban kerja dan status pekerjaan.

## 2. Tujuan

1. Menyediakan satu kanal resmi bagi staf untuk mengajukan permintaan (bug, penyesuaian alur, fitur baru, aplikasi baru).
2. Mengelola dan melacak setiap permintaan sebagai tiket, dikelompokkan per aplikasi.
3. Mencatat histori pekerjaan secara otomatis dan terstruktur (bukan laporan manual terpisah).
4. Memberikan visibilitas progres kerja kepada atasan tanpa perlu menanyakan langsung.

## 3. Target Pengguna

| Peran | Login? | Deskripsi |
|---|---|---|
| **Pelapor (Staf)** | Tidak | Staf rumah sakit yang mengajukan tiket. Identifikasi lewat nama + nomor WA, tanpa akun. |
| **Developer** | Ya | Pengelola tiket — kamu (dan developer lain jika ada di masa depan). |
| **Atasan** | Ya | Memantau progres pekerjaan secara read-only. |

## 4. Ruang Lingkup (Scope)

**Masuk dalam scope v1:**
- Form pengajuan tiket publik (tanpa login) dengan keterangan wajib + foto wajib.
- Manajemen tiket per aplikasi oleh developer.
- Perubahan status tiket dengan catatan pekerjaan wajib di setiap perubahan.
- Riwayat/log aktivitas otomatis per tiket.
- Dashboard read-only untuk atasan.
- Halaman lacak status tiket publik (nomor tiket + no WA).

**Di luar scope v1 (dikerjakan menyusul):**
- Integrasi WA gateway (dibangun terpisah oleh NiceKleng, sistem ini hanya menyediakan hook/trigger).
- Notifikasi email.
- Role granular (Spatie Permission) — cukup enum peran sederhana dulu.
- SLA otomatis / eskalasi.

## 5. Alur Pengguna (User Flow)

### 5.1 Staf mengajukan tiket
1. Staf membuka form publik → pilih aplikasi, isi judul, keterangan (wajib, min. 10 karakter), pilih jenis & prioritas, upload foto (wajib).
2. Staf isi nama & no WA.
3. Sistem cek `no_wa` di tabel `pelapor` — kalau sudah ada, pakai record lama (`firstOrCreate`); kalau belum, buat baru.
4. Sistem generate `nomor_tiket` unik (format: `TKT-YYYYMMDD-XXXX`).
5. Staf menerima nomor tiket untuk pelacakan mandiri di halaman `/lacak/{nomor_tiket}` (konfirmasi dengan no WA).

### 5.2 Developer mengelola tiket
1. Developer melihat daftar/papan tiket dikelompokkan per aplikasi.
2. Developer assign tiket ke diri sendiri, atur prioritas.
3. Setiap kali mengubah status, developer **wajib** mengisi catatan berisi apa yang sudah dikerjakan (min. 10 karakter) — tidak bisa mengubah status tanpa catatan.
4. Setiap perubahan status otomatis tercatat ke `riwayat_tiket` (status lama, status baru, siapa, kapan, catatan).

### 5.3 Atasan memantau
1. Atasan login, melihat dashboard read-only.
2. Bisa filter per aplikasi, per status, per rentang tanggal.
3. Bisa membuka detail tiket dan melihat timeline riwayat pekerjaan lengkap.
4. Tidak bisa mengedit apa pun.

## 6. Skema Database

### 6.1 `users` — developer & atasan (login)
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint | PK |
| nama | string | |
| email | string | unique |
| password | string | |
| peran | enum | `developer`, `atasan` |
| dibuat_pada, diperbarui_pada | timestamp | |

### 6.2 `application`
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint | PK |
| nama | string | |
| deskripsi | text | nullable |
| aktif | boolean | default true |
| dibuat_pada, diperbarui_pada | timestamp | |

### 6.3 `pelapor` — staf tanpa akun
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint | PK |
| nama | string | |
| no_wa | string | index, dipakai untuk WA gateway & pelacakan |
| dibuat_pada, diperbarui_pada | timestamp | |

### 6.4 `tiket`
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint | PK |
| nomor_tiket | string | unique, format `TKT-YYYYMMDD-XXXX` |
| aplikasi_id | FK → aplikasi | |
| pelapor_id | FK → pelapor | |
| ditugaskan_ke | FK → pengguna | nullable |
| judul | string | |
| keterangan | text | wajib |
| jenis | enum | `bug`, `penyesuaian_alur`, `permintaan_fitur`, `aplikasi_baru` |
| prioritas | enum | `rendah`, `sedang`, `tinggi`, `kritis` (default `sedang`) |
| status | enum | `baru`, `ditinjau`, `dikerjakan`, `pengujian`, `selesai`, `ditolak`, `ditahan` (default `baru`) |
| tanggal_target | date | nullable |
| selesai_pada | timestamp | nullable |
| dibuat_pada, diperbarui_pada | timestamp | |

### 6.5 `lampiran_tiket`
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint | PK |
| tiket_id | FK → tiket | cascade delete |
| path_file | string | |
| dibuat_pada | timestamp | |

### 6.6 `riwayat_tiket` — log progres otomatis + catatan
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint | PK |
| tiket_id | FK → tiket | cascade delete |
| pengguna_id | FK → pengguna | nullable, siapa yang melakukan aksi |
| aksi | enum | `perubahan_status`, `penugasan`, `perubahan_prioritas`, `catatan` |
| nilai_lama | string | nullable |
| nilai_baru | string | nullable |
| catatan | text | **wajib diisi saat aksi = perubahan_status** |
| dibuat_pada | timestamp | |

### 6.7 Diagram Relasi (ringkas)

```
pengguna 1---* tiket (ditugaskan_ke)
pengguna 1---* riwayat_tiket

aplikasi 1---* tiket

pelapor 1---* tiket

tiket 1---* lampiran_tiket
tiket 1---* riwayat_tiket
```

## 7. Kebutuhan Fungsional Utama

| ID | Kebutuhan |
|---|---|
| F-01 | Staf dapat mengajukan tiket tanpa login, wajib isi keterangan & foto |
| F-02 | Sistem menghasilkan nomor tiket unik untuk pelacakan |
| F-03 | Staf dapat melacak status tiket lewat nomor tiket + konfirmasi no WA |
| F-04 | Developer dapat melihat & mengelola tiket per aplikasi |
| F-05 | Developer wajib mengisi catatan pekerjaan setiap kali mengubah status tiket |
| F-06 | Sistem mencatat setiap perubahan status/penugasan/prioritas ke riwayat otomatis |
| F-07 | Atasan dapat melihat dashboard progres read-only, difilter per aplikasi/status/tanggal |
| F-08 | Atasan dapat membuka detail tiket dan melihat timeline riwayat pekerjaan |

## 8. Kebutuhan Non-Fungsional

- **Keamanan akses publik:** form pengajuan & halaman lacak perlu proteksi dari penyalahgunaan (captcha atau rate limit berdasarkan `no_wa`).
- **Privasi pelacakan:** halaman `/lacak/{nomor_tiket}` mengharuskan konfirmasi no WA agar tidak bisa dilihat sembarang orang.
- **Ekstensibilitas:** struktur `riwayat_tiket` dan enum `peran` dirancang agar mudah menambah developer lain atau menambah jenis aksi di kemudian hari.
- **Integrasi masa depan:** setiap perubahan status idealnya memicu hook/event yang nanti bisa dikonsumsi oleh WA gateway custom (dibangun terpisah).

## 9. Stack Teknis

- **Framework:** Laravel (starter kit Livewire)
- **Frontend interaktif:** Livewire + Alpine.js
- **Database:** MySQL/MariaDB (nama tabel & kolom dalam Bahasa Indonesia)
- **Penyimpanan file:** Laravel Storage (lokal atau S3-compatible), wajib `storage:link`

## 10. Urutan Pengembangan (Roadmap)

1. Setup Laravel + Livewire starter kit, migration seluruh tabel di atas
2. Model & relasi Eloquent
3. Form publik pengajuan tiket (tanpa login) + generate nomor tiket
4. Halaman lacak tiket publik
5. Autentikasi developer & atasan (peran)
6. Papan/daftar tiket developer per aplikasi + assign
7. Fitur ubah status dengan catatan wajib + riwayat otomatis
8. Dashboard atasan (read-only, filter, timeline)
9. Hook/event untuk integrasi WA gateway (disiapkan, integrasi menyusul)

## 11. Pertanyaan Terbuka (Open Items)

- Apakah urutan status perlu divalidasi ketat (tidak boleh lompat, misal dari `baru` langsung ke `selesai`)?
- Apakah satu tiket boleh punya lebih dari satu foto lampiran?
- Apakah developer lain (selain NiceKleng) akan ditambahkan di v1 atau menyusul?
