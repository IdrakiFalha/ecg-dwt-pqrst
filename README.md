Berikut adalah versi `README.md` yang telah dirapikan. Tanda-tanda khusus seperti asteris (`*`) dan dolar (`$`) telah dibersihkan atau disesuaikan dengan format Markdown standar agar terlihat lebih bersih, profesional, dan mudah dibaca (baik di editor teks maupun saat dirender).

---

# Proyek Pengolahan Sinyal EKG Berbasis Discrete Wavelet Transform (DWT db4) From Scratch

Proyek ini merupakan implementasi deteksi gelombang elektrokardiogram (**P-Q-R-S-T**) menggunakan metode **Discrete Wavelet Transform (DWT)** dengan basis **Daubechies 4 (db4)** yang dibangun secara mandiri (_from scratch_, tanpa pustaka komputasi wavelet otomatis).

Program ini dirancang untuk memproses sinyal EKG digital standar biomedis (format PhysioNet MIT-BIH Arrhythmia Database) dan menguraikan proses dekomposisi multiresolusi serta ekstraksi fitur titik gelombang jantung secara transparan dan visual.

---

## 1. Landasan Teori & Referensi Jurnal

Algoritma ini diadaptasi dari metode standar internasional deteksi delineasi gelombang EKG berbasis _wavelet_:

> **J. P. Martínez, R. Almeida, S. Olmos, A. P. Rocha, and P. Laguna**,
> _"A Wavelet-Based ECG Delineator: Evaluation on Open Databases,"_
> **IEEE Transactions on Biomedical Engineering**, Vol. 51, No. 4, pp. 570–581, April 2004.

### Karakteristik Matematis DWT db4:

- **Wavelet db4 (Daubechies 4):** Memiliki 4 _vanishing moments_ (8 koefisien filter). Bentuk gelombang fungsi wavelet db4 sangat menyerupai morfologi kompleks QRS pada EKG manusia, menjadikannya basis filter yang optimal untuk menyaring dan melokalisasi fitur detak jantung.
- **Algoritma Mallat / Algorithme à Trous:**
  Berbeda dengan DWT standar yang melakukan decimasi (_downsampling_ $2\downarrow$), algoritma _à trous_ menyisipkan nol (_upsampling_ filter) pada setiap skala $j$. Hal ini mempertahankan laju sampel sinyal di setiap level ($Q_1$ s/d $Q_8$), sehingga koordinat titik puncak gelombang tidak bergeser dari waktu aslinya.

---

## 2. Struktur Filter & Koefisien db4 (Manual)

Koefisien filter LPF (_Low Pass Filter_) $h[n]$ ortogonal db4 didefinisikan secara eksplisit:
`h = [0.23037781, 0.71484657, 0.63088077, -0.02798377, -0.18703481, 0.03084138, 0.03288301, -0.01059740]`

Filter HPF (_High Pass Filter_) $g[n]$ diturunkan melalui rumus _Quadrature Mirror Filter (QMF)_:
`g[n] = (-1)^n * h[L - 1 - n]`

### Peran Rentang Skala Dekomposisi ($Q_j$):

- **$Q_1 - Q_2$ (Frekuensi Sangat Tinggi):** Menampung derau frekuensi tinggi, artefak listrik otot (EMG), dan gangguan kabel/jaringan PLN.
- **$Q_3 - Q_4$ (Frekuensi Menengah, $\approx 10 - 45$ Hz):** Menampung energi utama **Kompleks QRS**.
- **$Q_5$ (Frekuensi Rendah-Sedang):** Dominan untuk energi **Gelombang P**.
- **$Q_6$ (Frekuensi Rendah):** Dominan untuk energi **Gelombang T**.
- **$Q_7 - Q_8$ (Frekuensi Sangat Rendah):** Menampung _baseline wander_ (pergeseran garis dasar akibat pernapasan atau pergerakan tubuh pasien).

---

## 3. Alur Komputasi & Deteksi PQRST (Step-by-Step)

```text
[File .dat / .hea / .atr]
         │
         ▼
1. Ekstraksi Sinyal Mentah (Format 212 PhysioNet)
         │
         ▼
2. Dekomposisi Multiresolusi DWT (Mallat a trous Q1 - Q8)
         │
         ▼
3. Kompensasi Penundaan Fasa (Group Delay Aligment: T_j + Delay_j = T_max)
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
   - Zero-Crossing & Refinement
         │
         ▼
7. Evaluasi Akurasi (Perbandingan dengan Anotasi Dokter .atr)
   - Sensitivitas (Se) & Positive Predictivity (PPV)
   - Perhitungan Heart Rate klinis (BPM & interval RR)

```

---

## 4. Penjelasan Parameter Kontrol

### A. Bagian PROCESS

- `File`: Memuat ketiga file wajib PhysioNet (`.hea`, `.dat`, `.atr`).
- `Record`: Menampilkan identitas pasien dan saluran (_lead_) aktif (misal `122 (MLII, V1)`).
- `NData`: Jumlah sampel data yang dianalisis dalam satu waktu (misal 999 sampel $\approx 2,77$ detik, 3600 sampel $\approx 10$ detik).
- `Start`: Titik awal pembacaan data rekaman (misal `Start = 626400` untuk menit ke-29).
- `Channel`: Pilihan sadapan elektroda (0 = sadapan utama MLII).

### B. Bagian T / DELAY

- `T1 - T8`: Pusat massa energi respon impuls filter $Q_j$.
- `Delay1 - Delay8`: Kompensasi pergeseran fasa ($Delay_j = T_{\max} - T_j$) agar hasil deteksi di setiap level tetap sinkron dengan sinyal aslinya.
- `T/Delay otomatis`: Mengaktifkan perhitungan pergeseran fasa secara otomatis sesuai rumus matematis filter bank.

### C. Bagian AMBANG & JENDELA

- `ThR` (0.4): Ambang batas relatif deteksi puncak R pada skala $Q_4$.
- `ThP` (0.15) & `ThT` (0.2): Ambang batas deteksi puncak gelombang P dan T.
- `Refr` (200 ms): _Refractory period_ (periode istirahat fisiologis jantung agar tidak terjadi deteksi ganda).
- `Pwin` (300 ms): Jendela waktu pencarian gelombang P ke arah kiri sebelum puncak R.
- `Twin` (450 ms): Jendela waktu pencarian gelombang T ke arah kanan setelah puncak R.

---

## 5. Fitur Antarmuka & Tab Visualisasi

1. **Tab Prosses:** Menampilkan sinyal mentah EKG bersama titik anotasi dokter (`.atr`).
2. **Tab Mallat:** Menampilkan dekomposisi 8 level koefisien $Q_1$ s/d $Q_8$.
3. **Tab Filter Bank:** Menampilkan grafik impuls $h[n]$, $g[n]$, respon frekuensi, dan spektrum ekuivalen filter bank (Hz).
4. **Tab Zero-cross:** Menampilkan titik potong nol (_zero-crossing_) pada skala $Q_3$, $Q_4$, $Q_5$, dan $Q_6$.
5. **Tab Gradient:** Menampilkan turunan pertama $dx/dn$ dan pasangan modulus maxima/minima.
6. **Tab Detection:** Visualisasi akhir lengkap dengan tanda bounding pulsa, label titik P-Q-R-S-T, akurasi $Se/PPV$, interval RR (ms), serta panel status detak jantung (**BPM**).

---

## 6. Petunjuk Menjalankan Program

### Menjalankan Program Web Interaktif:

1. Buka file `ecg dwt.html` di sembarang browser modern (Google Chrome, Microsoft Edge, Mozilla Firefox).
2. Di bagian **File**, klik tombol **Choose Files**, lalu pilih ketiga file sekaligus: `122.hea`, `122.dat`, dan `122.atr`.
3. Klik tombol biru **Action**.
4. Gunakan fitur _scroll_ mouse untuk memperbesar (_zoom_), seret mouse untuk menggeser (_pan_), dan klik ganda untuk mengembalikan tampilan (_reset zoom_).
5. Tersedia tombol **Reset Parameter Default** untuk mengembalikan semua nilai ambang ke konfigurasi standar.

### Menjalankan di Google Colab / Jupyter Notebook:

1. Buka Google Colab dan buka notebook `ecg_dwt_pqrst.ipynb`.
2. Unggah file `122.hea`, `122.dat`, dan `122.atr` ke session storage Colab.
3. Jalankan sel kode secara berurutan untuk memverifikasi proses pembacaan file dan eksekusi fungsi DWT.
