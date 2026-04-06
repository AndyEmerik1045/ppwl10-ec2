# Laporan Pengerjaan Proyek: Mono-Docker Monorepo

## 1. Deskripsi Proyek
Proyek ini bertujuan untuk melakukan **Dockerization** pada aplikasi berbasis **Monorepo** yang menggunakan **Bun** sebagai runtime.  
Aplikasi terdiri dari layanan **Frontend (React + Vite)** dan **Backend (Elysia + Prisma)** yang saling terintegrasi dalam satu ekosistem kontainer.

---

## 2. Langkah-Langkah Setup & Arsitektur
Pengerjaan dimulai dengan menyusun struktur folder yang efisien untuk mendukung **Docker Compose**.

### A. Struktur Folder

```text
.
├── apps
│   ├── frontend
│   │   └── (React + Vite app, served with Nginx)
│   │
│   └── backend
│       └── (Elysia API + Prisma ORM + SQLite)
│
├── packages
│   └── shared
│       └── (Shared TypeScript schemas)
│
└── docker-compose.yml
```

Penjelasan:

- **apps/frontend**  
  Berisi kode antarmuka pengguna, menggunakan **Nginx** untuk melayani file statis hasil build.

- **apps/backend**  
  Berisi logika API menggunakan **Elysia** dan **Prisma ORM** dengan database **SQLite (`dev.db`)**.

- **packages/shared**  
  Menyimpan skema **TypeScript** yang digunakan bersama agar tipe data antara Frontend dan Backend tetap sinkron.

---

### B. Konfigurasi Orchestration

Dibuat file **`docker-compose.yml`** untuk menghubungkan kedua layanan:

**Service Backend**
- Menjalankan API pada port **3000**

**Service Frontend**
- Menjalankan **Nginx**
- Port **5173** diarahkan dari port internal **80**

**Network**
- Menggunakan **bridge network** agar Frontend dapat berkomunikasi dengan Backend menggunakan **nama service**.

---

## 3. Modifikasi & Penyesuaian Sistem

Beberapa modifikasi penting dilakukan agar aplikasi dapat berjalan di dalam **kontainer Linux (Alpine)**.

### Nginx Routing
Menambahkan file **`nginx.conf`** khusus untuk menangani **Single Page Application (SPA)**.

Tanpa konfigurasi ini, halaman seperti:

```
/classroom
```

akan mengalami **error 404** saat halaman di-refresh secara manual di browser.

---

### Prisma Client Generation
Alur kerja diubah agar perintah berikut dijalankan **di dalam Dockerfile backend**:

```bash
prisma generate
```

Hal ini memastikan **binary engine Prisma** sesuai dengan sistem operasi kontainer (**Linux**).

---

### Environment Variables
Dilakukan penyesuaian **`DATABASE_URL`** agar tetap valid ketika database **SQLite** di-mount ke dalam **Docker volume**.

---

## 4. Bug Handling & Troubleshooting

Selama proses pengembangan, ditemukan beberapa kendala teknis yang berhasil diatasi.

---

### A. Masalah Arsitektur Prisma Engine

**Kendala**

Backend gagal dijalankan karena **Prisma mencari engine Windows** di dalam kontainer **Linux**.

**Solusi**

- Menambahkan paket **openssl** pada tahap instalasi di Dockerfile
- Menjalankan perintah berikut saat proses build image:

```bash
bunx prisma generate
```

---

### B. Optimasi Build (Dockerignore)

**Kendala**

- Proses build Docker sangat lambat  
- Ukuran image membengkak hingga **bergiga-giga**

**Solusi**

Membuat file **`.dockerignore`** untuk mengecualikan:

```text
node_modules
.git
*.db
```

agar tidak ikut tersalin ke dalam kontainer.

---

### C. Integrasi Google Classroom API

**Kendala**

Data tugas tidak muncul di halaman:

```
/classroom
```

meskipun login berhasil.

**Solusi**

- Memperbaiki mekanisme pengiriman **headers pada backend**
- Menyertakan **Bearer Token** saat melakukan request ke **Google API**
- Memastikan konfigurasi **OAuth Client ID** sudah mengarah ke:

```
http://localhost:5173
```

---

### D. Sinkronisasi Database

**Kendala**

Tabel **user** kosong saat aplikasi pertama kali dijalankan di Docker.

**Solusi**

Menjalankan perintah berikut untuk mensinkronkan skema database:

```bash
docker compose exec backend bunx prisma db push
```

Perintah ini membuat tabel baru di **SQLite database** yang berada di dalam **Docker volume**.

---

## 5. Kesimpulan Hasil Akhir

Proyek berhasil diselesaikan dengan hasil sebagai berikut:

**Frontend**
- Berhasil menampilkan tabel data **user** dari database SQLite.

**Google Integration**
- Berhasil menarik data tugas (**Assignment**) dari **Google Classroom** secara **real-time**.

**Docker Stability**
- Seluruh layanan dapat dijalankan dan dimatikan secara bersamaan dengan perintah:

```bash
docker compose up    ---> menjalankan server (seperti bun dev ini untuk docker)
docker compose down  ---> mematikan dan  membersihkan network dan container dari server
```

tanpa kehilangan data (**data persistence**).

---

## Catatan

Laporan ini dibuat sebagai **dokumentasi teknis pengerjaan proyek** dari tahap inisialisasi hingga penyelesaian bug fungsional.
