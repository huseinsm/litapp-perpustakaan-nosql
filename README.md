# 📚 Lit App — Basis Data Perpustakaan Digital Kampus

> Tugas Akhir **Teknologi Basis Data Non-Relasional (Kelas A)** — FILKOM Universitas Brawijaya

Lit App adalah rancangan **basis data** untuk perpustakaan digital kampus. Proyek ini **hanya berisi database** (tanpa tampilan/front end dan tanpa server aplikasi/back end). Semua database dijalankan dengan **Docker versi ringan (lite)** supaya tetap lancar di laptop dengan RAM terbatas.

Kami memakai **4 database NoSQL sekaligus** (disebut *polyglot persistence*), masing-masing dipakai untuk pekerjaan yang paling cocok dengannya:

| Database | Jenis | Dipakai untuk | Analogi sederhana |
|---|---|---|---|
| **MongoDB** | Document store | Data utama: pengguna, katalog, peminjaman, ulasan | Lemari arsip — tiap map berisi satu data lengkap |
| **Typesense** | Search engine | Pencarian judul/penulis, tahan salah ketik, autocomplete | Petugas yang hafal semua judul buku |
| **Neo4j** | Graph database | Rekomendasi buku ("yang meminjam ini juga meminjam…") | Peta pertemanan antara pembaca dan buku |
| **Redis** | Key-value store | Sesi login, stok cepat, dan cache | Papan tulis kecil di meja depan — cepat dibaca, cepat dihapus |

---

## 👥 Anggota Kelompok

| No | Nama | NIM |
|---|---|---|
| 1 | Muhammad Faza Husain Baihaqi | 245150700111044 |
| 2 | Husein Sidharta Muhammad | 245150707111040 |
| 3 | Reinhard Frano Randalinggi | 245150707111043 |

---

## 🎯 Gambaran Umum

**Pengguna sistem:**

- **Mahasiswa & Dosen (peminjam)** — mencari buku, meminjam, mengembalikan, memberi ulasan dan rating.
- **Admin & Pustakawan (pengelola)** — menambah/mengubah/menghapus katalog dan stok, serta memverifikasi peminjaman dan pengembalian.

**Kenapa tidak pakai database relasional (SQL) biasa?**

1. **Jenis koleksi berbeda-beda.** Buku teks punya *jumlah halaman & penerbit*, skripsi punya *pembimbing & abstrak*. Kalau dipaksa ke satu tabel SQL, akan banyak kolom kosong (NULL) dan tabel harus diubah setiap ada jenis koleksi baru.
2. **Pencarian harus pintar.** Mahasiswa sering salah ketik ("basis dta"). `LIKE '%...%'` di SQL tidak bisa menangani itu dengan cepat.
3. **Rekomendasi butuh menelusuri hubungan.** "Pengguna yang mirip denganmu juga meminjam…" di SQL butuh JOIN bertingkat yang berat; di database graf cukup menelusuri garis hubungan.

---

## 🏗️ Arsitektur: Multi Tenant — Sharding + Master-Slave

Lit App dirancang sebagai layanan **SaaS multi tenant**: satu sistem dipakai bersama oleh banyak kampus (**tenant**). Setiap data diberi penanda `id_kampus` agar data antar-kampus tidak tercampur.

- **Sharding** — data dibelah menjadi beberapa bagian (*shard*) berdasarkan kampus. Tiap shard menyimpan blok data yang berbeda, sehingga beban tidak menumpuk di satu mesin.
- **Master-Slave (Primary-Secondary)** — di setiap shard, **primary (master)** menerima semua penulisan, lalu datanya disalin ke **secondary (slave)** yang melayani pembacaan. Jika primary mati, secondary tetap menyimpan salinan data.

| Database | Bentuk di Lit App |
|---|---|
| **MongoDB** | Koleksi di-shard dengan shard key `id_kampus`; tiap shard berupa *replica set* (1 primary + secondary) |
| **Redis** | 1 master (sesi, stok, cache) + replica yang menyalin data master |
| **Typesense** | Indeks pencarian, disalin secara asinkron dari MongoDB |
| **Neo4j** | 1 instance; rekomendasi dibatasi maksimal 2 langkah (*hop*) dan hasilnya di-cache di Redis |

> Karena dijalankan di laptop dengan **Docker versi lite**, jumlah node dan RAM tiap container dibuat seminimal mungkin, tetapi pola sharding dan master-slave tetap ditunjukkan.

---

## 🗂️ Struktur Data (Kamus Istilah)

> Nama koleksi, atribut, dan kunci dibuat dalam **bahasa Indonesia** agar mudah dipahami. Istilah teknis yang sudah umum (`_id`, `email`, `password`, `rating`, `cache`, dll.) tetap dalam bahasa Inggris.

### 1. MongoDB — database `litapp`

Di MongoDB, "tabel" disebut **koleksi** dan "baris" disebut **dokumen**.

#### Koleksi `pengguna` — data akun anggota perpustakaan

| Atribut | Arti | Contoh |
|---|---|---|
| `_id` | Kode unik pengguna | `"u001"` |
| `id_kampus` | Kampus asal pengguna (sistem bisa dipakai banyak kampus) | `"ub"` |
| `nim_nip` | NIM untuk mahasiswa, NIP untuk dosen/staf | `"245150707111040"` |
| `nama` | Nama lengkap | `"Husein Sidharta Muhammad"` |
| `email` | Email kampus | `"husein@student.ub.ac.id"` |
| `password` | Kata sandi yang sudah di-*hash* (diacak) | `"$2b$10$..."` |
| `peran` | Hak akses: `mahasiswa`, `dosen`, `pustakawan`, `admin` | `"mahasiswa"` |
| `tanggal_daftar` | Kapan akun dibuat | `ISODate("2026-09-01")` |

#### Koleksi `katalog` — semua koleksi pustaka (buku, skripsi, jurnal, e-book, dst.)

| Atribut | Arti | Contoh |
|---|---|---|
| `_id` | Kode unik koleksi | `"b001"` |
| `id_kampus` | Perpustakaan kampus pemilik koleksi | `"ub"` |
| `judul` | Judul | `"Basis Data NoSQL"` |
| `penulis` | Daftar penulis | `["Andi", "Budi"]` |
| `tipe` | Jenis koleksi: `buku_teks`, `novel`, `skripsi`, `jurnal`, `ebook` | `"buku_teks"` |
| `kategori` | Topik/bidang ilmu | `["Basis Data", "Informatika"]` |
| `tahun_terbit` | Tahun terbit | `2024` |
| `stok_total` | Jumlah eksemplar yang dimiliki | `5` |
| `stok_tersedia` | Jumlah yang sedang bisa dipinjam | `3` |
| `detail_khusus` | **Atribut yang berbeda tiap jenis koleksi** (inilah kelebihan skema fleksibel) | lihat contoh di bawah |
| `rata_rata_rating` | Nilai rata-rata ulasan (1–5) | `4.5` |

Contoh `detail_khusus` yang berbeda bentuk dalam satu koleksi:

```json
{ "_id": "b001", "judul": "Basis Data NoSQL", "tipe": "buku_teks",
  "detail_khusus": { "halaman": 320, "penerbit": "Informatika", "isbn": "978-602-..." } }

{ "_id": "s012", "judul": "Sistem Rekomendasi Buku", "tipe": "skripsi",
  "detail_khusus": { "pembimbing": "Bu Rina", "program_studi": "Teknologi Informasi", "abstrak": "..." } }
```

#### Koleksi `peminjaman` — catatan pinjam & kembali

| Atribut | Arti | Contoh |
|---|---|---|
| `_id` | Kode unik transaksi | `"p0001"` |
| `id_kampus` | Kampus tempat transaksi | `"ub"` |
| `peminjam` | Ringkasan data peminjam (id, nama, NIM) — disalin agar tidak perlu JOIN | `{ "id": "u001", "nama": "Husein", "nim_nip": "2451..." }` |
| `koleksi` | Ringkasan data buku yang dipinjam (id, judul) | `{ "id": "b001", "judul": "Basis Data NoSQL" }` |
| `tanggal_pinjam` | Tanggal mulai pinjam | `ISODate("2026-11-20")` |
| `batas_kembali` | Tanggal paling lambat dikembalikan | `ISODate("2026-11-27")` |
| `tanggal_kembali` | Tanggal benar-benar dikembalikan (kosong jika belum) | `null` |
| `status` | `menunggu_verifikasi`, `dipinjam`, `dikembalikan`, `terlambat` | `"dipinjam"` |
| `denda` | Denda keterlambatan (Rupiah) | `0` |
| `diverifikasi_oleh` | Pustakawan yang menyetujui | `"u900"` |

#### Koleksi `ulasan` — ulasan dan rating dari pembaca

| Atribut | Arti | Contoh |
|---|---|---|
| `_id` | Kode unik ulasan | `"r0001"` |
| `id_pengguna` | Siapa yang menulis | `"u001"` |
| `id_koleksi` | Buku yang diulas | `"b001"` |
| `rating` | Nilai 1–5 | `5` |
| `komentar` | Isi ulasan | `"Penjelasannya mudah dipahami"` |
| `tanggal` | Kapan ulasan ditulis | `ISODate("2026-11-22")` |

### 2. Typesense — koleksi `katalog_pencarian`

Salinan **khusus baca** dari koleksi `katalog` MongoDB, hanya berisi atribut yang perlu dicari: `id`, `judul`, `penulis`, `tipe`, `kategori`, `tahun_terbit`, `stok_tersedia`, `rata_rata_rating`.
Dipakai untuk: pencarian tahan salah ketik, saran otomatis saat mengetik (*autocomplete*), dan filter (misal: tipe = skripsi, tahun ≥ 2020).

### 3. Neo4j — graf rekomendasi

**Simpul (node):**

| Label | Arti | Atribut utama |
|---|---|---|
| `Pengguna` | Anggota perpustakaan | `id`, `nim_nip`, `nama` |
| `Buku` | Koleksi pustaka | `id`, `judul`, `tipe` |
| `Kategori` | Bidang ilmu/topik | `nama` |

**Hubungan (relasi):**

| Relasi | Arti |
|---|---|
| `(Pengguna)-[:MEMINJAM {tanggal}]->(Buku)` | Pengguna pernah meminjam buku ini |
| `(Pengguna)-[:MENGULAS {rating}]->(Buku)` | Pengguna memberi rating pada buku |
| `(Buku)-[:TERMASUK]->(Kategori)` | Buku masuk dalam kategori tertentu |

```
(Husein)──MEMINJAM──▶(Basis Data NoSQL)◀──MEMINJAM──(Faza)──MEMINJAM──▶(Sistem Terdistribusi)
                                                                          ▲
                                         rekomendasi untuk Husein ────────┘
```

### 4. Redis — kunci (key) dan isinya

| Pola kunci | Isi | Umur data (TTL) |
|---|---|---|
| `sesi:{token}` | Data sesi login pengguna | 24 jam |
| `stok:{id_koleksi}` | Stok tersedia, dikurangi secara atomik saat dipinjam | permanen |
| `rekomendasi:{nim_nip}` | Hasil rekomendasi dari Neo4j yang disimpan sementara | 6 jam |
| `cari:{hash_kata_kunci}` | Hasil pencarian populer dari Typesense | 10 menit |

---

## 🧱 Struktur Folder

```
.
├── docker-compose.yml     # menjalankan seluruh database (versi lite)
├── mongodb/               # skrip MongoDB: klaster shard, koleksi, index, data contoh
├── typesense/             # skema koleksi pencarian + data contoh
├── neo4j/                 # skrip Cypher: node, relasi, query rekomendasi
├── redis/                 # contoh sesi, stok, cache
└── skenario/              # kueri uji end-to-end (lintas database) untuk laporan
```

---

## 🐳 Cara Menjalankan (Docker Lite)

### Kebutuhan

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows/Mac) atau Docker Engine (Linux)
- Git

### Langkah

```bash
# 1. Ambil repo
git clone https://github.com/<username>/litapp-perpustakaan-nosql.git
cd litapp-perpustakaan-nosql

# 2. Nyalakan semua database
docker compose up -d

# 3. Cek semua sudah berjalan
docker compose ps
```

**Laptop kurang kuat?** Nyalakan hanya service yang sedang dikerjakan, misalnya `docker compose up -d <nama-service>`, lalu `docker compose stop <nama-service>` jika sudah selesai.

### Mematikan

```bash
docker compose down      # matikan, data tetap tersimpan
docker compose down -v   # matikan DAN hapus semua data (mulai dari nol)
```

---

## 🔄 Alur Data Antar-Database

```
            tulis/ubah                 salin (sinkron berkala)
Pustakawan ───────────▶  MongoDB  ───────────────────────────▶  Typesense  ──▶ pencarian
                           │
                           │ setiap peminjaman/ulasan
                           ▼
                         Neo4j  ──── hasil rekomendasi ────▶  Redis (cache 6 jam)
                           
Login ─────────────────────────────────────────────────────▶  Redis (sesi 24 jam)
Pinjam buku ──▶ Redis kurangi stok (cepat) ──▶ MongoDB catat transaksi peminjaman
```

---

## 🧪 Skenario Pengujian

| Kode | Skenario | Database yang terlibat |
|---|---|---|
| TC-01 | Mahasiswa mencari buku (salah ketik), lalu melihat rekomendasi buku terkait | Typesense → MongoDB → Neo4j → Redis |
| TC-02 | Mahasiswa meminjam buku: stok berkurang, transaksi tercatat, graf rekomendasi bertambah | Redis → MongoDB → Neo4j |
| TC-03 | Pustakawan menambah jenis koleksi baru (misal: `jurnal`) tanpa mengubah skema | MongoDB → Typesense |

Kueri untuk tiap skenario ada di folder [`skenario/`](skenario/).

---

## 🛠️ Teknologi

`MongoDB 7` · `Neo4j 5 Community` · `Redis 7 (alpine)` · `Typesense 27` · `Docker Compose`

---

<sub>Dibuat untuk Tugas Akhir mata kuliah Teknologi Basis Data Non-Relasional — FILKOM UB, 2026.</sub>
