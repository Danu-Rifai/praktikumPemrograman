# Penjelasan Struktur Sistem Peminjaman Peralatan Laboratorium

Dokumen ini menjelaskan **bagaimana rancangan class kita bekerja**, mulai dari
gambaran besar, tugas tiap class, struktur data yang dipakai, sampai alur
langkah demi langkah saat meminjam dan mengembalikan alat. Tujuannya agar
semua anggota tim punya pemahaman yang sama sebelum mulai menulis kode.

---

## Daftar Isi

1. [Gambaran Besar](#1-gambaran-besar)
2. [Peta Class dan Relasinya](#2-peta-class-dan-relasinya)
3. [Tiga Enum (Daftar Nilai Tetap)](#3-tiga-enum-daftar-nilai-tetap)
4. [Penjelasan Tiap Class](#4-penjelasan-tiap-class)
5. [Struktur Data yang Dipakai](#5-struktur-data-yang-dipakai)
6. [Alur Kerja Utama](#6-alur-kerja-utama)
7. [Contoh Kasus Lengkap (Walk-through)](#7-contoh-kasus-lengkap-walk-through)
8. [Pemetaan Aturan Bisnis ke Rancangan](#8-pemetaan-aturan-bisnis-ke-rancangan)
9. [Pemetaan Menu Program ke Method](#9-pemetaan-menu-program-ke-method)
10. [Kerangka Kode Python](#10-kerangka-kode-python)
11. [Keputusan Desain dan Alasannya](#11-keputusan-desain-dan-alasannya)
12. [Hal yang Masih Perlu Diputuskan Tim](#12-hal-yang-masih-perlu-diputuskan-tim)
13. [Skalabilitas dan Pengembangan Lanjutan](#13-skalabilitas-dan-pengembangan-lanjutan)

---

## 1. Gambaran Besar

Sistem ini mencatat peminjaman alat laboratorium oleh mahasiswa. Ada tiga
hal pokok yang harus selalu benar:

- **Alat**: tahu apakah ia tersedia, sedang dipinjam, atau rusak.
- **Mahasiswa**: tahu berapa transaksi aktif yang ia punya (maks 2).
- **Transaksi**: tahu alat apa saja yang dipinjam, mana yang sudah kembali,
  dan apakah transaksi masih aktif.

Cara paling mudah memahami rancangan kita adalah dengan analogi **petugas
lab dengan buku catatan**:

| Di dunia nyata | Di program kita |
|---|---|
| Petugas lab yang mengatur semuanya | `LabManager` |
| Kartu data mahasiswa | `Mahasiswa` |
| Kartu data satu unit alat | `Peralatan` |
| Formulir peminjaman (satu lembar per peminjaman) | `Transaksi` |
| Satu baris di formulir (satu alat yang dipinjam) | `ItemPeminjaman` |
| Stempel status ("tersedia", "baik", dst.) | Enum |

**Prinsip utama rancangan:**

1. `LabManager` adalah *pengatur*. Dialah yang menyimpan semua data dan
   menjalankan aturan yang melibatkan banyak objek sekaligus.
2. Class lain adalah *pemilik data masing-masing* dan hanya mengubah data
   miliknya sendiri.
3. Satu transaksi bisa berisi banyak alat, dan tiap alat punya status
   kembalinya sendiri. Inilah yang memungkinkan **pengembalian sebagian**.

---

## 2. Peta Class dan Relasinya

```
                         ┌──────────────────────────┐
                         │        LabManager        │
                         │ dict mahasiswa           │
                         │ dict alat                │
                         │ dict transaksi           │
                         │ set  kategori            │
                         │ list log                 │
                         └─┬───────────┬──────────┬─┘
                  1 ◆      │           │ 1 ◆      │ 1 ◆
                           │           │          │
                    0..*   ▼    0..*   ▼   0..*   ▼
                   ┌───────────┐ ┌───────────┐ ┌────────────┐
                   │ Mahasiswa │ │ Peralatan │ │  Transaksi │
                   └─────┬─────┘ └─────▲─────┘ └──┬──────┬──┘
                         │ 1           │ 1        │ 1 ◆  │
                         │             │          │      │
                         └─────────────┼──────────┘      │ 1..*
                           0..* Transaksi                ▼
                                       │          ┌──────────────┐
                                       └──────────│ItemPeminjaman│
                                         0..*  1  └──────────────┘
```

Cara membaca relasi:

| Relasi | Arti dalam bahasa sehari-hari |
|---|---|
| `LabManager` 1 ◆ 0..* `Mahasiswa` | Satu LabManager menyimpan banyak mahasiswa |
| `LabManager` 1 ◆ 0..* `Peralatan` | Satu LabManager menyimpan banyak alat |
| `LabManager` 1 ◆ 0..* `Transaksi` | Satu LabManager menyimpan banyak transaksi |
| `Mahasiswa` 1 — 0..* `Transaksi` | Satu mahasiswa bisa punya banyak transaksi (aktif maksimal 2) |
| `Transaksi` 1 ◆ 1..* `ItemPeminjaman` | Satu transaksi berisi minimal 1 item. Item tidak berdiri sendiri tanpa transaksi |
| `ItemPeminjaman` 0..* — 1 `Peralatan` | Satu item menunjuk tepat ke satu alat. Alat yang sama bisa muncul di banyak item dari waktu ke waktu |

**Arti simbol:** ◆ (diamond) berarti "memiliki". Objek di sisi diamond
bertanggung jawab atas objek di sisi lain. Panah menunjukkan siapa yang
menyimpan referensi.

---

## 3. Tiga Enum (Daftar Nilai Tetap)

Enum dipakai agar nilai status tidak ditulis sebagai teks bebas yang rawan
salah ketik (misalnya `"dipinjam"` vs `"Dipinjam"` vs `"dipinjamm"`).

### `StatusTransaksi`: kondisi sebuah transaksi

| Nilai | Artinya |
|---|---|
| `DIPINJAM` | Belum ada satu pun alat yang dikembalikan |
| `SEBAGIAN_DIKEMBALIKAN` | Sebagian alat sudah kembali, sebagian belum |
| `SELESAI` | Semua alat sudah dikembalikan |

`DIPINJAM` dan `SEBAGIAN_DIKEMBALIKAN` sama-sama dianggap **transaksi aktif**.
Hanya `SELESAI` yang tidak aktif.

### `KondisiAlat`: kondisi fisik alat

| Nilai | Artinya |
|---|---|
| `BAIK` | Alat normal, boleh dipinjam lagi |
| `RUSAK_RINGAN` | Rusak sedikit, tidak boleh dipinjam |
| `RUSAK_BERAT` | Rusak parah, tidak boleh dipinjam |

### `StatusAlat`: ketersediaan alat

| Nilai | Artinya |
|---|---|
| `TERSEDIA` | Bisa dipinjam sekarang |
| `DIPINJAM` | Sedang dibawa mahasiswa |
| `TIDAK_TERSEDIA` | Tidak bisa dipinjam (misalnya karena rusak) |

> **Kenapa `KondisiAlat` dan `StatusAlat` dipisah?**
> Keduanya menjawab pertanyaan berbeda. `KondisiAlat` menjawab "bagaimana
> keadaan fisik alat?", sedangkan `StatusAlat` menjawab "bisa dipinjam
> sekarang atau tidak?". Alat bisa berkondisi `BAIK` tetapi berstatus
> `DIPINJAM`. Alat berkondisi `RUSAK_BERAT` otomatis berstatus
> `TIDAK_TERSEDIA`. Kalau digabung jadi satu, kita tidak bisa membedakan
> kedua situasi itu.

---

## 4. Penjelasan Tiap Class

### 4.1 `Mahasiswa`

**Tugas:** menyimpan data mahasiswa dan tahu transaksi aktif miliknya.

| Atribut | Tipe | Fungsi |
|---|---|---|
| `nim` | `str` | Identitas unik (dipakai sebagai key di dict `LabManager`) |
| `nama` | `str` | Nama mahasiswa |
| `nomor_hp` | `str` | Nomor HP. Disimpan sebagai `str` agar angka 0 di depan tidak hilang |
| `daftar_transaksi_aktif` | `list[Transaksi]` | Transaksi yang belum selesai milik mahasiswa ini |

| Method | Fungsi |
|---|---|
| `boleh_meminjam(): bool` | `True` jika transaksi aktifnya kurang dari 2 (Aturan 2) |
| `tambah_transaksi_aktif(transaksi)` | Memasukkan transaksi baru ke daftar aktif |
| `hapus_transaksi_aktif(transaksi)` | Mengeluarkan transaksi dari daftar aktif saat selesai |
| `punya_peminjaman_aktif(): bool` | `True` jika daftar aktif tidak kosong. Dipakai untuk melarang penghapusan (Aturan 6) |

**Catatan penting:** `Mahasiswa` tidak punya atribut `status_peminjaman: bool`.
Alasannya, aturan membolehkan hingga 2 transaksi aktif sehingga satu nilai
benar/salah tidak cukup. Cukup tanya `punya_peminjaman_aktif()`.

### 4.2 `Peralatan`

**Tugas:** mewakili **satu unit fisik** alat dan mengelola status serta
kondisinya.

| Atribut | Tipe | Fungsi |
|---|---|---|
| `kode_alat` | `str` | Identitas unik satu unit (key di dict `LabManager`) |
| `nama` | `str` | Nama alat, misalnya "Multimeter" |
| `kategori` | `str` | Kategori, misalnya "perangkat jaringan" |
| `kondisi` | `KondisiAlat` | Kondisi fisik terkini |
| `status` | `StatusAlat` | Ketersediaan terkini |

| Method | Fungsi |
|---|---|
| `tersedia(): bool` | `True` jika `status == TERSEDIA` |
| `sedang_dipinjam(): bool` | `True` jika `status == DIPINJAM` (hanya mengecek, tidak mengubah) |
| `tandai_dipinjam(): void` | Mengubah `status` menjadi `DIPINJAM` (aksi, mengubah data) |
| `terima_kembali(kondisi): void` | Mencatat kondisi baru. Jika `BAIK` maka status `TERSEDIA`, selain itu `TIDAK_TERSEDIA` (Aturan 5) |

> **Catatan: satu objek = satu unit fisik.** Kalau lab punya 2 Multimeter,
> buat 2 objek `Peralatan` dengan kode berbeda (misalnya `ALT-001` dan
> `ALT-002`), keduanya bernama "Multimeter". Dengan begitu kondisi dan status
> tiap unit dilacak sendiri-sendiri.

> **`sedang_dipinjam()` vs `tandai_dipinjam()`:** yang pertama *bertanya*
> (return `bool`, tidak mengubah apa pun), yang kedua *memerintah* (return
> `void`, mengubah status). Pemisahan ini mencegah method yang namanya
> terdengar seperti pengecekan tapi diam-diam mengubah data.

### 4.3 `ItemPeminjaman`

**Tugas:** satu baris di dalam transaksi, mencatat **satu alat** beserta
status pengembaliannya. Class inilah yang membuat pengembalian sebagian
mungkin dilakukan.

| Atribut | Tipe | Fungsi |
|---|---|---|
| `alat` | `Peralatan` | Alat yang dipinjam |
| `sudah_kembali` | `bool` | `False` saat dibuat, `True` setelah dikembalikan |
| `tanggal_kembali` | `date` | Terisi saat dikembalikan |
| `kondisi_kembali` | `KondisiAlat` | Kondisi alat saat dikembalikan |

| Method | Fungsi |
|---|---|
| `tandai_kembali(kondisi, tanggal)` | Mengisi tiga atribut di atas, lalu memanggil `alat.terima_kembali(kondisi)` |

Catatan: tidak ada `tandai_dipinjam()` di item. Saat item dibuat, state-nya
sudah benar (`sudah_kembali = False`). Perubahan status alat saat dipinjam
dilakukan oleh `LabManager.buat_transaksi()`.

### 4.4 `Transaksi`

**Tugas:** mewakili satu peminjaman. Memiliki daftar item dan mengetahui
statusnya sendiri.

| Atribut | Tipe | Fungsi |
|---|---|---|
| `id_transaksi` | `str` | Identitas unik transaksi |
| `mahasiswa` | `Mahasiswa` | Peminjam |
| `daftar_item` | `list[ItemPeminjaman]` | Alat-alat yang dipinjam (jumlahnya dinamis, Aturan 3) |
| `tanggal_peminjaman` | `date` | Kapan dipinjam |
| `batas_pengembalian` | `date` | Tenggat, maksimal 7 hari dari tanggal peminjaman |
| `status` | `StatusTransaksi` | `DIPINJAM`, `SEBAGIAN_DIKEMBALIKAN`, atau `SELESAI` |

| Method | Fungsi |
|---|---|
| `validasi_batas(): bool` | Memeriksa batas pengembalian tidak melebihi 7 hari |
| `kembalikan_alat(kode_alat, kondisi, tanggal)` | Mencari item dengan kode tersebut, memanggil `tandai_kembali()`, lalu `hitung_status()` |
| `hitung_status(): StatusTransaksi` | Menghitung status dari jumlah item yang sudah kembali |
| `masih_aktif(): bool` | `True` jika status bukan `SELESAI` |
| `alat_belum_kembali(): list[ItemPeminjaman]` | Daftar item yang belum dikembalikan |

Logika `hitung_status()`:

```
semua item sudah kembali      -> SELESAI
sebagian item sudah kembali   -> SEBAGIAN_DIKEMBALIKAN
belum ada yang kembali        -> DIPINJAM
```

Saat status menjadi `SELESAI`, transaksi memanggil
`mahasiswa.hapus_transaksi_aktif(self)` agar kuota peminjaman mahasiswa
kembali.

### 4.5 `LabManager`

**Tugas:** pusat data dan pengatur alur. Semua data disimpan di sini, dan
semua aturan yang melibatkan lebih dari satu objek dijalankan di sini.

| Atribut | Tipe | Isi |
|---|---|---|
| `mahasiswa` | `dict[nim, Mahasiswa]` | Semua mahasiswa, dicari lewat NIM |
| `alat` | `dict[kode_alat, Peralatan]` | Semua alat, dicari lewat kode |
| `transaksi` | `dict[id_transaksi, Transaksi]` | Semua transaksi, dicari lewat ID |
| `kategori` | `set[str]` | Kategori alat yang dikenal (tanpa duplikat) |
| `log` | `list[dict]` | Riwayat aktivitas |

| Kelompok method | Daftar |
|---|---|
| Kelola mahasiswa | `tambah_mahasiswa`, `edit_mahasiswa`, `hapus_mahasiswa`, `cari_mahasiswa` |
| Kelola alat | `tambah_alat`, `edit_alat`, `hapus_alat`, `cari_alat` |
| Kategori | `tambah_kategori` |
| Transaksi | `buat_transaksi`, `tampilkan_transaksi`, `proses_pengembalian` |
| Tampilan | `tampilkan_alat_tersedia`, `tampilkan_alat_dipinjam`, `tampilkan_alat_rusak` |
| Pencarian dan riwayat | `riwayat_mahasiswa`, `cari_transaksi_by_mahasiswa` |
| Bantu (privat) | `alat_dipakai_transaksi_aktif` |

---

## 5. Struktur Data yang Dipakai

Soal mewajibkan `list` dan/atau `dict`. Berikut alasan pemilihan tiap
struktur.

| Struktur | Dipakai di | Alasan |
|---|---|---|
| `dict` | `LabManager.mahasiswa`, `.alat`, `.transaksi` | Mencari data lewat key (NIM, kode, ID) langsung ketemu tanpa menelusuri satu per satu. Key juga otomatis mencegah duplikat |
| `list` | `Transaksi.daftar_item`, `Mahasiswa.daftar_transaksi_aktif`, `LabManager.log` | Urutan penting dan jumlah isinya berubah-ubah. Cocok untuk koleksi kecil yang sering ditelusuri berurutan |
| `set` | `LabManager.kategori` | Hanya perlu tahu "kategori ini ada atau tidak", dan tidak boleh ada kategori ganda |
| `Enum` | Status dan kondisi | Nilainya terbatas dan tetap, mencegah salah ketik |

**Bentuk data saat program berjalan** (contoh isi):

```python
lab.mahasiswa = {
    "M001": <Mahasiswa nim=M001 nama=Andi>,
    "M002": <Mahasiswa nim=M002 nama=Budi>,
}

lab.alat = {
    "ALT-001": <Peralatan Multimeter, BAIK, TERSEDIA>,
    "ALT-002": <Peralatan Tripod,     BAIK, DIPINJAM>,
}

lab.transaksi = {
    "T001": <Transaksi mahasiswa=M001, items=[...], status=DIPINJAM>,
}

lab.kategori = {"perangkat komputasi", "perangkat jaringan", "perangkat multimeter"}
```

**Mengapa dict untuk koleksi utama dan bukan list?** Bayangkan 10.000 alat.
Mencari alat berkode `ALT-7421` di dalam list harus memeriksa satu per satu
(lambat). Di dalam dict, hasilnya langsung ditemukan lewat key. Selain itu,
dict menjamin kode tidak kembar.

---

## 6. Alur Kerja Utama

### 6.1 Membuat Transaksi: `LabManager.buat_transaksi(nim, daftar_kode_alat, batas)`

Prinsipnya **validasi dulu semuanya, baru ubah data**. Dengan begitu kalau
alat ke-3 ternyata tidak tersedia, alat ke-1 dan ke-2 belum terlanjur
berubah status.

```
MULAI
  │
  ▼
Cari mahasiswa berdasarkan NIM ──(tidak ada)──► TOLAK: mahasiswa tidak ditemukan
  │
  ▼
mahasiswa.boleh_meminjam()? ─────(tidak)──────► TOLAK: sudah 2 transaksi aktif
  │
  ▼
Untuk setiap kode di daftar_kode_alat:
   alat ada? ───────────(tidak)───────────────► TOLAK: kode alat tidak ditemukan
   alat.tersedia()? ────(tidak)───────────────► TOLAK: alat tidak tersedia
  │
  ▼
Batas pengembalian valid (maks 7 hari)? ─(tidak)► TOLAK: batas melebihi 7 hari
  │
  ▼   ---- semua validasi lolos, baru ubah data ----
Buat satu ItemPeminjaman per alat
  │
  ▼
Buat objek Transaksi (status awal DIPINJAM)
  │
  ▼
Untuk setiap item: alat.tandai_dipinjam()
  │
  ▼
mahasiswa.tambah_transaksi_aktif(transaksi)
  │
  ▼
Simpan transaksi ke dict lab.transaksi
  │
  ▼
SELESAI: kembalikan objek Transaksi
```

### 6.2 Mengembalikan Alat: `LabManager.proses_pengembalian(id_transaksi, kode_alat, kondisi)`

```
MULAI
  │
  ▼
Cari transaksi berdasarkan ID ──(tidak ada)──► TOLAK: transaksi tidak ditemukan
  │
  ▼
transaksi.masih_aktif()? ───────(tidak)───────► TOLAK: transaksi sudah selesai
  │
  ▼
transaksi.kembalikan_alat(kode_alat, kondisi, tanggal_hari_ini)
  │    │
  │    ├─ cari item dengan kode_alat yang belum kembali
  │    │     (tidak ada) ──► TOLAK: alat tidak ada di transaksi / sudah kembali
  │    │
  │    ├─ item.tandai_kembali(kondisi, tanggal)
  │    │     ├─ sudah_kembali = True
  │    │     ├─ tanggal_kembali, kondisi_kembali terisi
  │    │     └─ alat.terima_kembali(kondisi)
  │    │            ├─ kondisi == BAIK ──► status TERSEDIA
  │    │            └─ rusak ringan/berat ► status TIDAK_TERSEDIA
  │    │
  │    └─ hitung_status()
  │          ├─ semua kembali  ──► SELESAI
  │          │        └─ mahasiswa.hapus_transaksi_aktif(transaksi)
  │          ├─ sebagian       ──► SEBAGIAN_DIKEMBALIKAN
  │          └─ belum ada      ──► DIPINJAM
  ▼
SELESAI
```

### 6.3 Menghapus Mahasiswa: `LabManager.hapus_mahasiswa(nim)`

```
Cari mahasiswa ──(tidak ada)──► TOLAK
mahasiswa.punya_peminjaman_aktif()? ──(ya)──► TOLAK (Aturan 6)
Hapus dari dict lab.mahasiswa
```

### 6.4 Menghapus Alat: `LabManager.hapus_alat(kode_alat)`

```
Cari alat ──(tidak ada)──► TOLAK
alat_dipakai_transaksi_aktif(kode_alat)? ──(ya)──► TOLAK (Aturan 6)
Hapus dari dict lab.alat
```

`alat_dipakai_transaksi_aktif()` menelusuri semua transaksi yang
`masih_aktif()`, lalu memeriksa apakah ada item yang memuat alat tersebut.

---

## 7. Contoh Kasus Lengkap (Walk-through)

**Data awal:**

| Kode | Nama | Kondisi | Status |
|---|---|---|---|
| ALT-001 | Kamera Digital | BAIK | TERSEDIA |
| ALT-002 | Tripod | BAIK | TERSEDIA |
| ALT-003 | Kabel LAN | BAIK | TERSEDIA |

Mahasiswa: `M001 Andi` dengan 0 transaksi aktif.

**Langkah 1. Andi meminjam 3 alat sekaligus**

`buat_transaksi("M001", ["ALT-001", "ALT-002", "ALT-003"], batas=+7 hari)`

| Objek | Sebelum | Sesudah |
|---|---|---|
| ALT-001, 002, 003 | TERSEDIA | **DIPINJAM** |
| Transaksi T001 | belum ada | status **DIPINJAM**, 3 item (semua `sudah_kembali=False`) |
| Andi | 0 aktif | 1 aktif (`[T001]`) |

**Langkah 2. Budi mencoba meminjam Tripod (ALT-002)**

`alat.tersedia()` bernilai `False` karena sedang `DIPINJAM`.
Hasilnya **ditolak** (Aturan 1). Tidak ada data yang berubah.

**Langkah 3. Andi mengembalikan Kamera dalam kondisi BAIK**

`proses_pengembalian("T001", "ALT-001", BAIK)`

| Objek | Perubahan |
|---|---|
| Item Kamera | `sudah_kembali=True`, kondisi kembali BAIK |
| ALT-001 | status **TERSEDIA** (bisa dipinjam lagi) |
| T001 | 1 dari 3 kembali, jadi **SEBAGIAN_DIKEMBALIKAN** |
| Andi | masih 1 transaksi aktif |

**Langkah 4. Andi mengembalikan Tripod dalam kondisi RUSAK_RINGAN**

| Objek | Perubahan |
|---|---|
| ALT-002 | kondisi RUSAK_RINGAN, status **TIDAK_TERSEDIA** (Aturan 5) |
| T001 | 2 dari 3 kembali, tetap SEBAGIAN_DIKEMBALIKAN |

**Langkah 5. Andi mencoba dihapus saat T001 masih aktif**

`hapus_mahasiswa("M001")`: `punya_peminjaman_aktif()` bernilai `True`.
Hasilnya **ditolak** (Aturan 6).

**Langkah 6. Andi mengembalikan Kabel LAN dalam kondisi BAIK**

| Objek | Perubahan |
|---|---|
| ALT-003 | status TERSEDIA |
| T001 | 3 dari 3 kembali, jadi **SELESAI** |
| Andi | transaksi aktif kembali 0, sekarang boleh dihapus |

**Hasil akhir:** Kamera dan Kabel LAN tersedia, Tripod tidak tersedia karena
rusak ringan. Tripod tidak muncul di `tampilkan_alat_tersedia()`, tetapi
muncul di `tampilkan_alat_rusak()`.

---

## 8. Pemetaan Aturan Bisnis ke Rancangan

| Aturan | Isi aturan | Ditangani oleh |
|---|---|---|
| 1 | Alat tidak tersedia tidak boleh dipinjam | `Peralatan.tersedia()`, dicek di `LabManager.buat_transaksi()` fase validasi |
| 2 | Maks 2 transaksi aktif per mahasiswa | `Mahasiswa.boleh_meminjam()` (hitung `daftar_transaksi_aktif`) |
| 3 | Satu transaksi bisa berisi banyak jenis alat, tanpa variabel terpisah | `Transaksi.daftar_item: list[ItemPeminjaman]` |
| 4 | Boleh mengembalikan sebagian, transaksi tetap aktif sampai semua kembali | `ItemPeminjaman` (status per alat) dan `Transaksi.hitung_status()` |
| 5 | Stok tersedia mengikuti kondisi alat saat kembali | `Peralatan.terima_kembali(kondisi)` |
| 6 | Mahasiswa/alat dengan transaksi aktif tidak boleh dihapus | `Mahasiswa.punya_peminjaman_aktif()` dan `LabManager.alat_dipakai_transaksi_aktif()` |
| Batas | Pengembalian maks 7 hari | `Transaksi.validasi_batas()` |
| Kategori | Kategori baru tanpa ubah struktur program | `kategori: set[str]` dan `LabManager.tambah_kategori()` |

---

## 9. Pemetaan Menu Program ke Method

| No | Menu | Method yang dipanggil |
|---|---|---|
| 1 | Kelola data mahasiswa | `tambah_mahasiswa`, `edit_mahasiswa`, `hapus_mahasiswa`, `cari_mahasiswa` |
| 2 | Kelola data alat | `tambah_alat`, `edit_alat`, `hapus_alat`, `cari_alat` |
| 3 | Buat transaksi peminjaman | `buat_transaksi` |
| 4 | Tampilkan transaksi | `tampilkan_transaksi` |
| 5 | Proses pengembalian alat | `proses_pengembalian` |
| 6 | Cari transaksi berdasarkan mahasiswa | `cari_transaksi_by_mahasiswa` |
| 7 | Tampilkan alat yang tersedia | `tampilkan_alat_tersedia` |
| 8 | Tampilkan alat yang sedang dipinjam | `tampilkan_alat_dipinjam` |
| 9 | Tampilkan alat yang rusak | `tampilkan_alat_rusak` |
| 10 | Tampilkan riwayat peminjaman mahasiswa | `riwayat_mahasiswa` |
| 11 | Keluar | Ditangani oleh loop menu di `main.py`, bukan oleh class |

---

## 10. Kerangka Kode Python

Ini kerangka awal untuk memulai. Isi method boleh disesuaikan. Yang penting
adalah **tanggung jawab tiap class tetap sama** dengan penjelasan di atas.

```python
from enum import Enum
from datetime import date, timedelta


# ---------- Enum ----------
class StatusTransaksi(Enum):
    DIPINJAM = "dipinjam"
    SEBAGIAN_DIKEMBALIKAN = "sebagian dikembalikan"
    SELESAI = "selesai"


class KondisiAlat(Enum):
    BAIK = "baik"
    RUSAK_RINGAN = "rusak ringan"
    RUSAK_BERAT = "rusak berat"


class StatusAlat(Enum):
    TERSEDIA = "tersedia"
    DIPINJAM = "dipinjam"
    TIDAK_TERSEDIA = "tidak tersedia"


# ---------- Mahasiswa ----------
class Mahasiswa:
    MAKS_TRANSAKSI_AKTIF = 2

    def __init__(self, nim, nama, nomor_hp):
        self._nim = nim
        self._nama = nama
        self._nomor_hp = nomor_hp
        self._daftar_transaksi_aktif = []

    def boleh_meminjam(self) -> bool:
        return len(self._daftar_transaksi_aktif) < self.MAKS_TRANSAKSI_AKTIF

    def tambah_transaksi_aktif(self, transaksi) -> None:
        self._daftar_transaksi_aktif.append(transaksi)

    def hapus_transaksi_aktif(self, transaksi) -> None:
        if transaksi in self._daftar_transaksi_aktif:
            self._daftar_transaksi_aktif.remove(transaksi)

    def punya_peminjaman_aktif(self) -> bool:
        return len(self._daftar_transaksi_aktif) > 0


# ---------- Peralatan ----------
class Peralatan:
    def __init__(self, kode_alat, nama, kategori, kondisi=KondisiAlat.BAIK):
        self._kode_alat = kode_alat
        self._nama = nama
        self._kategori = kategori
        self._kondisi = kondisi
        self._status = StatusAlat.TERSEDIA

    def tersedia(self) -> bool:
        return self._status == StatusAlat.TERSEDIA

    def sedang_dipinjam(self) -> bool:
        return self._status == StatusAlat.DIPINJAM

    def tandai_dipinjam(self) -> None:
        self._status = StatusAlat.DIPINJAM

    def terima_kembali(self, kondisi: KondisiAlat) -> None:
        self._kondisi = kondisi
        if kondisi == KondisiAlat.BAIK:
            self._status = StatusAlat.TERSEDIA
        else:
            self._status = StatusAlat.TIDAK_TERSEDIA


# ---------- ItemPeminjaman ----------
class ItemPeminjaman:
    def __init__(self, alat: Peralatan):
        self._alat = alat
        self._sudah_kembali = False
        self._tanggal_kembali = None
        self._kondisi_kembali = None

    def tandai_kembali(self, kondisi: KondisiAlat, tanggal: date) -> None:
        self._sudah_kembali = True
        self._tanggal_kembali = tanggal
        self._kondisi_kembali = kondisi
        self._alat.terima_kembali(kondisi)


# ---------- Transaksi ----------
class Transaksi:
    MAKS_HARI = 7

    def __init__(self, id_transaksi, mahasiswa, daftar_item,
                 tanggal_peminjaman, batas_pengembalian):
        self._id_transaksi = id_transaksi
        self._mahasiswa = mahasiswa
        self._daftar_item = daftar_item
        self._tanggal_peminjaman = tanggal_peminjaman
        self._batas_pengembalian = batas_pengembalian
        self._status = StatusTransaksi.DIPINJAM

    def validasi_batas(self) -> bool:
        selisih = self._batas_pengembalian - self._tanggal_peminjaman
        return timedelta(days=0) < selisih <= timedelta(days=self.MAKS_HARI)

    def kembalikan_alat(self, kode_alat, kondisi, tanggal) -> None:
        for item in self._daftar_item:
            if item._alat._kode_alat == kode_alat and not item._sudah_kembali:
                item.tandai_kembali(kondisi, tanggal)
                self.hitung_status()
                return
        raise ValueError("Alat tidak ada di transaksi ini atau sudah dikembalikan")

    def hitung_status(self) -> StatusTransaksi:
        jumlah_kembali = sum(1 for i in self._daftar_item if i._sudah_kembali)
        if jumlah_kembali == len(self._daftar_item):
            self._status = StatusTransaksi.SELESAI
            self._mahasiswa.hapus_transaksi_aktif(self)
        elif jumlah_kembali > 0:
            self._status = StatusTransaksi.SEBAGIAN_DIKEMBALIKAN
        else:
            self._status = StatusTransaksi.DIPINJAM
        return self._status

    def masih_aktif(self) -> bool:
        return self._status != StatusTransaksi.SELESAI

    def alat_belum_kembali(self) -> list:
        return [i for i in self._daftar_item if not i._sudah_kembali]


# ---------- LabManager ----------
class LabManager:
    def __init__(self):
        self._mahasiswa = {}   # nim -> Mahasiswa
        self._alat = {}        # kode_alat -> Peralatan
        self._transaksi = {}   # id_transaksi -> Transaksi
        self._kategori = set()
        self._log = []

    def buat_transaksi(self, nim, daftar_kode_alat, batas) -> Transaksi:
        # --- Fase 1: validasi (belum mengubah data apa pun) ---
        if nim not in self._mahasiswa:
            raise ValueError("Mahasiswa tidak ditemukan")
        mhs = self._mahasiswa[nim]
        if not mhs.boleh_meminjam():
            raise ValueError("Maksimal 2 transaksi aktif")
        for kode in daftar_kode_alat:
            if kode not in self._alat:
                raise ValueError(f"Alat {kode} tidak ditemukan")
            if not self._alat[kode].tersedia():
                raise ValueError(f"Alat {kode} tidak tersedia")

        # --- Fase 2: ubah data ---
        items = [ItemPeminjaman(self._alat[k]) for k in daftar_kode_alat]
        id_baru = f"T{len(self._transaksi) + 1:03d}"
        transaksi = Transaksi(id_baru, mhs, items, date.today(), batas)
        if not transaksi.validasi_batas():
            raise ValueError("Batas pengembalian maksimal 7 hari")
        for item in items:
            item._alat.tandai_dipinjam()
        mhs.tambah_transaksi_aktif(transaksi)
        self._transaksi[id_baru] = transaksi
        return transaksi

    def proses_pengembalian(self, id_transaksi, kode_alat, kondisi) -> None:
        if id_transaksi not in self._transaksi:
            raise ValueError("Transaksi tidak ditemukan")
        transaksi = self._transaksi[id_transaksi]
        if not transaksi.masih_aktif():
            raise ValueError("Transaksi sudah selesai")
        transaksi.kembalikan_alat(kode_alat, kondisi, date.today())

    def hapus_mahasiswa(self, nim) -> None:
        mhs = self._mahasiswa[nim]
        if mhs.punya_peminjaman_aktif():
            raise ValueError("Mahasiswa masih punya transaksi aktif")
        del self._mahasiswa[nim]

    def hapus_alat(self, kode_alat) -> None:
        if self._alat_dipakai_transaksi_aktif(kode_alat):
            raise ValueError("Alat masih tercatat di transaksi aktif")
        del self._alat[kode_alat]

    def _alat_dipakai_transaksi_aktif(self, kode_alat) -> bool:
        for t in self._transaksi.values():
            if t.masih_aktif():
                for item in t._daftar_item:
                    if item._alat._kode_alat == kode_alat:
                        return True
        return False
```

> Kode di atas sengaja mengakses atribut privat antar class (misalnya
> `item._alat`) agar ringkas. Saat implementasi sebenarnya, sebaiknya
> tambahkan properti atau method pembaca (`@property`) supaya enkapsulasi
> tetap terjaga.

---

## 11. Keputusan Desain dan Alasannya

Bagian ini bahan untuk **Dokumen Keputusan Desain** dan untuk menjawab
pertanyaan "mengapa struktur class ini dipilih?".

| Keputusan | Alasan | Alternatif yang tidak dipilih |
|---|---|---|
| `ItemPeminjaman` dipisah dari `Transaksi` | Tiap alat butuh status kembali, tanggal, dan kondisi sendiri. Wajib untuk pengembalian sebagian (Aturan 4) | Menyimpan alat sebagai list sederhana di transaksi. Tidak bisa mencatat pengembalian per alat |
| Tidak ada class `PengembalianAlat` terpisah | Data pengembalian (tanggal, kondisi) sudah menempel di `ItemPeminjaman`, jadi class tambahan hanya menduplikasi | Class `PengembalianAlat` terpisah. Terisolasi dan sulit dihubungkan ke alat serta transaksi |
| `kondisi` dan `status` alat dipisah | Keduanya menjawab pertanyaan berbeda. Alat berkondisi baik bisa sedang dipinjam | Satu atribut status saja. Tidak bisa membedakan "dipinjam" dan "rusak" secara bersih |
| `Mahasiswa.status_peminjaman: bool` diganti `daftar_transaksi_aktif` | Boolean tidak bisa mewakili batas 2 transaksi aktif | Boolean atau counter angka. Rawan tidak sinkron dengan daftar transaksi sebenarnya |
| `kategori: str` + `set` di `LabManager`, bukan Enum | Menambah kategori baru cukup menambah isi `set`, tanpa mengubah kode (syarat soal) | `KategoriAlat` sebagai Enum. Tiap kategori baru berarti mengubah kode |
| `LabManager` mengubah status alat saat transaksi dibuat | Validasi dan perubahan data butuh melihat banyak objek sekaligus, jadi tepat dipegang pengatur pusat | Item memanggil `tandai_dipinjam()` di konstruktornya. Efek samping tersembunyi dan validasi sulit diselesaikan dulu |
| Aturan penghapusan dipusatkan di `LabManager` | Mahasiswa dan alat memakai pola yang sama. Class domain hanya menyediakan fakta (`punya_peminjaman_aktif()`) | Method `boleh_dihapus()` di tiap class. Duplikat, dan `Peralatan` tidak punya akses ke data transaksi |
| Validasi dulu, ubah data kemudian, di `buat_transaksi()` | Mencegah sebagian alat terlanjur berstatus `DIPINJAM` jika transaksi akhirnya gagal | Ubah data sambil validasi. Perlu mekanisme rollback |
| `dict` untuk koleksi utama | Pencarian lewat key cepat dan mencegah duplikat | `list`. Pencarian harus menelusuri satu per satu |

**Sinkronisasi dua arah:** `Mahasiswa.daftar_transaksi_aktif` dan
`Transaksi.mahasiswa` saling menunjuk. Tanggung jawab menjaga keduanya
sinkron dibagi jelas:

- `LabManager.buat_transaksi()` memanggil `mahasiswa.tambah_transaksi_aktif()`.
- `Transaksi.hitung_status()` memanggil `mahasiswa.hapus_transaksi_aktif()`
  saat menjadi `SELESAI`.

---

## 12. Hal yang Masih Perlu Diputuskan Tim

### a. Tafsir Aturan 6 untuk penghapusan alat

Bunyi soal: alat tidak boleh dihapus jika *masih tercatat dalam transaksi
aktif*.

| Pilihan | Cara kerja | Konsekuensi |
|---|---|---|
| **Ketat** (disarankan) | `_alat_dipakai_transaksi_aktif()` memeriksa semua item di semua transaksi aktif, termasuk item yang sudah dikembalikan | Sesuai bunyi soal. Perlu akses ke daftar item (misalnya method `memuat_alat(kode_alat)` di `Transaksi`) |
| **Longgar** | Hanya memeriksa `alat_belum_kembali()` | Lebih sederhana, tetapi alat yang sudah dikembalikan lebih dulu pada transaksi yang masih aktif bisa terhapus |

Apa pun pilihannya, tulis di dokumen keputusan desain.

### b. Tantangan tambahan (minimal 2 dari A-D)

Belum diterapkan di rancangan saat ini. Titik masuk yang sudah tersedia:

| Tantangan | Titik masuk di rancangan |
|---|---|
| A. Pencarian fleksibel | `cari_alat(kode_alat, nama, kategori)` sudah ada, tinggal pastikan ketiga parameter opsional |
| B. Statistik | Dihitung dari `lab.transaksi` dan `lab.alat`. Perlu method baru di `LabManager` |
| C. Pemeliharaan | Alat `RUSAK_BERAT` sudah berstatus `TIDAK_TERSEDIA`. Perlu daftar pemeliharaan dan method pemulihan |
| D. Log aktivitas | Atribut `log: list[dict]` sudah ada. Perlu method `catat_log()` dan format `[YYYY-MM-DD HH:MM] pesan` |

### c. Satu objek = satu unit alat

Rancangan ini menganggap tiap unit fisik punya kode sendiri. Pastikan tim
sepakat, karena memengaruhi cara data awal dibuat.

---

## 13. Skalabilitas dan Pengembangan Lanjutan

Soal meminta evaluasi bagaimana sistem berkembang. Berikut jawaban awal dari
rancangan ini.

| Kebutuhan baru | Kesiapan rancangan | Yang perlu diubah |
|---|---|---|
| Alat naik jadi 10.000 | Baik. Pencarian lewat key di `dict` tetap cepat | Pencarian berdasarkan nama/kategori masih menelusuri semua alat, bisa ditambah indeks per kategori |
| Mahasiswa naik jadi 20.000 | Baik, alasannya sama | `riwayat_mahasiswa()` menelusuri semua transaksi. Bisa ditambah indeks `nim -> daftar transaksi` |
| Kategori alat baru | Sudah siap | Cukup `tambah_kategori()` |
| Jenis pengguna baru (dosen, staf) | Cukup siap | Buat class induk `Pengguna`, lalu `Mahasiswa` mewarisinya |
| Aturan peminjaman baru | Cukup siap | Aturan terpusat di `boleh_meminjam()` dan `buat_transaksi()`. Kalau bertambah banyak, pisahkan jadi class aturan tersendiri |
| Denda keterlambatan | Cukup siap | Tambah `hitung_denda()` di `Transaksi`. Datanya sudah ada (`batas_pengembalian` dan `tanggal_kembali`) |
| Penyimpanan database | Perlu penyesuaian | Pisahkan penyimpanan dari `LabManager` ke class repository, supaya `LabManager` tidak tahu cara data disimpan |

**Titik yang paling mungkin menjadi berat dirawat:** `LabManager` memegang
banyak tanggung jawab (CRUD tiga jenis data, transaksi, tampilan, riwayat).
Jika proyek membesar, pertimbangkan memecahnya menjadi beberapa pengelola
(misalnya pengelola mahasiswa, pengelola alat, dan pengelola transaksi) di
bawah satu koordinator. Hal ini layak dicatat sebagai jawaban pertanyaan
refleksi "apakah ada class dengan terlalu banyak tanggung jawab?".

---

## Ringkasan Singkat untuk Dibaca Cepat

1. **`LabManager`** menyimpan semua data (3 dict, 1 set, 1 list) dan menjalankan aturan lintas objek.
2. **`Transaksi`** memiliki banyak **`ItemPeminjaman`**, satu item per alat, masing-masing dengan status kembali sendiri.
3. **`Peralatan`** memisahkan **kondisi** (fisik) dan **status** (ketersediaan).
4. Meminjam: **validasi semua dulu**, baru ubah data.
5. Mengembalikan: item dicatat, alat diperbarui sesuai kondisi, status transaksi dihitung ulang. Jika semua kembali, transaksi selesai dan kuota mahasiswa pulih.
6. Penghapusan mahasiswa dan alat dicek di `LabManager` terhadap transaksi aktif.
