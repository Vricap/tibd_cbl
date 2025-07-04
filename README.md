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
