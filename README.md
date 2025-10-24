feat: Implementasi Klasifikasi Teks Zero-Shot

Menambahkan fungsionalitas klasifikasi teks zero-shot di Node-RED menggunakan library `transformers.js` dengan model yang di-hosting secara lokal.

### 📝 Deskripsi Perubahan

Implementasi ini mencakup beberapa komponen utama untuk memungkinkan inferensi model Transformer langsung di dalam Node-RED:

1.  **Konfigurasi `setting.js`:**
    * Menambahkan `process`, `path`, dan `transformers` (wrapper kustom) ke `functionGlobalContext` agar dapat diakses di dalam *function node*.
    * Menambahkan `global.self = global;` sebagai *polyfill* yang mungkin diperlukan oleh beberapa library.
    * Meningkatkan `functionTimeout` menjadi 300000 ms (5 menit) untuk memberi waktu yang cukup bagi model untuk dimuat saat *startup*.

2.  **Logika Node Klasifikasi (`Zero-Shot Classifier`):**
    * Menggunakan `pipeline` dari `transformers.js` untuk memuat model klasifikasi zero-shot (`Xenova/distilbert-base-uncased-mnli`).
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
        * `POST /analyze-async-mqtt`: Menerima *request*, membuat *job*, dan mem-PC*publish* ke *topic* MQTT (`actions/analyze/new`).
        * *Subscriber* pada *topic* yang sama akan menjalankan *worker* untuk analisis.

### ✅ TODO (Tugas Selesai)

-   [x] Modifikasi `setting.js` untuk *global context* dan *timeout*.
-   [x] Buat skrip *wrapper* `lib/transformers-wrapper.js` (diasumsikan).
-   [x] Unduh model `Xenova/distilbert-base-uncased-mnli` ke direktori `.node-red/models/Xenova/`.
-   [x] Implementasikan logika *function node* untuk *zero-shot classifier* dengan *caching* dan *local model path*.
-   [x] Bangun *flow* Node-RED dengan *endpoint* HTTP dan MQTT.
-   [x] Implementasikan *flow* *worker* asinkron untuk menangani tugas analisis yang berat.
-   [x] Konfigurasi Git *repository* dan *access token* untuk *push* ke *remote*.

### 📂 File yang Diubah/Ditambahkan

* `setting.js` (Dimodifikasi)
* `flows.json` (Dimodifikasi - berisi semua *node* dan *flow* dari gambar)
* `lib/transformers-wrapper.js` (Baru/Dimodifikasi - *file* yang memuat `transformers.js`)
* `models/Xenova/distilbert-base-uncased-mnli/...` (Baru - berisi file model, *harus* ditambahkan ke `.gitignore` jika berukuran besar)
* `package.json` (Dimodifikasi - untuk menambahkan `@xenova/transformers` sebagai dependensi)