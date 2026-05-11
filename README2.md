# DuckDB - Panduan Lengkap

## Daftar Isi
1. [Pengenalan DuckDB](#pengenalan-duckdb)
2. [Setup & Inisiasi Project](#setup--inisiasi-project)
3. [Perbedaan SQL Biasa vs Columnar Database](#perbedaan-sql-biasa-vs-columnar-database)
4. [Sintaks Dasar DuckDB](#sintaks-dasar-duckdb)
5. [Contoh Praktis](#contoh-praktis)

---

## Pengenalan DuckDB

DuckDB adalah database management system yang dirancang untuk analytical queries dengan performa tinggi. DuckDB menggunakan **columnar storage** yang membuatnya sangat efisien untuk data analysis.

**Keunggulan DuckDB:**
- ✅ Columnari storage (analisis data super cepat)
- ✅ In-process database (tidak perlu server terpisah)
- ✅ SQL-compliant
- ✅ Lightweight dan portable
- ✅ Support untuk berbagai format data (CSV, Parquet, JSON, dll)

---

## Setup & Inisiasi Project

### Cara 1: Cara Paling Mudah (Lewat File Explorer)

**Langkah-langkah:**

1. **Buka File Explorer Windows** (`Win + E`)

2. **Navigasi ke folder tujuan:**
   ```
   D:\ITT-TELU-SM-6\ASPRAK BASIS DATA\duckdb
   ```

3. **Buka Terminal Command Prompt:**
   - Klik pada Address Bar di bagian atas
   - Hapus alamat yang ada
   - Ketik: `cmd`
   - Tekan `Enter`

4. **Setelah terminal terbuka, buat database:**
   ```bash
   duckdb asisten_db.db
   ```
   > **Catatan:** Anda bebas mengganti `asisten_db` dengan nama database yang Anda inginkan

5. **Database Anda telah dibuat!** 🎉
   - File `.db` akan tersimpan permanen di folder tersebut
   - Anda dapat membuka kembali dengan perintah yang sama

### Cara 2: Via PowerShell atau CMD dari Folder Manapun

```bash
# Navigasi ke folder
cd D:\ITT-TELU-SM-6\ASPRAK BASIS DATA\duckdb

# Buat/buka database
duckdb asisten_db.db
```

### Cara 3: Dengan Python Script

```python
import duckdb

# Membuat/membuka database
conn = duckdb.connect('asisten_db.db')

# Membuat tabel
conn.execute("""
    CREATE TABLE IF NOT EXISTS siswa (
        id INTEGER PRIMARY KEY,
        nama VARCHAR,
        nilai DECIMAL(5,2)
    )
""")

print("Database berhasil dibuat!")
conn.close()
```

---

## Perbedaan SQL Biasa vs Columnar Database

### SQL Biasa (Row-Oriented Storage)

**Cara Penyimpanan:**
```
Tabel: Siswa
┌─────────────┬───────────┬────────┐
│ ID  │ Nama        │ Nilai  │
├─────────────┼───────────┼────────┤
│ 1   │ Budi        │ 85     │ <- Baris 1 disimpan bersama
│ 2   │ Ani         │ 90     │ <- Baris 2 disimpan bersama
│ 3   │ Citra       │ 88     │ <- Baris 3 disimpan bersama
└─────────────┴───────────┴────────┘

Urutan penyimpanan di disk:
[1, Budi, 85, 2, Ani, 90, 3, Citra, 88]
```

**Karakteristik:**
- Data disimpan per **baris** (row-by-row)
- Cocok untuk **OLTP** (Online Transaction Processing)
- Baik untuk: INSERT, UPDATE, DELETE individual records
- Kurang efisien untuk analisis data besar

**Contoh Database:** MySQL, PostgreSQL, SQL Server

### Columnar Database (Column-Oriented Storage)

**Cara Penyimpanan:**
```
Tabel: Siswa
┌─────────────┬───────────┬────────┐
│ ID  │ Nama        │ Nilai  │
├─────────────┼───────────┼────────┤
│ 1   │ Budi        │ 85     │
│ 2   │ Ani         │ 90     │
│ 3   │ Citra       │ 88     │
└─────────────┴───────────┴────────┘

Urutan penyimpanan di disk:
Column ID   : [1, 2, 3]
Column Nama : [Budi, Ani, Citra]
Column Nilai: [85, 90, 88]
```

**Karakteristik:**
- Data disimpan per **kolom** (column-by-column)
- Cocok untuk **OLAP** (Online Analytical Processing)
- Baik untuk: Aggregation, filtering, analysis
- **Super cepat** untuk query yang hanya butuh beberapa kolom
- Kompresi data lebih baik

**Contoh Database:** DuckDB, Parquet, ClickHouse, Apache Arrow

### Perbandingan Visual

| Aspek | Row-Oriented | Column-Oriented |
|-------|--------------|-----------------|
| **Penyimpanan** | Per baris | Per kolom |
| **Kompresi** | Sedang | Sangat baik |
| **Query tunggal baris** | ⚡ Cepat | Lambat |
| **Aggregation (SUM, AVG)** | Lambat | ⚡⚡ Super cepat |
| **INSERT individual** | ⚡ Cepat | Lambat |
| **Analisis data besar** | Lambat | ⚡⚡ Super cepat |
| **Penggunaan RAM** | Lebih banyak | Lebih sedikit |

### Contoh Kecepatan Nyata

**Query: Hitung rata-rata nilai dari 1 juta siswa**

```sql
SELECT AVG(nilai) FROM siswa;
```

- **Row-Oriented:** Harus membaca semua 1 juta baris lengkap ❌
- **Column-Oriented:** Hanya baca kolom `nilai` ✅ (50x lebih cepat!)

---

## Sintaks Dasar DuckDB

### 1. Membuat Tabel

#### Syntax Dasar
```sql
CREATE TABLE nama_tabel (
    kolom1 TIPE_DATA PRIMARY KEY,
    kolom2 TIPE_DATA,
    kolom3 TIPE_DATA
);
```

#### Contoh
```sql
CREATE TABLE siswa (
    id INTEGER PRIMARY KEY,
    nama VARCHAR,
    email VARCHAR UNIQUE,
    tanggal_lahir DATE,
    nilai DECIMAL(5,2),
    is_aktif BOOLEAN DEFAULT TRUE
);
```

### 2. Tipe Data DuckDB

```sql
-- Numerik
INTEGER, BIGINT, SMALLINT, TINYINT  -- Bilangan bulat
DECIMAL(p,s), FLOAT, DOUBLE         -- Desimal dan floating point

-- String
VARCHAR, TEXT, STRING               -- Teks

-- Date & Time
DATE, TIME, TIMESTAMP               -- Tanggal dan waktu

-- Boolean
BOOLEAN                             -- TRUE/FALSE

-- Array & Struct
INTEGER[], VARCHAR[]                -- Array
STRUCT(name VARCHAR, age INTEGER)   -- Struktur kompleks
```

### 3. INSERT Data

```sql
-- Insert single record
INSERT INTO siswa (id, nama, email, nilai)
VALUES (1, 'Budi Santoso', 'budi@email.com', 85.50);

-- Insert multiple records
INSERT INTO siswa (id, nama, email, nilai) VALUES
(2, 'Ani Wijaya', 'ani@email.com', 90.75),
(3, 'Citra Dewi', 'citra@email.com', 88.25),
(4, 'Doni Hermawan', 'doni@email.com', 92.00);

-- Insert dari SELECT
INSERT INTO siswa_backup
SELECT * FROM siswa WHERE nilai > 85;
```

### 4. SELECT / Query Data

```sql
-- Select semua kolom
SELECT * FROM siswa;

-- Select kolom tertentu (LEBIH CEPAT di columnar!)
SELECT nama, nilai FROM siswa;

-- Filter data
SELECT * FROM siswa WHERE nilai > 85;

-- Sorting
SELECT * FROM siswa ORDER BY nilai DESC;

-- Limit hasil
SELECT * FROM siswa LIMIT 5;
```

### 5. Aggregation (Sangat Cepat di DuckDB!)

```sql
-- Count
SELECT COUNT(*) as total_siswa FROM siswa;

-- Sum
SELECT SUM(nilai) as total_nilai FROM siswa;

-- Average
SELECT AVG(nilai) as rata_rata FROM siswa;

-- Min & Max
SELECT MIN(nilai) as nilai_min, MAX(nilai) as nilai_max FROM siswa;

-- Group By
SELECT nilai, COUNT(*) as jumlah 
FROM siswa 
GROUP BY nilai 
ORDER BY nilai DESC;
```

### 6. UPDATE Data

```sql
-- Update satu record
UPDATE siswa 
SET nilai = 95.00 
WHERE id = 1;

-- Update multiple records
UPDATE siswa 
SET is_aktif = FALSE 
WHERE nilai < 70;
```

### 7. DELETE Data

```sql
-- Delete dengan kondisi
DELETE FROM siswa WHERE id = 1;

-- Delete semua data (HATI-HATI!)
DELETE FROM siswa;
```

### 8. JOIN Query

```sql
-- Create related table
CREATE TABLE mata_pelajaran (
    id INTEGER PRIMARY KEY,
    siswa_id INTEGER,
    pelajaran VARCHAR,
    nilai DECIMAL(5,2)
);

-- INNER JOIN
SELECT 
    s.nama,
    m.pelajaran,
    m.nilai
FROM siswa s
INNER JOIN mata_pelajaran m ON s.id = m.siswa_id;

-- LEFT JOIN
SELECT 
    s.nama,
    COUNT(m.id) as jumlah_pelajaran
FROM siswa s
LEFT JOIN mata_pelajaran m ON s.id = m.siswa_id
GROUP BY s.nama;
```

### 9. Export & Import Data

```sql
-- Export ke CSV
COPY (SELECT * FROM siswa) TO 'siswa.csv' WITH (FORMAT CSV, HEADER TRUE);

-- Import dari CSV
COPY siswa FROM 'siswa.csv' WITH (FORMAT CSV, HEADER TRUE);

-- Export ke Parquet (format columnar)
COPY (SELECT * FROM siswa) TO 'siswa.parquet';

-- Import dari Parquet
COPY siswa FROM 'siswa.parquet';

-- Read CSV tanpa import
SELECT * FROM read_csv('siswa.csv');

-- Read JSON
SELECT * FROM read_json('data.json');
```

### 10. View & Temporary Tables

```sql
-- Create view
CREATE VIEW siswa_terbaik AS
SELECT * FROM siswa WHERE nilai > 85;

-- Use view
SELECT * FROM siswa_terbaik;

-- Temporary table (exists hanya di session ini)
CREATE TEMPORARY TABLE temp_data AS
SELECT * FROM siswa WHERE nilai > 90;
```

---

## Contoh Praktis

### Scenario: Database Nilai Siswa

#### 1. Setup & Inisiasi

```bash
# Buka terminal di folder duckdb
duckdb asisten_db.db
```

#### 2. Buat Tabel

```sql
-- Tabel Siswa
CREATE TABLE siswa (
    id INTEGER PRIMARY KEY,
    nama VARCHAR NOT NULL,
    email VARCHAR UNIQUE,
    tanggal_lahir DATE,
    is_aktif BOOLEAN DEFAULT TRUE
);

-- Tabel Mata Pelajaran
CREATE TABLE mata_pelajaran (
    id INTEGER PRIMARY KEY,
    nama VARCHAR NOT NULL
);

-- Tabel Nilai (Junction Table)
CREATE TABLE nilai_siswa (
    id INTEGER PRIMARY KEY,
    siswa_id INTEGER,
    mata_pelajaran_id INTEGER,
    nilai DECIMAL(5,2),
    tanggal DATE DEFAULT CURRENT_DATE,
    FOREIGN KEY (siswa_id) REFERENCES siswa(id),
    FOREIGN KEY (mata_pelajaran_id) REFERENCES mata_pelajaran(id)
);
```

#### 3. Insert Sample Data

```sql
-- Insert Siswa
INSERT INTO siswa VALUES
(1, 'Budi Santoso', 'budi@school.com', '2005-03-15', TRUE),
(2, 'Ani Wijaya', 'ani@school.com', '2005-07-22', TRUE),
(3, 'Citra Dewi', 'citra@school.com', '2005-11-08', TRUE),
(4, 'Doni Hermawan', 'doni@school.com', '2006-01-30', TRUE),
(5, 'Eka Putri', 'eka@school.com', '2005-05-12', TRUE);

-- Insert Mata Pelajaran
INSERT INTO mata_pelajaran VALUES
(1, 'Matematika'),
(2, 'Bahasa Indonesia'),
(3, 'Bahasa Inggris'),
(4, 'IPA'),
(5, 'IPS');

-- Insert Nilai
INSERT INTO nilai_siswa VALUES
(1, 1, 1, 85.50, '2024-05-10'),
(2, 1, 2, 88.00, '2024-05-10'),
(3, 1, 3, 80.25, '2024-05-10'),
(4, 2, 1, 92.75, '2024-05-10'),
(5, 2, 2, 90.50, '2024-05-10'),
(6, 2, 3, 94.00, '2024-05-10'),
(7, 3, 1, 78.50, '2024-05-10'),
(8, 3, 2, 82.25, '2024-05-10'),
(9, 3, 3, 79.75, '2024-05-10'),
(10, 4, 1, 95.00, '2024-05-10'),
(11, 4, 2, 93.50, '2024-05-10'),
(12, 4, 3, 96.25, '2024-05-10'),
(13, 5, 1, 88.75, '2024-05-10'),
(14, 5, 2, 86.50, '2024-05-10'),
(15, 5, 3, 89.00, '2024-05-10');
```

#### 4. Query Analisis Data

```sql
-- Rata-rata nilai per siswa
SELECT 
    s.nama,
    ROUND(AVG(ns.nilai), 2) as rata_rata_nilai,
    COUNT(ns.id) as jumlah_pelajaran
FROM siswa s
LEFT JOIN nilai_siswa ns ON s.id = ns.siswa_id
GROUP BY s.id, s.nama
ORDER BY rata_rata_nilai DESC;

-- Siswa dengan nilai tertinggi per mata pelajaran
SELECT 
    mp.nama as mata_pelajaran,
    s.nama as nama_siswa,
    ns.nilai
FROM nilai_siswa ns
JOIN siswa s ON ns.siswa_id = s.id
JOIN mata_pelajaran mp ON ns.mata_pelajaran_id = mp.id
WHERE ns.nilai = (
    SELECT MAX(nilai) 
    FROM nilai_siswa 
    WHERE mata_pelajaran_id = mp.id
)
ORDER BY mp.nama;

-- Statistik lengkap per mata pelajaran
SELECT 
    mp.nama as mata_pelajaran,
    COUNT(ns.id) as jumlah_siswa,
    ROUND(AVG(ns.nilai), 2) as rata_rata,
    MIN(ns.nilai) as nilai_terendah,
    MAX(ns.nilai) as nilai_tertinggi,
    ROUND(STDDEV(ns.nilai), 2) as standar_deviasi
FROM mata_pelajaran mp
LEFT JOIN nilai_siswa ns ON mp.id = ns.mata_pelajaran_id
GROUP BY mp.id, mp.nama
ORDER BY rata_rata DESC;
```

#### 5. Export Hasil Analisis

```sql
-- Export ke CSV
COPY (
    SELECT 
        s.nama,
        ROUND(AVG(ns.nilai), 2) as rata_rata_nilai
    FROM siswa s
    LEFT JOIN nilai_siswa ns ON s.id = ns.siswa_id
    GROUP BY s.id, s.nama
    ORDER BY rata_rata_nilai DESC
) TO 'laporan_nilai.csv' WITH (FORMAT CSV, HEADER TRUE);

-- Export ke JSON
COPY (SELECT * FROM siswa) TO 'siswa.json' WITH (FORMAT JSON);

-- Export ke Parquet (optimal untuk data besar)
COPY (SELECT * FROM siswa) TO 'siswa.parquet';
```

---

## Tips & Trik DuckDB

### 1. Performa Query
```sql
-- Gunakan SELECT kolom spesifik, bukan SELECT *
-- ❌ Lambat
SELECT * FROM siswa WHERE nilai > 85;

-- ✅ Cepat (columnar advantage!)
SELECT nama, nilai FROM siswa WHERE nilai > 85;
```

### 2. Format Parquet untuk Data Besar
```sql
-- Parquet lebih cepat dan lebih kecil dari CSV
COPY (SELECT * FROM siswa) TO 'siswa.parquet';
SELECT * FROM 'siswa.parquet' WHERE nilai > 85;
```

### 3. Aggregate Functions yang Berguna
```sql
-- List aggregation (kumpulkan data dalam array)
SELECT 
    mata_pelajaran_id,
    LIST(nilai) as semua_nilai
FROM nilai_siswa
GROUP BY mata_pelajaran_id;

-- String concatenation
SELECT 
    siswa_id,
    STRING_AGG(DISTINCT pelajaran, ', ') as pelajaran_list
FROM mata_pelajaran
GROUP BY siswa_id;
```

### 4. Window Functions
```sql
-- Ranking siswa per mata pelajaran
SELECT 
    s.nama,
    mp.nama as mata_pelajaran,
    ns.nilai,
    ROW_NUMBER() OVER (
        PARTITION BY mp.id 
        ORDER BY ns.nilai DESC
    ) as ranking
FROM nilai_siswa ns
JOIN siswa s ON ns.siswa_id = s.id
JOIN mata_pelajaran mp ON ns.mata_pelajaran_id = mp.id;
```

### 5. Recursive Query (Advanced)
```sql
-- Contoh: Generate angka 1-10
WITH RECURSIVE cte AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM cte WHERE n < 10
)
SELECT * FROM cte;
```

---

## Cheat Sheet Singkat

| Perintah | Fungsi |
|----------|--------|
| `duckdb database.db` | Buka/buat database |
| `.tables` | Lihat semua tabel |
| `.schema tabel_name` | Lihat struktur tabel |
| `.quit` atau `.exit` | Keluar dari DuckDB |
| `DESCRIBE tabel_name;` | Info kolom tabel |
| `SELECT COUNT(*) FROM tabel;` | Hitung baris |
| `PRAGMA table_info(tabel_name);` | Detail kolom |

---

## Referensi & Dokumentasi

- **Website Resmi:** https://duckdb.org
- **Documentation:** https://duckdb.org/docs/
- **SQL Reference:** https://duckdb.org/docs/sql/introduction
- **Python API:** https://duckdb.org/docs/api/python/overview

---

**Dibuat:** 12 Mei 2026  
**Untuk:** ITT TELU - Asprak Basis Data  
**DuckDB Version:** Latest

