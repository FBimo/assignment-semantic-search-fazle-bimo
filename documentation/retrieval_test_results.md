# Hasil Uji Relevansi Retrieval & Response AI

Konfigurasi: embedding `gemini-embedding-001` (3072 dimensi, `RETRIEVAL_DOCUMENT` / `RETRIEVAL_QUERY`), FAISS `IndexFlatIP` dengan vektor dinormalisasi (skor = cosine similarity), `top_k = 3`, LLM `gemini-3.8-flash` (temperature 0.2).
Output mentah lengkap (top-3 dan jawaban LLM per query) ada di [`test_run_raw.json`](test_run_raw.json) dan di output cell Section 8 notebook.

## Ringkasan

| Metrik | Nilai |
|---|---|
| Jumlah query uji | 8 (2 direct, 4 paraphrase, 2 out-of-KB) |
| Hit@1 (6 query yang ada jawabannya) | **1.00** (6/6) |
| Hit@3 | **1.00** |
| MRR | **1.000** |
| LLM menolak menjawab saat informasi tidak ada | 2/2 |
| LLM menjawab (tidak menolak) saat informasi ada | 6/6 |
| Keyword baseline Hit@1 (6 query yang sama) | 1/6. Gagal di **semua** query parafrase |

## Detail per query

| # | Tipe | Query | Ekspektasi | Top-3 (FAQ id: skor) | Hasil |
|---|---|---|---|---|---|
| 1 | direct | Bagaimana cara membatalkan pesanan saya? | #3 | **3**: 0.778 · 24: 0.718 · 44: 0.698 | ✅ Akurat |
| 2 | direct | Metode pembayaran apa saja yang bisa saya gunakan? | #17 | **17**: 0.788 · 22: 0.727 · 20: 0.700 | ✅ Akurat |
| 3 | paraphrase | Duit saya kapan balik kalau barangnya sudah saya retur? | #25 | **25**: 0.786 · 27: 0.732 · 24: 0.729 | ✅ Akurat |
| 4 | paraphrase | Saya nggak ingat kata sandi, gimana caranya bisa masuk lagi? | #32 | **32**: 0.757 · 55: 0.701 · 31: 0.652 | ✅ Akurat |
| 5 | paraphrase | Bisa nyicil nggak kalau belanja di sini? | #20 | **20**: 0.797 · 22: 0.700 · 17: 0.695 | ✅ Akurat |
| 6 | paraphrase | Kiriman saya udah telat seminggu dari perkiraan, harus ngapain? | #12 | **12**: 0.786 · 14: 0.765 · 52: 0.722 | ✅ Akurat, margin tipis |
| 7 | out_of_kb | Apakah platform ini juga menjual tiket pesawat dan hotel? | – | 1: 0.677 · 17: 0.667 · 2: 0.659 | ✅ LLM menolak |
| 8 | out_of_kb (near-miss) | Apakah bisa bayar di tempat (COD) saat barang sampai? | – | 17: 0.691 · 11: 0.688 · 1: 0.686 | ✅ LLM menolak |

### Jawaban LLM (ringkas)

1. Pembatalan lewat 'Pesanan Saya' selama status 'Diproses'; tidak bisa setelah 'Dikirim'. `[FAQ #3]`
2. Transfer bank, kartu kredit/debit, e-wallet, minimarket rekanan. `[FAQ #17]`
3. Refund diproses 3–7 hari kerja setelah barang retur diterima & diverifikasi penjual. `[FAQ #25]`
4. Langkah bernomor: klik 'Lupa Password' → masukkan email → ikuti instruksi di email. `[FAQ #32]`
5. Bisa, untuk kartu kredit tertentu dengan nominal minimal dari bank. `[FAQ #20]`
6. Karena sudah lewat >3 hari dari estimasi, hubungi CS dengan menyertakan nomor resi. `[FAQ #12]`
7. "Maaf, informasi tersebut belum tersedia di FAQ kami." + saran menghubungi CS.
8. "Maaf, informasi tersebut belum tersedia di FAQ kami." + saran menghubungi CS untuk memastikan ketersediaan COD.

## Analisis

### 1. Retrieval akurat, termasuk pada parafrase
Keenam query yang punya jawaban menempatkan FAQ yang benar di peringkat 1. Query parafrase sengaja tidak memakai kata kunci FAQ:

| Query | Kata di query | Kata di FAQ |
|---|---|---|
| #3 | *duit … balik*, *retur* | *dana refund*, *pengembalian* |
| #4 | *nggak ingat kata sandi*, *masuk* | *lupa password*, *login* |
| #5 | *nyicil* | *cicilan* |
| #6 | *kiriman … telat seminggu dari perkiraan* | *paket belum sampai melebihi estimasi* |

Keyword baseline (overlap token) **gagal di keempat query ini**. Pada #3 dan #6 overlap-nya 0 sehingga baseline praktis memilih acak, dan pada #4 baseline memilih FAQ #41 (cashback) karena kata "masuk". Ini bukti bahwa sistem menangkap makna dan tidak hanya mencocokkan kata. Skor top-1 query parafrase (rata-rata 0.782) setara dengan query direct (0.783), jadi bahasa informal tidak menurunkan kualitas retrieval.

### 2. Di mana retrieval kurang tajam
- **#6 hampir tertukar.** Selisih FAQ #12 (0.786) dengan #14 *"status pengiriman tidak berubah selama beberapa hari"* (0.765) hanya **0.021**. Keduanya memang topik yang bertetangga. Dengan formulasi query sedikit berbeda (misalnya "paket nggak gerak-gerak"), urutannya bisa terbalik dan jawabannya menjadi "tunggu 1x24 jam" alih-alih "hubungi CS". Untungnya FAQ #14 tetap masuk context, sehingga LLM masih bisa memilih yang tepat.
- **Peringkat 2–3 sering noise.** Untuk #1 (batal pesanan), peringkat 2 adalah FAQ #24 *pengembalian barang*. Secara semantik dekat ("membatalkan transaksi"), tetapi bukan jawaban. LLM berhasil mengabaikannya berkat aturan 2 di prompt.
- **Pada #4, FAQ #55 (akun tidak bisa login) di peringkat 2 sebenarnya juga relevan.** LLM hanya menjawab dari #32. Ini tepat untuk "lupa kata sandi", tetapi menunjukkan bahwa jawaban terbaik kadang perlu menggabungkan beberapa FAQ.

### 3. Query di luar knowledge base: retrieval tidak bisa "bilang tidak tahu"
FAISS selalu mengembalikan top-k. Untuk #7 dan #8 yang dikembalikan adalah FAQ generik (*cara pemesanan*, *metode pembayaran*) dengan skor 0.66–0.69. Ada dua sinyal yang membedakannya dari query yang ada jawabannya:

| Sinyal | Query in-KB | Query out-of-KB |
|---|---|---|
| Skor top-1 | 0.757 – 0.797 | 0.677 – 0.691 |
| Gap top-1 vs top-2 | 0.021 – 0.097 | 0.003 – 0.011 |

Pemisahnya ada, tetapi margin skor absolut hanya ~0.07. Threshold di sekitar 0.72 akan bekerja pada 8 query ini, tetapi terlalu sedikit data untuk dikalibrasi dengan yakin. Karena itu prototype ini **menyerahkan keputusan "tidak tahu" ke LLM** melalui prompt, dan skor similarity ikut disertakan di context sebagai petunjuk.

### 4. Kualitas jawaban LLM
- **Grounded dan faktual.** Semua angka di jawaban (3–7 hari kerja, >3 hari, 1x24 jam) persis dari FAQ. Tidak ada informasi tambahan dari pengetahuan umum, padahal untuk COD dan tiket pesawat LLM jelas "tahu" jawaban umumnya.
- **Format konsisten.** Setiap jawaban memakai sitasi `[FAQ #id]`. Prosedur (#4) otomatis menjadi langkah bernomor, dan nadanya empatik pada keluhan (#6: "mohon maaf atas ketidaknyamanan…").
- **Inferensi ringan yang masih valid.** Pada #6 LLM menghubungkan "telat seminggu" dengan aturan "melebihi estimasi lebih dari 3 hari". Penalaran ini benar dan tetap didukung context.
- **Near-miss COD (#8) dijawab terlalu konservatif.** FAQ #17 mendaftar metode pembayaran tanpa COD. Jawaban yang ideal: *"Metode yang tersedia adalah transfer bank, kartu, e-wallet, dan minimarket; COD tidak tercantum di FAQ kami."* LLM memilih menolak sepenuhnya. Ini aman (tidak mengklaim "COD tidak tersedia" tanpa dasar eksplisit), tetapi kurang membantu. Aturan 3 (jawab sebagian) di prompt belum cukup kuat untuk kasus "informasi tidak disebut" vs "informasi dinyatakan tidak ada".

## Keterbatasan & ide perbaikan
1. **Tidak ada relevance threshold.** Out-of-KB hanya ditangani oleh LLM. Perbaikan: kalibrasi threshold skor/gap dengan set query yang lebih besar, lalu lewati LLM (langsung fallback) jika di bawah threshold. Ini juga menghemat biaya.
2. **Set uji kecil (8 query).** Hit@1 = 1.00 belum bisa digeneralisasi. Perlu ±50–100 query berlabel, termasuk query multi-intent ("batal pesanan dan refund ke mana?") dan typo.
3. **Topik bertetangga** (#12 vs #14 vs #52; #24 vs #54) rawan tertukar. Bisa ditambah re-ranking (cross-encoder / LLM re-rank) atau hybrid search (BM25 + embedding).
4. **Jawaban LLM non-deterministik.** Walaupun temperature 0.2, redaksi jawaban bisa sedikit berbeda setiap run. Hasil retrieval sendiri deterministik.
5. **Ketergantungan API & rate limit.** Saat pengujian sempat muncul 429/503. Notebook menangani ini dengan retry + exponential backoff.
6. **Tidak ada konteks percakapan/akun.** Sistem tidak bisa menjawab pertanyaan spesifik pesanan pengguna ("pesanan #123 sampai mana?"). Itu memerlukan integrasi sistem lain, di luar cakupan FAQ.
