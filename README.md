# Analisis dan Peramalan PDB Industri Makanan dan Minuman Indonesia (2010 Q1 – 2026 Q3)

Daffa Muhammad Khrisadi (`24083010062`) — Analisis Deret Waktu A

Repositori ini berisi analisis dan pemodelan deret waktu pada data **Produk Domestik Bruto (PDB) Seri 2010 Atas Dasar Harga Konstan (ADHK 2010) Subsektor Industri Makanan dan Minuman** di Indonesia.

---

## 1. Latar Belakang

Subsektor Industri Makanan dan Minuman merupakan kontributor terbesar pada PDB Industri Pengolahan Non-Migas di Indonesia dengan rata-rata pangsa sekitar **34,04%** (mencapai **39,08%** pada periode terbaru). Peramalan kinerja subsektor ini penting untuk memantau pertumbuhan riil manufaktur dan mendukung perencanaan kebijakan industri.

Dari sisi deret waktu, data ini memiliki tiga karakteristik utama:
- **Tidak stasioner:** Nilai produksi riil terus tumbuh lebih dari tiga kali lipat dari **Rp85.184,0 Miliar** (`2010-Q1`) menjadi **Rp261.225,3 Miliar** (`2026-Q3`), disertai fluktuasi kuartalan yang makin melebar.
- **Musiman kuartalan:** Produksi cenderung meningkat pada pertengahan tahun (Q2–Q3) dan melandai pada akhir maupun awal tahun (Q4–Q1).
- **Efek pergeseran hari raya:** Periode Ramadan dan Idul Fitri maju sekitar 11 hari setiap tahun (bergeser dari Q3 di `2010–2013`, ke Q2 di `2017–2022`, hingga masuk ke Q1 pada `2025–2026`) serta adanya perubahan kemiringan tren pasca-pandemi COVID-19.

---

## 2. Dataset

Data bersumber dari **Badan Pusat Statistik (BPS)** melalui **Pusat Data dan Informasi (Pusdatin) Kementerian Perindustrian**, tersimpan pada [`data/pdb_industri_makanan_minuman_bps.csv`](data/pdb_industri_makanan_minuman_bps.csv). Dataset berfrekuensi **triwulanan** dari **`2010-Q1` hingga `2026-Q3`** (**67 observasi**, tanpa *missing value* maupun duplikasi).

| Kolom | Deskripsi Singkat |
| :--- | :--- |
| `periode`, `tanggal`, `tahun`, `triwulan` | Penanda waktu kuartalan (`2010-Q1` s.d. `2026-Q3`). |
| `pdb_mamin_adhk_miliar_rp` | **Variabel Target:** PDB Industri Makanan & Minuman Harga Konstan 2010 (Miliar Rupiah). |
| `pdb_mamin_adhb_miliar_rp` | PDB Industri Makanan & Minuman Harga Berlaku (Miliar Rupiah) sebagai pembanding nominal. |
| `pdb_industri_nonmigas_adhk_miliar_rp` | Total PDB Industri Pengolahan Non-Migas ADHK 2010 (Miliar Rupiah). |
| `hari_ramadan_idul_fitri` | Jumlah hari periode Ramadan dan Idul Fitri yang jatuh pada triwulan terkait. |

---

## 3. Metode

Alur analisis disusun secara bertahap tanpa kebocoran data masa depan (*no data leakage*):

1. **Pembagian Data Kronologis:** Data dibagi berurutan menjadi **Training Set** (`2010-Q1`–`2022-Q3`, 51 observasi), **Validation Set** (`2022-Q4`–`2024-Q3`, 8 observasi), dan **Test Set** (`2024-Q4`–`2026-Q3`, 8 observasi).
2. **Eksplorasi Data (EDA):** Meliputi pemeriksaan kualitas data, pemisahan komponen tren dan musiman dengan dekomposisi **STL**, analisis pertumbuhan antar-triwulan (*q-to-q*), pemetaan pergeseran hari raya, serta deteksi *outlier* menggunakan *Robust Z-Score*.
3. **Uji Stasioneritas:** Dilakukan pada *Training Set* melalui visualisasi *Rolling Mean* dan *Rolling Standard Deviation*, evaluasi stabilitas varians dengan **Box-Cox**, plot **ACF & PACF**, serta uji statistik **ADF** dan **KPSS**.
4. **Model Baseline Awal:** Membangun tiga model acuan sederhana pada *Training Set* untuk dievaluasi di *Validation Set*, yaitu **Naive (Last Value)**, **Seasonal Naive**, dan **Regresi Log-Linear Trend + Seasonal Dummy (OLS)**.
5. **Model Utama & Pembanding (Tahap Lanjutan):** Mengembangkan **SARIMA / RegARIMA** (dengan proses *differencing* dan variabel kalender hari raya, disertai uji diagnostik residual *Ljung-Box*, normalitas, dan *ARCH-LM*) untuk dibandingkan terhadap **Support Vector Regression (`SVR-RBF`)** berbasis fitur *lag* musiman.

---

## 4. Evaluasi Model (Sesuai Hasil Notebook)

Performa peramalan diukur dalam satuan asli (Miliar Rupiah) menggunakan **RMSE** dan **MAE** (rata-rata besar kesalahan nominal), serta **MAPE** dan **sMAPE** (rata-rata persentase kesalahan relatif).

### Hasil Uji Stasioneritas (*Training Set*)
Estimasi Box-Cox menghasilkan nilai optimal `lambda = 0,6817`. Uji akar unit pada data level menunjukkan bahwa baik data asli maupun data tertransformasi logaritma **belum stasioner** (`ADF p-value > 0,05` dan `KPSS p-value < 0,05`), sehingga diperlukan proses *differencing* pada pemodelan utama:

| Deret Uji | ADF *p-value* (`c`) | KPSS *p-value* (`c`) | Kesimpulan |
| :--- | ---: | ---: | :---: |
| **Data Asli (Miliar Rupiah)** | 0,9058 | 0,0100 | **Tidak Stasioner** |
| **Transformasi Logaritma** | 0,3564 | 0,0100 | **Tidak Stasioner** |

### Hasil Evaluasi Baseline Awal (*Validation Set*: `2022-Q4` – `2024-Q3`)

| Model Baseline Awal | RMSE (Miliar Rp) | MAE (Miliar Rp) | MAPE (%) | sMAPE (%) |
| :--- | ---: | ---: | ---: | ---: |
| **Baseline 1 — Naive (Last Value)** | 9.407,24 | 7.475,00 | 3,387% | 3,474% |
| **Baseline 2 — Seasonal Naive** | 17.789,37 | 16.540,02 | 7,627% | 7,977% |
| **Baseline 3 — Log-Linear + Seasonal OLS** | 23.407,15 | 22.724,00 | 10,481% | 9,935% |

- **Naive (Last Value)** menghasilkan galat paling kecil di antara ketiganya semata-mata karena titik akhir data latih (`2022-Q3`) berada di puncak musiman, namun prediksinya berupa garis datar yang tidak mampu mengikuti pola kuartalan maupun kenaikan tren berikutnya.
- **Seasonal Naive** selalu memprediksi terlalu rendah (*under-predict*) karena hanya mengulang angka tahun sebelumnya tanpa memperhitungkan pertumbuhan tahunan.
- **Log-Linear + Seasonal OLS** memprediksi terlalu tinggi (*over-predict*) karena menarik satu garis tren konstan sejak 2010 tanpa beradaptasi terhadap pelandaian tren pasca-pandemi 2020.

Hasil ketiga *baseline* ini menjadi batas acuan awal yang akan diperbaiki oleh model **RegARIMA/SARIMA** dan **SVR-RBF** pada tahap selanjutnya.

---

## 5. Struktur Folder & Cara Menjalankan

```text
sarima-pdb-mamin-indonesia/
├── data/
│   └── pdb_industri_makanan_minuman_bps.csv
├── notebook/
│   └── Analisis_Deret_Waktu_PDB_Mamin.ipynb
├── requirements.txt
└── README.md
```

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter lab notebook/Analisis_Deret_Waktu_PDB_Mamin.ipynb
```
