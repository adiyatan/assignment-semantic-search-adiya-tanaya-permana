# Prototype Aplikasi Semantic Search & LLM untuk Customer Service E-Commerce

> **Individual Assignment - AI Engineering Bootcamp**  
> **Topik:** Advanced Prompt Engineering & Context Management (Semantic Search & LLM Integration)

Proyek ini membangun prototype asisten cerdas berbasis *Semantic Search* dan *Large Language Model* (LLM) untuk menjawab pertanyaan pelanggan e-commerce secara otomatis, akurat, dan ramah berdasarkan *knowledge base* FAQ internal.

---

## 1. Problem Statement & Objective

### Latar Belakang & Objective
Tim *customer service* (CS) sebuah platform e-commerce menghadapi lonjakan ratusan pertanyaan repetitif setiap harinya seputar status pemesanan, kebijakan pengiriman, metode pembayaran, hingga klaim pengembalian barang. Pencarian tradisional berbasis pencocokan kata kunci (*keyword search*) kerap gagal ketika pelanggan menggunakan bahasa sehari-hari, sinonim, atau struktur kalimat yang tidak identik dengan teks FAQ.

**Objective Proyek:**
1. Membangun sistem pencarian semantik (*semantic search*) yang dapat memahami makna/intensi pertanyaan pengguna meskipun kata-kata yang digunakan berbeda dengan teks FAQ asli.
2. Mengintegrasikan hasil pencarian semantik ke dalam prompt terstruktur (*Persona, Task, Context, Format/Rules*) untuk LLM.
3. Menghasilkan jawaban yang santun, akurat, kontekstual (*grounded*), serta bebas dari halusinasi (mampu menolak secara sopan apabila informasi tidak terdapat di *knowledge base*).

### Sumber & Struktur Knowledge Base
* **Sumber Data**: `ecommerce_faq_knowledge_base.csv`
* **Jumlah Entri**: 59 entri FAQ
* **Struktur Kolom**:
  * `faq_id`: Identitas unik entri FAQ (integer 1-59).
  * `category`: Kategori topik layanan e-commerce.
  * `question`: Pertanyaan resmi yang sering diajukan pelanggan.
  * `answer`: Jawaban resmi dan kebijakan platform.
* **Distribusi Kategori**:
  * *Pemesanan* (9 entri), *Pengiriman* (9 entri), *Pembayaran* (8 entri), *Pengembalian* (8 entri), *Akun* (7 entri), *Produk* (7 entri), *Promo* (6 entri), *Customer Service* (5 entri).
  * Tidak terdapat missing value pada dataset (100% lengkap).

### Batasan & Asumsi
1. **Asumsi Grounding**: Sistem diasumsikan sebagai *closed-domain assistant*. Jawaban yang diberikan kepada pelanggan harus bersumber secara eksklusif dari informasi FAQ yang berhasil di-retrieve, bukan pengetahuan umum internal LLM.
2. **Unit Retrievable**: Karena setiap entri FAQ relatif ringkas dan mandiri, teknik *chunking* dokumen panjang tidak diterapkan; setiap entri digabungkan menjadi unit tunggal representatif (`Q: ... A: ...`).
3. **Bahasa**: Seluruh basis pengetahuan dan interaksi pengguna menggunakan Bahasa Indonesia.

---

## 2. Arsitektur & Alur Sistem (End-to-End Pipeline)

Sistem dibangun dengan alur 5 tahapan utama:

```
[ Knowledge Base CSV ]
          │
          ▼ (Ingest)
[ Unit Retrievable: Q + A ]
          │
          ▼ (Embed: Gemini Embeddings API - gemini-embedding-001)
[ 59 Vektor Embedding (Dim: 3072) ]
          │
          ▼ (Store)
[ FAISS Vector Database (IndexFlatL2) ]
          │
  Query Pengguna
          │
          ▼ (Retrieve)
[ Top-k Most Relevant FAQ Contexts ]
          │
          ▼ (Generate: Gemini 3.5 Flash + Structured Prompt)
[ Respon AI Ramah, Akurat & Terverifikasi ]
```

1. **Ingest (Data Preprocessing)**:
   Membaca file `ecommerce_faq_knowledge_base.csv`. Tiap baris digabungkan menjadi format `Q: {question} A: {answer}` agar representasi vektor merangkum hubungan antara pertanyaan dan jawabannya.
2. **Embed (Vector Generation)**:
   Mengubah unit retrievable menjadi vektor densitas tinggi menggunakan Google Gemini API model `gemini-embedding-001` (menghasilkan representasi vektor 3072 dimensi, `float32`).
3. **Store (Vector Database Indexing)**:
   Vektor disimpan ke dalam index vector database berbasis FAISS (`faiss.IndexFlatL2`). Index diverifikasi memuat tepat 59 vektor dan disimpan secara persisten ke disk sebagai `faiss_index.bin`.
4. **Retrieve (Semantic Search)**:
   Saat query masuk, query di-embed menjadi vektor, lalu dilakukan pencarian *nearest neighbor* terdekat menggunakan jarak Euclidean ($L_2$ distance) untuk mengambil $Top\text{-}k$ entri FAQ paling relevan.
5. **Generate (Prompt Engineering & LLM Integration)**:
   Konteks hasil retrieval disisipkan ke dalam template prompt terstruktur (*Persona, Task, Context, Format/Rules*), lalu dikirim ke model LLM `gemini-3.5-flash` dengan konfigurasi `temperature=0.2` untuk menghasilkan jawaban yang presisi dan konsisten.

---

## 3. Struktur Direktori Proyek

```
semantic-search-assignment/
├── data/
│   └── ecommerce_faq_knowledge_base.csv     # Knowledge base FAQ asli
├── notebook/
│   └── starter_notebook_semantic_search.ipynb # Notebook utama dengan eksekusi & output lengkap
├── documentation/
│   └── retrieval_test_results.md            # Laporan pengujian 5 skenario query secara mendalam
├── faiss_index.bin                          # File database index FAISS yang telah di-generate
├── embeddings.npy                           # Backup array numpy embedding vektor
├── run_pipeline.py                          # Script standalone untuk menjalankan pipeline end-to-end
├── create_notebook.py                       # Script generator notebook terotomatisasi
├── requirements.txt                         # Daftar dependensi pustaka Python
└── README.md                                # Dokumentasi komprehensif proyek
```

---

## 4. Hasil Pengujian Relevansi & Kualitas Respon AI

Sistem diuji dengan 5 skenario query yang mencakup pertanyaan langsung, variasi sinonim/parafrase, dan pertanyaan di luar domain (*out-of-scope*):

| # | Tipe Query | Query Pengguna | Top Retrieved FAQ | L2 Distance | Kualitas Jawaban LLM |
| :-: | :--- | :--- | :--- | :-: | :--- |
| **1** | **Direct Match** | *"Bagaimana cara membatalkan pesanan yang sudah dibuat?"* | FAQ ID 3 (Pemesanan) | **0.2392** | **Sangat Akurat**: Menjelaskan pembatalan via 'Pesanan Saya' saat status 'Diproses'. |
| **2** | **Direct Match** | *"Berapa lama estimasi waktu pengiriman barang untuk area luar pulau?"* | FAQ ID 9 (Pengiriman) | **0.2212** | **Sangat Presisi**: Mengambil angka 5-10 hari kerja khusus luar pulau. |
| **3** | **Paraphrased Match** | *"Apakah toko menerima opsi pembayaran bertahap atau sistem kredit kartu?"* | FAQ ID 20 (Pembayaran - Cicilan) | **0.4325** | **Pemahaman Semantik Tinggi**: Mengenali bahwa pembayaran bertahap = cicilan kartu kredit bank. |
| **4** | **Paraphrased Match** | *"Barang pesanan yang saya terima ternyata rusak dan cacat, bagaimana prosedur komplain dan retur?"* | FAQ ID 24, 54, 16 (Retur & Rusak) | **0.3526 – 0.3674** | **Sintesis Kontekstual Sempurna**: Merangkum panduan 3 langkah komplain foto/video (2x24 jam) & retur (3 hari). |
| **5** | **Out-of-Scope** | *"Apakah kantor e-commerce ini sedang membuka lowongan magang untuk posisi AI Engineer?"* | FAQ ID 45, 51 (Tidak Relevan) | **> 0.85** | **Anti-Halusinasi Sukses**: Jujur menyatakan tidak memiliki info tersebut dan mengarahkan ke CS resmi. |

> Detail lengkap pengujian per query beserta log konteks dan analisis dapat dilihat pada [`documentation/retrieval_test_results.md`](documentation/retrieval_test_results.md).

---

## 5. Keterbatasan Sistem & Peluang Pengembangan

1. **Jarak Tanpa Ambang Batas Otomatis (Distance Thresholding)**:
   FAISS `IndexFlatL2` selalu mengembalikan $k$ tetangga terdekat berapapun nilai jaraknya. Pada query *out-of-scope*, dokumen terdekat tetap dikirim ke LLM (jarak $> 0.85$). Meskipun LLM berhasil menolak menjawab berkat batasan prompt, menerapkan *distance threshold* (misal batas cut-off $\le 0.65$) akan lebih efisien dalam memfilter query sebelum memanggil API LLM.
2. **Ketergantungan Representasi Dense Murni**:
   Model embedding semantik sangat unggul dalam menangkap makna/parafrase, tetapi untuk pencarian nomor seri spesifik, kode voucher tertentu, atau SKU produk acak, pendekatan *Hybrid Search* (kombinasi Dense Embedding + Sparse BM25) akan memberikan akurasi yang lebih seimbang.
3. **Status Sistem Masih Prototype Satu Putaran**:
   Sistem saat ini belum mengelola *multi-turn conversational memory* (riwayat percakapan sebelumnya).

---

## 6. Panduan Menjalankan Proyek

### Prasyarat Lingkungan
* Python versi 3.10 atau yang lebih baru.
* Google Gemini API Key aktif.

### Langkah Instalasi & Eksekusi

1. **Clone Repository & Siapkan Virtual Environment**:
   ```bash
   git clone <link-repo-anda>
   cd semantic-search-assignment
   python -m venv venv
   # Di Windows:
   venv\Scripts\activate
   # Di Linux/macOS:
   source venv/bin/activate
   ```

2. **Instal Dependensi**:
   ```bash
   pip install -r requirements.txt
   ```

3. **Set API Key**:
   Pastikan environment variable `GEMINI_API_KEY` telah disetel:
   ```bash
   # Di Windows PowerShell:
   $env:GEMINI_API_KEY="YOUR_GEMINI_API_KEY"

   # Di Windows Command Prompt:
   set GEMINI_API_KEY=YOUR_GEMINI_API_KEY

   # Di Linux/macOS:
   export GEMINI_API_KEY="YOUR_GEMINI_API_KEY"
   ```

4. **Menjalankan Melalui Jupyter Notebook**:
   Buka VS Code atau jalankan Jupyter Lab / Notebook:
   ```bash
   jupyter notebook notebook/starter_notebook_semantic_search.ipynb
   ```
   Seluruh cell dapat dieksekusi secara berurutan (*Run All*) tanpa error dari awal hingga akhir.

5. **Menjalankan Melalui Script Python Standalone**:
   Anda juga dapat menjalankan pengujian pipeline end-to-end secara langsung melalui terminal:
   ```bash
   python run_pipeline.py
   ```
