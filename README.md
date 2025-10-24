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