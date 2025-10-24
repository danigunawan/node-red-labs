# Proyek: Layanan Klasifikasi Teks Zero-Shot di Node-RED

Proyek ini mengimplementasikan layanan klasifikasi teks *zero-shot* (nol-tembak) secara *real-time* di dalam Node-RED. Dengan menggunakan *library* `@xenova/transformers.js`, *flow* ini mampu menjalankan model *machine learning* canggih langsung di *server* Node-RED, memungkinkan analisis teks tanpa bergantung pada API eksternal.

## ✨ Fitur Utama

* **Klasifikasi Teks Zero-Shot:** Mampu mengklasifikasikan teks ke dalam kategori (label) yang disediakan secara *on-the-fly*, bahkan jika model belum pernah dilatih pada label tersebut.
* **Inferensi Lokal:** Menjalankan model `Xenova/distilbert-base-uncased-mnli` langsung dari *filesystem* lokal (`.node-red/models`), menghilangkan ketergantungan pada internet dan latensi API.
* **API Ganda:** Menyediakan *endpoint* HTTP untuk pemrosesan sinkron (`/classify`) dan asinkron (`/analyze-async`) untuk tugas yang lebih lama.
* **Pola Worker Asinkron:** Menggunakan alur *background worker* (Flow 2) untuk memproses tugas berat tanpa memblokir *request* utama, lengkap dengan *endpoint* untuk mengecek status *job*.
* **Integrasi MQTT:** Menawarkan *endpoint* asinkron alternatif (`/analyze-async-mqtt`) yang berkomunikasi melalui *topics* MQTT.
* **Caching Model:** Memuat model ke dalam memori (node *context*) saat pertama kali dijalankan untuk inferensi yang sangat cepat pada *request* berikutnya.

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

## 🧠 Penjelasan Arsitektur Flow

Arsitektur *flow* ini dibagi menjadi beberapa alur logika untuk menangani berbagai jenis *request* dan pola eksekusi:

### 1. Alur Sinkron & Testing

* **`POST /classify`**: Ini adalah *endpoint* utama untuk klasifikasi sinkron. Sebuah *request* HTTP POST masuk, diproses secara langsung oleh *node* `Offline Zero-Shot Classifier`, dan respons yang berisi hasil klasifikasi dikirim kembali dalam *request* yang sama.
* **`Inject Test Data`**: *Node* `Inject` ini digunakan untuk melakukan *debugging* dan pengujian cepat pada *node* `Zero-Shot Classifier` tanpa perlu mengirim *request* HTTP dari luar.

### 2. Alur Asinkron (Pola Worker & Job)

Ini adalah pola yang ideal untuk tugas analisis yang memakan waktu lama, agar tidak membuat klien menunggu:

1.  **`POST /analyze-async`** (Flow 1): Klien mengirim *request* ke *endpoint* ini.
2.  **`Create Job & Start Process`**: *Node* ini segera membuat *job* baru (misalnya di *database* atau *flow context*) dengan status "pending" dan menghasilkan ID unik (`jobId`).
3.  **`Send Job ID`**: Klien *langsung* menerima respons HTTP 200 yang berisi `jobId` tersebut. *Request* klien selesai di sini.
4.  **`Flow 2: BACKGROUND WORKER`**: Secara terpisah, *flow* ini (yang dipicu oleh *node* "Create Job") mengambil tugasnya. *Node* `Action Category Analyzer (Worker)` melakukan pemrosesan yang berat.
5.  **`Update Job Result`**: Setelah selesai, hasilnya disimpan, dan status *job* diperbarui menjadi "completed".
6.  **`GET /analyze-status/:jobId`** (Flow 3): Klien dapat kapan saja memanggil *endpoint* ini menggunakan `jobId` yang didapatnya untuk mengecek status dan mengambil hasil jika sudah "completed".

### 3. Alur Asinkron (MQTT)

Ini adalah pola asinkron alternatif yang menggunakan *message broker* (MQTT) untuk komunikasi antar *service*:

1.  **`POST /analyze-async-mqtt`**: Mirip dengan alur sebelumnya, klien mengirim *request* ke *endpoint* ini.
2.  **`Create Job & Prepare MQTT`**: *Job* dibuat dan datanya diformat menjadi pesan MQTT.
3.  **`Publish to actions/analyze/new`**: Pesan *job* dipublikasikan ke *topic* MQTT `actions/analyze/new`. Klien kemudian dikirimi `jobId`.
4.  **`Subscribe to actions/analyze/new`**: Di bagian lain (bisa di *flow* yang sama atau *instance* Node-RED yang berbeda), sebuah *node* *subscriber* "mendengarkan" *topic* ini.
5.  **`Action Category Analyzer (Worker)`**: Saat pesan baru masuk, *worker* mengambilnya, memproses analisis, dan kemudian memublikasikan hasilnya ke *topic* lain seperti `actions/analyze/completed`.

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


2. Unduh Model
Unduh file model Xenova/distilbert-base-uncased-mnli dari Hugging Face. Letakkan semua file (termasuk config.json, model.onnx, tokenizer.json, dll.) di dalam direktori .node-red Anda dengan struktur berikut:

.node-red/
├── models/
│   └── Xenova/
│       └── distilbert-base-uncased-mnli/
│           ├── config.json
│           ├── model.onnx
│           ├── tokenizer.json
│           └── ... (file lainnya)
├── node_modules/
├── flows.json
└── setting.js
3. Konfigurasi setting.js
Edit file setting.js Anda untuk memuat library ke dalam global context dan mengatur timeout.

module.exports = {
    // ... (konfigurasi lainnya)

    // Timeout fungsi ditingkatkan untuk memberi waktu model dimuat
    functionTimeout: 300000, // 5 menit

    functionGlobalContext: {
        // Polyfill yang mungkin diperlukan
        global: global, 
        
        // Modul yang diekspos ke Function Node
        process: process,
        path: require('path'),
        
        // Wrapper untuk library transformers
        // (Diasumsikan Anda memiliki file ini atau memuatnya langsung)
        transformers: require('./lib/transformers-wrapper') 
        // ATAU: transformers: require('@xenova/transformers')
    },

    // ... (konfigurasi lainnya)
}
4. Impor Flow
Impor flows.json yang berisi alur dari gambar di atas ke editor Node-RED Anda.
5. Jalankan
Jalankan Node-RED dari direktori yang benar:

cd C:\Users\ryzen\.node-red
node-red
🚀 Penggunaan (API Endpoints)
POST /classify (Sinkron)
Mengklasifikasikan teks secara langsung dan mengembalikan hasilnya.
Request Body:

{
    "text": "Saya sangat kecewa dengan kualitas produk ini.",
    "labels": ["positif", "negatif", "netral"]
}
Response Sukses (200 OK):

{
    "sequence": "Saya sangat kecewa dengan kualitas produk ini.",
    "labels": ["negatif", "netral", "positif"],
    "scores": [0.987, 0.008, 0.005]
}
POST /analyze-async (Asinkron)
Memulai job klasifikasi dan langsung mengembalikan jobId.
Request Body:

{
    "text": "Analisis ini membutuhkan waktu yang lama.",
    "labels": ["teknis", "penjualan", "keuangan"]
}
Response Sukses (200 OK):

{
    "jobId": "job_abc123456"
}
GET /analyze-status/:jobId
Mengecek status dari job yang sedang berjalan atau sudah selesai.
Request:GET /analyze-status/job_abc123456
Response (Contoh Selesai):

{
    "status": "completed",
    "jobId": "job_abc123456",
    "result": {
        "sequence": "Analisis ini membutuhkan waktu yang lama.",
        "labels": ["teknis", "keuangan", "penjualan"],
        "scores": [0.85, 0.10, 0.05]
    }
}
📄 Kode Inti: Node "Zero-Shot Classifier"
Berikut adalah kode JavaScript yang digunakan di dalam function node utama.
<details>
<summary>Klik untuk melihat kode node Zero-Shot Classifier</summary>

// Memuat library dari global context (setting.js)
const { pipeline, env } = await global.get('transformers');
const path = global.get('path');
const get_process = global.get("process");
// Mengatur path model lokal
const cwd = get_process.cwd();
const localModelPath = path.join(cwd, 'models');
node.log(`Mencari model di path: ${localModelPath}`);
// Memaksa library untuk menggunakan model lokal, bukan mengunduh
env.localModelPath = localModelPath;
env.allowRemoteModels = false;

node.log("Library 'transformers' berhasil dimuat dari global context.");
// Caching: Cek apakah model sudah dimuat di context
let classifier = context.get('zero_shot_classifier');
if (!classifier) {
    node.status({ fill: "blue", shape: "dot", text: "Loading classifier model..." });
    try {
        // Memuat pipeline model. Ini hanya terjadi sekali.
        classifier = await pipeline('zero-shot-classification', 'Xenova/distilbert-base-uncased-mnli');
        
        // Simpan model yang sudah dimuat ke context untuk request berikutnya
        context.set('zero_shot_classifier', classifier);
        node.status({ fill: "green", shape: "dot", text: "Classifier ready" });
    } catch (err) {
        node.error("Gagal memuat model klasifikasi", err);
        node.status({ fill: "red", shape: "dot", text: "Model load failed" });
        msg.payload = {
            error: "Initialization failed on server",
            details: {
                message: err.message,
                name: err.name
            }
        };
        return msg;
    }
}
// Validasi input
const text = msg.payload.text;
const labels = msg.payload.labels;
if (!text || typeof text !== 'string' || !Array.isArray(labels) || labels.length === 0) {
    node.warn('Input tidak valid. msg.payload harus berisi "text" (string) dan "labels" (array of string).');
    msg.payload = { error: 'Input tidak valid. Pastikan msg.payload memiliki format { "text": "...", "labels": ["...", "..."] }' };
    return msg;
}

node.status({ fill: "blue", shape: "dot", text: "Classifying text..." });
// Menjalankan klasifikasi
try {
    const output = await classifier(text, labels);
    msg.payload = output;
    node.status({ fill: "green", shape: "dot", text: "Classifier ready" });
} catch (error) {
    node.error("Error selama proses klasifikasi teks", error);
    node.status({ fill: "red", shape: "dot", text: "Classification error" });
    msg.payload = {
        error: "Error during classification",
        details: {
            message: error.message,
            name: error.name
        }
    };
}
return msg;
</details>