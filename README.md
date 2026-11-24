# Peramalan PDB Triwulanan Industri Makanan dan Minuman Indonesia Menggunakan Metode SARIMA dan Support Vector Regression (SVR)

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
5. **Model Statistik Utama — SARIMA & RegARIMA (Progress 3):** Melakukan transformasi logaritma dan uji diferensiasi ($d, D, s=4$) pada *Training Set*, identifikasi kandidat orde via ACF/PACF, estimasi serta seleksi model **SARIMA** dan **RegARIMA** (eksogen `hari_ramadan_idul_fitri`) berdasarkan AIC/AICc/BIC per rezim, validasi diagnostik residual (*Ljung-Box*, *Shapiro-Wilk*, *Jarque-Bera*, dan *ARCH-LM*), serta evaluasi 8 langkah pada *Validation Set*.
6. **Model Pembanding & Evaluasi Akhir (Tahap Lanjutan):** Membangun **Support Vector Regression (`SVR-RBF`)** berbasis fitur *lag* musiman dan kalender pada *Validation Set* (Progress 4), lalu melatih ulang kedua model final pada gabungan `Train + Validation` untuk perbandingan akhir pada *Test Set* (`2024-Q4`–`2026-Q3`) yang hingga saat ini tetap dikunci (*no data leakage*).

---

## 4. Evaluasi Model (Sesuai Hasil Notebook)

Performa peramalan diukur dalam satuan asli (Miliar Rupiah) menggunakan **RMSE** dan **MAE** (rata-rata besar kesalahan nominal), **MAPE** dan **sMAPE** (rata-rata persentase kesalahan relatif), serta **MASE** (rasio MAE terhadap kesalahan *in-sample Seasonal Naive* $s=4$ pada *Training Set* sebesar `9.277,56` miliar rupiah).

### Hasil Uji Stasioneritas (*Training Set*, $N = 51$)
Estimasi Box-Cox menghasilkan nilai optimal `lambda = 0,6817`. Data level asli maupun logaritma **belum stasioner** (`ADF p > 0,05`, `KPSS p = 0,01`). Setelah diferensiasi ganda pada skala logaritma ($\Delta\Delta_4 \ln Y_t$), kedua uji menyatakan deret telah **stasioner penuh**:

| Deret Uji (*Training Set*) | ADF *Stat* (`c`) | ADF *p-value* (`c`) | KPSS *p-value* (`c`) | Kesimpulan ($\alpha = 5\%$) |
| :--- | ---: | ---: | ---: | :---: |
| **Data Asli $Y_t$ (Level)** | -0,4254 | 0,9058 | 0,0100 | Tidak Stasioner |
| **Transformasi Logaritma $\ln(Y_t)$** | -1,8490 | 0,3564 | 0,0100 | Tidak Stasioner |
| **Diferensiasi Non-Musiman $\Delta\ln(Y_t)$** | -1,1178 | 0,7079 | 0,1000 | Parsial (KPSS lolos) |
| **Diferensiasi Musiman $\Delta_4\ln(Y_t)$** | -1,1546 | 0,6928 | 0,0231 | Tidak Stasioner |
| **Diferensiasi Ganda $\Delta\Delta_4\ln(Y_t)$** | **-6,3315** | **0,0000** | **0,1000** | **Stasioner** |

### Hasil Evaluasi Baseline & SARIMA/RegARIMA (*Validation Set*: `2022-Q4` – `2024-Q3`)

| Model | Diagnostik | RMSE (Miliar Rp) | MAE (Miliar Rp) | MAPE (%) | sMAPE (%) | MASE |
| :--- | :---: | ---: | ---: | ---: | ---: | ---: |
| **`SARIMA(0,1,0)(1,0,0)_4 + drift` (Log) — Terpilih** | **Lolos** | **3.189,90** | **2.177,65** | **1,023%** | **1,033%** | **0,235** |
| `SARIMA(0,1,0)(2,0,0)_4 + drift` (Log) | Lolos | 4.004,44 | 3.251,33 | 1,511% | 1,528% | 0,350 |
| `RegARIMA(0,1,0)(1,0,0)_4 + drift + Ramadan` (Log) | Lolos | 4.044,19 | 3.030,57 | 1,416% | 1,433% | 0,327 |
| `SARIMA(1,0,0)(1,1,0)_4 + drift` (Log) | Lolos | 4.475,78 | 3.763,61 | 1,735% | 1,735% | 0,406 |
| `SARIMA(0,1,0)(0,1,1)_4` (Log) | Tidak Lolos | 6.448,36 | 5.542,14 | 2,568% | 2,613% | 0,597 |
| Baseline 1 — Naive (Last Value) | Baseline | 9.407,24 | 7.475,00 | 3,387% | 3,474% | 0,806 |
| Baseline 2 — Seasonal Naive ($s = 4$) | Baseline | 17.789,37 | 16.540,02 | 7,627% | 7,977% | 1,783 |
| Baseline 3 — Log-Linear + Seasonal OLS | Baseline | 23.407,15 | 22.724,00 | 10,481% | 9,935% | 2,449 |

- **`SARIMA(0,1,0)(1,0,0)_4 + drift` pada skala logaritma** menjadi model statistik terbaik pada tahap validasi: memangkas RMSE sebesar **66,1%** dibanding *baseline* Naive (`9.407,24`) dan **82,1%** dibanding Seasonal Naive (`17.789,37`), dengan `MASE = 0,235`, cakupan interval prediksi 95% sempurna (**8/8**), serta lolos penuh uji diagnostik residual (*Ljung-Box* lag 4/8/12 $p \ge 0,1037$; *Shapiro-Wilk* $p = 0,2128$; *Jarque-Bera* $p = 0,2185$; *ARCH-LM* $p = 0,6727$).
- **Uji RegARIMA (`hari_ramadan_idul_fitri`):** Koefisien linier hari Ramadan tidak signifikan secara statistik pada *Training Set* ($p = 0,4858$–$0,9979$) dan tidak melampaui akurasi SARIMA murni pada validasi karena pergeseran Ramadan ke Q1 baru dimulai sejak `2023-Q1`.
- **Tahap Berikutnya:** Membangun model pembanding **SVR-RBF** pada *Validation Set* yang sama (Progress 4) sebelum membuka *Test Set* (`2024-Q4`–`2026-Q3`) untuk perbandingan final.

---

## 5. Struktur Folder & Cara Menjalankan

```text
Time-Series_ADW-A/
├── Poster/
│   └── 24083010062_Daffa Muhammad Khrisadi_Progress2.pdf
├── data/
│   └── pdb_industri_makanan_minuman_bps.csv
├── notebook/
│   ├── Analisis_Deret_Waktu_PDB_Mamin.ipynb          # Progress 1: EDA, Stasioneritas Awal, & Baseline
│   └── Progress3_Pemodelan_SARIMA_RegARIMA.ipynb     # Progress 3: Diferensiasi, SARIMA/RegARIMA, Diagnostik, & Validasi
├── requirements.txt
└── README.md
```

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter lab notebook/Progress3_Pemodelan_SARIMA_RegARIMA.ipynb
```
