# History Pengembangan What-If Simulator

## Ringkasan Perubahan

### 1. Inisialisasi & Eksplorasi
- Eksplorasi struktur proyek `ai-agent-belanja-negara`
- Framework: Python + Streamlit + XGBoost + Plotly
- Menemukan file `dashboard/app.py`, `src/predict.py`, `src/utils.py`, `src/feature_engineering.py`

### 2. Penyempurnaan Tata Letak What-If Simulator
- Membuat layout 3 kolom: Peta (kiri), Slider (tengah), Hasil (kanan)
- Menambahkan peta Indonesia choropleth dengan highlight provinsi terpilih
- Filter provinsi dan tahun
- Small box naratif hasil analisis di bawah peta

### 3. Penambahan Slider Parameter
Menambahkan 5 slider dinamis:
- **Belanja APBN** — persentase terhadap PDRB (0-50%, default 10%)
- **Penyaluran KUR** — persentase terhadap PDRB (0-50%, default 5%)
- **Transfer Daerah** — persentase terhadap PDRB (0-50%, default 15%)
- **KURS USD ke Rp** — nilai tukar (5.000-20.000, default data asli)
- **Jumlah Penduduk** — ribuan jiwa (50%-150% dari data asli)

### 4. Integrasi Data Pendukung

#### File: pendukung/dasar analisis.txt
Koefisien elastisitas untuk simulasi:
- Belanja APBN naik 1% → PDRB naik 0.119361%
- Penyaluran KUR naik 1% → PDRB naik 0.319629%
- Transfer Daerah (TKD) naik 1% → PDRB naik 0.172685%

Diimplementasikan di `src/predict.py` sebagai konstanta:
```python
KOEF_APBN = 0.119361
KOEF_KUR = 0.319629
KOEF_TKD = 0.172685
```

#### File: pendukung/indikator.xlsx
Target kenaikan kelas:
| Kelas Saat Ini | Naik Kelas | Target PDRB/kap | Growth Diperlukan |
|---|---|---|---|
| LOW | LOM | $1,146 | 1.86% |
| LOM | UPM | $4,516 | 1.69% |
| UPM | HIGH | $14,005 | 1.70% |

Threshold World Bank:
- LOM = < $905
- UPM = $905 - $3,595
- HIGH = > $3,595

#### File: pendukung/metadata.xlsx
Deskripsi 21 kolom dataset (PROVINSI, REG, TAHUN, PDRB_IDR_MLY, KURS, PDRB_USD, PENDUDUK_RB, PDRBKAP_USD, G_PDRB_IDR, G_KURS, G_PDRB_USD, G_PENDUDUK, G_PDRBKAP_USD, LOM, UPM, HIGH, KLASIFIKASI, KELAS SAAT INI, NAIK KELAS, TARGET_PDRB, G_PDRB)

### 5. Fitur Analisis by System (OpenAI / DeepSeek / SumoPod)
- Membuat `dashboard/ai_analysis.py`
- Integrasi dengan OpenAI-compatible API (bisa pakai OpenAI, DeepSeek, SumoPod)
- Konfigurasi via file `.env`:
  ```
  OPENAI_API_KEY=sk-xxx
  OPENAI_BASE_URL=https://api.deepseek.com/v1
  OPENAI_MODEL=deepseek-chat
  ```
- Prompt analisis dalam Bahasa Indonesia dengan struktur:
  1. Ringkasan Eksekutif
  2. Analisis Dampak
  3. Klasifikasi Ekonomi
  4. Rekomendasi

### 6. Logika Prediksi Economic Trap
Formula ekstrapolasi linear:
```
gap = UPM_THRESHOLD (3595) - PDRB/kap saat ini
tahun_prediksi = (gap / delta_pdrbkap) + 1
```

Tiga kondisi tampilan:
| Klasifikasi | Delta | Tampilan |
|---|---|---|
| LOM | Positif | "Prediksi Capai Upper-Middle Income: X tahun" |
| UPM | Positif | "Prediksi Capai High Income: X tahun" |
| LOM | Negatif/nol | "Tidak terprediksi — berpotensi terjebak" |
| UPM/HIGH | Negatif/nol | "Stabil — berada di kategori X" |

### 7. Redesign CSS & Tampilan
- `param-card` — slider container dengan glass effect + accent color
- `result-card` — card hasil dengan padding ringan
- `glass-narrative` — box naratif dengan border-left accent
- `class-badge` — badge klasifikasi (LOM merah, UPM kuning, HIGH hijau)
- `info-strip` — strip data provinsi
- Slider premium — thumb bulat 18px gradien emas, value badge

### 8. Fitur Reset Hasil
- Saat provinsi atau tahun diubah, seluruh hasil simulasi dan analisis by system otomatis di-reset
- Menggunakan `on_change` callback `clear_ws_results()`

### 9. Perbaikan Performa
- Peta di-cache di `session_state` (hanya render ulang saat provinsi/tahun berubah)
- Slider dibungkus `st.form()` (tidak trigger rerun saat digeser)
- Hapus blok kode duplikat

### 10. Perbaikan Bug
- Fix `ImportError: TARGET_NAIK_KELAS` — ganti inline dict
- Fix `NameError: name 'r' is not defined`
- Fix `KeyError: ws_result` — tambah pengecekan session_state
- Fix `st.form_submit_button` hilang
- Fix prediksi economic trap untuk klasifikasi UPM/HIGH
- Fix slider penduduk persentase ke ribuan jiwa

### 11. Perbaikan Prediksi Economic Trap (Final)
- **Masalah**: Prediksi hanya menampilkan "Bebas" jika sudah UPM/HIGH, dan "> 200 tahun" jika delta negatif
- **Solusi**: Menggunakan hierarki 3 tingkat untuk annual delta:
  1. Delta dari simulasi (jika > 0)
  2. Rata-rata growth historis PDRB/kap 5 tahun terakhir
  3. Fallback 1% per tahun
- **Formula final**:
  ```
  annual_delta = max(delta_simulasi, pdrbkap * growth_historis)
  tahun_prediksi = (target_threshold - pdrbkap_saat_ini) / annual_delta + 1
  ```
- **Target**: 1 tingkat di atas klasifikasi **awal** (bukan setelah simulasi)
  - LOW → Lower-Middle Income ($905)
  - LOM → Upper-Middle Income ($3,595)
  - UPM → High Income ($11,115)
- **Hasil**: Semua kondisi menampilkan angka pasti tahun (tidak ada "Tidak terprediksi" atau "> 200 tahun")
  - Default NTT: delta kecil → **85 tahun** (pakai growth historis)
  - NTT agresif: delta +$948 → **2 tahun** (pakai delta simulasi)
  - NTT negatif: delta -$218 → **110 tahun** (pakai growth historis)

### 12. Penambahan File Dokumentasi
- **`prd.md`**: Product Requirements Document lengkap
- **`history.md`**: Riwayat pengembangan (file ini)

### 13. File yang Diubah
| File | Status | Keterangan |
|---|---|---|
| `src/utils.py` | Diubah | Tambah DATA_PENDUKUNG, TARGET_NAIK_KELAS |
| `src/predict.py` | Diubah | Tambah KOEF_APBN/KOEF_KUR/KOEF_TKD, update rumus PDRB |
| `dashboard/app.py` | Diubah | Layout baru, CSS, form, reset, prediksi trap, formula hierarki |
| `dashboard/ai_analysis.py` | Baru | Modul analisis OpenAI/DeepSeek |
| `requirements.txt` | Diubah | Tambah python-dotenv, openai |
| `.env` | Baru | Konfigurasi API key |
| `.gitignore` | Diubah | Tambah .env |
| `prd.md` | Baru | Product Requirements Document |
| `history.md` | Baru | Dokumentasi ini |
