# Laporan Pengujian Relevansi Semantic Search & Respon LLM

Dokumen ini menyajikan hasil pengujian komprehensif terhadap sistem **Semantic Search & LLM Integration** pada *knowledge base* FAQ e-commerce.

---

## 1. Metodologi Pengujian

Pengujian dilakukan untuk menguji dua pilar utama sistem:
1. **Performa Semantic Retrieval**: Seberapa baik representasi vektor `gemini-embedding-001` dan pencarian berbasis Euclidean Distance (`IndexFlatL2` FAISS) dalam memetakan intensi query pengguna ke entri FAQ yang tepat (terutama saat pengguna menggunakan sinonim atau variasi bahasa).
2. **Kualitas Generasi LLM**: Seberapa patuh LLM (`gemini-3.5-flash`) terhadap batasan konteks FAQ (*grounded generation*), kemampuannya merangkai jawaban yang ramah dan solutif, serta keandalannya dalam menolak pertanyaan di luar domain (*out-of-scope rejection*).

Sistem diuji dengan **5 variasi query representatif**:
* **2 Query Langsung (Direct Match)**: Query yang memiliki padanan kata dan struktur kalimat yang sangat dekat dengan entri FAQ asli.
* **2 Query Parafrase / Sinonim (Paraphrased Match)**: Query yang menggunakan sinonim, istilah alternatif, atau menggabungkan beberapa konsep tanpa menyebut kata kunci persis pada FAQ.
* **1 Query di Luar Cakupan (Out-of-Scope / Not in FAQ)**: Pertanyaan yang topiknya tidak ada di dalam *knowledge base* e-commerce.

---

## 2. Hasil Pengujian Rinci

### Pengujian #1: Direct FAQ Query (Kategori: Pemesanan)

* **Query Pengguna**: `"Bagaimana cara membatalkan pesanan yang sudah dibuat?"`
* **Tipe Pengujian**: Direct Match
* **Hasil Retrieval (Top-3)**:

| Rank | FAQ ID | Kategori | Pertanyaan FAQ | L2 Distance | Status Relevansi |
| :---: | :---: | :---: | :--- | :---: | :---: |
| **1** | **3** | **Pemesanan** | **Bagaimana cara membatalkan pesanan yang sudah dibuat?** | **0.2392** | **Sangat Relevan (Exact)** |
| 2 | 24 | Pengembalian | Bagaimana prosedur pengembalian barang yang tidak sesuai? | 0.5069 | Kurang Relevan |
| 3 | 44 | Produk | Bagaimana jika produk yang saya pesan ternyata kehabisan stok? | 0.5349 | Tidak Relevan |

* **Jawaban LLM**:
  > *"Halo! Untuk membatalkan pesanan yang sudah dibuat, Anda dapat mengajukannya melalui halaman **'Pesanan Saya'** selama status pesanan Anda masih **'Diproses'**.*
  >
  > *Mohon diperhatikan bahwa jika status pesanan sudah berubah menjadi **'Dikirim'**, maka pembatalan pesanan sudah tidak dapat dilakukan.*
  >
  > *Semoga informasi ini membantu! Silakan hubungi saya kembali jika ada hal lain yang ingin ditanyakan."*

* **Evaluasi & Analisis**:
  * **Retrieval**: Sangat presisi. Model embedding menempatkan entri target di Rank 1 dengan skor jarak L2 yang sangat rendah (**0.2392**). Selisih jarak (*margin*) ke Rank 2 cukup lebar ($\Delta = 0.2677$), menandakan keyakinan sistem yang tinggi.
  * **Generasi LLM**: Respon sangat akurat, langsung menjawab inti syarat pembatalan (status 'Diproses' vs 'Dikirim'), serta diformat dengan *tone* customer service yang ramah dan sopan.

---

### Pengujian #2: Direct FAQ Query (Kategori: Pengiriman)

* **Query Pengguna**: `"Berapa lama estimasi waktu pengiriman barang untuk area luar pulau?"`
* **Tipe Pengujian**: Direct Match
* **Hasil Retrieval (Top-3)**:

| Rank | FAQ ID | Kategori | Pertanyaan FAQ | L2 Distance | Status Relevansi |
| :---: | :---: | :---: | :--- | :---: | :---: |
| **1** | **9** | **Pengiriman** | **Berapa lama estimasi waktu pengiriman barang?** | **0.2212** | **Sangat Relevan (Target)** |
| 2 | 12 | Pengiriman | Bagaimana jika paket saya belum sampai melebihi estimasi? | 0.5051 | Terkait Konteks |
| 3 | 14 | Pengiriman | Kenapa status pengiriman saya tidak berubah selama beberapa hari? | 0.5708 | Terkait Konteks |

* **Jawaban LLM**:
  > *"Halo! Untuk estimasi pengiriman barang ke area luar pulau, umumnya membutuhkan waktu **5-10 hari kerja**, tergantung pada jasa kirim yang Anda pilih.*
  >
  > *Ada lagi yang bisa saya bantu?"*

* **Evaluasi & Analisis**:
  * **Retrieval**: Berhasil mengambil FAQ ID 9 di Rank 1 dengan jarak L2 terendah (**0.2212**). Seluruh Top-3 berasal dari kategori yang sama (*Pengiriman*).
  * **Generasi LLM**: LLM menyaring informasi spesifik yang diminta pengguna (luar pulau: 5-10 hari kerja) tanpa mengikutsertakan informasi dalam kota yang tidak relevan dengan pertanyaan, menunjukkan pemahaman kontekstual yang tajam.

---

### Pengujian #3: Paraphrased Query (Kategori: Pembayaran - Cicilan/Kredit)

* **Query Pengguna**: `"Apakah toko menerima opsi pembayaran bertahap atau sistem kredit kartu?"`
* **Tipe Pengujian**: Paraphrased / Semantic Matching (Tanpa kata "cicilan")
* **Hasil Retrieval (Top-3)**:

| Rank | FAQ ID | Kategori | Pertanyaan FAQ | L2 Distance | Status Relevansi |
| :---: | :---: | :---: | :--- | :---: | :---: |
| **1** | **20** | **Pembayaran** | **Apakah saya bisa membayar dengan cicilan?** | **0.4325** | **Sangat Relevan (Target)** |
| 2 | 17 | Pembayaran | Metode pembayaran apa saja yang tersedia? | 0.6342 | Terkait Konteks |
| 3 | 18 | Pembayaran | Apakah ada biaya tambahan untuk pembayaran menggunakan kartu kredit? | 0.6356 | Terkait Konteks |

* **Jawaban LLM**:
  > *"Halo! Terima kasih telah menghubungi kami.*
  >
  > *Ya, kami mendukung pembayaran menggunakan kartu kredit. Untuk opsi pembayaran bertahap (cicilan), fitur tersebut tersedia untuk transaksi dengan kartu kredit tertentu pada nominal minimal yang ditentukan oleh pihak bank.*
  >
  > *Jika ada hal lain yang ingin ditanyakan, silakan beri tahu kami. Semoga informasi ini membantu!"*

* **Evaluasi & Analisis**:
  * **Retrieval**: Meskipun pengguna tidak menyebut kata *"cicilan"*, melainkan *"pembayaran bertahap"* dan *"sistem kredit kartu"*, semantic search berhasil mengenali kesamaan konsep dan menempatkan FAQ ID 20 di Rank 1 dengan skor **0.4325**. Pendekatan berbasis keyword matching murni (seperti TF-IDF atau BM25) kemungkinan besar akan gagal pada skenario ini.
  * **Generasi LLM**: LLM mengonfirmasi dukungan kartu kredit dan secara eksplisit menghubungkannya dengan fitur cicilan sesuai FAQ konteks.

---

### Pengujian #4: Paraphrased Query (Kategori: Pengembalian & Komplain Rusak)

* **Query Pengguna**: `"Barang pesanan yang saya terima ternyata rusak dan cacat, bagaimana prosedur komplain dan retur?"`
* **Tipe Pengujian**: Paraphrased / Multi-intent Concept
* **Hasil Retrieval (Top-3)**:

| Rank | FAQ ID | Kategori | Pertanyaan FAQ | L2 Distance | Status Relevansi |
| :---: | :---: | :---: | :--- | :---: | :---: |
| **1** | **24** | **Pengembalian** | **Bagaimana prosedur pengembalian barang yang tidak sesuai?** | **0.3526** | **Sangat Relevan** |
| **2** | **54** | **Pengembalian** | **Saya menerima barang yang salah, apa yang harus saya lakukan?** | **0.3650** | **Sangat Relevan** |
| **3** | **16** | **Pengiriman** | **Apa yang harus dilakukan jika paket diterima dalam kondisi rusak?** | **0.3674** | **Sangat Relevan** |

* **Jawaban LLM**:
  > *"Halo Kak, mohon maaf atas ketidaknyamanan yang dialami terkait pesanan tersebut.*
  >
  > *Jika Kakak menerima barang dalam kondisi rusak, berikut adalah prosedur komplain dan pengembalian yang dapat dilakukan:*
  >
  > 1. ***Ambil Bukti:*** *Segera ambil foto dan/atau video kondisi paket/barang saat Kakak terima.*
  > 2. ***Laporkan Komplain:*** *Laporkan masalah tersebut melalui menu komplain dalam waktu maksimal **2x24 jam** sejak barang diterima.*
  > 3. ***Ajukan Pengembalian:*** *Kakak dapat mengajukan pengembalian melalui menu **'Pesanan Saya'** dengan memilih alasan yang sesuai dan mengunggah bukti foto barang tersebut, maksimal **3 hari** sejak barang diterima.*
  >
  > *Semoga informasi ini membantu. Jika ada pertanyaan lebih lanjut, jangan ragu untuk memberi tahu kami!"*

* **Evaluasi & Analisis**:
  * **Retrieval**: Performa luar biasa. Seluruh Top-3 hasil retrieval memiliki jarak yang sangat rapat (**0.3526 s.d. 0.3674**) dan semuanya saling melengkapi: FAQ 16 mencakup instruksi penanganan barang rusak (foto/video dalam 2x24 jam), sedangkan FAQ 24 mencakup menu pengajuan komplain (menu 'Pesanan Saya' maksimal 3 hari).
  * **Generasi LLM**: LLM menyintesis ketiga konteks tersebut menjadi panduan bertahap (langkah 1-2-3) yang terstruktur, kohesif, dan mudah dipahami oleh pengguna.

---

### Pengujian #5: Out-of-Scope Query (Tidak Ada di FAQ)

* **Query Pengguna**: `"Apakah kantor e-commerce ini sedang membuka lowongan magang untuk posisi AI Engineer?"`
* **Tipe Pengujian**: Out-of-Scope / Rejection Handling
* **Hasil Retrieval (Top-3)**:

| Rank | FAQ ID | Kategori | Pertanyaan FAQ | L2 Distance | Status Relevansi |
| :---: | :---: | :---: | :--- | :---: | :---: |
| 1 | 45 | Produk | Apakah saya bisa request custom produk kepada penjual? | 0.8598 | Tidak Relevan |
| 2 | 51 | Customer Service | Apakah tersedia layanan pengaduan 24 jam? | 0.8622 | Tidak Relevan |
| 3 | 50 | Customer Service | Bagaimana cara melaporkan penjual yang menipu? | 0.8734 | Tidak Relevan |

* **Jawaban LLM**:
  > *"Halo! Terima kasih telah menghubungi kami.*
  >
  > *Mohon maaf, saat ini saya belum memiliki informasi mengenai lowongan magang untuk posisi AI Engineer di dalam sistem knowledge base kami.*
  >
  > *Untuk mendapatkan informasi lebih lanjut terkait lowongan pekerjaan atau magang, kami menyarankan Anda untuk menghubungi tim customer service kami melalui saluran bantuan resmi. Terima kasih dan semoga harimu menyenangkan!"*

* **Evaluasi & Analisis**:
  * **Retrieval**: Nilai jarak L2 berada pada rentang **0.8598 – 0.8734**, jauh lebih tinggi daripada query in-scope (0.22 – 0.43). Ini adalah karakteristik tipikal representasi vektor ketika tidak ada pasangan semantik yang cocok.
  * **Generasi LLM**: Berkat instruksi batasan ketat pada prompt (*persona, task, context, rules*), model **tidak mengalami halusinasi** (tidak mengarang lowongan magang). Model secara jujur menyatakan bahwa informasi tidak tersedia di knowledge base dan mengarahkan pengguna ke saluran resmi.

---

## 3. Matriks Perbandingan & Kesimpulan

| Kategori Pengujian | Nilai Jarak L2 (Top-1) | Akurasi Retrieval | Kualitas Respon LLM | Kepatuhan Konteks |
| :--- | :---: | :---: | :---: | :---: |
| **Direct Match** | 0.22 – 0.24 | 100% (Exact match) | Sangat Akurat & Ringkas | 100% Grounded |
| **Paraphrased Match** | 0.35 – 0.43 | 100% (Intent match) | Komprehensif & Solutif | 100% Grounded |
| **Out-of-Scope** | > 0.85 | N/A (Far neighbors) | Penolakan Santun & Anti-Halusinasi | 100% Grounded |

### Insight Rekomendasi (Distance Thresholding)
Hasil pengujian empiris membuktikan bahwa nilai L2 distance dapat digunakan sebagai indikator keandalan retrieval:
- **L2 Distance $\le 0.50$**: Tingkat relevansi tinggi. Konteks dapat langsung disuplai ke LLM.
- **$0.50 <$ L2 Distance $\le 0.70$**: Tingkat relevansi menengah / parsial.
- **L2 Distance $> 0.75$**: Konteks di luar cakupan (*out-of-scope*).

Pada arsitektur produksi, disarankan menerapkan **Gatekeeper Threshold** pada nilai $0.70$ untuk langsung mengembalikan jawaban fallback otomatis tanpa perlu memanggil LLM API, sehingga menghemat biaya API dan memangkas latensi sistem.
