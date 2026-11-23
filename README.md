# Lit App

Basis data untuk perpustakaan digital kampus. Tugas akhir mata kuliah Teknologi Basis Data Non-Relasional (Kelas A), FILKOM Universitas Brawijaya.

Repo ini hanya berisi database. Tidak ada front end maupun back end.

## Anggota

| Nama | NIM |
|---|---|
| Muhammad Faza Husain Baihaqi | 245150700111044 |
| Husein Sidharta Muhammad | 245150707111040 |
| Reinhard Frano Randalinggi | 245150707111043 |

## Database yang dipakai

| Database | Kegunaan |
|---|---|
| MongoDB | Data utama: pengguna, katalog, peminjaman, ulasan |
| Typesense | Pencarian katalog (tetap ketemu walau salah ketik, ada autocomplete) |
| Neo4j | Rekomendasi buku berdasarkan riwayat pinjam pengguna lain |
| Redis | Sesi login, stok, dan cache |

Alasan memakai beberapa database sekaligus:

- Jenis koleksi berbeda-beda atributnya. Buku teks punya penerbit dan jumlah halaman, skripsi punya pembimbing dan abstrak. Di MongoDB semuanya bisa disimpan di satu koleksi tanpa kolom kosong.
- Pencarian dengan `LIKE` di SQL lambat dan tidak bisa menangani salah ketik. Typesense dibuat untuk itu.
- Rekomendasi seperti "yang meminjam buku ini juga meminjam..." lebih ringan dijalankan di graf daripada dengan JOIN bertingkat.

## Arsitektur

Sistem ini multi tenant: satu sistem dipakai banyak kampus. Setiap data punya `id_kampus` supaya data antar-kampus terpisah.

- **Sharding**: data MongoDB dibagi ke beberapa shard berdasarkan `id_kampus`.
- **Master-slave**: setiap shard punya satu primary untuk menulis dan secondary yang menyalin datanya untuk membaca. Redis juga memakai satu master dan satu replica.
- Typesense mengambil salinan katalog dari MongoDB. Neo4j berjalan sebagai satu instance.

Semua dijalankan dengan Docker versi ringan karena tidak semua laptop anggota kuat. Jumlah node dan batas RAM tiap container dibuat sekecil mungkin.

## Struktur data

### MongoDB

**pengguna**: akun anggota perpustakaan

| Atribut | Keterangan |
|---|---|
| `_id` | Kode pengguna |
| `id_kampus` | Kampus asal |
| `nim_nip` | NIM (mahasiswa) atau NIP (dosen/staf) |
| `nama` | Nama lengkap |
| `email` | Email kampus |
| `password` | Kata sandi yang sudah di-hash |
| `peran` | `mahasiswa`, `dosen`, `pustakawan`, atau `admin` |

**katalog**: buku, skripsi, jurnal, e-book, dan lainnya

| Atribut | Keterangan |
|---|---|
| `_id` | Kode koleksi |
| `id_kampus` | Perpustakaan pemilik |
| `judul` | Judul |
| `penulis` | Daftar penulis |
| `tipe` | Jenis koleksi, misalnya `buku_teks` atau `skripsi` |
| `kategori` | Bidang ilmu |
| `tahun_terbit` | Tahun terbit |
| `stok_total` | Jumlah eksemplar |
| `stok_tersedia` | Jumlah yang bisa dipinjam |
| `detail_khusus` | Atribut yang berbeda untuk tiap jenis koleksi |

Contoh `detail_khusus`:

```json
{ "_id": "b001", "tipe": "buku_teks", "judul": "Basis Data NoSQL",
  "detail_khusus": { "halaman": 320, "penerbit": "Informatika" } }

{ "_id": "s012", "tipe": "skripsi", "judul": "Sistem Rekomendasi Buku",
  "detail_khusus": { "pembimbing": "Bu Rina", "abstrak": "..." } }
```

**peminjaman**: catatan pinjam dan kembali

| Atribut | Keterangan |
|---|---|
| `_id` | Kode transaksi |
| `id_kampus` | Kampus |
| `peminjam` | Id, nama, dan NIM peminjam |
| `koleksi` | Id dan judul yang dipinjam |
| `tanggal_pinjam` | Tanggal pinjam |
| `batas_kembali` | Tanggal paling lambat kembali |
| `tanggal_kembali` | Tanggal dikembalikan (kosong jika belum) |
| `status` | `menunggu_verifikasi`, `dipinjam`, `dikembalikan`, atau `terlambat` |
| `denda` | Denda keterlambatan dalam rupiah |

**ulasan**: rating dan komentar pembaca

| Atribut | Keterangan |
|---|---|
| `_id` | Kode ulasan |
| `id_pengguna` | Penulis ulasan |
| `id_koleksi` | Koleksi yang diulas |
| `rating` | Nilai 1 sampai 5 |
| `komentar` | Isi ulasan |

### Typesense

Koleksi `katalog_pencarian` berisi salinan atribut katalog yang perlu dicari: judul, penulis, tipe, kategori, tahun terbit, dan stok tersedia.

### Neo4j

- Node: `Pengguna`, `Buku`, `Kategori`
- Relasi: `(Pengguna)-[:MEMINJAM]->(Buku)`, `(Pengguna)-[:MENGULAS]->(Buku)`, `(Buku)-[:TERMASUK]->(Kategori)`

### Redis

| Key | Isi | Kedaluwarsa |
|---|---|---|
| `sesi:{token}` | Data login | 24 jam |
| `stok:{id_koleksi}` | Stok tersedia | - |
| `rekomendasi:{nim_nip}` | Hasil rekomendasi dari Neo4j | 6 jam |
| `cari:{kata_kunci}` | Hasil pencarian populer | 10 menit |

## Menjalankan

Butuh Docker Desktop dan Git.

```bash
git clone https://github.com/huseinsm/litapp-perpustakaan-nosql.git
cd litapp-perpustakaan-nosql
docker compose up -d
docker compose ps
```

Kalau laptop berat, nyalakan service yang diperlukan saja dengan `docker compose up -d <nama-service>`.

Untuk mematikan:

```bash
docker compose down      # data tetap tersimpan
docker compose down -v   # data ikut dihapus
```

## Skenario pengujian

| Kode | Skenario | Database |
|---|---|---|
| TC-01 | Mencari buku dengan salah ketik, lalu melihat rekomendasi | Typesense, MongoDB, Neo4j, Redis |
| TC-02 | Meminjam buku: stok berkurang, transaksi tercatat, graf bertambah | Redis, MongoDB, Neo4j |
| TC-03 | Menambah jenis koleksi baru tanpa mengubah skema | MongoDB, Typesense |
