# Proyek: Layanan Klasifikasi Teks Zero-Shot di Node-RED

Proyek ini mengimplementasikan layanan klasifikasi teks *zero-shot* (nol-tembak) secara *real-time* di dalam Node-RED. Dengan menggunakan *library* `@xenova/transformers.js`, *flow* ini mampu menjalankan model *machine learning* canggih langsung di *server* Node-RED, memungkinkan analisis teks tanpa bergantung pada API eksternal.

## 📝 Deskripsi Perubahan

Implementasi ini mencakup beberapa komponen utama untuk memungkinkan inferensi model Transformer langsung di dalam Node-RED:

1.  **Konfigurasi `setting.js`:**
    * Menambahkan `process`, `path`, dan `transformers` (wrapper kustom) ke `functionGlobalContext` agar dapat diakses di dalam *function node*.
    * Menambahkan `global.self = global;` sebagai *polyfill* yang mungkin diperlukan oleh beberapa *library*.
    * Meningkatkan `functionTimeout` menjadi 300000 ms (5 menit) untuk memberi waktu yang cukup bagi model untuk dimuat saat *startup*.

2.  **Logika Node Klasifikasi (`Zero-Shot Classifier`):**
    * Menggunakan `pipeline` dari `transformers.js` untuk memuat model klasifikasi *zero-shot* (`Xenova/distilbert-base-uncased-mnli`).
    * **Model Lokal:** Mengatur `env.localModelPath` ke `./models` dan memaksa `env.allowRemoteModels = false` untuk memastikan model dimuat dari *filesystem* lokal, bukan diunduh.
    * **Caching Model:** Model yang sudah dimuat disimpan dalam *context* node (`context.set('zero_shot_classifier', classifier)`) untuk menghindari proses *loading* ulang pada setiap *request*.
    * **Input:** Menerima `msg.payload.text` (teks yang akan diklasifikasi) dan `msg.payload.labels` (array *string* untuk kategori).
    * **Error Handling:** Dilengkapi dengan penanganan *error* yang robust untuk kegagalan saat memuat model atau selama proses klasifikasi.
    * **Status Node:** Memberikan *feedback* visual di editor Node-RED (Loading, Ready, Error).

3.  **Struktur Flow & API (Berdasarkan Gambar):**
    * **API Sinkron:** `POST /classify` untuk klasifikasi *on-demand* secara langsung.
    * **API Asinkron (Worker):**
        * `POST /analyze-async`: Menerima *request*, membuat *job*, dan mengembalikan `jobId`.
        * `Flow 2: BACKGROUND WORKER`: Memproses *job* secara terpisah.
        * `GET /analyze-status/:jobId`: API untuk mengecek status pekerjaan.
    * **API Asinkron (MQTT):**
        * `POST /analyze-async-mqtt`: Menerima *request*, membuat *job*, dan mem-*publish* ke *topic* MQTT (`actions/analyze/new`).
        * *Subscriber* pada *topic* yang sama akan menjalankan *worker* untuk analisis.

## ✅ TODO (Tugas Selesai)

-   [x] Modifikasi `setting.js` untuk *global context* dan *timeout*.
-   [x] Buat skrip *wrapper* `lib/transformers-wrapper.js` (diasumsikan).
-   [x] Unduh model `Xenova/distilbert-base-uncased-mnli` ke direktori `.node-red/models/Xenova/`.
-   [x] Implementasikan logika *function node* untuk *zero-shot classifier* dengan *caching* dan *local model path*.
-   [x] Bangun *flow* Node-RED dengan *endpoint* HTTP dan MQTT.
-   [x] Implementasikan *flow* *worker* asinkron untuk menangani tugas analisis yang berat.
-   [x] Konfigurasi Git *repository* dan *access token* untuk *push* ke *remote*.

## 📂 File yang Diubah/Ditambahkan

* `setting.js` (Dimodifikasi)
* `flows.json` (Dimodifikasi - berisi semua *node* dan *flow* dari gambar)
* `lib/transformers-wrapper.js` (Baru/Dimodifikasi - *file* yang memuat `transformers.js`)
* `models/Xenova/distilbert-base-uncased-mnli/...` (Baru - berisi file model, *harus* ditambahkan ke `.gitignore` jika berukuran besar)
* `package.json` (Dimodifikasi - untuk menambahkan `@xenova/transformers` sebagai dependensi)

---

## 🧠 Arsitektur & Cara Kerja (Visual)

Bagian ini menjelaskan alur kerja visual dari *flow* yang diimplementasikan, melengkapi poin-poin di atas.

![Arsitektur Flow Node-RED](https://i.imgur.com/50967b.png)

### 1. Alur Sinkron & Testing

* **`POST /classify`**: *Endpoint* utama untuk klasifikasi sinkron. *Request* masuk, diproses oleh *node* `Offline Zero-Shot Classifier`, dan respons dikirim kembali secara langsung.
* **`Inject Test Data`**: *Node* untuk melakukan *debugging* cepat pada *classifier* tanpa perlu *request* HTTP.

### 2. Alur Asinkron (Pola Worker & Job)

Ini adalah pola yang ideal untuk tugas yang memakan waktu lama:
1.  **`POST /analyze-async`**: Klien mengirim *request* ke *endpoint* ini.
2.  **`Create Job & Start Process`**: Sebuah *job* baru dibuat dengan status "pending" dan ID unik. `msg` diteruskan (kemungkinan ke alur *worker*).
3.  **`Send Job ID`**: Klien *langsung* menerima respons berisi `jobId` tanpa menunggu analisis selesai.
4.  **`Flow 2: BACKGROUND WORKER`**: Alur terpisah (dapat berjalan di *thread* lain atau *instance* terpisah) mengambil *job*, menjalankan `Action Category Analyzer (Worker)`, dan memperbarui hasil *job* di *database/context*.
5.  **`GET /analyze-status/:jobId`**: Klien menggunakan *endpoint* ini untuk mengecek status pekerjaan menggunakan `jobId` yang didapat.

### 3. Alur Asinkron (MQTT)

Pola alternatif yang menggunakan *message broker*:
1.  **`POST /analyze-async-mqtt`**: *Endpoint* menerima *request* dan membuat *job*.
2.  **`Publish to.../new`**: *Job* dipublikasikan ke *topic* MQTT `actions/analyze/new`.
3.  **`Subscribe to.../new`**: Sebuah *worker* yang "mendengarkan" *topic* ini akan mengambil *job*, memprosesnya, dan mempublikasikan hasilnya ke `actions/analyze/completed`.

---

## 💡 Penjelasan: Cara Kerja Zero-Shot Classification

Kemampuan *zero-shot* bukanlah sihir; ini adalah cara cerdas memanfaatkan model yang telah dilatih untuk tugas lain, yaitu **Natural Language Inference (NLI)**.

Model yang kita gunakan adalah `Xenova/distilbert-base-uncased-mnli`. Bagian **`mnli`** menunjukkan bahwa model ini telah di-***fine-tuning*** pada *dataset* Multi-Genre Natural Language Inference.

1.  **Tugas NLI:** Model NLI dilatih untuk menerima dua kalimat: sebuah **premis** (pernyataan) dan sebuah **hipotesis** (dugaan). Model kemudian menentukan hubungan antara keduanya:
    * **Entailment (Terkandung):** Premis *mendukung* hipotesis.
    * **Contradiction (Bertentangan):** Premis *bertentangan* dengan hipotesis.
    * **Neutral (Netral):** Tidak ada hubungan.

2.  **Penerapan di Zero-Shot:** Kita "menipu" model NLI ini untuk melakukan klasifikasi.
    * **Teks Input** Anda dijadikan sebagai **Premis**.
        * Contoh Premis: `"Barang ini rusak dan tidak berfungsi."`
    * **Setiap Label** yang Anda berikan diubah menjadi **Hipotesis** menggunakan sebuah *template* (secara *default*: `"This text is about {label}."`).
        * Hipotesis 1: `"This text is about komplain."`
        * Hipotesis 2: `"This text is about pujian."`
        * Hipotesis 3: `"This text is about pertanyaan."`

3.  **Proses Klasifikasi:** Model akan membandingkan Premis dengan *setiap* Hipotesis dan menghitung skor **"entailment"** (seberapa besar Premis mendukung Hipotesis).

4.  **Hasil:** *Label* yang hipotesisnya memiliki skor *entailment* tertinggi akan dipilih sebagai pemenangnya.
    * Dalam contoh di atas, Hipotesis 1 (`"komplain"`) akan mendapatkan skor *entailment* yang jauh lebih tinggi.

## 🛠️ Instalasi & Konfigurasi

Untuk menjalankan proyek ini, ikuti langkah-langkah berikut di dalam direktori `.node-red` Anda (mis. `C:\Users\ryzen\.node-red`):

### 1. Instal Dependensi

Pastikan Anda telah menginstal *library* `transformers.js`:
```bash
npm install @xenova/transformers