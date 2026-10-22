# Semantic Search & LLM Application: Asisten FAQ E-Commerce

Prototype asisten customer service yang menjawab pertanyaan pelanggan e-commerce berdasarkan knowledge base FAQ. Pipeline-nya: **semantic search (Gemini Embeddings + FAISS)** dan **LLM (Gemini)**.

> Assignment *Semantic Search & LLM Application*, AI Engineering Bootcamp, oleh Fazle Bimo.

## 1. Problem Statement

### Objective
Tim customer service e-commerce menerima ratusan pertanyaan repetitif setiap hari seputar pemesanan, pengiriman, pembayaran, dan pengembalian barang. Prototype ini bertujuan untuk:
1. **Menemukan entri FAQ yang paling relevan** terhadap pertanyaan pelanggan, **meskipun pelanggan tidak memakai kata-kata yang sama** dengan FAQ (misalnya "duit saya kapan balik?" → FAQ tentang *refund*).
2. **Menyusun jawaban yang natural** dengan LLM berdasarkan FAQ yang ditemukan, sebagai fondasi sebelum dikembangkan menjadi RAG system penuh.

### Knowledge base
| | |
|---|---|
| Sumber | `data/ecommerce_faq_knowledge_base.csv`, FAQ e-commerce yang dibagikan pada assignment (tidak diubah; hanya dibaca) |
| Jumlah entri | 59 FAQ, tanpa missing value dan tanpa duplikat |
| Kolom | `faq_id` (int, unik), `category`, `question`, `answer` |
| Kategori (8) | Pemesanan (9), Pengiriman (9), Pembayaran (8), Pengembalian (8), Akun (7), Produk (7), Promo (6), Customer Service (5) |
| Panjang teks | question ±55 karakter, answer ±128 karakter (maks 181) |
| Bahasa | Indonesia |

### Batasan & asumsi
- Sistem **hanya menjawab berdasarkan FAQ yang tersedia**. Jika informasi tidak ada, sistem harus mengatakannya ("Maaf, informasi tersebut belum tersedia di FAQ kami.") dan mengarahkan ke customer service. Sistem tidak boleh menebak dari pengetahuan umum LLM.
- **1 FAQ = 1 unit retrievable** (question + answer digabung). Tidak ada chunking karena setiap entri sudah pendek.
- Sistem **tidak punya akses ke data pesanan/akun** pengguna, sehingga pertanyaan seperti "pesanan saya #123 sampai mana?" di luar cakupan.
- Satu pertanyaan per query (single-turn), tanpa memori percakapan.
- Pertanyaan pelanggan diasumsikan berbahasa Indonesia (formal maupun informal/slang).

## 2. Alur Sistem

```
            ┌──────────────────────── offline (sekali) ────────────────────────┐
 CSV FAQ ──►│ INGEST ──► EMBED (RETRIEVAL_DOCUMENT) ──► STORE (FAISS + .faiss) │
            └──────────────────────────────────────────────────────────────────┘
                                                             │
 Query ──► EMBED (RETRIEVAL_QUERY) ──► RETRIEVE top-k ◄──────┘
                                           │
                                           ▼
                     PROMPT (Persona · Task · Context · Format) ──► GENERATE (Gemini) ──► Jawaban + [FAQ #id]
```

| Tahap | Implementasi | Lokasi di notebook |
|---|---|---|
| **Ingest** | `pandas` membaca CSV, memeriksa struktur (kolom, missing, duplikat, distribusi kategori, panjang teks), lalu membentuk satu teks per entri: `Kategori: … / Pertanyaan: … / Jawaban: …` | Section 1–2 |
| **Embed** | Gemini Embeddings API `gemini-embedding-001` (3072 dimensi). Dokumen memakai `task_type=RETRIEVAL_DOCUMENT` dan query memakai `RETRIEVAL_QUERY` (asymmetric retrieval). Embedding dikirim per batch, dinormalisasi L2, dan shape-nya diverifikasi `(59, 3072)` | Section 3 |
| **Store** | FAISS `IndexIDMap(IndexFlatIP)`. Karena vektor ternormalisasi, inner product = **cosine similarity**, dan ID vektor = `faq_id`. Jumlah vektor diverifikasi `ntotal == 59`. Index disimpan ke `vector_store/faq_index.faiss` beserta `index_metadata.json` | Section 4 |
| **Retrieve** | `semantic_search(query, top_k=3)` meng-embed query, menjalankan `index.search`, lalu mengembalikan FAQ + skor similarity | Section 5 |
| **Generate** | `build_prompt()` menyusun prompt terstruktur. `generate_answer()` menjalankan query → retrieval → prompt → Gemini `gemini-3.8-flash` (temperature 0.2) | Section 6–7 |

### Desain prompt
- **Persona** (lewat `system_instruction` dan section `### PERSONA`): "Asisten CS" yang ramah, empatik, ringkas, dan tidak punya akses ke data pesanan.
- **Task**: jawab **hanya** dari CONTEXT; jangan menambah fakta, angka, atau kebijakan dari pengetahuan umum; abaikan FAQ yang tidak relevan (karena retrieval selalu mengembalikan top-k); jawab sebagian jika hanya sebagian yang didukung; jika tidak ada, mulai dengan kalimat fallback baku dan sarankan menghubungi CS.
- **Context**: hasil retrieval dibungkus `<faq id=".." kategori=".." similarity="..">`. Skor ikut disertakan sebagai sinyal kepercayaan.
- **Format**: Bahasa Indonesia sopan, maks. 4 kalimat atau langkah bernomor, sitasi `[FAQ #id]`, dan tidak menyebut istilah teknis ("CONTEXT", "skor") ke pengguna.

## 3. Hasil Pengujian

8 query diuji: 2 *direct*, 4 *paraphrase* (bahasa sehari-hari/sinonim), dan 2 *out-of-KB* (1 di luar domain, 1 *near-miss*). Setiap query diberi ground truth FAQ sehingga relevansi bisa diukur.

| # | Tipe | Query | FAQ diharapkan | Top-1 (skor) | Respons LLM |
|---|---|---|---|---|---|
| 1 | direct | Bagaimana cara membatalkan pesanan saya? | #3 | ✅ #3 (0.778) | Benar, sitasi [FAQ #3] |
| 2 | direct | Metode pembayaran apa saja yang bisa saya gunakan? | #17 | ✅ #17 (0.788) | Benar |
| 3 | paraphrase | Duit saya kapan balik kalau barangnya sudah saya retur? | #25 | ✅ #25 (0.786) | Benar (3–7 hari kerja) |
| 4 | paraphrase | Saya nggak ingat kata sandi, gimana caranya bisa masuk lagi? | #32 | ✅ #32 (0.757) | Benar, langkah bernomor |
| 5 | paraphrase | Bisa nyicil nggak kalau belanja di sini? | #20 | ✅ #20 (0.797) | Benar |
| 6 | paraphrase | Kiriman saya udah telat seminggu dari perkiraan, harus ngapain? | #12 | ✅ #12 (0.786), #14 di 0.765 | Benar, empatik |
| 7 | out_of_kb | Apakah platform ini juga menjual tiket pesawat dan hotel? | – | #1 (0.677) | ✅ Fallback, tidak mengarang |
| 8 | out_of_kb | Apakah bisa bayar di tempat (COD) saat barang sampai? | – | #17 (0.691) | ✅ Fallback (terlalu konservatif) |

**Metrik retrieval (6 query in-KB):** Hit@1 = **1.00**, Hit@3 = **1.00**, MRR = **1.000**. Sebagai pembanding, keyword search (overlap token) hanya mendapat Hit@1 = 1/6 dan **gagal di semua query parafrase**.

### Analisis singkat
- **Semantic search menangkap makna.** Query slang tanpa kata kunci FAQ (*duit balik*, *retur*, *kata sandi*, *nyicil*, *telat dari perkiraan*) tetap menemukan FAQ yang benar, dengan skor setara query formal (rata-rata 0.782 vs 0.783).
- **Area rawan: topik bertetangga.** Pada query #6, FAQ #12 dan #14 hanya berselisih 0.021. Peringkat 2–3 sering berisi FAQ yang mirip tetapi bukan jawaban (misalnya *pengembalian* untuk query *pembatalan*). LLM berhasil menyaringnya.
- **Out-of-KB tidak bisa ditolak oleh retrieval sendiri.** FAISS selalu mengembalikan top-k. Skor top-1 out-of-KB lebih rendah (0.68–0.69 vs 0.76–0.80) dan gap top-1/top-2 hampir nol, tetapi marginnya tipis. Saat ini penolakan dilakukan oleh LLM melalui prompt, dan pada 2/2 kasus berhasil.
- **Kualitas jawaban.** Semua jawaban grounded (angka persis dari FAQ), bersitasi, dan nadanya sesuai. Kelemahan: pada *near-miss* COD, LLM menolak total padahal bisa menyebut metode pembayaran yang tersedia menurut FAQ #17.

Analisis lengkap ada di [documentation/retrieval_test_results.md](documentation/retrieval_test_results.md), dan output mentah di [documentation/test_run_raw.json](documentation/test_run_raw.json).

## 4. Keterbatasan Sistem
1. **Belum ada relevance threshold.** Query di luar KB tetap mengirim 3 FAQ "terdekat" ke LLM, sehingga penolakan bergantung pada kepatuhan LLM terhadap prompt.
2. **Margin skor sempit.** Selisih skor antara query relevan dan tidak relevan hanya ±0.07, sehingga threshold perlu dikalibrasi dengan lebih banyak data.
3. **Set uji kecil (8 query).** Hit@1 sempurna belum tentu bertahan pada query multi-intent, typo, atau campuran bahasa.
4. **Topik bertetangga** (keterlambatan paket vs status tidak update vs resi belum bisa dilacak) rawan tertukar. Kandidat perbaikan: re-ranking atau hybrid search (BM25 + embedding).
5. **Satu FAQ per jawaban.** LLM cenderung menjawab dari satu FAQ, padahal kadang lebih lengkap jika menggabungkan dua (misalnya #32 lupa password + #55 akun terkunci).
6. **Redaksi jawaban LLM tidak sepenuhnya deterministik** (temperature 0.2). Hasil retrieval sendiri deterministik.
7. **Bergantung pada API eksternal.** Rate limit (429) dan overload (503) bisa terjadi. Notebook menangani ini dengan retry + exponential backoff.
8. **Tanpa konteks pengguna atau percakapan.** Sistem tidak bisa menjawab hal spesifik pesanan dan tidak mengingat pertanyaan sebelumnya.

## 5. Cara Menjalankan

**Prasyarat:** Python 3.10+ dan Gemini API key ([Google AI Studio](https://aistudio.google.com/apikey)).

```bash
git clone https://github.com/FBimo/assignment-semantic-search-fazle-bimo.git
cd assignment-semantic-search-fazle-bimo

python -m venv .venv
# Windows: .venv\Scripts\activate    |    macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt

# Set API key: salin .env.example menjadi .env lalu isi GEMINI_API_KEY
cp .env.example .env
```

Jalankan notebook:
- **Interaktif:** `jupyter lab notebook/starter_notebook_semantic_search.ipynb`, lalu *Run All*.
- **Headless** (menjalankan seluruh notebook dari awal hingga akhir dan menyimpan output):
  ```bash
  cd notebook
  python -m nbconvert --to notebook --execute --inplace starter_notebook_semantic_search.ipynb
  ```

Notebook dapat dijalankan dari root project maupun dari folder `notebook/` karena path di-resolve otomatis. Model bisa diganti tanpa mengubah kode lewat env var `EMBEDDING_MODEL` dan `LLM_MODEL` (default `gemini-embedding-001` dan `gemini-3.8-flash`).

> Catatan: starter notebook awalnya memakai `text-embedding-004` dan `gemini-2.0-flash`. Kedua model itu sudah tidak tersedia, sehingga diganti dengan model yang aktif.

## 6. Struktur Repository

```
assignment-semantic-search-fazle-bimo/
├── data/
│   └── ecommerce_faq_knowledge_base.csv      # knowledge base asli (tidak diubah)
├── notebook/
│   └── starter_notebook_semantic_search.ipynb # seluruh pipeline + output eksekusi
├── vector_store/
│   ├── faq_index.faiss                       # FAISS index (59 vektor × 3072 dim), dibuat oleh notebook Section 4
│   └── index_metadata.json                   # model, dimensi, metrik, jumlah vektor
├── documentation/
│   ├── retrieval_test_results.md             # hasil uji + analisis lengkap
│   └── test_run_raw.json                     # output mentah pengujian
├── .env.example
├── requirements.txt
└── README.md
```

| Output yang diminta | Lokasi |
|---|---|
| Knowledge base processing code | Notebook Section 1–2 |
| Embedding pipeline code | Notebook Section 3 (`embed_texts`, `get_embedding`) |
| FAISS index + script pembuatannya | `vector_store/faq_index.faiss` + Notebook Section 4 |
| Semantic search function | Notebook Section 5 (`semantic_search`) |
| LLM-powered response function | Notebook Section 6–7 (`build_prompt`, `generate_answer`) |
| Hasil uji relevansi + analisis | Notebook Section 8, `documentation/retrieval_test_results.md`, README §3 |
| requirements.txt | `requirements.txt` |
