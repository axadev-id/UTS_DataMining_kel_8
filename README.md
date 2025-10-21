# UTS Data Mining — Analisis Pembelian, Outlier, dan Klasifikasi Permintaan

Dokumentasi resmi proyek analitik untuk mem-parsing data pembelian dari sumber teks/TSV, melakukan pembersihan dan agregasi, deteksi outlier berbasis Z-Score, visualisasi, serta membangun model klasifikasi sederhana (baseline) untuk kategorisasi permintaan produk.

## Daftar Isi

- Ringkasan Proyek
- Dataset & Asumsi
- Metodologi
  - Ingest & Parsing
  - Perekayasaan Fitur & Outlier
  - Agregasi & Visualisasi
  - Pemodelan (Baseline)
- Reproduksibilitas
- Struktur Repositori
- Artefak Keluaran
- Interpretasi & Kesimpulan
- Keterbatasan
- Rencana Lanjutan

## Ringkasan Proyek

Notebook utama (`tes.ipynb`) mengimplementasikan pipeline end-to-end:

1) Membaca dataset mentah dari `data/pembelian.tsv` dan memvalidasi keberadaan file.
2) Mem-parsing struktur baris menjadi entitas transaksi dan ringkasan produk.
3) Menyiapkan fitur numerik utama (Jumlah_Beli), mendeteksi outlier dengan Z-Score, dan melakukan agregasi per produk.
4) Memvisualisasikan distribusi dan hubungan fitur kunci.
5) Membangun baseline klasifikasi kategori permintaan dengan Random Forest.
6) Menyimpan artefak keluaran untuk analisis lanjutan.

## Dataset & Asumsi

- Sumber data: `data/pembelian.tsv` (teks/TSV) berisi header produk, transaksi per baris, dan ringkasan total.
- Encoding pembacaan: UTF-8 dengan `errors='ignore'` untuk toleransi karakter tidak dikenal.
- Heuristik pemisahan transaksi masuk vs. keluar ditentukan oleh lebar spasi di antara kolom (parameter `KELUAR_SPACE_THRESHOLD`).
- Nilai kuantitas/nilai dinormalisasi dari format lokal (titik/koma) menjadi numerik float.

## Metodologi

### 1) Ingest & Parsing

- Header Produk: diekstrak menggunakan regex menjadi `kode`, `nama_produk`, `unit`.
- Transaksi: mengekstrak `tanggal`, `no_transaksi`, `qty`, `nilai`; lalu diberi atribut masuk/keluar.
- Total Produk: baris ringkasan untuk `total_qty_masuk` dan `total_qty_keluar`.
- Hasil parsing awal diserialisasi ke CSV:
  - `pembelian_transaksi.csv` (baris transaksi)
  - `pembelian_total.csv` (ringkasan per produk)

### 2) Perekayasaan Fitur & Outlier

- `Jumlah_Beli` dibentuk dari `qty_masuk` jika tersedia; jika tidak, diisi dari `qty_keluar`.
- Deteksi outlier menggunakan Z-Score klasik dengan ambang |Z| > 3 pada nilai valid.
- Flag outlier per baris disusun ke level produk (produk ber-flag jika memiliki setidaknya satu transaksi outlier).

### 3) Agregasi & Visualisasi

- Agregasi per produk mencakup metrik berikut:
  - `trx_count`, `qty_sum`, `qty_mean`, `qty_median`, `qty_std`, `qty_max`, `qty_p75`, `qty_p95`.
- Visualisasi utama:
  - Histogram `Jumlah_Beli` dengan KDE, mean, dan overlay outlier (merah).
  - Scatter plot berwarna berdasarkan flag outlier, mis. `qty_median` vs `qty_p95`, `trx_count` vs `qty_max`.

### 4) Pemodelan (Baseline)

- Label target `permintaan_kategori` diturunkan dari median `qty_mean` (threshold adaptif).
- Fitur model: `[trx_count, qty_sum, qty_median, qty_std, qty_max, qty_p75, qty_p95]`.
- Split train/test terstratifikasi; standarisasi fitur; model `RandomForestClassifier` (n_estimators=200, random_state=42).
- Laporan evaluasi yang ditampilkan di notebook: confusion matrix, classification report, akurasi.

## Reproduksibilitas

Prasyarat: Python 3.10+ dan Jupyter.

Langkah penyiapan (Windows PowerShell):

```powershell
# (Opsional) Buat dan aktivasi virtual environment
python -m venv .venv ; .\.venv\Scripts\Activate.ps1

# Instal dependensi inti
python -m pip install --upgrade pip
pip install pandas numpy seaborn matplotlib scipy scikit-learn jupyter

# Jalankan Jupyter / buka notebook di VS Code
jupyter notebook
```

Eksekusi:
1) Pastikan `data/pembelian.tsv` tersedia.
2) Buka dan jalankan `tes.ipynb` dari atas ke bawah.
3) Verifikasi artefak keluaran di root repositori (lihat bagian berikutnya).

## Struktur Repositori

```
.
├─ data/
│  └─ pembelian.tsv                # Sumber data mentah (input)
├─ tes.ipynb                       # Notebook utama (ETL, EDA, outlier, ML)
├─ pembelian_transaksi.csv         # Hasil parsing transaksi
├─ pembelian_total.csv             # Hasil ringkasan total per produk
├─ pembelian_transaksi_with_outlier_flag.csv
├─ pembelian_agregat_with_outlier_flag.csv
└─ pembelian_transaksi_final.csv
```

## Artefak Keluaran

- `pembelian_transaksi_with_outlier_flag.csv` — Transaksi dengan kolom indikator outlier.
- `pembelian_agregat_with_outlier_flag.csv` — Agregat per produk beserta flag outlier produk.
- `pembelian_transaksi_final.csv` — Transaksi tanpa outlier (direkomendasikan untuk analisis/prediksi lanjutan).

## Interpretasi & Kesimpulan

- Deteksi outlier (|Z| > 3) efektif untuk menandai observasi ekstrem yang berpotensi disebabkan oleh anomali data atau kejadian operasional khusus.
- Agregasi per produk menyediakan ringkasan permintaan yang informatif (tengah, sebaran, ekor distribusi) dan memudahkan segmentasi risiko outlier.
- Baseline Random Forest dengan label berbasis median memberikan titik awal yang adaptif. Hasil evaluasi (confusion matrix, classification report, akurasi) tercetak di notebook dan bergantung pada karakteristik dataset.

Secara keseluruhan, pipeline ini:
1) Meningkatkan kualitas data melalui penandaan dan penyaringan outlier,
2) Menyediakan fitur agregat yang relevan untuk analisis permintaan,
3) Menawarkan baseline klasifikasi yang siap dikembangkan lebih lanjut.

## Keterbatasan

- Skema pelabelan menggunakan median `qty_mean` (unsupervised target derivation) — belum memanfaatkan sinyal domain khusus (mis. SLA stok, target KPI, musiman).
- Z-Score sensitif terhadap distribusi tidak normal dan outlier ganda; metode robust dapat memberikan hasil lebih stabil.
- Model baseline belum melalui tuning hyperparameter dan validasi silang menyeluruh.

## Rencana Lanjutan

- Uji metode outlier alternatif (IQR, robust Z-Score/MAD) dan bandingkan dampaknya terhadap downstream task.
- Tambahkan fitur waktu (musiman, tren), harga, dan stok; evaluasi variasi per kategori produk.
- Terapkan cross-validation dan hyperparameter tuning (Grid/Random/Bayesian Search).
- Kelola pipeline dengan `sklearn` Pipeline dan simpan artefak (`joblib`) untuk replikasi/deployment.
- Dokumentasikan eksperimen dan parameterisasi melalui config (YAML) untuk memudahkan reproducibility.
