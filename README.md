# CardioWave DWT • Deteksi Gelombang P-Q-R-S-T EKG Berbasis DWT db4 (From Scratch)

Proyek ini merupakan implementasi sistem deteksi delineasi gelombang elektrokardiogram (**P-Q-R-S-T**) menggunakan metode **Discrete Wavelet Transform (DWT)** dengan basis **Daubechies 4 (db4)** yang dibangun secara mandiri (_from scratch_, tanpa pustaka komputasi wavelet otomatis).

Aplikasi web interaktif **CardioWave DWT** dirancang dengan antarmuka telemetri klinis modern untuk memproses sinyal EKG digital standar biomedis dari **MIT-BIH Arrhythmia Database (PhysioNet)**, khususnya **Record 122**, serta menguraikan proses dekomposisi multiresolusi dan ekstraksi fitur titik gelombang jantung secara transparan, akurat, dan visual.

![Hasil Deteksi Gelombang P-Q-R-S-T EKG](hasil%20DWT.png)
*Gambar 1: Hasil visualisasi deteksi gelombang P-Q-R-S-T pada Record 122 MIT-BIH Arrhythmia Database.*

---

## 1. Dataset: MIT-BIH Arrhythmia Database (Record 122)

Eksperimen dan pengujian pada proyek ini menggunakan data klinis nyata dari repositori resmi **PhysioNet MIT-BIH Arrhythmia Database**:

* **Sumber Dataset:** [PhysioNet MIT-BIH Arrhythmia Database](https://physionet.org/content/mitdb/)
* **Identitas Rekaman:** **Record 122**
* **Karakteristik Sinyal:**
  * Frekuensi Sampling ($f_s$): **360 Hz** (360 sampel per detik).
  * Saluran Sadapan: Sadapan bipolar **MLII** (*Modified Limb Lead II*) dan sadapan prekordial **V1**.
  * Total Panjang Rekaman: 650.000 sampel ($\approx 30$ menit).

### Tiga Berkas Format PhysioNet yang Digunakan:

1. **`122.hea` (Header File):**  
   Berkas teks ASCII deskriptif yang memuat spesifikasi teknis sinyal: jumlah sinyal (2 channel), frekuensi cuplik (360 Hz), format penyimpanan (Format 212), faktor konversi ADC (*gain* = 200 ADC units/mV), garis dasar (*baseline*), serta nama kanal elektroda (`MLII`, `V1`).
2. **`122.dat` (Signal Data File):**  
   Berkas biner yang menyimpan data amplitudo EKG mentah dengan pengemasan terkompresi *Format 212* (dua sampel 12-bit dikemas dalam 3 byte berselingan).
3. **`122.atr` (Reference Annotation File):**  
   Berkas biner penanda referensi (*ground truth*) yang diverifikasi secara independen oleh setidaknya dua dokter spesialis kardiologi. Berkas ini digunakan untuk menguji akurasi algoritma melalui metrik **Sensitivitas ($Se$)** dan **Nilai Prediktif Positif ($PPV$)**.

---

## 2. Landasan Teori & Referensi Jurnal

Algoritma ini diadaptasi dari metode standar internasional deteksi delineasi gelombang EKG berbasis _wavelet_:

> **J. P. Martínez, R. Almeida, S. Olmos, A. P. Rocha, and P. Laguna**,  
> _"A Wavelet-Based ECG Delineator: Evaluation on Open Databases,"_  
> **IEEE Transactions on Biomedical Engineering**, Vol. 51, No. 4, pp. 570–581, April 2004.

### Karakteristik Matematis DWT db4:

- **Wavelet db4 (Daubechies 4):** Memiliki 4 _vanishing moments_ (8 koefisien filter). Morfologi fungsi wavelet db4 sangat menyerupai gelombang kompleks QRS pada EKG manusia, menjadikannya basis filter yang optimal untuk melokalisasi fitur detak jantung.
- **Algoritma Mallat / Algorithme à Trous:**  
  Berbeda dengan DWT standar yang melakukan decimasi (_downsampling_ $2\downarrow$), algoritma _à trous_ menyisipkan nol (_upsampling_ filter) pada setiap skala $j$. Hal ini mempertahankan laju sampel sinyal di setiap level ($Q_1$ s/d $Q_8$), sehingga koordinat titik puncak gelombang tidak bergeser dari waktu aslinya.

---

## 3. Struktur Filter & Koefisien db4 (Manual)

Koefisien filter LPF (_Low Pass Filter_) $h[n]$ ortogonal db4 didefinisikan secara eksplisit:
```text
h = [0.23037781, 0.71484657, 0.63088077, -0.02798377, -0.18703481, 0.03084138, 0.03288301, -0.01059740]
```

Filter HPF (_High Pass Filter_) $g[n]$ diturunkan melalui rumus _Quadrature Mirror Filter (QMF)_:
```text
g[n] = (-1)^n * h[L - 1 - n]
```

### Peran Rentang Skala Dekomposisi ($Q_j$):

- **$Q_1 - Q_2$ (Frekuensi Sangat Tinggi):** Menampung derau frekuensi tinggi, artefak listrik otot (EMG), dan gangguan listrik jala-jala (50/60 Hz).
- **$Q_3 - Q_4$ (Frekuensi Menengah, $\approx 10 - 45$ Hz):** Menampung energi utama **Kompleks QRS**.
- **$Q_5$ (Frekuensi Rendah-Sedang):** Dominan untuk energi **Gelombang P**.
- **$Q_6$ (Frekuensi Rendah):** Dominan untuk energi **Gelombang T**.
- **$Q_7 - Q_8$ (Frekuensi Sangat Rendah):** Menampung _baseline wander_ (pergeseran garis dasar akibat respirasi atau pergerakan pasien).

---

## 4. Alur Komputasi & Deteksi PQRST (Step-by-Step)

```text
[File .dat / .hea / .atr / Demo]
         │
         ▼
1. Ekstraksi Sinyal Mentah (Format 212 PhysioNet)
         │
         ▼
2. Dekomposisi Multiresolusi DWT (Mallat à trous Q1 - Q8)
         │
         ▼
3. Kompensasi Penundaan Fasa (Group Delay Alignment: T_j + Delay_j = T_max)
         │
         ▼
4. Deteksi Kompleks QRS (Skala Q4 & Q3)
   - Pasangan Modulus Maxima/Minima melewati Ambang ThR
   - Zero-Crossing sebagai penanda letak R
   - Lembah kiri & kanan sebagai penanda Q dan S
         │
         ▼
5. Isolasi & Pemotongan QRS (Linear Interpolation)
         │
         ▼
6. Deteksi Gelombang T (Skala Q6) & Gelombang P (Skala Q5)
   - Pencarian pada jendela waktu sebelum R (Pwin) dan sesudah R (Twin)
   - Zero-Crossing & Refinement dengan sinyal aproksimasi
         │
         ▼
7. Evaluasi Akurasi Klinis (Perbandingan dengan Anotasi Ahli .atr)
   - Sensitivitas (Se) & Positive Predictivity (PPV)
   - Kalkulasi Laju Jantung (BPM), Interval R-R, dan Ekspor Spreadsheet CSV
```

---

## 5. Fitur Antarmuka Modern (CardioWave DWT)

Aplikasi web interaktif [ecg dwt.html](ecg%20dwt.html) telah diperbarui dengan tampilan telemetri klinis mutakhir:

1. **Desain Telemetri & Dukungan Tema Ganda**:
   - Tampilan gelap (*Dark Mode Telemetry*) bernuansa monitor bedside ICU/klinis.
   - Pilihan *Light Mode* untuk pencahayaan laboratorium bersih, lengkap dengan penyimpanan preferensi otomatis.
2. **Clinical KPI Telemetry Ribbon**:
   - **Laju Detak Jantung (BPM):** Dilengkapi diagnosis ritme otomatis (*Sinus Normal 60–100*, *Bradikardia*, atau *Takikardia*).
   - **Total Detak (Beats):** Penghitungan total puncak R dan durasi rekaman sinyal dalam detik.
   - **Interval R-R Rata-rata:** Analisis variabilitas inter-beat (ms).
   - **Akurasi Anotasi (.atr):** Persentase *Sensitivity* ($Se$) dan *Positive Predictive Value* ($PPV$) terhadap anotasi standar PhysioNet.
3. **Fleksibilitas Input Dataset**:
   - **Drag-and-Drop Dropzone:** Unggah seret-lepas berkas rekaman `.hea`, `.dat`, dan `.atr` secara bersamaan dengan chip status hijau aktif.
   - **Tombol Pintasan "Muat 122 Lokal":** Memuat rekaman pasien MIT-BIH `122` dari direktori secara langsung hanya dengan 1 klik.
   - **Sinyal Demo Sintetis:** Uji coba instan tanpa perlu menyediakan file eksternal.
4. **Interactive Waveform Viewport**:
   - Kanvas beresolusi tinggi dengan penskalaan vertikal adaptif tanpa pemotongan amplitudo gelombang.
   - Toolbar navigasi: **Zoom +**, **Zoom -**, dan **Reset Skala**, didukung gesture *mouse wheel* dan *drag pan*.
5. **Tabel Koordinat Gelombang & Ekspor CSV**:
   - Drawer tabel di bawah kanvas menampilkan indeks sampel $P, Q, R, S, T$ per-detak, interval R-R lokal, serta estimasi BPM lokal.
   - Tombol **Unduh CSV** di bar navigasi untuk menyimpan hasil ke file spreadsheet.

---

## 6. 6 Tab Visualisasi Dekomposisi & Analisis

1. **Tab 1 - Masukan:** Visualisasi sinyal EKG mentah (mV) beserta titik anotasi ahli dari `.atr`.
2. **Tab 2 - Mallat Q1–8:** Penguraian 8 skala koefisien wavelet $Q_1$ s/d $Q_8$ yang telah diselaraskan penundaan fasanya ($T_j + Delay_j = T_{\max}$).
3. **Tab 3 - Filter Bank:** Karakteristik filter db4 $h[n]$ (LPF), $g[n]$ (HPF), respon frekuensi magnitude, dan spektrum pita frekuensi filter bank (Hz).
4. **Tab 4 - Zero-Crossing:** Titik potong sumbu nol (*zero-crossing*) pada skala $Q_3$ (Q & S), $Q_4$ (R), $Q_5$ (P), dan $Q_6$ (T).
5. **Tab 5 - Gradien:** Analisis turunan pertama sinyal $dx/dn$ serta pasangan modulus maxima-minima pada skala $W_4$.
6. **Tab 6 - Deteksi PQRST:** Hasil akhir deteksi lengkap dengan *bounding box* pulsa gelombang, penanda titik $P, Q, R, S, T$, nilai akurasi $Se/PPV$, dan tabel koordinat.

---

## 7. Penjelasan Parameter Kontrol

### A. Dataset & Sinyal
- `N Data`: Jumlah sampel sinyal yang diproses (default: 999 sampel $\approx 2.77$ detik pada $fs = 360$ Hz).
- `Mulai (Start)`: Titik awal indeks sampel pembacaan rekaman.
- `Kanal (Channel)`: Saluran sadapan elektroda (0 = sadapan utama MLII).

### B. Ambang & Jendela (Slider + Input)
- `ThR` (0.4): Ambang batas relatif deteksi puncak R pada skala $Q_4$.
- `ThP` (0.15) & `ThT` (0.2): Ambang batas modulus maxima gelombang P dan T.
- `Refr` (200 ms): *Refractory period* untuk mencegah deteksi ganda.
- `Pwin` (300 ms): Jendela pencarian gelombang P sebelum puncak R.
- `Twin` (450 ms): Jendela pencarian gelombang T setelah puncak R.

### C. Filter Bank $Q_j$ (Accordion)
- `T1 - T8` & `Delay1 - Delay8`: Pusat energi impuls respons dan nilai delay kompensasi fasa.
- `Hitung T & Delay Otomatis`: Mengaktifkan perhitungan matematis otomatis db4.

---

## 8. Petunjuk Menjalankan Program

### Cara 1: Menggunakan Aplikasi Web Interaktif
1. Buka file `ecg dwt.html` langsung di browser modern (Google Chrome, Microsoft Edge, Firefox) atau melalui local server (misal: `python -m http.server 8080`).
2. Klik tombol **Sinyal Demo** untuk mencoba simulasi, atau klik **Muat 122 Lokal** untuk langsung menganalisis rekaman pasien MIT-BIH 122.
3. Anda juga dapat memilih file `.hea`, `.dat`, dan `.atr` sendiri melalui area *drag-and-drop*.
4. Klik **Proses Analisis** untuk mengeksekusi delineasi gelombang.
5. Unduh hasil pengujian melalui tombol **Unduh CSV**.

### Cara 2: Menggunakan Jupyter Notebook / Google Colab
1. Buka file `ecg_dwt_pqrst.ipynb` melalui Jupyter Lab / Notebook atau unggah ke Google Colab.
2. Pastikan file `122.hea`, `122.dat`, dan `122.atr` berada pada direktori yang sama atau telah diunggah ke *runtime* Colab.
3. Jalankan sel kode secara berurutan untuk menelaah setiap tahap kalkulasi matematis DWT db4.
