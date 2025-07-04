# CBL TIBD Kel. 1  

## A. Persiapan Lingkungan HDFS  
Langkah awal adalah menyiapkan Hadoop Distributed File System (HDFS) agar siap untuk pengolahan data.  

Pertama, keluar dahulu dari *safe mode* (read-only) agar kita bisa melakukan aksi menulis (write) atau menghapus (delete) file di HDFS:  
```
$ hadoop dfsadmin -safemode leave
```  

Selanjutnya membuat direktori dalam HDFS tempat peyimpanan input. Disini kita memilih `/user/kelompok1/input`:  
```
$ hadoop fs -mkdir -p /user/kelompok1/input
```  

Mengunggah dataset `vgsales.csv` ke dalam direktori input HDFS yang sudah dibuat:  
```
$ hadoop fs -put vgsales.csv /user/kelompok1/input
```  

Verifikasi keberadaan file dataset dalam direktori `/user/kelompok1/input` untuk memastikan proses unggah berhasil:  
```
$ hadoop fs -ls /user/kelompok1/input
```
> Output seharusnya menampilkan file `vgsales.csv`  
  

## Implementasi Hive untuk Analisis Data  
Analisis data terstruktur dilakukan menggunakan Apache Hive.  

Jalankan **WordCount** bawaan Hadoop:  
```
$ hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount /user/kelompok1/input /user/kelompok1/output/wordcount
```  

Cek hasil output di direktori `/user/kelompok1/output/wordcount`:  
```
$ hadoop fs -ls /user/kelompok1/output/wordcount
```  

Salin hasil ke *Shared Folder* supaya bisa dibuka di Windows:  
```
$ hadoop fs -get /user/kelompok1/output/wordcount/part-r-00000 /media/sf_kelompok1/output/wordcount_result.txt
```  

Buat skrip **Hive** `hive_query.hql` yang berisi perintah *HQL* untuk membuat struktur data, memuat data, dan menjalankan query agregasi:  
```
$ hadoop fs -touch /media/sf_kelompok/script/hive_query.hql
```  

Buka file dengan text editor **nano**:  
```
$ nano /media/sf_kelompok/script/hive_query.hql
```  

Masukan skrip HQL seperti di bawah:  
```sql
-- Buat database
CREATE DATABASE IF NOT EXISTS kelompok1_db;
USE kelompok1_db;

-- Buat tabel
CREATE TABLE IF NOT EXISTS vgsales (
    Rank INT,
    Name STRING,
    Platform STRING,
    Year INT,
    Platform STRING,
    Year INT,
    Genre STRING,
    Publisher STRING,
    NA_Sales FLOAT,
    EU_Sales FLOAT,
    JP_Sales FLOAT,
    Other_Sales FLOAT,
    Global_Sales FLOAT
)

ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

-- Load data dari HDFS ke tabel Hive
LOAD DATA INPATH '/user/kelompok1/input/vgsales.csv' OVERWRITE INTO TABLE vgsales;

-- Query 1: Total Global Sales per Genre
INSERT OVERWRITE DIRECTORY '/user/kelompok1/output/hive_genre'
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
SELECT Genre, SUM(Global_Sales) FROM vgsales GROUP BY Genre;

-- Query 2: Total Global Sales per Platform
INSERT OVERWRITE DIRECTORY '/user/kelompok1/output/hive_platform'
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
SELECT Platform, SUM(Global_Sales) FROM vgsales GROUP BY Platform;
```  

Eksekusi skrip **Hive** `hive_query.hql`:  
```
$ hive -f /media/sf_kelompok1/scripts/hive_query.hql
```  

Mengambil hasi query dengan disalin ke *shared folder* untuk diolah lebih lanjut:  
```
$ hadoop fs -get /user/kelompok1/output/hive_genre/000000_0 /media/sf_kelompok1/output/hive_genre.txt
```  
  

## Implementasi dan Perbandingan Kinerja WordCount  
Eksperimen ini membandingkan kinerja MapReduce dengan konfigurasi default dan yang di-tuning.  

Menjalankan WordCount Default: Job dijalankan dengan 1 reducer dan waktunya dicatat:  
```
$ time hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount /user/kelompok1/input /user/kelompok1/output/wordcount_default
```  

> Hasil WordCount default:  
> ![wordcount result default](/resource/wordcount_result_default.png)  

Hapus output default yang pertama (Jika belum pernah dibuat, tidak masalah jika terjadi error):  
```
$ hadoop fs -rm /user/kelompok1/output/wordcount_tuned
```  

Kemudian menjalankan **WordCount** lagi, tetapi kali ini dilakukan *tuning*, job dilakukan dengan 4 *reducer*:  
```
$ time hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount -D mapreduce.job reduces=4 /user/kelompok1/input /user/kelompok1/output/wordcount_default
```  

> Hasil WordCount Tuned:  
> ![wordcount result tuned](/resource/wordcount_result_tuned.png)  

Salin hasil dari kedua job ke *shared folder*:  
```
$ hadoop fs -get /user/kelompok1/output/wordcount_tuned/part-r-00000 /media/sf_kelompok1/output/wordcount_tuned.txt
```  

## Visualisasi Platform  
```python
from google.colab import drive
drive.mount('/content/drive')

import pandas as pd
import matplotlib.pyplot as plt
import numpy as np

# File path di Google Drive
platform_path = '/content/drive/MyDrive/dataset/hive_platform.csv'
genre_path = '/content/drive/MyDrive/dataset/hive_genre.csv'

# Baca file CSV dengan delimiter tab
platform_df = pd.read_csv(platform_path, delimiter='\t', header=None)
genre_df = pd.read_csv(genre_path, delimiter='\t', header=None)

# Tambahkan nama kolom
platform_df.columns = ['Platform', 'Jumlah']
genre_df.columns = ['Genre', 'Jumlah']

# Bersihkan teks dan ubah tipe data
platform_df['Platform'] = platform_df['Platform'].str.replace('"', '').str.strip()
genre_df['Genre'] = genre_df['Genre'].str.replace('"', '').str.strip()
platform_df['Jumlah'] = pd.to_numeric(platform_df['Jumlah'], errors='coerce')
genre_df['Jumlah'] = pd.to_numeric(genre_df['Jumlah'], errors='coerce')

# Platform valid
valid_platforms = [
    '2600', '3DO', '3DS', 'DC', 'DS', 'GB', 'GBA', 'GC', 'GEN', 'GG',
    'N64', 'NES', 'NG', 'PC', 'PCFX', 'PS', 'PS2', 'PS3', 'PS4', 'PSP',
    'PSV', 'SAT', 'SCD', 'SNES', 'TG16', 'WS', 'Wii', 'WiiU', 'X360',
    'XB', 'XOne'
]

# Filter platform dan genre
platform_valid = platform_df[platform_df['Platform'].isin(valid_platforms)]
platform_sorted = platform_valid.sort_values(by='Jumlah', ascending=False).head(15)

genre_df = genre_df.dropna(subset=['Genre'])
genre_df = genre_df[genre_df['Genre'].apply(lambda x: isinstance(x, str))]
genre_sorted = genre_df[genre_df['Jumlah'] > 0].sort_values(by='Jumlah', ascending=False).head(15)

plt.figure(figsize=(14, 7))
bars = plt.bar(platform_sorted['Platform'], platform_sorted['Jumlah'], color='steelblue')
plt.title('Top 15 Platform dengan Jumlah Game Terbanyak', fontsize=16)
plt.xlabel('Platform', fontsize=14)
plt.ylabel('Jumlah Game', fontsize=14)
plt.xticks(fontsize=12)
plt.yticks(fontsize=12)
plt.grid(axis='y', linestyle='--', alpha=0.7)

# Label di atas bar
for bar in bars:
    plt.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 10,
             f'{bar.get_height():.0f}', ha='center', fontsize=10)

plt.tight_layout()
plt.show()
```  
> Top 15 Platform dengan Jumlah Game Terbanyak  
> ![top 15 platform](/resource/top_15_platform.png)  

```python
plt.figure(figsize=(12, 8))
bars = plt.barh(genre_sorted['Genre'], genre_sorted['Jumlah'], color='mediumseagreen')
plt.title('Top 15 Genre Game Terpopuler', fontsize=16)
plt.xlabel('Jumlah Game', fontsize=14)
plt.ylabel('Genre', fontsize=14)
plt.xticks(fontsize=12)
plt.yticks(fontsize=12)
plt.grid(axis='x', linestyle='--', alpha=0.7)

# Label di ujung bar
for bar in bars:
    plt.text(bar.get_width() + 10, bar.get_y() + bar.get_height()/2,
             f'{bar.get_width():.0f}', va='center', fontsize=10)

plt.tight_layout()
plt.show()
```  
> Top 15 Genre Game Terpopuler  
> ![top 15 genre](/resource/top_15_genre.png)  

```python
top_genres_pie = genre_sorted.head(10)
colors = plt.cm.Set3(np.linspace(0, 1, 10))

plt.figure(figsize=(10, 10))
plt.pie(top_genres_pie['Jumlah'], labels=top_genres_pie['Genre'],
        autopct='%1.1f%%', startangle=140, colors=colors, textprops={'fontsize': 12})
plt.title('Distribusi 10 Genre Terpopuler', fontsize=16)
plt.axis('equal')
plt.show()
```  
> Distribusi 10 Genre Terpopuler  
> ![Distribusi 10 Genre](/resource/distribusi_10_genre.png)  

```python
plt.figure(figsize=(14, 6))
plt.plot(platform_sorted['Platform'], platform_sorted['Jumlah'], marker='o', color='darkorange')
plt.title('Jumlah Game per Platform (Top 15)', fontsize=16)
plt.xlabel('Platform', fontsize=14)
plt.ylabel('Jumlah Game', fontsize=14)
plt.xticks(rotation=45, fontsize=12)
plt.yticks(fontsize=12)
plt.grid(True, linestyle='--', alpha=0.7)
plt.tight_layout()
plt.show()
```  
> Jumlah Game per Platform (Top 15)  
> ![game per platform](/resource/game_per_platform.png)  
