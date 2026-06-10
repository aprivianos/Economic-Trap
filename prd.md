# Product Requirements Document (PRD)
## What-If Simulator — Regional Economic Trap Early Warning System

---

## 1. Tujuan
Menyediakan simulator interaktif bagi pengguna untuk mengubah parameter fiskal dan demografi, lalu melihat dampaknya terhadap prediksi pertumbuhan ekonomi dan klasifikasi provinsi.

---

## 2. Fitur Utama

### 2.1 Peta Interaktif
- Peta choropleth seluruh Indonesia
- Provinsi terpilih diberi highlight biru terang dengan border tebal
- Provinsi lain tampil transparan (border tipis)
- Center peta otomatis ke provinsi yang dipilih
- Peta static (tanpa drag/zoom/modebar)

### 2.2 Parameter Slider (5 Variabel)

| Variabel | Rentang | Default | Dampak |
|---|---|---|---|
| Belanja APBN | 0% - 50% | 10% | +0.119361% PDRB per 1% |
| Penyaluran KUR | 0% - 50% | 5% | +0.319629% PDRB per 1% |
| Transfer Daerah (TKD) | 0% - 50% | 15% | +0.172685% PDRB per 1% |
| KURS USD ke Rp | 5.000 - 20.000 | Data asli | Konversi PDRB/kap ke USD |
| Jumlah Penduduk | 50% - 150% (ribuan jiwa) | Data asli | PDRB per kapita |

### 2.3 Hasil Simulasi (3 Kolom)
- **Prediksi Pertumbuhan**: nilai pertumbuhan asli vs skenario + delta perubahan
- **Economic Trap**: status trap + target naik kelas dari indikator.xlsx
- **Klasifikasi**: badge LOM/UPM/HIGH + naratif perubahan kelas

### 2.4 Prediksi Capai Kelas Berikutnya
Menampilkan estimasi **jangka waktu tahunan** untuk mencapai 1 tingkat di atas klasifikasi awal.

**Formula hierarki (prioritas):**
1. Jika delta PDRB/kap dari simulasi > 0 → gunakan delta simulasi
2. Jika delta simulasi ≤ 0 → gunakan rata-rata growth historis PDRB/kap (5 tahun terakhir)
3. Jika data historis tidak tersedia → fallback 1% per tahun

**Target per kelas awal:**
| Kelas Awal | Target | Threshold |
|---|---|---|
| LOW | Lower-Middle Income | $905 |
| LOM | Upper-Middle Income | $3,595 |
| UPM | High Income | $11,115 |

**Rumus:**
```
annual_delta = max(delta_simulasi, pdrbkap_saat_ini * growth_historis)
tahun_prediksi = (target_threshold - pdrbkap_saat_ini) / annual_delta + 1
```

### 2.5 Reset Hasil
Saat provinsi atau tahun diubah, seluruh hasil simulasi dan analisis by system otomatis di-reset.

### 2.6 Analisis by System (Opsional)
- Integrasi dengan OpenAI / DeepSeek / SumoPod API
- Mengirim data simulasi ke LLM untuk analisis naratif
- Output dalam Bahasa Indonesia (4 bagian: Ringkasan, Analisis, Klasifikasi, Rekomendasi)
- Tombol "Analisis by System" di kolom kanan

---

## 3. Sumber Data

### 3.1 Data Utama
- **File**: `data/processed/data_processed.parquet`
- **Pipeline**: `src/data_pipeline.py` (load, clean, lag, rolling, ratio)
- **Model**: XGBoost (models/model_xgboost.pkl)

### 3.2 Data Pendukung (`pendukung/`)

**dasar analisis.txt** — Koefisien elastisitas:
| Variabel | Dampak per 1% |
|---|---|
| Belanja APBN | +0.119361% PDRB |
| Penyaluran KUR | +0.319629% PDRB |
| Transfer Daerah (TKD) | +0.172685% PDRB |

**indikator.xlsx** — Threshold dan target kenaikan kelas:
| Kelas Saat Ini | Naik Kelas | Target PDRB/kap | Growth Diperlukan |
|---|---|---|---|
| LOW | LOM | $1,146 | 1.86% |
| LOM | UPM | $4,516 | 1.69% |
| UPM | HIGH | $14,005 | 1.70% |

**metadata.xlsx** — Deskripsi 21 kolom dataset.

### 3.3 Threshold Klasifikasi (World Bank)
- **LOM** (Lower-Middle Income): < $905
- **UPM** (Upper-Middle Income): $905 - $3,595
- **HIGH** (High Income): > $3,595

---

## 4. Struktur Kode

### Dashboard
| File | Fungsi |
|---|---|
| `dashboard/app.py` | Main Streamlit app (3 menu, layout, CSS, logika UI) |
| `dashboard/ai_analysis.py` | Analisis LLM via OpenAI-compatible API |

### Sumber
| File | Fungsi |
|---|---|
| `src/predict.py` | Model loading, what-if scenario, klasifikasi, koefisien elastisitas |
| `src/utils.py` | Konstanta path, threshold, default parameter |
| `src/feature_engineering.py` | Persiapan fitur untuk model (encoding, scaling) |
| `src/data_pipeline.py` | Pipeline data (load, clean, lag, rolling, ratio) |
| `src/train_model.py` | Training XGBoost + grid search + SHAP |

### Konfigurasi
| File | Fungsi |
|---|---|
| `.env` | API key, base URL, model name untuk AI |
| `requirements.txt` | Python dependencies |
| `.gitignore` | Ignore __pycache__, .env |

---

## 5. Alur Simulasi

1. **Input**: User menggeser 5 slider + pilih provinsi + tahun
2. **Proses**: 
   - Hitung delta dari nilai default untuk APBN/KUR/TKD
   - Kalikan delta × koefisien elastisitas
   - Tambahkan efek ke PDRB asli
   - Update KURS dan PENDUDUK_RB langsung
   - Hitung ulang PDRB per kapita, PDRBKAP_USD, fitur lag/rolling
3. **Prediksi**: Model XGBoost memprediksi pertumbuhan PDRB (G_PDRB_IDR)
4. **Klasifikasi**: Bandingkan PDRBKAP_USD dengan threshold World Bank
5. **Prediksi Trap**: Ekstrapolasi linear untuk estimasi tahun capai kelas berikutnya

---

## 6. Pengembangan Selanjutnya (Backlog)

- [ ] Data real-time APBN/KUR/TKD per provinsi
- [ ] Multi-year simulasi (bukan hanya 1 tahun)
- [ ] Export hasil simulasi ke PDF/CSV
- [ ] Perbandingan antar provinsi dalam satu tampilan
- [ ] Sensitivity analysis (tornado chart)
- [ ] Dark mode toggle
