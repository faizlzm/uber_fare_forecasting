# Uber Fare Forecasting
Pembersihan 200.000 sampel perjalanan Uber di New York City (2009–2015), agregasi menjadi total fare harian, lalu peramalan 30 hari ke depan dengan membandingkan lima model time series (Naive, Seasonal Naive, ARIMA, SARIMA, Prophet). Hasilnya disajikan dalam dashboard Power BI.

## Daftar Isi

- [Latar Belakang](#latar-belakang)
- [Problem Statement](#problem-statement)
- [Struktur Repositori](#struktur-repositori)
- [Dataset](#dataset)
- [Metodologi](#metodologi)
- [Temuan Utama](#temuan-utama)
- [Hasil Forecasting](#hasil-forecasting)
- [Rekomendasi](#rekomendasi)
- [Keterbatasan](#keterbatasan)
- [Cara Menjalankan](#cara-menjalankan)
- [Penulis](#penulis)

---

## Latar Belakang

Permintaan pada layanan ride-hailing berfluktuasi menurut waktu: ada tren yang berubah dan pola musiman mingguan, sehingga kebutuhan operasional sulit diperkirakan. Ketidakpastian yang sama berlaku untuk total nilai tarif (*total fare*) harian. Peramalan berbasis data historis dapat menjadi acuan perencanaan yang lebih terukur.

## Problem Statement

> Bagaimana memperkirakan total fare harian Uber untuk 30 hari ke depan, dan model time series mana yang paling akurat untuk tujuan tersebut?

## Struktur Repositori

```text
uber_fare_forecasting/
├── analysis_process/
│   └── uber_fare_forecasting.ipynb                       # Cleaning, EDA, dan forecasting
├── data/
│   └── uber.csv                                          # Dataset mentah (200.000 baris, 9 kolom)
├── dashboard/
│   └── dashboard_final_project.pbix                      # Dashboard Power BI
├── report/
│   └── ahmad_faiz_ali_azmi_final_project_dibimbing.pdf   # Laporan presentasi (28 slide)
└── README.md
```

## Dataset

Uber Fares Dataset (Kaggle): sampel perjalanan Uber di New York City.

- **Jumlah baris:** 200.000 perjalanan
- **Jumlah kolom:** 9
- **Periode penjemputan:** 1 Januari 2009 – 30 Juni 2015

| Kolom | Keterangan |
| --- | --- |
| `Unnamed: 0` | Indeks baris sumber (diganti nama menjadi `order_id`) |
| `key` | Identifier berbasis timestamp (dibuang karena redundan) |
| `fare_amount` | Tarif perjalanan dalam USD |
| `pickup_datetime` | Waktu penjemputan (UTC) |
| `pickup_longitude`, `pickup_latitude` | Koordinat penjemputan |
| `dropoff_longitude`, `dropoff_latitude` | Koordinat tujuan |
| `passenger_count` | Jumlah penumpang |

## Metodologi

### 1. Data Cleaning

| Masalah | Penanganan |
| --- | --- |
| `pickup_datetime` bertipe teks | Dikonversi ke datetime (UTC), lalu ke waktu lokal New York |
| Duplikasi | 0 baris duplikat; kolom `key` dibuang karena redundan |
| Missing value | 1 baris (koordinat tujuan kosong); ikut tereliminasi pada filter wilayah |
| Koordinat tertukar (lintang dan bujur terbalik) | 102 baris **diperbaiki** dengan menukar kembali |
| Koordinat `0.0` atau di luar wilayah New York | Baris di luar *bounding box* (lat 40,4–41,1; lon −74,3 hingga −73,6) dibuang: **4.255 baris** |
| Anomali tarif, jarak, dan penumpang | Dibuang: tarif ≤ $0 atau ≥ $499, jarak 0 km, penumpang di luar 0–6 orang: **1.990 baris** |

Hasil: **200.000 → 195.745 → 193.755 baris** bersih.

### 2. Feature Engineering

`distance_km` (formula Haversine), komponen waktu lokal New York (hari, bulan, kuartal, tahun, jam, AM/PM), dan `pickup_area` serta `dropoff_area` (berdasarkan perkiraan *bounding box* wilayah), lalu *one-hot encoding* untuk korelasi.

### 3. Agregasi Harian

Perjalanan diagregasi per **tanggal lokal New York** menjadi `total_fare`, `total_trips`, dan `avg_fare`: **2.373 hari** (31 Desember 2008 – 30 Juni 2015), tanpa hari kosong.

### 4. Analisis dan Pemodelan Time Series

STL decomposition (periode 7 hari), uji stasioneritas ADF, ACF dan PACF, lalu lima model dibandingkan pada satu *holdout* kronologis:

| Pembagian data | Jumlah | Periode |
| --- | ---: | --- |
| Training | 2.343 hari (98,74%) | s.d. 31 Mei 2015 |
| Testing | 30 hari (1,26%) | 1–30 Juni 2015 |

Metrik evaluasi: MAE, RMSE, dan WAPE.

## Temuan Utama

> Angka dicocokkan ulang dengan menjalankan ulang pipeline dari `data/uber.csv`. Hasil Prophet diambil dari notebook karena tidak dijalankan ulang.

### 1. Jarak berkorelasi kuat dengan tarif

Korelasi Pearson `distance_km` dengan `fare_amount` sebesar **0,88**. Sebagian besar perjalanan berjarak dan bertarif rendah (distribusi miring ke kanan; rata-rata tarif $11,32, median $8,50). Korelasi ini belum membuktikan bahwa jarak satu-satunya penentu tarif.

### 2. Total fare harian cukup volatil

- Rata-rata total fare harian **$924,87** (tanpa hari pertama yang parsial), dengan *coefficient of variation* 24,84%.
- Rata-rata perubahan absolut harian 22,14%; **43,3%** perubahan harian melebihi 20%.
- Total fare berkorelasi dengan jumlah perjalanan (r = 0,784) dan dengan rata-rata tarif (r = 0,532).

### 3. Pola mingguan konsisten: Jumat tertinggi, Minggu terendah

| Hari | Rata-rata total fare |
| --- | ---: |
| Jumat | $1.077,28 |
| Kamis | $1.013,09 |
| Sabtu | $968,77 |
| Rabu | $951,42 |
| Selasa | $905,66 |
| Senin | $817,49 |
| Minggu | $740,47 |

Jumat lebih tinggi sekitar **45,5%** dari Minggu, dan pola ini muncul pada setiap tahun 2009–2015 (selisih 38,2%–53,9%). Secara bulanan, Oktober tertinggi ($992,29) dan Agustus terendah ($823,49).

### 4. Level naik sekitar akhir 2012, didorong tarif rata-rata

Rata-rata total fare harian naik dari sekitar $813 (2009) menjadi $1.055 (2013), lalu turun ke $960 (2015, Januari–Juni). Kenaikan ini datang dari **rata-rata tarif per perjalanan** (sekitar $10,0 pada 2009 menjadi $13,0 pada 2014), sedangkan jumlah perjalanan per hari di sampel relatif datar (sekitar 80–85).

### 5. Struktur time series

- **ADF:** statistik −3,6213, p = 0,0054 (mendukung stasioneritas, meski level bergeser sekitar 2012).
- **ACF:** autokorelasi positif pada lag 1 (0,4643) dan lag 7 (0,4619), berulang pada kelipatan 7 hari, sehingga ada seasonality mingguan.
- **STL:** Trend Strength 0,3989 dan Seasonal Strength 0,3714; sebagian besar fluktuasi harian belum dijelaskan (simpangan baku residual sekitar $153).

## Hasil Forecasting

### Perbandingan model (holdout 1–30 Juni 2015)

| Model | MAE | RMSE | WAPE |
| --- | ---: | ---: | ---: |
| Naive | 153,48 | 188,68 | 16,17% |
| Seasonal Naive | 197,31 | 232,51 | 20,78% |
| ARIMA(2,1,1) | 132,30 | 162,38 | 13,94% |
| Prophet | 106,32 | 124,80 | 11,20% |
| **SARIMA(1,0,1)(0,1,1,7)** | **102,65** | **122,89** | **10,81%** |

SARIMA memiliki error terendah pada ketiga metrik dan menurunkan MAE sekitar 33% dibandingkan Naive. ARIMA tanpa komponen musiman cenderung menghasilkan prediksi mendatar.

### Forecast 1–30 Juli 2015

Setelah dilatih ulang pada seluruh 2.373 hari, SARIMA memperkirakan:

- **Total fare 30 hari: sekitar $27.629** (rata-rata sekitar $921 per hari), dengan interval prediksi 95% per hari di notebook.
- Dibandingkan 30 hari aktual terakhir ($28.481,85), ini berarti **penurunan sekitar 3,0%**.

## Rekomendasi

1. **Rencanakan kapasitas dan promosi menurut hari.** Siapkan layanan lebih pada Jumat, dan evaluasi promosi pada Minggu lewat uji coba terukur.
2. **Pantau harian** jumlah perjalanan, rata-rata tarif, dan total fare. Saat terjadi perubahan besar, bandingkan dengan hari yang sama pada minggu sebelumnya.
3. **Pakai forecast 30 hari sebagai acuan awal, bukan target pasti,** dan evaluasi akurasinya secara berkala sambil mencoba model lain.

## Penulis

**Ahmad Faiz Ali Azmi**
Lulusan Teknik Informatika, Universitas Brawijaya. Peserta Offline Bootcamp Data Analyst Batch 4 di dibimbing.id.

GitHub: [@faizlzm](https://github.com/faizlzm)
