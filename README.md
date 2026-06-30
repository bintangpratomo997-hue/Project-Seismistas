# 🌏 Visual Analytics Seismisitas Indonesia

Sistem **business intelligence & visual analytics** untuk menganalisis dan memvisualisasikan data seismisitas (kegempaan) Indonesia secara geospasial. Aplikasi mengintegrasikan katalog gempa BMKG dengan data patahan aktif PSG/Badan Geologi ESDM, lalu menyajikannya melalui dashboard interaktif berbasis **Plotly Dash** yang mencakup peta interaktif, estimasi kepadatan (KDE), dan analisis klaster spasial (DBSCAN).

> Dikembangkan sebagai bagian dari skripsi S1 Sistem Informasi, Universitas Tarumanagara.
> **Demo:** https://project-seismistas-production.up.railway.app

---

## ✨ Fitur Utama

Dashboard terdiri atas tiga halaman:

1. **Ikhtisar (Overview)** — ringkasan statistik seismisitas: total kejadian, distribusi per kategori magnitudo/kedalaman, tren tahunan, dan kejadian per provinsi, dengan filter rentang tahun.
2. **Peta Interaktif** — sebaran episenter gempa di atas basemap OpenStreetMap, dengan overlay **patahan aktif** dan **heatmap KDE** (Kernel Density Estimation) yang dapat diaktif/nonaktifkan, serta filter magnitudo, kedalaman, dan tahun.
3. **Analisis Klaster (DBSCAN)** — pengelompokan zona seismik aktif secara *real-time* dengan parameter ε (epsilon) dan `min_samples` yang dapat diatur pengguna, lengkap dengan metrik evaluasi (Silhouette Score, Davies–Bouldin Index), peta zona klaster, bar chart jumlah gempa per klaster, scatter plot magnitudo–kedalaman, dan boxplot distribusi atribut per klaster.

---

## 🧱 Arsitektur & Pipeline Data

```
┌─────────────┐   EXTRACT    ┌─────────────┐  TRANSFORM   ┌──────────────────┐
│  BMKG CSV   │ ───────────▶ │   ETL       │ ───────────▶ │  PostgreSQL       │
│  Patahan    │              │ pipeline    │   • bersih   │  Star Schema      │
│  GeoJSON    │              │             │   • CRS      │  FACT_GEMPA +     │
└─────────────┘              └─────────────┘   • fitur    │  6 DIM_*          │
                                               LOAD ─────▶ └──────────────────┘
                                                                    │
                                       ┌────────────────────────────┴───────────┐
                                       ▼                                          ▼
                                 DBSCAN (zona klaster)                 KDE (kepadatan episenter)
                                       └────────────────────┬─────────────────────┘
                                                            ▼
                                            Dashboard Visual Analytics (Plotly Dash)
```

- **Extract** — `etl_pipeline.py` membaca katalog gempa BMKG (CSV) dan patahan aktif PSG/ESDM (GeoJSON).
- **Transform** — pembersihan & deduplikasi, normalisasi CRS (EPSG:4326 → EPSG:32750 / UTM 50S), kategorisasi magnitudo & kedalaman, perhitungan jarak episenter ke patahan terdekat, dan ekstraksi komponen waktu.
- **Load** — pemuatan ke PostgreSQL berskema bintang (*star schema*): tabel fakta `FACT_GEMPA` + enam tabel dimensi (`DIM_WAKTU`, `DIM_LOKASI`, `DIM_MAGNITUDE`, `DIM_KEDALAMAN`, `DIM_PATAHAN`, `DIM_KLUSTER`).
- **Modeling** — `dbscan_clustering.py` menjalankan grid search DBSCAN dan menyimpan hasil klaster final; KDE dihitung *real-time* di dashboard.

---

## 📊 Hasil Klastering (Parameter Final)

Parameter final: **ε = 200 km**, **min_samples = 1000** → **4 zona seismik** (noise 6,9%); Silhouette Score = 0,4604; Davies–Bouldin Index = 2,4327.

| Klaster | Wilayah | Jumlah Gempa |
|:-------:|---------|:------------:|
| 0 | Sulawesi Utara–Maluku | 20.079 |
| 1 | Papua | 4.332 |
| 2 | Sumatra | 15.321 |
| 3 | Nusa Tenggara | 4.847 |

---

## 🛠️ Teknologi

| Kategori | Teknologi |
|----------|-----------|
| Bahasa | Python 3.11 |
| Dashboard | Dash 2.18, Dash Bootstrap Components, Plotly 5.22 |
| Data & Analitik | pandas, NumPy, scikit-learn (DBSCAN), SciPy (KDE) |
| Geospasial | GeoPandas, Shapely, pyproj, Fiona |
| Basis Data | PostgreSQL + SQLAlchemy + psycopg2 |
| Deployment | Gunicorn, Railway |

---

## 📁 Struktur Berkas

```
Project-Seismistas/
├── app.py                              # Aplikasi dashboard Plotly Dash (entry point)
├── etl_pipeline.py                     # Pipeline ETL: CSV/GeoJSON → PostgreSQL star schema
├── dbscan_clustering.py                # Grid search & clustering DBSCAN
├── seismisitas_db_dump.sql             # Dump basis data (skema + data) untuk restore
├── bmkg_gempa_indonesia_Ja2000_Fe2026.csv   # Katalog gempa BMKG (2000–2026)
├── Patahan Aktif geoportal.esdm.go.id.geojson  # Data patahan aktif PSG/ESDM
├── assets/                             # Aset statis (CSS/gambar) untuk Dash
├── requirements.txt                    # Dependensi Python
├── runtime.txt                         # Versi Python (python-3.11.9)
└── Procfile                            # Perintah start untuk deployment
```

---

## 🚀 Menjalankan Secara Lokal

### 1. Prasyarat
- Python 3.11
- PostgreSQL (lokal atau remote)

### 2. Kloning & instalasi dependensi
```bash
git clone https://github.com/bintangpratomo997-hue/Project-Seismistas.git
cd Project-Seismistas
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Siapkan basis data
Buat database lalu pulihkan dari dump:
```bash
createdb seismisitas_db
psql -d seismisitas_db -f seismisitas_db_dump.sql
```
Atau bangun ulang dari sumber: jalankan `python etl_pipeline.py` lalu `python dbscan_clustering.py`.

### 4. Konfigurasi koneksi
Sesuaikan kredensial PostgreSQL (host, port, dbname, user, password) pada konfigurasi koneksi di `app.py`, `etl_pipeline.py`, dan `dbscan_clustering.py`. **Disarankan** memakai *environment variable* (mis. `DATABASE_URL`) alih-alih menuliskan kata sandi langsung di kode.

### 5. Jalankan dashboard
```bash
python app.py
# buka http://127.0.0.1:8050
```

---

## ☁️ Deployment (Railway)

Aplikasi siap di-deploy ke Railway/Heroku-style PaaS:
- `Procfile` → `web: gunicorn app:server --bind 0.0.0.0:$PORT --workers 1 --timeout 120`
- `runtime.txt` → `python-3.11.9`
- Tambahkan PostgreSQL plugin dan set variabel koneksi basis data pada *service variables*.

---

## 📚 Sumber Data

- **Katalog Gempa** — Badan Meteorologi, Klimatologi, dan Geofisika (BMKG), DataOnline, periode 2000–2026.
- **Patahan Aktif** — Pusat Survei Geologi (PSG) / Badan Geologi, Kementerian ESDM (geoportal.esdm.go.id).

---

## 👤 Penulis

**Sri Bintang Pratomo** (NIM 825220070)
Program Studi Sistem Informasi — Universitas Tarumanagara

---

> Repositori ini merupakan artefak penelitian akademik.
