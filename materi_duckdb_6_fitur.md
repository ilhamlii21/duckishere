# Materi Praktikum: 6 Fitur Unik DuckDB
## Implementasi dengan Database Toko (produk, pelanggan, transaksi)

> **Dosen:** —  
> **Mata Kuliah:** Columnar Database  
> **Tools:** DuckDB CLI  

---

## Setup Awal: Membuat Database & Tabel

Sebelum mencoba fitur-fitur di bawah, jalankan script berikut untuk membuat database dan mengisi data:

```sql
-- Buat dan masuk ke database
-- $ duckdb asisten_db

CREATE TABLE produk (
    id_produk     INTEGER PRIMARY KEY,
    nama_produk   VARCHAR,
    kategori      VARCHAR,
    harga_satuan  DECIMAL
);

CREATE TABLE pelanggan (
    id_pelanggan  INTEGER PRIMARY KEY,
    nama_pelanggan VARCHAR,
    kota          VARCHAR,
    level_member  VARCHAR  -- Bronze, Gold, Platinum
);

CREATE TABLE transaksi (
    id_transaksi       INTEGER PRIMARY KEY,
    id_produk          INTEGER,
    id_pelanggan       INTEGER,
    jumlah_beli        INTEGER,
    tanggal_transaksi  DATE,
    FOREIGN KEY (id_produk)    REFERENCES produk(id_produk),
    FOREIGN KEY (id_pelanggan) REFERENCES pelanggan(id_pelanggan)
);

INSERT INTO produk VALUES
    (1, 'Kopi', 'Minuman', 25000),
    (2, 'Roti', 'Makanan', 15000);

INSERT INTO pelanggan VALUES
    (10, 'Ilham', 'Purwokerto', 'Gold'),
    (11, 'Budi',  'Jakarta',    'Bronze');

INSERT INTO transaksi VALUES
    (101, 1, 10, 2, '2026-05-12'),
    (102, 2, 11, 5, '2026-05-12');
```

---

## Fitur 1 — `SUMMARIZE`: Profiling Data Instan

### Penjelasan
`SUMMARIZE` adalah perintah unik DuckDB yang langsung menghasilkan ringkasan statistik setiap kolom: jumlah baris, nilai min, max, rata-rata, hingga persentase null — **dalam satu query**.

Di Oracle, Anda harus menjalankan `COUNT`, `MIN`, `MAX`, `AVG` secara terpisah per kolom.

### Sintaks
```sql
SUMMARIZE tabel;
SUMMARIZE SELECT * FROM tabel;
```

### Implementasi

```sql
-- Profiling tabel produk
SUMMARIZE produk;

-- Profiling tabel transaksi
SUMMARIZE transaksi;

-- Profiling hasil JOIN
SUMMARIZE
SELECT t.id_transaksi, p.nama_produk, t.jumlah_beli,
       (p.harga_satuan * t.jumlah_beli) AS total_harga
FROM transaksi t
JOIN produk p ON t.id_produk = p.id_produk;
```

### Output yang Dihasilkan

| column_name | column_type | min | max | approx_unique | avg | std | ... |
|---|---|---|---|---|---|---|---|
| id_produk | INTEGER | 1 | 2 | 2 | 1.5 | 0.7 | ... |
| harga_satuan | DECIMAL | 15000 | 25000 | 2 | 20000 | 7071 | ... |

### Latihan
> Jalankan `SUMMARIZE pelanggan;` — kolom apa yang memiliki nilai null? Berapa distinct value pada kolom `level_member`?

---

## Fitur 2 — Query Langsung ke File (CSV / Parquet / JSON)

### Penjelasan
DuckDB bisa membaca file eksternal **tanpa proses import**. Cukup tulis nama file dalam tanda kutip seperti nama tabel. Ini sangat berguna untuk analisis data cepat.

Di Oracle, Anda harus membuat **External Table** dengan konfigurasi direktori yang panjang.

### Sintaks
```sql
SELECT * FROM 'nama_file.csv';
SELECT * FROM read_csv('file.csv', header=true);
SELECT * FROM read_parquet('file.parquet');
SELECT * FROM read_json('file.json');
```

### Implementasi

**Langkah 1 — Export tabel ke file CSV terlebih dahulu:**
```sql
-- Export tabel transaksi ke CSV
COPY transaksi TO 'transaksi_export.csv' (HEADER, DELIMITER ',');

-- Export hasil JOIN ke Parquet
COPY (
    SELECT t.id_transaksi, p.nama_produk, pl.nama_pelanggan,
           t.jumlah_beli, p.harga_satuan,
           (t.jumlah_beli * p.harga_satuan) AS total
    FROM transaksi t
    JOIN produk p    ON t.id_produk    = p.id_produk
    JOIN pelanggan pl ON t.id_pelanggan = pl.id_pelanggan
) TO 'rekap_transaksi.parquet' (FORMAT PARQUET);
```

**Langkah 2 — Query langsung dari file:**
```sql
-- Baca CSV tanpa import
SELECT * FROM 'transaksi_export.csv';

-- Baca Parquet dan filter langsung
SELECT * FROM 'rekap_transaksi.parquet'
WHERE jumlah_beli > 3;

-- Gabungkan file CSV dengan tabel di database
SELECT f.*, p.kategori
FROM 'transaksi_export.csv' f
JOIN produk p ON f.id_produk = p.id_produk;
```

### Latihan
> Export tabel `pelanggan` ke CSV, lalu query file tersebut untuk menampilkan hanya pelanggan dari kota 'Jakarta'.

---

## Fitur 3 — `PIVOT` Native

### Penjelasan
`PIVOT` mengubah nilai unik dalam satu kolom menjadi kolom-kolom baru (transformasi baris ke kolom). DuckDB mendukung `PIVOT` dengan sintaks yang bersih dan ringkas.

Di Oracle, sintaks `PIVOT` jauh lebih verbose dan membutuhkan daftar nilai yang di-hardcode secara eksplisit.

### Sintaks DuckDB
```sql
PIVOT tabel
ON kolom_yang_dipivot
USING fungsi_agregasi(kolom_nilai)
GROUP BY kolom_dimensi;
```

### Implementasi

**Skenario:** Tampilkan total pembelian per produk, dipivot berdasarkan `level_member` pelanggan.

```sql
-- Buat view rekap dulu
CREATE OR REPLACE VIEW rekap AS
SELECT pl.level_member,
       p.nama_produk,
       SUM(t.jumlah_beli) AS total_beli
FROM transaksi t
JOIN produk    p  ON t.id_produk    = p.id_produk
JOIN pelanggan pl ON t.id_pelanggan = pl.id_pelanggan
GROUP BY pl.level_member, p.nama_produk;

-- Tampilkan data sebelum pivot
SELECT * FROM rekap;

-- PIVOT: setiap nama_produk menjadi kolom
PIVOT rekap
ON nama_produk
USING SUM(total_beli)
GROUP BY level_member;
```

### Hasil

| level_member | Kopi | Roti |
|---|---|---|
| Gold | 2 | NULL |
| Bronze | NULL | 5 |

### Latihan
> Pivot tabel transaksi sehingga setiap `id_pelanggan` menjadi kolom, dengan nilai adalah total `jumlah_beli` per produk.

---

## Fitur 4 — `SELECT * EXCLUDE` dan `SELECT * REPLACE`

### Penjelasan
Dua sintaks ini mempermudah manipulasi kolom tanpa harus menuliskan semua nama kolom secara manual:

- `EXCLUDE` — pilih semua kolom **kecuali** kolom tertentu
- `REPLACE` — pilih semua kolom, tapi **ganti nilai** kolom tertentu dengan ekspresi baru

Oracle tidak memiliki sintaks ini sama sekali.

### Sintaks
```sql
-- Exclude: semua kolom kecuali yang disebutkan
SELECT * EXCLUDE (kolom1, kolom2) FROM tabel;

-- Replace: semua kolom, ganti satu kolom dengan ekspresi
SELECT * REPLACE (ekspresi AS nama_kolom) FROM tabel;
```

### Implementasi

```sql
-- EXCLUDE: tampilkan produk tanpa kolom id_produk
SELECT * EXCLUDE (id_produk) FROM produk;

-- EXCLUDE: tampilkan pelanggan tanpa kolom sensitif
SELECT * EXCLUDE (id_pelanggan, level_member) FROM pelanggan;

-- REPLACE: tampilkan produk, tapi harga sudah termasuk PPN 11%
SELECT * REPLACE (harga_satuan * 1.11 AS harga_satuan) FROM produk;

-- REPLACE: ubah nama_pelanggan menjadi huruf besar
SELECT * REPLACE (UPPER(nama_pelanggan) AS nama_pelanggan) FROM pelanggan;

-- Kombinasi EXCLUDE + REPLACE
SELECT * EXCLUDE (id_produk) REPLACE (harga_satuan * 1.11 AS harga_satuan)
FROM produk;
```

### Latihan
> Tampilkan semua data transaksi, **kecuali** `id_transaksi`, dan **ganti** `jumlah_beli` dengan `jumlah_beli * 2` (simulasi double order).

---

## Fitur 5 — Tipe Data LIST & STRUCT + Lambda Function

### Penjelasan
DuckDB mendukung tipe data kolumnar tingkat lanjut:

- **LIST** — kolom berisi array nilai
- **STRUCT** — kolom berisi objek key-value (seperti JSON)
- **Lambda** — fungsi anonim untuk memfilter atau memetakan list

Di Oracle, tidak ada dukungan native untuk list/struct dan lambda dalam SQL.

### Sintaks
```sql
-- List
[nilai1, nilai2, nilai3]
list_sum([...])
list_filter(list, x -> kondisi)
list_transform(list, x -> ekspresi)

-- Struct
{'key': nilai, 'key2': nilai2}
struct_extract(kolom, 'key')
```

### Implementasi

```sql
-- LIST: kumpulkan semua produk yang dibeli pelanggan
SELECT id_pelanggan,
       list(id_produk) AS daftar_produk_dibeli
FROM transaksi
GROUP BY id_pelanggan;

-- LIST aggregate dengan operasi
SELECT id_pelanggan,
       list(jumlah_beli)         AS semua_jumlah,
       list_sum(list(jumlah_beli)) AS total_beli
FROM transaksi
GROUP BY id_pelanggan;

-- STRUCT: gabungkan info produk menjadi satu objek
SELECT nama_produk,
       {'kategori': kategori, 'harga': harga_satuan} AS info_produk
FROM produk;

-- Lambda — filter harga di atas threshold
SELECT list_filter([25000, 15000, 30000, 5000], x -> x > 10000) AS harga_valid;

-- Lambda — transform: hitung harga setelah diskon 10%
SELECT list_transform([25000, 15000], x -> x * 0.9) AS harga_diskon;

-- Kombinasi: kumpulkan harga, lalu filter yang mahal
SELECT list_filter(list(harga_satuan), x -> x > 20000) AS produk_mahal
FROM produk;
```

### Latihan
> Gunakan `list()` untuk mengumpulkan semua `tanggal_transaksi` per pelanggan, lalu gunakan `list_sort()` untuk mengurutkannya.

---

## Fitur 6 — Perintah CLI: `.mode`, `.timer`, `.output`

### Penjelasan
DuckDB CLI memiliki perintah meta (diawali titik) yang mengontrol **format output**, **pengukuran waktu**, dan **ekspor hasil** — langsung dari terminal tanpa kode tambahan.

Oracle SQL*Plus memiliki perintah serupa tapi jauh lebih rumit dan terbatas.

### Perintah CLI Utama

| Perintah | Fungsi |
|---|---|
| `.mode table` | Output tabel ASCII (default) |
| `.mode markdown` | Output format Markdown |
| `.mode csv` | Output format CSV |
| `.mode json` | Output format JSON |
| `.timer on/off` | Tampilkan/sembunyikan waktu eksekusi |
| `.output nama_file` | Redirect output ke file |
| `.output` | Kembalikan output ke layar |
| `.help` | Tampilkan semua perintah CLI |

### Implementasi

```sql
-- Mode 1: Output tabel biasa (default)
.mode table
SELECT * FROM produk;

-- Mode 2: Output Markdown (berguna untuk dokumentasi)
.mode markdown
SELECT p.nama_produk, SUM(t.jumlah_beli) AS total_terjual
FROM transaksi t
JOIN produk p ON t.id_produk = p.id_produk
GROUP BY p.nama_produk;

-- Mode 3: Aktifkan timer untuk melihat performa query
.timer on
SELECT * FROM transaksi
JOIN produk    p  ON transaksi.id_produk    = p.id_produk
JOIN pelanggan pl ON transaksi.id_pelanggan = pl.id_pelanggan;
-- Output: Run Time (s): real 0.001 user 0.000 sys 0.000

-- Mode 4: Export hasil query ke file CSV
.mode csv
.output laporan_penjualan.csv
SELECT p.nama_produk,
       pl.nama_pelanggan,
       t.jumlah_beli,
       (p.harga_satuan * t.jumlah_beli) AS total_bayar
FROM transaksi t
JOIN produk    p  ON t.id_produk    = p.id_produk
JOIN pelanggan pl ON t.id_pelanggan = pl.id_pelanggan;
.output   -- kembalikan output ke layar

-- Mode 5: Export ke JSON
.mode json
.output rekap.json
SELECT * FROM pelanggan;
.output
```

### Latihan
> Gunakan `.mode markdown` dan `.output` untuk mengekspor hasil `SUMMARIZE transaksi;` ke file `profil_transaksi.md`.

---

## Ringkasan Perbandingan DuckDB vs Oracle

| # | Fitur | DuckDB | Oracle |
|---|---|---|---|
| 1 | Profiling kolom otomatis | ✅ `SUMMARIZE` | ❌ Manual per kolom |
| 2 | Query file eksternal | ✅ Langsung (`'file.csv'`) | ❌ Perlu External Table |
| 3 | PIVOT native | ✅ Sintaks ringkas | ⚠️ Verbose, nilai hardcode |
| 4 | SELECT * EXCLUDE/REPLACE | ✅ | ❌ Tidak ada |
| 5 | LIST, STRUCT, Lambda | ✅ Native | ❌ Tidak ada |
| 6 | CLI mode & output | ✅ Ringan & fleksibel | ⚠️ SQL*Plus terbatas |

---

## Referensi

- [DuckDB Documentation](https://duckdb.org/docs/)
- [DuckDB SQL Reference](https://duckdb.org/docs/sql/introduction)
- [DuckDB CLI Guide](https://duckdb.org/docs/api/cli/overview)
