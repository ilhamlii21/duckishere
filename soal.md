# Soal Praktikum: 6 Fitur Unik DuckDB
## Dataset: Jakarta Busway Transaction Log

> **Mata Kuliah:** Columnar Database  
> **Database:** Busway Ticketing System  
> **Tools:** DuckDB CLI

---

## Informasi Dataset

Anda diberikan dua tabel:

### Tabel 1: `halte_busway` (30 halte aktif)
Informasi tentang halte (stasiun) Busway Jakarta:
- `id_halte` - ID unik halte (H01-H30)
- `nama_halte` - Nama halte (Blok M, Harmoni, Ancol, dll)
- `koridor_utama` - Koridor yang dilayani (1-13)
- `lokasi_lat`, `lokasi_lon` - Koordinat GPS
- `tipe_halte` - Ujung / Transit / Sentral / Reguler
- `fasilitas_toilet`, `fasilitas_mushola` - Fasilitas ketersediaan (Boolean)
- `status_operasional` - Status halte
- `alamat_lengkap` - Alamat lengkap
- `kecamatan` - Kecamatan
- `kota_administrasi` - Kota administrasi

### Tabel 2: `tap_in_out_log` (30 transaksi sampel)
Log transaksi tap in/out pengguna Busway:
- `id_transaksi` - UUID unik transaksi
- `nomor_kartu` - Nomor kartu pembayaran (K-001234, dll)
- `id_halte_asal` - Halte keberangkatan
- `id_halte_tujuan` - Halte tujuan
- `waktu_tap_in` - Waktu naik bus
- `waktu_tap_out` - Waktu turun bus
- `tarif_debet` - Tarif yang dipotong (3500 IDR standar)
- `sisa_saldo_kartu` - Saldo kartu setelah transaksi
- `tipe_pembayaran` - JakLingko, BCA Flazz, Mandiri E-money, BNI TapCash
- `jenis_perjalanan` - Reguler, Pelajar, Lansia
- `catatan_sistem` - Catatan (Tap berhasil, Subsidi, Saldo Kritis)

---

# BAGIAN 1: SUMMARIZE (Data Profiling Instan)

## Soal 1.1 - Basic SUMMARIZE
**Level:** Dasar

Jalankan perintah `SUMMARIZE` pada tabel `halte_busway` dan jawab:
1. Berapa jumlah halte dalam database?
2. Kolom mana yang memiliki nilai NULL?
3. Berapa nilai rata-rata (`avg`) dan standar deviasi (`std`) dari `koridor_utama`?

**Hint:** Gunakan `SUMMARIZE halte_busway;`

---

## Soal 1.2 - SUMMARIZE pada Kolom Spesifik
**Level:** Dasar

Jalankan `SUMMARIZE` pada tabel `tap_in_out_log` dan tentukan:
1. Berapa banyak distinct value dalam kolom `tipe_pembayaran`?
2. Berapakah nilai minimum dari `sisa_saldo_kartu`?
3. Berapakah nilai maksimum dari `tarif_debet`?

**Hint:** Gunakan `SUMMARIZE tap_in_out_log;`

---

## Soal 1.3 - SUMMARIZE pada Hasil JOIN
**Level:** Menengah

Gunakan SUMMARIZE pada hasil JOIN berikut:
```sql
SELECT h_asal.nama_halte AS halte_asal,
       h_tujuan.nama_halte AS halte_tujuan,
       t.tarif_debet,
       t.sisa_saldo_kartu,
       EXTRACT(HOUR FROM t.waktu_tap_in) AS jam_perjalanan
FROM tap_in_out_log t
JOIN halte_busway h_asal ON t.id_halte_asal = h_asal.id_halte
JOIN halte_busway h_tujuan ON t.id_halte_tujuan = h_tujuan.id_halte;
```

Jawab:
1. Kolom apa yang NOT NULL di semua baris?
2. Berapa distinct halte_asal dan halte_tujuan?
3. Berapa rata-rata jam_perjalanan (dalam jam)?

---

---

# BAGIAN 2: Query Langsung ke File (CSV / Parquet)

## Soal 2.1 - Export Tabel ke CSV
**Level:** Dasar

Lakukan langkah-langkah berikut:
1. **Export tabel `halte_busway` ke file CSV:**
   ```sql
   COPY halte_busway TO 'halte_busway.csv' (HEADER, DELIMITER ',');
   ```

2. Verifikasi file sudah tercreate dengan perintah CLI:
   ```
   .shell dir halte_busway.csv
   ```

3. Buka file `halte_busway.csv` dan cek:
   - Berapa baris data yang ter-export?
   - Apa delimiter yang digunakan?

---

## Soal 2.2 - Query File CSV Langsung
**Level:** Dasar

Setelah file CSV terbuat, jalankan query:
```sql
SELECT * FROM 'halte_busway.csv' LIMIT 5;
```

Pertanyaan:
1. Apakah semua kolom terbaca dengan benar?
2. Cobalah filter halte hanya dari kota 'Jakarta Pusat':
   ```sql
   SELECT * FROM 'halte_busway.csv'
   WHERE kota_administrasi = 'Jakarta Pusat';
   ```
   Berapa banyak halte?

---

## Soal 2.3 - Export JOIN Result ke Parquet
**Level:** Menengah

Export hasil JOIN ke format Parquet:
```sql
COPY (
    SELECT t.id_transaksi,
           h_asal.nama_halte AS halte_asal,
           h_tujuan.nama_halte AS halte_tujuan,
           t.tipe_pembayaran,
           t.jenis_perjalanan,
           t.tarif_debet,
           t.sisa_saldo_kartu,
           CAST((t.waktu_tap_out - t.waktu_tap_in) AS VARCHAR) AS durasi_perjalanan
    FROM tap_in_out_log t
    JOIN halte_busway h_asal ON t.id_halte_asal = h_asal.id_halte
    JOIN halte_busway h_tujuan ON t.id_halte_tujuan = h_tujuan.id_halte
) TO 'rekap_transaksi.parquet' (FORMAT PARQUET);
```

Kemudian, query file parquet untuk:
1. Menampilkan 5 transaksi pertama
2. Hitung total transaksi per `tipe_pembayaran`
3. Filter transaksi dengan `jenis_perjalanan = 'Subsidi Pelajar'`

---

## Soal 2.4 - Gabungkan CSV + Tabel Database
**Level:** Menengah

Gunakan JOIN antara file CSV dan tabel database:
```sql
SELECT f.id_halte,
       f.nama_halte,
       COUNT(t.id_transaksi) AS jumlah_transaksi_keluar
FROM 'halte_busway.csv' f
LEFT JOIN tap_in_out_log t ON f.id_halte = t.id_halte_asal
GROUP BY f.id_halte, f.nama_halte
ORDER BY jumlah_transaksi_keluar DESC
LIMIT 10;
```

Jawab:
1. Halte mana yang paling sering menjadi asal perjalanan?
2. Halte mana yang tidak memiliki transaksi sama sekali?

---

---

# BAGIAN 3: PIVOT Native

## Soal 3.1 - Basic PIVOT: Perjalanan per Tipe Pembayaran
**Level:** Menengah

Buat view rekap dulu:
```sql
CREATE OR REPLACE VIEW rekap_pembayaran AS
SELECT t.tipe_pembayaran,
       h.nama_halte,
       COUNT(t.id_transaksi) AS jumlah_transaksi
FROM tap_in_out_log t
JOIN halte_busway h ON t.id_halte_asal = h.id_halte
GROUP BY t.tipe_pembayaran, h.nama_halte;
```

Kemudian pivot sehingga setiap `tipe_pembayaran` menjadi kolom:
```sql
PIVOT rekap_pembayaran
ON tipe_pembayaran
USING SUM(jumlah_transaksi)
GROUP BY nama_halte;
```

Pertanyaan:
1. Halte mana yang paling digunakan dengan pembayaran JakLingko?
2. Pembayaran apa yang paling jarang digunakan?
3. Halte mana yang hanya memiliki satu jenis pembayaran?

---

## Soal 3.2 - PIVOT: Analisis per Tipe Perjalanan
**Level:** Menengah**

Buat view untuk menggabungkan data perjalanan:
```sql
CREATE OR REPLACE VIEW analisis_perjalanan AS
SELECT t.jenis_perjalanan,
       DATE(t.waktu_tap_in) AS tanggal,
       t.tipe_pembayaran,
       COUNT(t.id_transaksi) AS transaksi
FROM tap_in_out_log t
GROUP BY t.jenis_perjalanan, DATE(t.waktu_tap_in), t.tipe_pembayaran;
```

Pivot sehingga setiap `jenis_perjalanan` menjadi kolom:
```sql
PIVOT analisis_perjalanan
ON jenis_perjalanan
USING SUM(transaksi)
GROUP BY tanggal, tipe_pembayaran;
```

Analisis:
1. Perjalanan jenis apa yang mendapat subsidi (berdasarkan catatan sistem)?
2. Pada tanggal mana, perjalanan reguler paling tinggi?
3. Apakah ada perjalanan Lansia menggunakan semua tipe pembayaran?

---

## Soal 3.3 - PIVOT: Perbandingan Saldo
**Level:** Lanjut

Buat kategori saldo (High/Medium/Low):
```sql
CREATE OR REPLACE VIEW kategori_saldo AS
SELECT t.nomor_kartu,
       CASE
           WHEN t.sisa_saldo_kartu >= 50000 THEN 'High'
           WHEN t.sisa_saldo_kartu >= 10000 THEN 'Medium'
           ELSE 'Low'
       END AS kategori_saldo,
       t.jenis_perjalanan,
       COUNT(t.id_transaksi) AS freq
FROM tap_in_out_log t
GROUP BY t.nomor_kartu, kategori_saldo, t.jenis_perjalanan;
```

Pivot sehingga kategori_saldo menjadi kolom:
```sql
PIVOT kategori_saldo
ON kategori_saldo
USING SUM(freq)
GROUP BY nomor_kartu, jenis_perjalanan;
```

Insight:
1. Kartu mana yang paling sering transaksi dengan saldo Low?
2. Apakah pengguna dengan saldo High lebih sering melakukan perjalanan reguler?
3. Adakah kartu yang hanya bertransaksi pada satu kategori saldo saja?

---

---

# BAGIAN 4: SELECT * EXCLUDE dan REPLACE

## Soal 4.1 - EXCLUDE Kolom Sensitif
**Level:** Dasar

Tampilkan semua halte **kecuali** koordinat GPS (dianggap sensitif):
```sql
SELECT * EXCLUDE (lokasi_lat, lokasi_lon) FROM halte_busway LIMIT 5;
```

Pertanyaan:
1. Berapa kolom yang ditampilkan?
2. Apakah kolom `id_halte` dan `nama_halte` masih ada?

---

## Soal 4.2 - EXCLUDE dari Transaksi
**Level:** Dasar

Tampilkan transaksi tanpa kolom internal (id_transaksi, catatan_sistem):
```sql
SELECT * EXCLUDE (id_transaksi, catatan_sistem) 
FROM tap_in_out_log 
LIMIT 10;
```

Analisis:
1. Informasi apa yang masih ditampilkan untuk pengguna akhir?
2. Adakah data duplikat dalam 10 baris pertama?

---

## Soal 4.3 - REPLACE: Kalkulasi Harga Dengan PPN
**Level:** Menengah**

Gunakan REPLACE untuk menampilkan kolom-kolom tertentu dengan nilai yang sudah dimodifikasi. Hitung durasi perjalanan dari `waktu_tap_in` dan `waktu_tap_out`:

```sql
SELECT id_halte_asal,
       id_halte_tujuan,
       * EXCLUDE (id_transaksi, waktu_tap_in, waktu_tap_out, catatan_sistem)
       REPLACE (
           EXTRACT(EPOCH FROM (waktu_tap_out - waktu_tap_in))/60 AS durasi_menit,
           tarif_debet * 1.10 AS tarif_dengan_pajak
       )
FROM tap_in_out_log
LIMIT 10;
```

Pertanyaan:
1. Berapa durasi rata-rata perjalanan dalam menit?
2. Berapakah tarif dengan pajak untuk transaksi pertama?
3. Apakah ada transaksi dengan durasi negatif (error)?

---

## Soal 4.4 - REPLACE: Transformasi Nama Halte
**Level:** Menengah**

Ubah nama halte menjadi UPPERCASE dan tambahkan tipe halte:
```sql
SELECT * REPLACE (
    UPPER(nama_halte) AS nama_halte,
    CONCAT(koridor_utama, ' - ', tipe_halte) AS info_koridor
)
EXCLUDE (lokasi_lat, lokasi_lon, alamat_lengkap)
FROM halte_busway
WHERE status_operasional = 'Aktif'
ORDER BY koridor_utama;
```

Insight:
1. Halte mana dari Koridor 1 yang ada?
2. Tipe halte apa saja yang ada di Koridor 3?
3. Berapa halte yang bukan tipe 'Reguler'?

---

---

# BAGIAN 5: LIST, STRUCT, dan Lambda Function

## Soal 5.1 - LIST: Kumpulkan Halte per Koridor
**Level:** Menengah

Kumpulkan semua nama halte per koridor menggunakan `list()`:
```sql
SELECT koridor_utama,
       list(nama_halte) AS daftar_halte,
       COUNT(*) AS jumlah_halte
FROM halte_busway
GROUP BY koridor_utama
ORDER BY koridor_utama;
```

Pertanyaan:
1. Koridor berapa yang memiliki halte paling banyak?
2. Berapa halte di Koridor 1?
3. Adakah koridor yang hanya memiliki 1 halte?

---

## Soal 5.2 - LIST: Kumpulkan Transaksi per Pengguna
**Level:** Menengah

Kumpulkan semua halte tujuan yang pernah dikunjungi per nomor kartu:
```sql
SELECT nomor_kartu,
       list(DISTINCT id_halte_tujuan) AS halte_tujuan_dikunjungi,
       list_unique_count(list(id_halte_tujuan)) AS jumlah_halte_unik,
       COUNT(*) AS total_perjalanan
FROM tap_in_out_log
GROUP BY nomor_kartu
ORDER BY total_perjalanan DESC
LIMIT 10;
```

Analisis:
1. Kartu mana yang mengunjungi halte paling banyak?
2. Adakah kartu yang hanya mengunjungi 1 halte?
3. Rata-rata berapa halte yang dikunjungi per kartu?

---

## Soal 5.3 - STRUCT: Gabungkan Info Halte
**Level:** Menengah**

Buat STRUCT untuk menyimpan info halte lengkap:
```sql
SELECT nama_halte,
       {
           'tipe': tipe_halte,
           'kota': kota_administrasi,
           'fasilitas_toilet': fasilitas_toilet,
           'fasilitas_mushola': fasilitas_mushola
       } AS info_halte
FROM halte_busway
WHERE kota_administrasi = 'Jakarta Pusat'
LIMIT 5;
```

Pertanyaan:
1. Berapa halte di Jakarta Pusat yang memiliki toilet?
2. Berapa halte yang memiliki KEDUA fasilitas (toilet AND mushola)?
3. Tipe halte apa saja di Jakarta Pusat?

---

## Soal 5.4 - Lambda: Filter Halte dengan Fasilitas Lengkap
**Level:** Lanjut**

Gunakan lambda untuk filter halte yang memiliki KEDUA fasilitas:
```sql
SELECT 
    FILTER(
        list(nama_halte),
        x -> (
            (SELECT fasilitas_toilet AND fasilitas_mushola 
             FROM halte_busway WHERE nama_halte = x LIMIT 1)
        )
    ) AS halte_fasilitas_lengkap
FROM halte_busway
LIMIT 1;
```

**Alternative (lebih mudah):**
```sql
SELECT list_filter(
    list(nama_halte),
    x -> (x IN (
        SELECT nama_halte FROM halte_busway 
        WHERE fasilitas_toilet = true AND fasilitas_mushola = true
    ))
) AS halte_lengkap
FROM halte_busway
LIMIT 1;
```

Analisis:
1. Berapa halte yang memiliki fasilitas lengkap?
2. Halte-halte mana saja itu?

---

## Soal 5.5 - Lambda: Transform Harga Diskon
**Level:** Lanjut**

Simulasi: dari list saldo pengguna, hitung saldo setelah dikurangi tarif:
```sql
SELECT nomor_kartu,
       list(sisa_saldo_kartu) AS saldo_historis,
       list_transform(
           list(sisa_saldo_kartu),
           x -> CASE
               WHEN x < 10000 THEN 'LOW'
               WHEN x < 50000 THEN 'MEDIUM'
               ELSE 'HIGH'
           END
       ) AS kategorisasi_saldo
FROM tap_in_out_log
GROUP BY nomor_kartu
LIMIT 5;
```

Insight:
1. Kartu mana yang pernah memiliki saldo LOW?
2. Berapa kartu yang konsisten HIGH (semua transaksi saldo >= 50000)?
3. Adakah kartu yang belum pernah mencapai saldo MEDIUM atau LOW?

---

---

# BAGIAN 6: CLI Commands (.mode, .timer, .output)

## Soal 6.1 - Ubah Mode Output ke Markdown
**Level:** Dasar

Jalankan perintah berikut:
```
.mode markdown
SELECT koridor_utama, 
       COUNT(*) AS jumlah_halte,
       list(nama_halte) AS halte_list
FROM halte_busway
GROUP BY koridor_utama
ORDER BY koridor_utama;
```

Pertanyaan:
1. Apakah output berbentuk tabel Markdown?
2. Salin hasil dan paste ke file editor, apakah formatnya valid untuk dokumentasi?

---

## Soal 6.2 - Timer: Ukur Performa Query JOIN
**Level:** Dasar

Aktifkan timer dan jalankan query JOIN kompleks:
```
.timer on
SELECT h_asal.nama_halte,
       h_tujuan.nama_halte,
       COUNT(t.id_transaksi) AS jumlah
FROM tap_in_out_log t
JOIN halte_busway h_asal ON t.id_halte_asal = h_asal.id_halte
JOIN halte_busway h_tujuan ON t.id_halte_tujuan = h_tujuan.id_halte
GROUP BY h_asal.nama_halte, h_tujuan.nama_halte
ORDER BY jumlah DESC;
.timer off
```

Jawab:
1. Berapa lama eksekusi query (real time)?
2. Apakah time user atau sys lebih tinggi?

---

## Soal 6.3 - Export ke CSV dengan .output
**Level:** Dasar

Redirect output hasil query ke file CSV:
```
.mode csv
.output laporan_halte_fasilitas.csv
SELECT id_halte,
       nama_halte,
       kota_administrasi,
       CASE WHEN fasilitas_toilet THEN 'Ya' ELSE 'Tidak' END AS toilet,
       CASE WHEN fasilitas_mushola THEN 'Ya' ELSE 'Tidak' END AS mushola,
       tipe_halte
FROM halte_busway
WHERE status_operasional = 'Aktif'
ORDER BY kota_administrasi;
.output
```

Verifikasi:
1. Apakah file `laporan_halte_fasilitas.csv` terbuat?
2. Buka file tersebut di Excel, apakah data terbaca dengan benar?
3. Berapa baris data yang di-export?

---

## Soal 6.4 - Export ke JSON
**Level:** Menengah**

Export hasil analisis transaksi ke JSON:
```
.mode json
.output analisis_pembayaran.json
SELECT tipe_pembayaran,
       COUNT(*) AS total_transaksi,
       ROUND(AVG(tarif_debet), 2) AS tarif_rata,
       ROUND(AVG(sisa_saldo_kartu), 2) AS saldo_rata,
       MIN(sisa_saldo_kartu) AS saldo_min,
       MAX(sisa_saldo_kartu) AS saldo_max
FROM tap_in_out_log
GROUP BY tipe_pembayaran
ORDER BY total_transaksi DESC;
.output
```

Analisis:
1. Buka file `analisis_pembayaran.json` dengan text editor, apakah format valid JSON?
2. Tipe pembayaran mana yang paling sering digunakan?
3. Pembayaran mana yang memiliki saldo rata-rata tertinggi?

---

## Soal 6.5 - Export SUMMARIZE ke File
**Level:** Menengah**

Gunakan kombinasi `.mode markdown`, `SUMMARIZE`, dan `.output` untuk dokumentasi:
```
.mode markdown
.output profil_data_halte.md
SUMMARIZE halte_busway;
.output
```

Kemudian:
```
.mode markdown
.output profil_data_transaksi.md
SUMMARIZE tap_in_out_log;
.output
```

Tugas:
1. Buka kedua file `.md` yang dihasilkan
2. Salin dan paste ke dokumentasi proyek
3. Jelaskan kolom mana yang memiliki NULL value
4. Identifikasi data outlier berdasarkan min/max value

---

---

# BONUS: Challenge (Kombinasi Multiple Fitur)

## Challenge 1 - Profiling + Pivot + Export
**Level:** Lanjut**

1. Gunakan `SUMMARIZE` untuk profil tabel `tap_in_out_log`
2. Buat PIVOT untuk menampilkan count transaksi per `jenis_perjalanan` × `tipe_pembayaran`
3. Export hasil PIVOT ke CSV dan JSON

```sql
-- Langkah 1: Profiling
SUMMARIZE tap_in_out_log;

-- Langkah 2: Buat view untuk pivot
CREATE OR REPLACE VIEW pivot_analisis AS
SELECT t.jenis_perjalanan,
       t.tipe_pembayaran,
       COUNT(*) AS freq
FROM tap_in_out_log t
GROUP BY t.jenis_perjalanan, t.tipe_pembayaran;

-- Langkah 3: Pivot
PIVOT pivot_analisis
ON tipe_pembayaran
USING SUM(freq)
GROUP BY jenis_perjalanan;

-- Langkah 4: Export ke CSV
.mode csv
.output pivot_pembayaran.csv
PIVOT (SELECT * FROM pivot_analisis)
ON tipe_pembayaran
USING SUM(freq)
GROUP BY jenis_perjalanan;
.output
```

Pertanyaan analisis:
1. Subsidi untuk jenis perjalanan apa saja?
2. Pembayaran apa yang paling adaptif (diterima di berbagai jenis perjalanan)?

---

## Challenge 2 - LIST + PIVOT + File Query
**Level:** Lanjut**

1. Kumpulkan halte dengan `list()` per kota
2. Pivot perjalanan per kota
3. Query file CSV dari hasil sebelumnya

```sql
-- Langkah 1: Kumpulkan halte per kota
CREATE OR REPLACE VIEW halte_per_kota AS
SELECT kota_administrasi,
       list(nama_halte) AS halte_list,
       COUNT(*) AS jumlah
FROM halte_busway
GROUP BY kota_administrasi;

-- Langkah 2: Export halte list
COPY halte_per_kota TO 'halte_per_kota.csv' (HEADER, DELIMITER ',');

-- Langkah 3: Pivot transaksi per kota
CREATE OR REPLACE VIEW transaksi_kota AS
SELECT h.kota_administrasi,
       t.jenis_perjalanan,
       COUNT(t.id_transaksi) AS freq
FROM tap_in_out_log t
JOIN halte_busway h ON t.id_halte_asal = h.id_halte
GROUP BY h.kota_administrasi, t.jenis_perjalanan;

-- Langkah 4: Pivot
PIVOT transaksi_kota
ON jenis_perjalanan
USING SUM(freq)
GROUP BY kota_administrasi;

-- Langkah 5: Query file CSV yang sudah diekspor
SELECT * FROM 'halte_per_kota.csv'
WHERE kota_administrasi IN ('Jakarta Pusat', 'Jakarta Selatan');
```

Insight:
1. Kota mana yang memiliki halte paling banyak?
2. Kota mana dengan transaksi reguler paling tinggi?
3. Adakah kota dengan hanya transaksi subsidi (tanpa reguler)?

---

## Challenge 3 - EXCLUDE + REPLACE + STRUCT + Export
**Level:** Lanjut**

Kombinasikan EXCLUDE, REPLACE, STRUCT, dan export:

```sql
-- Buat view dengan EXCLUDE + REPLACE
CREATE OR REPLACE VIEW transaksi_summary AS
SELECT * EXCLUDE (id_transaksi, catatan_sistem)
       REPLACE (
           EXTRACT(HOUR FROM waktu_tap_in) AS jam_keberangkatan,
           ROUND(CAST(waktu_tap_out - waktu_tap_in AS INTERVAL) EXTRACT(EPOCH) / 60, 1) AS durasi_menit
       )
FROM tap_in_out_log;

-- Tambahkan STRUCT info pembayaran
SELECT *,
       {
           'tipe': tipe_pembayaran,
           'jenis': jenis_perjalanan,
           'tarif': tarif_debet
       } AS payment_info
FROM transaksi_summary
LIMIT 20;

-- Export hasil
.mode json
.output transaksi_dengan_struct.json
SELECT *,
       {
           'tipe': tipe_pembayaran,
           'jenis': jenis_perjalanan,
           'tarif': tarif_debet
       } AS payment_info
FROM transaksi_summary;
.output
```

Evaluasi:
1. Kolom mana yang sudah di-exclude?
2. Kolom baru apa yang ter-create oleh REPLACE?
3. Apakah JSON output valid dan readable?

---

---

# Kriteria Penilaian

| Aspek | Skor | Kriteria |
|---|---|---|
| **SUMMARIZE** | 10 | Memahami output profiling, mengidentifikasi NULL, ekstremitas |
| **Query File** | 10 | Export-import CSV/Parquet, JOIN file+DB, filter akurat |
| **PIVOT** | 15 | Pivot syntax benar, hasil akurat, interpretasi insight |
| **EXCLUDE/REPLACE** | 10 | Kolom yang di-exclude/replace benar, transformasi tepat |
| **LIST/STRUCT/Lambda** | 15 | Aggregasi list benar, STRUCT well-formed, lambda logic tepat |
| **CLI Commands** | 10 | Mode switching benar, timer tepat, export format valid |
| **Challenge** | 20 | Kombinasi fitur, insight mendalam, doc berkualitas |
| **Dokumentasi** | 10 | Laporan jelas, screenshoot, penjelasan lengkap |
| **Total** | **100** | |

---

## Deliverables

Setiap praktikan harus submit:
1. **File SQL** - Semua query yang dijalankan (dapat di-copy dari `.sql_history`)
2. **File CSV/JSON** - Hasil export dari `.output`
3. **File Markdown** - Dokumentasi dengan analisis (bisa dari `.mode markdown` output)
4. **Laporan Analisis** - Jawaban semua soal di atas

Nama file format: `NAMA_NIM_duckdb_praktikum.zip` berisi:
```
├── queries.sql
├── laporan.md
├── hasil_export/
│   ├── halte_busway.csv
│   ├── rekap_transaksi.parquet
│   ├── laporan_halte_fasilitas.csv
│   ├── analisis_pembayaran.json
│   └── profil_data_transaksi.md
└── screenshots/
    ├── challenge_1_output.png
    ├── challenge_2_pivot.png
    └── challenge_3_json.png
```

---

## Referensi

- [DuckDB Official Docs](https://duckdb.org/docs/)
- [DuckDB SUMMARIZE](https://duckdb.org/docs/guides/data_profiling/summarize)
- [DuckDB File Formats](https://duckdb.org/docs/data/overview)
- [DuckDB PIVOT](https://duckdb.org/docs/sql/statements/pivot)
- [DuckDB Complex Types](https://duckdb.org/docs/sql/data_types/nested)
- [DuckDB CLI](https://duckdb.org/docs/api/cli/overview)

---

**Selamat Mengerjakan! 🚌📊**
