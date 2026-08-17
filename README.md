# ERP SSI — Aplikasi Percobaan (1 Bulan)

Aplikasi web internal untuk Spice Solution Indonesia (SSI) — jasa maklon & produksi/penjualan
herb and spices bulky, pasar lokal & ekspor. Dibangun dari desain database yang sama dengan
`Desain_Database_ERP_SSI_DataDictionary.xlsx` (34 tabel, 8 modul).

## Arsitektur singkat

- **Backend**: Flask (Python) — server-rendered (Jinja2), tanpa build step JS (Tailwind via CDN).
- **Database**: abstraksi tunggal yang mendukung SQLite (dev lokal, otomatis, tanpa setup) dan
  PostgreSQL (produksi, mis. Neon/Render Postgres) — aktif otomatis kalau env var `DATABASE_URL` diisi.
- **CRUD generik**: satu mesin CRUD (`app/crud.py` + `app/templates/list.html` & `form.html`) yang
  membaca skema dari `app/db/tables_data.json` (sumber yang sama dgn Data Dictionary Excel) —
  bukan 34 halaman hand-coded terpisah. Field referensi/dropdown (mis. Status PO, Grade Kualitas)
  ditarik dari tabel `reference_values` (hasil normalisasi sheet "Master Data"); field FK ke tabel
  lain (mis. No Buyer di PO) dirender sbg dropdown yang menarik data tabel tujuan.
- **Auth & role**: login berbasis session, password di-hash dgn PBKDF2 (`app/auth_utils.py`, tanpa
  dependency eksternal). 10 role (Admin + 9 role sesuai domain "Departemen" di desain: Produksi,
  PPIC, QC, Gudang, Finance, Purchasing, Sales/Marketing, Ekspor/Logistik, HR). Matriks akses per
  modul ada di `app/permissions.py`.

## Penyederhanaan yang disengaja (MVP 1 bulan)

Karena ini versi percobaan yang dibangun cepat, beberapa hal disederhanakan dari desain penuh
di Data Dictionary Excel (poin-poin ini konsisten dgn sheet "Rekomendasi Penyesuaian" di sana):

1. **Relasi antar tabel bersifat "soft link"** (disimpan sbg kode teks, divalidasi lewat dropdown
   di form), bukan foreign key constraint keras di database — supaya input data lebih fleksibel.
2. **Kolom hasil formula** (Sisa Stok, Rendemen, Nilai PO, dst.) bisa diisi/ubah manual di versi
   ini, belum dihitung otomatis dari kolom lain. Ditandai "(biasanya otomatis/agregat)" di form.
3. Setiap tabel dapat kolom teknis `id` (auto-increment) sbg primary key aplikasi; kolom kode
   bisnis asli (No PO, No Batch, dst) tetap ada sbg kolom biasa + unique index.
4. PO Index, Dashboard, Rekap Pengeluaran (yang di desain asli bersifat VIEW/agregat) di versi
   ini disederhanakan atau digantikan tampilan dashboard bawaan aplikasi.

Semua ini aman untuk masa percobaan; kalau nanti lanjut ke versi produksi jangka panjang,
rujuk sheet "Rekomendasi Penyesuaian" di file Excel utk daftar lengkap perbaikan yang disarankan.

## Menjalankan secara lokal

```bash
cd erp-ssi-app
python3 scripts/seed.py   # buat tabel + isi data referensi + user default (aman diulang)
python3 run.py            # buka http://localhost:5000
```

Tanpa konfigurasi apa pun, aplikasi otomatis pakai SQLite (file `erp_dev.db`) — cocok utk
coba-coba lokal. Untuk pakai Postgres (mis. saat sudah deploy), set env var `DATABASE_URL`.

## Login default

> **Ganti semua password ini sebelum dipakai tim secara nyata.** Ini hanya utk masa percobaan.

| Username     | Password         | Role             | Akses modul |
|--------------|------------------|------------------|-------------|
| `admin`      | `Admin#SSI2026`      | Admin            | Semua modul, full akses |
| `produksi`   | `Produksi#2026`      | Produksi         | Edit: Produksi, WMS. Lihat: lainnya |
| `ppic`       | `Ppic#2026`          | PPIC             | Edit: Penjualan/PO, Produksi, Administrasi. Lihat: lainnya |
| `qc`         | `Qc#2026`            | QC               | Edit: Produksi. Lihat: lainnya |
| `gudang`     | `Gudang#2026`        | Gudang           | Edit: WMS. Lihat: lainnya |
| `finance`    | `Finance#2026`       | Finance          | Edit + Lihat: Pembiayaan, Keuangan Operasional, Penjualan/PO. **Tidak lihat**: HR |
| `purchasing` | `Purchasing#2026`    | Purchasing       | Edit: WMS, Keuangan Operasional. Lihat: lainnya |
| `sales`      | `Sales#2026`         | Sales/Marketing  | Edit: Penjualan/PO, Administrasi. Lihat: lainnya |
| `logistik`   | `Logistik#2026`      | Ekspor/Logistik  | Edit: Administrasi & Legal. Lihat: lainnya |
| `hr`         | `Hr#2026`            | HR               | Edit + Lihat: HR. **Tidak lihat**: Pembiayaan, Keuangan Operasional |

Modul **Pembiayaan (Funding)** dan **Keuangan Operasional** dibatasi hanya utk Admin + Finance
(data sensitif — sesuai rekomendasi role-based access control di Data Dictionary Excel).

## Deploy gratis ke internet (Neon + Render, tanpa kartu kredit)

### 1. Database — Neon (Postgres gratis permanen, bukan trial)
1. Buka https://neon.tech → daftar (bisa pakai akun Google/GitHub, tanpa kartu kredit).
2. Buat project baru → beri nama mis. `erp-ssi`.
3. Di halaman project, salin **Connection string** (format `postgresql://...`).

### 2. Hosting — Render (750 jam gratis/bulan, cukup utk 1 aplikasi nonstop 1 bulan)
1. Push folder ini ke repository GitHub (bisa private).
2. Buka https://render.com → daftar (bisa pakai akun GitHub).
3. **New** → **Web Service** → hubungkan repo GitHub tadi.
4. Render otomatis mendeteksi `render.yaml` di repo ini — cukup klik **Apply**.
5. Saat diminta, isi env var `DATABASE_URL` dengan connection string dari Neon (langkah 1).
6. Deploy pertama akan otomatis menjalankan `scripts/seed.py` (bikin tabel + data referensi +
   user default) sebelum aplikasi jalan.
7. Setelah selesai, Render kasih URL publik (`https://erp-ssi-app-xxxx.onrender.com`) — itu
   alamat aplikasinya.

Catatan: web service gratis Render tidur otomatis setelah ±15 menit tanpa aktivitas, dan perlu
~30-60 detik utk "bangun" lagi saat diakses — normal utk tier gratis, cukup wajar utk pemakaian
trial internal.

### Kalau tidak mau pakai GitHub
Render & platform sejenis pada umumnya perlu source dari Git repository utk auto-deploy. Kalau
kamu belum punya akun GitHub, bikin dulu di https://github.com/signup (gratis), lalu push folder
`erp-ssi-app` ini ke repo baru sebelum lanjut ke langkah Render di atas.

## Struktur folder

```
erp-ssi-app/
├── app/
│   ├── db/
│   │   ├── tables_data.json          # sumber tunggal skema (sama dgn Data Dictionary Excel)
│   │   ├── master_data_dropdowns.json# 57 domain nilai referensi (dari sheet Master Data)
│   │   ├── build_registry.py         # ubah json -> registry siap pakai (field, tipe, FK, dst)
│   │   ├── ddl.py                    # generate CREATE TABLE (SQLite & Postgres)
│   │   └── conn.py                   # lapisan akses DB (dual dialect)
│   ├── templates/                    # base, login, dashboard, list, form
│   ├── static/css/app.css
│   ├── auth.py, auth_utils.py, permissions.py, crud.py, dashboard.py
│   └── module_colors.py
├── scripts/
│   ├── seed.py                       # bikin tabel + isi referensi + user default
│   └── schema_postgres.sql           # DDL Postgres siap pakai (sudah divalidasi via psql lokal)
├── requirements.txt / Procfile / render.yaml / .env.example
└── run.py                            # entrypoint (dev server / gunicorn via `run:app`)
```

## Verifikasi yang sudah dilakukan

- DDL Postgres divalidasi langsung terhadap PostgreSQL 16 asli (`psql`) — 36 tabel + index
  berhasil dibuat tanpa error.
- Alur fungsional diuji end-to-end via SQLite lokal: login, lihat dashboard, cari data, tambah,
  ubah, hapus baris, dropdown referensi & FK terisi otomatis dari tabel terkait, serta pembatasan
  akses per role (role HR ditolak saat mencoba membuka modul Pembiayaan, dsb).
- **Belum diuji**: jalur Python ke PostgreSQL (`psycopg2`) — sandbox pengembangan ini tidak
  mengizinkan instalasi paket baru (`pip`/`npm`/`apt` diblokir kebijakan jaringan), jadi driver
  Postgres utk Python tidak bisa dipasang & dites lokal. Kode ditulis mengikuti pola standar
  `psycopg2`, dan `requirements.txt` akan otomatis terpasang penuh saat build di Render (server
  build Render punya akses internet normal). Disarankan pantau log Render setelah deploy pertama.
