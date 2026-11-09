# Analisis dan Peramalan Deret Waktu PDB Industri Makanan dan Minuman Indonesia (2010 Q1 – 2026 Q3)

**Penulis:** Daffa Muhammad Khrisadi (`24083010062`) — Analisis Deret Waktu A

Repositori ini mendokumentasikan penelitian analisis dan peramalan deret waktu (_time series forecasting_) pada **Produk Domestik Bruto (PDB) Seri 2010 Atas Dasar Harga Konstan (ADHK 2010) Subsektor Industri Makanan dan Minuman** di Indonesia. Dokumen ini menguraikan latar belakang permasalahan, spesifikasi dataset, kerangka metodologi secara menyeluruh (mulai dari eksplorasi data, uji ketidakstasioneran, pemodelan _baseline_, hingga spesifikasi model utama **RegARIMA/SARIMA** dan metode pembanding **SVR-RBF**), serta hasil evaluasi kuantitatif yang selaras dengan tahapan pada _notebook_.

---

## 1. Latar Belakang

Sektor industri pengolahan non-migas merupakan motor utama pertumbuhan ekonomi Indonesia, dan di dalamnya **Subsektor Industri Makanan dan Minuman** memegang peranan paling dominan. Sepanjang periode `2010-Q1` hingga `2026-Q3`, subsektor ini menyumbang rata-rata **34,04%** terhadap total PDB Industri Pengolahan Non-Migas nasional — bahkan meningkat hingga mencapai **39,08%** pada periode terkini. Mengingat besarnya kontribusi tersebut, penyediaan proyeksi kuartalan yang akurat menjadi instrumen krusial dalam pemantauan kinerja sektor industri, perencanaan kapasitas produksi, serta perumusan kebijakan ketahanan pangan dan hilirisasi industri.

Secara statistik dan ekonometrika, peramalan output riil Industri Makanan dan Minuman menghadirkan sejumlah tantangan struktural yang tidak dapat diselesaikan oleh model ekstrapolasi sederhana:

1. **Ketidakstasioneran pada Rata-Rata dan Varians:** Nilai tambah riil (ADHK 2010) meningkat lebih dari tiga kali lipat dari **Rp85.184,00 Miliar** pada `2010-Q1` menjadi **Rp261.225,30 Miliar** pada `2026-Q3`. Kenaikan skala produksi ini diikuti oleh pelebaran simpangan fluktuasi kuartalan dari tahun ke tahun.
2. **Perubahan Kemiringan Tren Pasca-Pandemi (_Structural Regime Shift_):** Setelah tumbuh dengan laju eksponensial yang relatif stabil pada dekade `2010–2019`, guncangan pandemi COVID-19 pada `2020–2021` menggeser level dan kemiringan lintasan pertumbuhan jangka menengah, sehingga model tren deterministik tunggal mengalami bias proyeksi yang besar.
3. **Siklus Musiman Kuartalan ($s = 4$) dan Pergeseran Kalender Hari Raya (_Moving Holiday_):** Permintaan dan produksi makanan-minuman berfluktuasi secara musiman setiap empat triwulan (menguat pada pertengahan tahun dan melandai di akhir/awal tahun). Di samping siklus kalender Masehi, lonjakan konsumsi bulan **Ramadan dan Hari Raya Idul Fitri** bergeser maju sekitar 11 hari setiap tahun — jatuh penuh di **Q3** pada `2010–2013`, berpindah ke **Q2** pada `2017–2022`, terbagi di perbatasan `Q1–Q2` pada `2023–2024`, dan masuk penuh ke **Q1** pada `2025–2026`.

Karakteristik tersebut menuntut kerangka pemodelan deret waktu yang mampu menangani akar unit (_unit root_), ketergantungan autokorelasi musiman, serta efek variasi kalender secara simultan.

---

## 2. Sumber dan Deskripsi Dataset

### 2.1 Sumber Data

Dataset yang digunakan dalam penelitian ini bersumber dari **Badan Pusat Statistik (BPS) Republik Indonesia** yang diperoleh melalui **Pusat Data dan Informasi (Pusdatin) Kementerian Perindustrian**. Data mencakup deret waktu kuartalan ($s = 4$) selama **67 observasi berurutan** mulai **`2010-Q1` hingga `2026-Q3`** yang tersimpan pada berkas [`data/pdb_industri_makanan_minuman_bps.csv`](data/pdb_industri_makanan_minuman_bps.csv).

Berdasarkan audit integritas data pada _notebook_:

- **Nilai hilang (_missing / NULL values_):** `0` pada seluruh 8 kolom (`0,0%`).
- **Kontinuitas rentang waktu:** `0` triwulan terlewat sepanjang `2010-Q1` s.d. `2026-Q3`.
- **Duplikasi periode:** `0` baris duplikat.

### 2.2 Struktur Variabel & Statistik Deskriptif ($N = 67$)

| Nama Kolom                             | Tipe Data  |        Peran Variabel        | Deskripsi                                                                                                                               |
| :------------------------------------- | :--------: | :--------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------- |
| `periode`                              |   `str`    |        Penanda Waktu         | Kode kuartal kalender dalam format `YYYY-QN` (`2010-Q1` s.d. `2026-Q3`).                                                                |
| `tanggal`                              | `datetime` |         Indeks Waktu         | Tanggal awal masing-masing triwulan (`YYYY-MM-DD`).                                                                                     |
| `tahun`                                |  `int64`   |       Atribut Kalender       | Tahun kalender Masehi (`2010`–`2026`).                                                                                                  |
| `triwulan`                             |  `int64`   |        Indeks Musiman        | Urutan triwulan dalam satu tahun (`1`, `2`, `3`, `4`).                                                                                  |
| `pdb_mamin_adhk_miliar_rp`             | `float64`  |   **Target Utama ($Y_t$)**   | PDB Industri Makanan & Minuman Atas Dasar Harga Konstan 2010 (Miliar Rupiah), mencerminkan volume produksi riil tanpa distorsi inflasi. |
| `pdb_mamin_adhb_miliar_rp`             | `float64`  |      Pembanding Nominal      | PDB Industri Makanan & Minuman Atas Dasar Harga Berlaku (Miliar Rupiah).                                                                |
| `pdb_industri_nonmigas_adhk_miliar_rp` | `float64`  |       Konteks Sektoral       | Total PDB Industri Pengolahan Non-Migas ADHK 2010 (Miliar Rupiah) untuk menghitung pangsa subsektor.                                    |
| `hari_ramadan_idul_fitri`              |  `int64`   | **Eksogen Kalender ($X_t$)** | Jumlah hari periode bulan Ramadan dan Idul Fitri (maksimum 32 hari per siklus Hijriah) yang jatuh pada triwulan bersangkutan.           |

Ringkasan statistik deskriptif dari hasil eksekusi _notebook_ adalah sebagai berikut:

| Indikator                                  |       Mean |   Std. Dev |   Minimum | Median (Q2) |   Maksimum |
| :----------------------------------------- | ---------: | ---------: | --------: | ----------: | ---------: |
| **`pdb_mamin_adhk_miliar_rp` (Miliar Rp)** | 166.771,15 |  48.653,10 | 85.184,00 |  171.813,30 | 261.225,30 |
| **`pdb_mamin_adhb_miliar_rp` (Miliar Rp)** | 236.709,51 | 110.887,49 | 84.636,90 |  231.521,00 | 487.258,10 |
| **Pangsa thd. Industri Non-Migas (%)**     |      34,04 |       3,58 |     27,34 |       35,05 |      39,08 |
| **Hari Ramadan & Idul Fitri (Hari)**       |       8,12 |      12,95 |      0,00 |        0,00 |      32,00 |

---

## 3. Metodologi Penelitian

Seluruh tahapan metodologi dirancang mengikuti prinsip pencegahan kebocoran data masa depan (_no look-ahead bias_ dan _strict featurization ordering_), di mana parameter statistik maupun pra-pemrosesan hanya diestimasi dari data latih.

### 3.1 Pembagian Data Kronologis (_Chronological Split_)

Dataset sepanjang 67 triwulan dibagi secara berurutan menurut waktu menjadi tiga himpunan independen:

1. **Training Set (`2010-Q1` s.d. `2022-Q3` — 51 observasi):** Digunakan untuk analisis ketidakstasioneran, estimasi parameter transformasi (_Box-Cox_), serta pelatihan parameter model.
2. **Validation Set (`2022-Q4` s.d. `2024-Q3` — 8 observasi / 2 tahun):** Digunakan untuk mengevaluasi model _baseline_ awal, memilih kandidat orde terbaik, dan menala _hyperparameter_ secara _out-of-sample_.
3. **Test Holdout Set (`2024-Q4` s.d. `2026-Q3` — 8 observasi / 2 tahun):** Disimpan sepenuhnya steril (_untouched_) selama proses seleksi model dan hanya digunakan satu kali pada pengujian akhir.

### 3.2 Eksplorasi Deret Waktu, Dekomposisi STL, & Deteksi Guncangan Struktural

- **Dekomposisi STL (_Seasonal-Trend Decomposition using LOESS_):** Memisahkan deret $Y_t = T_t + S_t + R_t$ dengan periode musiman $s = 4$ dan pembobotan _robust_ (`robust=True`) agar estimasi tren dan musiman tidak terdistorsi oleh _outlier_.
- **Analisis Laju Pertumbuhan Kuartalan (_q-to-q_) & Pemetaan Kalender:** Mengukur distribusi pertumbuhan $\frac{Y_t - Y_{t-1}}{Y_{t-1}} \times 100\%$ pada masing-masing kuartal (`Q1`–`Q4`) serta memetakan distribusi hari Ramadan–Idul Fitri per kuartal sepanjang `2010–2026`.
- **Deteksi Anomali / Guncangan Struktural (_Robust Z-Score_):** Menggunakan _Median Absolute Deviation_ (MAD) pada komponen ireguler $R_t$:
  $$Z_{\text{robust}, t} = \frac{0,6745 \cdot (R_t - \text{median}(R))}{\text{median}(|R_t - \text{median}(R)|)}$$
  Observasi dengan $|Z_{\text{robust}, t}| > 2,8$ ditandai sebagai guncangan struktural ekonomi (misalnya kontraksi akhir tahun selama pandemi pada `2020-Q4` dan `2021-Q4`), namun tetap dipertahankan karena merepresentasikan dinamika makroekonomi aktual.

### 3.3 Pengujian Ketidakstasioneran & Transformasi Pra-Pemrosesan

Pengujian dilakukan secara berlapis murni pada **Training Set ($N = 51$)**:

1. **Evaluasi Visual Rata-Rata & Varians Bergulir:** Menghitung _Rolling Mean_ dan _Rolling Standard Deviation_ berjendela 4 triwulan untuk memeriksa apakah ekspektasi $E[Y_t]$ dan simpangan baku fluktuasi berubah terhadap waktu.
2. **Transformasi Box-Cox:** Mengestimasi parameter $\lambda$ yang memaksimumkan fungsi _profile log-likelihood_ pada _Training Set_:
   $$Y_t^{(\lambda)} = \begin{cases} \dfrac{Y_t^\lambda - 1}{\lambda}, & \lambda \neq 0 \\ \ln(Y_t), & \lambda = 0 \end{cases}$$
   serta mengevaluasi rentang kepercayaan 95% berbasis distribusi $\chi^2_{(1)}$.
3. **Analisis Correlogram (ACF & PACF):** Memeriksa pola peluruhan koefisien autokorelasi hingga 16 _lag_ triwulan untuk mendeteksi keberadaan tren stokastik (_slow linear decay_ pada ACF dan lonjakan mendekati $1$ pada PACF _lag_ 1).
4. **Uji Akar Unit Dua Arah (ADF & KPSS):** Mengombinasikan dua uji formal dengan hipotesis nol yang saling melengkapi pada taraf nyata $\alpha = 5\%$:
   - **Augmented Dickey-Fuller (ADF)** (spesifikasi konstanta `c` serta konstanta + tren `ct`): $H_0$ menyatakan deret mengandung _unit root_ (**tidak stasioner**).
   - **Kwiatkowski-Phillips-Schmidt-Shin (KPSS)**: $H_0$ menyatakan deret **stasioner**.
     Deret dinyatakan stasioner hanya apabila secara konsisten menolak $H_0$ ADF ($p < 0,05$) sekaligus gagal menolak $H_0$ KPSS ($p \ge 0,05$). Untuk deret yang tidak stasioner, tahap pemodelan utama menerapkan _regular differencing_ $(1 - B)^d$ dan _seasonal differencing_ $(1 - B^4)^D$.

### 3.4 Spesifikasi Model Baseline Awal

Sebagai tolok ukur dasar (_benchmark_ minimum) yang telah diimplementasikan dan dievaluasi pada _notebook_, dibangun tiga model _baseline_ yang dilatih murni pada _Training Set_:

1. **Baseline 1 — Naive (Last Observed Value):** Memproyeksikan seluruh titik pada horizon validasi sama dengan observasi terakhir _Training Set_ (`2022-Q3`):
   $$\hat{Y}_{T+h} = Y_T, \qquad h = 1, 2, \dots, H$$
2. **Baseline 2 — Seasonal Naive ($s = 4$):** Mengulang pola 4 triwulan terakhir dari _Training Set_ (`2021-Q4` s.d. `2022-Q3`) untuk kuartal yang bersesuaian:
   $$\hat{Y}_{T+h} = Y_{T + ((h-1)\bmod 4) - 3}$$
3. **Baseline 3 — Log-Linear Trend + Seasonal Dummy (OLS):** Model regresi linier parametrik pada domain logaritma natural dengan indeks waktu linier $t$ dan variabel _dummy_ kuartal (`Q2`, `Q3`, `Q4`; dengan `Q1` sebagai kategori acuan):
   $$\ln(Y_t) = \beta_0 + \beta_1 t + \gamma_2 D_{2,t} + \gamma_3 D_{3,t} + \gamma_4 D_{4,t} + u_t, \qquad \hat{Y}_t = \exp\left(\widehat{\ln Y_t}\right)$$

### 3.5 Kerangka Model Utama (`RegARIMA / SARIMA`) & Metode Pembanding (`SVR-RBF`)

Berlandaskan temuan ketidakstasioneran dan keterbatasan _baseline_ awal, tahapan pemodelan lanjutan membandingkan dua paradigma berbeda:

1. **Model Utama — `SARIMA` / `RegARIMA`:** Memodelkan ketergantungan _Autoregressive_ dan _Moving Average_ non-musiman maupun musiman setelah diferensiasi $(d, D)$ dengan tambahan regresor kalender hari raya $X_t$ (`hari_ramadan_idul_fitri`):
   $$Y_t^{(\lambda)} = \beta X_t + \eta_t, \qquad \phi_p(B)\,\Phi_P(B^4)\,(1 - B)^d (1 - B^4)^D \eta_t = \theta_q(B)\,\Theta_Q(B^4)\,\varepsilon_t$$
   Seleksi orde dilakukan menggunakan kriteria `AIC`/`BIC`, akurasi _Validation Set_, serta pemeriksaan diagnostik residual yang mencakup uji kebebasan autokorelasi **Ljung-Box** (_White Noise_), uji kenormalan **Shapiro-Wilk / Jarque-Bera**, dan uji homoskedastisitas **ARCH-LM**.
2. **Metode Pembanding — _Support Vector Regression_ (`SVR-RBF`):** Pendekatan pembelajaran mesin non-parametrik berbasis kernel _Radial Basis Function_ $K(x_i, x_j) = \exp(-\gamma \|x_i - x_j\|^2)$ yang dilatih pada deret perubahan logaritma stasioner $\Delta \ln(Y_t)$ menggunakan fitur _Autoregressive Seasonal Lags_ ($1..4$), gelombang _Harmonik Fourier_ kuartalan, indikator kuartal, dan perubahan hari raya $\Delta X_t$ melalui strategi peramalan multi-langkah rekursif.

---

## 4. Evaluasi Model & Temuan Empiris pada Notebook

### 4.1 Metrik Evaluasi Performa

Seluruh prediksi dievaluasi kembali pada skala asli (**Miliar Rupiah**) menggunakan empat metrik galat standar agar perbandingan antarmodel bersifat setara dan mudah diinterpretasikan secara ekonomi:

- **RMSE (_Root Mean Squared Error_):** $\sqrt{\frac{1}{H}\sum_{t=1}^{H}(Y_t - \hat{Y}_t)^2}$ — memberikan bobot penalti lebih besar pada kesalahan prediksi ekstrem.
- **MAE (_Mean Absolute Error_):** $\frac{1}{H}\sum_{t=1}^{H}|Y_t - \hat{Y}_t|$ — mengukur rata-rata selisih absolut dalam Miliar Rupiah.
- **MAPE (_Mean Absolute Percentage Error_):** $\frac{100\%}{H}\sum_{t=1}^{H}\left|\frac{Y_t - \hat{Y}_t}{Y_t}\right|$ — mengukur persentase kesalahan relatif terhadap nilai aktual.
- **sMAPE (_Symmetric Mean Absolute Percentage Error_):** $\frac{100\%}{H}\sum_{t=1}^{H}\frac{2|Y_t - \hat{Y}_t|}{|Y_t| + |\hat{Y}_t|}$ — menyeimbangkan penalti antara _over-prediction_ dan _under-prediction_.

### 4.2 Hasil Pengujian Stasioneritas pada _Training Set_ (`2010-Q1` – `2022-Q3`)

Estimasi parameter **Box-Cox** pada _Training Set_ menghasilkan $\hat{\lambda} = 0,6817$ (interval kepercayaan 95%: $[-0,538,\; 1,950]$). Selanjutnya, hasil pengujian akar unit formal pada data level sebelum diferensiasi adalah sebagai berikut:

| Deret Uji (_Training Set_, $N = 51$) | ADF Stat (`c`) | ADF _p-value_ (`c`) | ADF _p-value_ (`ct`) | KPSS Stat (`c`) | KPSS _p-value_ (`c`) | Kesimpulan ($\alpha = 5\%$) |
| :----------------------------------- | -------------: | ------------------: | -------------------: | --------------: | -------------------: | :-------------------------: |
| **Level Asli $Y_t$ (Miliar Rupiah)** |        -0,4254 |              0,9058 |               0,0571 |          1,1251 |               0,0100 |     **TIDAK STASIONER**     |
| **Level Tertransformasi $\ln(Y_t)$** |        -1,8490 |              0,3564 |               0,9946 |          1,1192 |               0,0100 |     **TIDAK STASIONER**     |

Baik pada skala asli $Y_t$ maupun setelah transformasi logaritma $\ln(Y_t)$, nilai _p-value_ uji ADF tetap berada di atas $0,05$ (gagal menolak $H_0$ adanya _unit root_) dan nilai _p-value_ uji KPSS berada di bawah $0,05$ (menolak $H_0$ stasioneritas). Selaras dengan _Rolling Mean_ yang menanjak monoton dan peluruhan ACF yang sangat lambat, terbukti secara konklusif bahwa data level **memang tidak stasioner** dan membutuhkan proses _differencing_.

### 4.3 Hasil Evaluasi Model Baseline Awal pada _Validation Set_ (`2022-Q4` – `2024-Q3`)

Ketiga model _baseline_ awal yang dilatih pada _Training Set_ (`2010-Q1` – `2022-Q3`) menghasilkan performa _out-of-sample_ sepanjang 8 triwulan _Validation Set_ sebagai berikut:

| Peringkat | Model Baseline Awal                                      | RMSE (Miliar Rp) | MAE (Miliar Rp) | MAPE (%) | sMAPE (%) |
| :-------: | :------------------------------------------------------- | ---------------: | --------------: | -------: | --------: |
|   **1**   | **Baseline 1 — Naive (Last Value)**                      |         9.407,24 |        7.475,00 |   3,387% |    3,474% |
|   **2**   | **Baseline 2 — Seasonal Naive ($s = 4$)**                |        17.789,37 |       16.540,02 |   7,627% |    7,977% |
|   **3**   | **Baseline 3 — Log-Linear Trend + Seasonal Dummy (OLS)** |        23.407,15 |       22.724,00 |  10,481% |    9,935% |

**Interpretasi Empiris Evaluasi Baseline:**

1. **Mengapa _Log-Linear Trend + Seasonal Dummy (OLS)_ memiliki galat tertinggi (`MAPE 10,481%`)?** Model OLS mengestimasi satu kemiringan pertumbuhan eksponensial konstan sejak `2010-Q1`. Akibat terjadinya pelandaian kemiringan tren setelah guncangan pandemi `2020–2021`, garis ekstrapolasi deterministik OLS melambung terlalu tinggi (_systematic over-prediction_) sepanjang `2022-Q4` s.d. `2024-Q3` dengan selisih rata-rata mencapai Rp22.724,00 Miliar.
2. **Mengapa _Seasonal Naive_ tertinggal di bawah aktual (`MAPE 7,627%`)?** Karena mengulang mentah-mentah siklus empat triwulan `2021-Q4`–`2022-Q3` selama dua tahun ke depan tanpa komponen tren, _Seasonal Naive_ mengalami _systematic under-prediction_ yang melebar pada tahun kedua validasi (`2023-Q4`–`2024-Q3`).
3. **Keterbatasan _Naive (Last Value)_ meskipun meraih `MAPE 3,387%`:** Angka galat yang relatif rendah pada _Naive_ semata-mata terjadi karena titik terakhir _Training Set_ (`2022-Q3`) bertepatan dengan puncak musiman kuartal ketiga, sehingga proyeksi garis datarnya kebetulan memotong rata-rata awal periode validasi. Namun secara struktural, model ini menghasilkan prediksi datar yang gagal menangkap dinamika kuartalan maupun kenaikan tren lanjutan menuju `2024-Q3`.

Temuan evaluasi ini menetapkan tolok ukur kuantitatif (`MAPE < 3,387%` dan `RMSE < Rp9.407,24 Miliar` pada _Validation Set_) sekaligus membuktikan perlunya model **RegARIMA / SARIMA** dan **SVR-RBF** yang mampu beradaptasi terhadap tren stokastik pasca-pandemi, struktur autokorelasi musiman, serta pergeseran kalender hari raya.
