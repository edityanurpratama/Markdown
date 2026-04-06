# Ringkasan Eksekutif  
Dokumen ini menjelaskan langkah terperinci melakukan klasterisasi K-means pada data RT tahunan dan analisis deret waktu per klaster. Data diasumsikan lengkap untuk tahun 2021–2025 (tiap RT punya 5 titik data). Langkah pra-pemrosesan meliputi agregasi kolom Male dan Female menjadi total populasi per tahun per RT, normalisasi fitur (z-score), dan ekstraksi fitur seperti proporsi laki-laki, tren (slope), atau perbedaan tahun sebelumnya. Untuk klasterisasi, K-means dipilih karena sederhana dan skala baik untuk data relatif kecil, dengan jumlah klaster ditentukan melalui metode *elbow*, *silhouette*, atau *gap statistic*. Validasi klaster dilakukan dengan skor Silhouette dan indeks Davies–Bouldin【19†L679-L688】【27†L680-L688】; nilai Silhouette mendekati 1 menunjukkan klaster berjarak baik (baik homogenitas internal), sedangkan Davies–Bouldin lebih rendah lebih baik. Hasil klaster divisualkan dalam grafik *line plot* per RT per klaster dan pusat klaster (centroid).  

Selanjutnya, analisis deret waktu dilakukan per klaster: data tiap RT dalam klaster diagregasi (median atau rata-rata per tahun) untuk membentuk satu rangkaian waktu per klaster. Metode peramalan yang direkomendasikan mencakup dekomposisi (mis. STL), ARIMA/SARIMA (jika data stasioner), Prophet (Model aditif Facebook yang tangguh terhadap outlier dan data hilang【21†L31-L39】), atau ETS (Exponential Smoothing) yang mudah diinterpretasi. Model dipilih sesuai karakteristik: misalnya ARIMA fleksibel dan akurat untuk pola sederhana【9†L43-L51】, sedangkan Prophet unggul menangani tren berubah dan outlier【9†L43-L51】【21†L31-L39】. Setiap model di-fit dengan validasi (sisa, MAE, RMSE, MAPE), lalu dibuat peramalan 1–3 tahun ke depan. Deteksi anomali bisa dilakukan dengan memeriksa residual atau outlier pada input/forecast. Hasil klasterisasi dapat membantu menyusun kebijakan: klaster dengan tren negatif atau fluktuasi tinggi diprioritaskan intervensi. Reproduksibilitas dijaga dengan *seed* acak tetap, pembagian data (mis. training historis, test untuk evaluasi), serta pencatatan versi pustaka (pandas, scikit-learn, Prophet).  

【38†embed_image】 *Contoh visualisasi hasil analisis: sebuah grafik time series hasil klasterisasi (contoh ilustratif).*

## Tabel Perbandingan Metode Clustering dan Time-Series  
| **Metode**         | **Kelebihan**                                                   | **Kekurangan**                                                   | **Kebutuhan Data**                               | **Kompleksitas**                 |
|--------------------|-----------------------------------------------------------------|-------------------------------------------------------------------|--------------------------------------------------|----------------------------------|
| **K-means**        | Sederhana, cepat, skalabel ke data besar, konvergen【17†L118-L122】. Mendukung cluster berbentuk elips (dengan generalisasi)【17†L118-L122】. | Harus tentukan *K* manual; sensitif inisialisasi awal dan outlier【17†L126-L133】【17†L136-L139】; sulit untuk cluster dengan ukuran/kepadatan berbeda. Prone *curse of dimensionality*【17†L126-L133】. Hanya bentuk cluster bulat (spherical) dengan baik. | Data numerik, tidak terlalu besar (m *features* rendah hingga menengah). | Relatif rendah (O(n·k·d·iter) per iterasi). |
| **Hierarki (Agglomerative)** | Tak perlu tentukan *K* di awal; dendrogram memvisualisasi struktur berjenjang; bisa deteksi cluster arbitrer bentuk. | Tidak efisien untuk data besar (O(n³)); perlu simpan matriks jarak; sensitif ke ukuran/kepadatan data. | Data kecil–sedang; mendukung semua jenis metrik jarak. | Tinggi (O(n³) waktu dan O(n²) memori). |
| **DBSCAN** (jika relevan) | Dapat temukan cluster bentuk arbitrer, tidak perlu K, tahan outlier. | Tidak cocok untuk data dengan densitas bervariasi tinggi; perlu atur parameter *eps*; tidak langsung untuk time series. | Data cukup banyak; banyak sampel untuk definisi densitas. | O(n log n) (dengan struktur KD-tree). |
| **ARIMA/SARIMA**   | Model klasik untuk time-series; fleksibel (ARIMA) dan mampu akurasi tinggi jika data stasioner【9†L43-L51】. SARIMA mendukung perioda musiman. | Perlu differencing agar stasioner; pemilihan parameter (p,d,q) non-trivial; asumsi linearitas; kurang baik pada outlier/tren tak terduga. Kompleksitas formulasi tinggi【14†L237-L240】. | Data berurutan (≥ 30–50 titik umum disarankan【6†L229-L238】); pola tren/musiman. | Menengah: fitting ARIMA mungkin memakan waktu dengan evaluasi grid search. |
| **Prophet**        | Otomatis, tangguh terhadap *missing data*, outlier, tren berubah【21†L31-L39】. Mudah penggunaan (masukan kolom waktu & nilai). Dukung musiman harian/mingguan/tahunan dan hari libur. | Kurang optimal jika data sangat sedikit (<sejumlah musim penuh). Kurang transparan (model aditif berbasis Stan). | Data berurutan dengan banyak musim (≥ 3 siklus minimal); pola musiman kuat. | Menengah: fit cukup cepat (Stan) tapi tunning parameter (changepoint) perlu perhatian. |
| **ETS (Holt-Winters)** | Memodel komponen level, tren, musiman secara eksplisit【14†L225-L233】. Mudah diinterpretasi dan cocok untuk pola musiman stabil【14†L225-L233】. | Kurang responsif terhadap lonjakan tiba-tiba (shock)【14†L233-L240】. Asumsi aditif (varian konstan) atau multiplikatif perlu dipilih. | Data berurutan dengan tren/musiman konsisten. | Rendah–menengah: parameter λ smoothing sederhana, pemodelan cepat. |
| **Dekomposisi (STL, dll.)** | Analisis pola: pisahkan tren/musiman. Berguna untuk deteksi anomali dan transformasi lanjut. | Bukan model prediksi langsung; perlu langkah lanjutan setelah dekomposisi. | Data berkala (musiman); cukup banyak titik untuk estimasi musiman. | Rendah: hanya dekomposisi; O(n) per iterasi. |

Tabel di atas merangkum karakteristik setiap metode. ARIMA/SARIMA unggul untuk pola pendek dan stasioner, sedangkan Prophet dan ETS unggul menangani tren/musim. K-means sederhana tapi perlu *K* ditentukan【17†L118-L122】【17†L126-L133】. Metode hierarki memberi fleksibilitas cluster tanpa *K*, namun dengan kompleksitas tinggi.

## Persiapan Data (Pra-Pemrosesan)  
Data diasumsikan lengkap tanpa *missing* untuk semua RT tahun 2021–2025. Langkah utama pra-pemrosesan:  
- **Agregasi**: Hitung total populasi per RT-tahun: `Total = Male + Female`. Bisa juga ekstrak proporsi laki-laki (`Male/Total`) dan wanita (`Female/Total`).  
- **Normalisasi / Standarisasi**: Karena K-means berbasis jarak, fitur harus dinormalisasi (mis. Z-score) supaya unit berbeda (jumlah, proporsi, slope) sebanding【30†L396-L405】. Scaling mencegah variabel bervariasi besar mendominasi jarak.  
- **Ekstraksi Fitur**: Ubah deret waktu menjadi fitur statistik, misalnya: tren linier (slope) tiap RT, indeks musiman (jika ada pola periodik), nilai lag (mis. Total(t) – Total(t–1)), atau statistik ringkasan (mean, varian). Sebagai contoh: menghitung slope garis regresi Total versus Tahun untuk setiap RT memberi fitur tren【27†L680-L688】. Feature ini membantu K-means menangkap pola umum (naik/turun).  
- **Pemisahan Fitur Clustering vs Forecasting**: Fitur yang dipakai K-means dapat berbeda dari yang dipakai di model waktu. Untuk klasterisasi gunakan fitur statis (mis. nilai setiap tahun atau ringkasan); untuk pemodelan waktu gunakan urutan aktual (forecast memerlukan deret chronologis).

## Klasterisasi K-means  
Proses klasterisasi dilakukan setelah data disiapkan:  

1. **Pilih Fitur dan Skala** – Bentuk matriks fitur (mis. pivot data sehingga tiap RT punya kolom total tahun 2021–2025, ditambah fitur lain seperti slope). Lakukan standarisasi ( `StandardScaler` ).  
2. **Tentukan Jumlah Klaster (K)** – Evaluasi grafik *elbow* (inertia vs K) atau skor *silhouette* untuk berbagai K. Skor *silhouette* (rata-rata [–1,1]) diukur dengan `silhouette_score` dari scikit-learn【19†L679-L688】. Skor mendekati 1 menandakan cluster terpisah jelas; ~0 berarti tumpang tindih; negatif artinya salah klaster. Metode *gap statistic* juga dapat digunakan sebagai pembanding.  
3. **Jalankan Algoritma K-means** – Gunakan `sklearn.cluster.KMeans` dengan `n_init` beberapa kali untuk stabilitas. Contoh kode sederhana:  

```python
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler

# Contoh pipeline sederhana:
X = df_pivot.values  # misalnya array (RT x fitur)
X_scaled = StandardScaler().fit_transform(X)
kmeans = KMeans(n_clusters=3, random_state=0, n_init=10)
labels = kmeans.fit_predict(X_scaled)
df_pivot['cluster'] = labels  # simpan label klaster
```

   *Pseudocode:* untuk tiap RT, hitung jarak Euclidean ke pusat klaster (centroid) terbaik, assign ulang, dan ulangi hingga konvergen.  

4. **Validasi Klaster** – Hitung *silhouette score* dan *Davies–Bouldin* (`davies_bouldin_score`)【27†L680-L688】. Davies–Bouldin mengukur rasio jarak intra-cluster terhadap antar-cluster (nilainya ≥0; makin kecil makin baik)【27†L680-L688】. Silhouette dan Davies–Bouldin mengukur kohesi dan separasi klaster. Jika skor buruk, eksperimen dengan K lain atau metode clustering berbeda.  

5. **Visualisasi Hasil Klaster** – Untuk mengecek pola klaster:  
   - **Line Plot per RT**: Gambarkan masing-masing seri waktu (Total populasi per tahun) untuk RT-RT dalam satu klaster agar pola umum terlihat (mis. sebagian naik, sebagian turun). Pusat klaster (centroid) juga digambarkan (nilai rata-rata tiap tahun untuk klaster).  
   - **Heatmap**: Tampilan matriks (RT vs fitur) dengan pengelompokan klaster, untuk melihat fitur dominan tiap klaster.  
   - **Contoh:** Mulai dari DataFrame `df` yang telah berisi kolom `cluster`, gunakan `plt.plot()` atau `seaborn.lineplot` terpisah per klaster. Misal:  

   ```python
   import matplotlib.pyplot as plt
   for c in sorted(df_pivot['cluster'].unique()):
       subset = df_pivot[df_pivot['cluster']==c]
       for _, row in subset.iterrows():
           plt.plot(years, row[years], color=f"C{c}", alpha=0.3)
       # centroid
       centroid = kmeans.cluster_centers_[c]
       plt.plot(years, centroid[:-1], color=f"C{c}", lw=3, label=f"Centroid {c}")
   plt.xlabel('Tahun'); plt.ylabel('Populasi Total'); plt.legend()
   plt.show()
   ```
   Ini memberikan gambaran pola trend per klaster.  

## Analisis Deret Waktu per Klaster  
Setelah klasterisasi, lakukan analisis deret waktu pada setiap klaster:

- **Agregasi** – Gabungkan data RT dalam klaster: misalnya hitung *median* (lebih robust) atau *mean* populasi tiap tahun per klaster. Hasilnya satu deret waktu per klaster (nilai per tahun).  

- **Metode Pemodelan** – Pilih metode sesuai sifat data:  
  - **Dekomposisi (STL)** untuk menilai komponen tren/musiman. Baik untuk visualisasi dan deteksi anomali.  
  - **ARIMA/SARIMA**: jika data tampak linier/tren, lakukan uji stasioneritas (ADF) dan tentukan ordo p,d,q (Box-Jenkins). SARIMA bila ada musiman tahunan. ARIMA baik untuk pola jangka pendek; perhitungkan bahwa ARIMA memerlukan stasioner dan p/d/q complex【14†L237-L240】.  
  - **ETS (Holt-Winters)**: bila musiman/pola tren jelas. Lebih mudah tunning (jika Auto-ETS tersedia) dan langsung model level/tren/musim【14†L225-L233】. Lebih tidak responsif ke shock singkat【14†L233-L240】.  
  - **Prophet**: untuk data dengan banyak musiman dan perubahan tren. Set up *ds* (waktu) dan *y* (nilai) untuk tiap klaster, lalu `m = Prophet()`. Prophet otomatis memasukkan musiman tahunan/dan mingguan (jika frekuensi mendukung), robust ke *missing* dan outlier【21†L31-L39】.  

- **Forecasting & Anomali**: Fit model masing-masing klaster hingga tahun 2025, lalu lakukan peramalan 1–3 tahun ke depan. Buatlah prediksi dengan fungsi *predict()* (statsmodels) atau `m.predict(future_df)` (Prophet). Evaluasi model: hitung MAE, RMSE, dan MAPE (scikit-learn menyediakan `mean_absolute_error` dan `mean_absolute_percentage_error`【39†L0-L4】). MAPE berguna interpretasi relatif, tapi hati-hati kalau nilainya kecil. Untuk anomali, periksa residual: misalnya titik dengan error jauh > 2σ bisa dianggap anomali.  

- **Contoh Ringkas (Python)**:  
   ```python
   # Asumsi df_cluster_ts: DataFrame dengan kolom ['cluster','Year','Total']
   from prophet import Prophet
   results = {}
   for c in sorted(df_cluster_ts['cluster'].unique()):
       df_c = df_cluster_ts[df_cluster_ts['cluster']==c].rename(columns={'Year':'ds','Total':'y'})
       df_c['ds'] = pd.to_datetime(df_c['ds'], format='%Y')
       m = Prophet(yearly_seasonality=True)  # aktifkan musiman tahunan
       m.fit(df_c)
       future = m.make_future_dataframe(periods=3, freq='Y')
       forecast = m.predict(future)
       # Ambil prediksi tahun 2026–2028 mis.
       print(f"Klaster {c} forecast:", forecast[['ds','yhat','yhat_lower','yhat_upper']].tail(3))
   ```  
   *(Kode contoh: siapkan DataFrame `df_cluster_ts` hasil agregasi klaster, lalu per klaster buat model Prophet.)*  

## Integrasi Hasil dan Rekomendasi Kebijakan  
Hasil klasterisasi dapat menjadi basis kebijakan: misalnya **klaster yang menunjukkan penurunan populasi** (slope negatif) atau fluktuasi tajam bisa diidentifikasi memerlukan intervensi prioritas (pendataan ulang, peningkatan fasilitas, dsb). Klaster dengan tren positif/tinggi pun perlu dipersiapkan infrastruktur. Dengan menggabungkan klasterisasi dan hasil peramalan, dapat disusun strategi alokasi sumber daya sesuai pola daerah.  

**Reproduksibilitas:** Set *random_state* di KMeans dan split data (mis. latih:2021–2023, uji:2024–2025) untuk evaluasi model ts. Gunakan `sklearn.preprocessing.StandardScaler` dan pipeline untuk konsistensi transformasi. Catat versi library (pandas, sklearn, prophet) agar eksperimen dapat diulang.  

## Alur Kerja Analisis (Contoh Mermaid)  
```mermaid
graph TD
A[Load Data RT] --> B[Pra-pemrosesan & Normalisasi]
B --> C[Feature Engineering (Total, Proporsi, Slope)]
C --> D[K-Means (tentukan K) & Labeling]
D --> E[Validasi Klaster (Silhouette, Davies-Bouldin)]
E --> F[Visualisasi Klaster (Line plot/Heatmap)]
F --> G[Agregasi Data per Klaster]
G --> H[Analisis Deret Waktu (Dekomposisi, ARIMA/SARIMA, ETS, Prophet)]
H --> I[Forecast & Deteksi Anomali]
I --> J[Rekomendasi Kebijakan berdasarkan Klaster]
```  

## Contoh Kode Pipeline End-to-End  
```python
import pandas as pd
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score, davies_bouldin_score
from prophet import Prophet
import matplotlib.pyplot as plt

# 1. Load data
df = pd.read_csv('data_rt.csv')  # kolom: RT, Year, Male, Female
df['Total'] = df['Male'] + df['Female']

# 2. Pivot: RT x Tahun, ambil Total
df_pivot = df.pivot(index='RT', columns='Year', values='Total').fillna(0)
years = sorted(df['Year'].unique())
# 3. Ekstrak fitur tambahan: tren (slope) tiap RT
slopes = []
for rt, row in df_pivot.iterrows():
    z = row.values.astype(float)
    m = np.polyfit(years, z, 1)[0]
    slopes.append(m)
df_pivot['slope'] = slopes

# 4. Normalisasi
scaler = StandardScaler()
X = scaler.fit_transform(df_pivot.values)

# 5. Klasterisasi K-means
k = 3
kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
labels = kmeans.fit_predict(X)
df_pivot['cluster'] = labels

# 6. Validasi
sil = silhouette_score(X, labels)
db = davies_bouldin_score(X, labels)
print(f"Silhouette = {sil:.2f}, Davies-Bouldin = {db:.2f}")

# 7. Visualisasi: contoh plot tiap cluster
for c in range(k):
    plt.figure(figsize=(6,4))
    subset = df_pivot[df_pivot['cluster']==c]
    for idx, row in subset.iterrows():
        plt.plot(years, row[years], alpha=0.5, label=f"RT {idx}")
    centroid = scaler.inverse_transform(kmeans.cluster_centers_)[c][:-1]
    plt.plot(years, centroid, 'k--', lw=3, label="Centroid")
    plt.title(f"Cluster {c}: Time Series RT")
    plt.xlabel('Tahun'); plt.ylabel('Total Populasi')
    plt.legend(fontsize=6)
    plt.show()

# 8. Agregasi per klaster (median per tahun)
df['cluster'] = df['RT'].map(df_pivot['cluster'])
df_clust = df.groupby(['cluster','Year'])['Total'].median().reset_index()

# 9. Forecast per klaster menggunakan Prophet
for c in df_clust['cluster'].unique():
    df_c = df_clust[df_clust['cluster']==c][['Year','Total']].rename(columns={'Year':'ds','Total':'y'})
    df_c['ds'] = pd.to_datetime(df_c['ds'], format='%Y')
    m = Prophet(yearly_seasonality=True)
    m.fit(df_c)
    future = m.make_future_dataframe(periods=3, freq='Y')
    forecast = m.predict(future)
    print(f"\nForecast Cluster {c}:")
    print(forecast[['ds','yhat']].tail(3))
```
*Penjelasan singkat:* Script di atas memuat data, menyiapkan fitur, melakukan K-means, memvalidasi (silhouette, Davies–Bouldin), memvisualkan hasil klaster dengan *line plot*, dan melakukan forecast menggunakan Prophet untuk setiap klaster.  

## Sumber Prioritas  
- **Dokumentasi Resmi Python/Scikit-Learn:** Panduan `sklearn.cluster.KMeans`, `silhouette_score`, `davies_bouldin_score`, `StandardScaler` (scikit-learn)【19†L679-L688】【27†L680-L688】. Dok. resmi pandas dan matplotlib.  
- **Prophet (Facebook/Meta):** Dokumentasi Prophet (penjelasan model aditif, robust terhadap outlier/missing)【21†L31-L39】.  
- **Referensi Akademik:** Rousseeuw (1987) untuk *silhouette*【19†L679-L688】; Davies & Bouldin (1979)【27†L680-L688】; literatur perbandingan ARIMA vs Prophet【9†L43-L51】【21†L31-L39】; Hyndman & Athanasopoulos (Forecasting: Principles and Practice) bagi konteks ETS vs ARIMA【14†L225-L233】【14†L237-L240】. (Sumber lokal/Indonesia: riset penerapan Prophet vs ARIMA di jurnal Indonesia【9†L43-L51】.)  

