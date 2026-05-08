# Pengenalan DuckDB dan Database Columnar

Selamat datang di panduan singkat mengenai DuckDB! Panduan ini akan membahas konsep dasar DuckDB, sintaks yang sering digunakan, serta perbedaan mendasar antara database *columnar* (berbasis kolom) dan database biasa (*row-oriented* / berbasis baris).

---

## 1. Apa itu DuckDB?

**DuckDB** adalah sistem manajemen database relasional (RDBMS) *open-source* yang dirancang khusus untuk beban kerja analitik (OLAP - *Online Analytical Processing*). 
DuckDB sering disebut sebagai "SQLite untuk analitik" karena ia bersifat *in-process* (berjalan di dalam proses aplikasi yang sama) dan tidak memerlukan instalasi server terpisah.

**Keunggulan DuckDB:**
- Sangat cepat untuk kueri analitik (seperti agregasi, pengelompokan data, *join* pada data besar).
- Mampu membaca dan memproses file format CSV, JSON, dan Parquet secara langsung tanpa harus mengimpornya terlebih dahulu.
- Ringan, mudah dipasang, dan tidak butuh setup server yang rumit.

---

## 2. Perbedaan: Columnar Database vs Database Biasa (Row-Oriented)

Database tradisional (seperti MySQL, PostgreSQL) umumnya adalah *Row-Oriented*, sedangkan database yang berfokus pada analitik (seperti DuckDB, ClickHouse) adalah *Columnar*.

| Fitur | Database Biasa (Row-Oriented) | Database Columnar (DuckDB) |
|---|---|---|
| **Penyimpanan Data** | Disimpan **baris demi baris** (*row by row*). | Disimpan **kolom demi kolom** (*column by column*). |
| **Kasus Penggunaan Utama** | **OLTP** (*Online Transaction Processing*). Cocok untuk transaksi harian (seperti *insert*, *update*, *delete* per baris), misal: e-commerce, sistem login. | **OLAP** (*Online Analytical Processing*). Cocok untuk analisis data, *data science*, pembuatan laporan agregasi data yang sangat besar. |
| **Kecepatan Baca (Query)** | Cepat jika Anda ingin mengambil semua kolom dari beberapa baris (misal: `SELECT * FROM users WHERE id = 5`). | Sangat cepat jika Anda ingin mengambil sedikit kolom dari jutaan baris (misal: `SELECT AVG(gaji) FROM karyawan`). |
| **Kompresi Data** | Kurang efisien karena tipe data dalam satu blok memori (satu baris) bervariasi (teks, angka, tanggal dicampur). | Sangat efisien karena setiap blok memori hanya berisi satu kolom dengan tipe data yang sama. |
| **Contoh Database** | MySQL, PostgreSQL, SQLite, Oracle, SQL Server. | DuckDB, ClickHouse, Amazon Redshift, Google BigQuery. |

---

## 3. Sintaks Dasar DuckDB

Karena DuckDB menggunakan standar SQL, sintaksnya sangat familier jika Anda pernah menggunakan PostgreSQL. Berikut adalah beberapa perintah dan fitur sintaks yang sering digunakan di DuckDB.

### Membaca File Secara Langsung (Kelebihan DuckDB)

Anda tidak perlu membuat tabel terlebih dahulu untuk membaca file. DuckDB bisa melakukan kueri langsung terhadap file!

```sql
-- Membaca dari file CSV (DuckDB akan menebak tipe data secara otomatis)
SELECT * FROM read_csv_auto('data.csv');
-- Sintaks pintasan yang lebih singkat:
SELECT * FROM 'data.csv';

-- Membaca dari file Parquet
SELECT * FROM read_parquet('data.parquet');
-- Atau pintasan:
SELECT * FROM 'data.parquet';

-- Membaca dari JSON
SELECT * FROM read_json_auto('data.json');
```

### Membuat Tabel dan Memasukkan Data

```sql
-- Membuat tabel baru
CREATE TABLE pengguna (
    id INTEGER,
    nama VARCHAR,
    umur INTEGER,
    kota VARCHAR
);

-- Memasukkan data ke dalam tabel
INSERT INTO pengguna VALUES 
    (1, 'Budi', 25, 'Jakarta'),
    (2, 'Siti', 22, 'Bandung'),
    (3, 'Andi', 30, 'Surabaya');
```

### Membuat Tabel langsung dari File (CTAS - Create Table As Select)

```sql
-- Membuat tabel 'data_sales' dan langsung mengisi datanya dari file CSV
CREATE TABLE data_sales AS SELECT * FROM 'sales.csv';
```

### Mengekspor Hasil Query ke File

```sql
-- Menyimpan hasil query ke file CSV baru
COPY (SELECT nama, kota FROM pengguna WHERE umur > 20) 
TO 'hasil.csv' (HEADER, DELIMITER ',');

-- Menyimpan hasil query ke format Parquet (sangat efisien untuk analitik)
COPY (SELECT * FROM pengguna) TO 'hasil.parquet' (FORMAT PARQUET);
```

### Query Analitik dan Agregasi

```sql
-- Menghitung rata-rata umur dan total pengguna berdasarkan kota
SELECT 
    kota, 
    AVG(umur) AS rata_rata_umur, 
    COUNT(*) AS jumlah_orang
FROM pengguna
GROUP BY kota
ORDER BY jumlah_orang DESC;
```

---

## Kesimpulan

- Gunakan **Database Row-Oriented (Biasa)** jika Anda membuat aplikasi operasional yang perlu mengelola banyak data dari perorangan (*insert* / *update* cepat).
- Gunakan **Database Columnar (seperti DuckDB)** jika Anda bekerja sebagai Data Analyst/Scientist yang perlu menarik kesimpulan dari jutaan baris data, membaca file besar secara efisien, dan melakukan perhitungan statistik dengan cepat.
