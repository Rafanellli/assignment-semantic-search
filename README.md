# Semantic Search dan LLM untuk FAQ E-commerce

Prototype asisten customer service yang mencari FAQ berdasarkan kemiripan makna, kemudian menyusun jawaban menggunakan Gemini berdasarkan FAQ yang ditemukan. Project ini dibuat untuk **Assignment 3 — Semantic Search & LLM Application** pada AI Engineering Bootcamp Reworkacademy.

## Tujuan dan batasan use case

Tim customer service perlu menjawab pertanyaan berulang mengenai pemesanan, pengiriman, pembayaran, pengembalian barang, akun, promo, dan produk. Pengguna dapat menyampaikan masalah dengan bahasa yang berbeda dari FAQ asli.

Tujuan prototype adalah menemukan FAQ yang relevan, menghasilkan jawaban yang mudah dipahami, serta menyatakan keterbatasan informasi apabila konteks tidak dapat menjawab pertanyaan. Sistem tidak mengakses pesanan, pembayaran, stok, atau akun pelanggan secara langsung. Informasi kebijakan dalam project ini berasal dari dataset latihan, bukan kebijakan toko yang telah diverifikasi untuk penggunaan produksi.

## Knowledge base

Sumber data adalah file FAQ asli yang disediakan mentor: `data/ecommerce_faq_knowledge_base.csv`. File CSV sumber dipertahankan tanpa perubahan langsung.

- Jumlah entri: **59 FAQ**.
- Kategori: Pemesanan, Pengiriman, Pembayaran, Pengembalian, Akun, Promo, Produk, dan Customer Service.
- Setiap pasangan pertanyaan dan jawaban menjadi satu unit retrievable.

| Kolom | Isi |
|---|---|
| `faq_id` | ID FAQ |
| `category` | Kategori pertanyaan |
| `question` | Pertanyaan FAQ |
| `answer` | Jawaban FAQ |

Kolom `content` dibuat di memori menggunakan format `Q: <question> A: <answer>`. Seluruh FAQ tidak digabung menjadi satu dokumen; terdapat satu teks untuk setiap FAQ. Chunking dokumen panjang tidak digunakan.

## Alur sistem

| Tahap | Implementasi |
|---|---|
| Ingest | Membaca CSV dengan Pandas, memeriksa struktur data, dan menggabungkan question serta answer per baris. |
| Embed | Mengubah unit FAQ menjadi embedding melalui Gemini Embeddings API. |
| Store | Menyimpan vektor float32 pada `faiss.IndexFlatL2` dan menulis index ke file. |
| Retrieve | Membuat embedding query, mencari tiga vektor terdekat, lalu mengambil teks FAQ berdasarkan posisi barisnya. |
| Generate | Menyisipkan FAQ hasil retrieval ke prompt dan meminta Gemini menjawab hanya berdasarkan konteks tersebut. |

## Konfigurasi eksperimen

| Komponen | Konfigurasi |
|---|---|
| Lingkungan pengerjaan | Windows, Python 3.11.9, virtual environment `tugas2` |
| Model embedding | `gemini-embedding-001` |
| Dimensi embedding default | 3072 |
| Vector index | `faiss.IndexFlatL2` |
| Jumlah hasil retrieval | `top_k=3` |
| ID model generatif dalam kode yang berhasil diuji | `gemini-3.5-flash` |
| Temperature dalam pengujian | 0.3 |

Nama model pada starter notebook disesuaikan karena `text-embedding-004` dan `gemini-2.0-flash` menghasilkan error 404 pada saat pengerjaan. Percobaan menggunakan `gemini-3.8-flash` sempat berjalan lebih dari 10 menit tanpa hasil, sedangkan percobaan contoh menggunakan `gemini-3.5-flash` selesai sekitar 5,3 detik. Angka tersebut merupakan pengamatan satu percobaan, bukan benchmark kecepatan atau bukti penyebab keterlambatan model lain.

Ketersediaan model dan kuota dapat berubah. FAQ serta query wajib menggunakan model embedding dan dimensi yang sama. Jika model embedding diganti, buat ulang embedding FAQ dan index. Mengganti model penyusun jawaban saja tidak mengharuskan pembuatan ulang index.

`IndexFlatL2` mengembalikan **squared L2 distance**. Nilai lebih kecil menunjukkan vektor lebih dekat dalam index yang sama; nilainya bukan probabilitas atau persentase keyakinan. Teks FAQ diambil menggunakan `df.iloc[idx]`, sehingga urutan baris CSV harus tetap sesuai dengan urutan saat index dibuat.

## Prompt dan konteks

Prompt menetapkan persona asisten customer service, tugas menjawab pertanyaan, dan konteks dari tiga FAQ hasil retrieval. Instruksi membatasi jawaban pada informasi FAQ serta meminta model mengakui apabila informasi tidak tersedia dan menyarankan menghubungi customer service.

Instruksi prompt membantu membatasi jawaban, tetapi belum merupakan jaminan bahwa semua jawaban selalu didukung sumber. Hasil generasi tetap perlu diperiksa terhadap seluruh konteks retrieval.

## File project

| Lokasi | Fungsi |
|---|---|
| `data/ecommerce_faq_knowledge_base.csv` | Dataset asli dari mentor |
| `data/ecommerce_faq.index` | Index FAISS yang disimpan setelah pembuatan embedding |
| `notebook/starter_notebook_semantic_search.ipynb` | Pipeline, output pengujian, dan analisis relevansi |
| `documentation/` | Dokumentasi tambahan apabila diperlukan |
| `README.md` | Penjelasan use case, alur, hasil, batasan, dan cara menjalankan |
| `requirements.txt` | Dependency Python |
| `.env` | API key lokal; tidak disertakan dalam Git |

## Cara menjalankan

### 1. Siapkan environment

Buka terminal PowerShell di folder utama repository. Gunakan Python 3.11 agar mengikuti versi lingkungan pengerjaan, lalu buat environment:

```powershell
py -3.11 -m venv tugas2
.\tugas2\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

Jika PowerShell menolak skrip aktivasi, izinkan skrip lokal untuk sesi terminal tersebut lalu aktifkan kembali:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
.\tugas2\Scripts\Activate.ps1
```

File requirements awal mencantumkan dependency langsung tanpa mengarang versi dari laptop peserta. Setelah environment yang digunakan telah berhasil menjalankan notebook, kunci versi yang terpasang dari terminal venv tersebut:

```powershell
python -m pip freeze | Set-Content -Encoding utf8 requirements.txt
```

Perintah ini mengganti daftar awal dengan versi paket yang benar-benar terpasang, termasuk dependency turunannya. Commit file requirements yang sudah dikunci bersama hasil final.

### 2. Siapkan API key lokal

Buat file bernama `.env` di folder utama repository dengan isi berikut. Ganti nilai contoh menggunakan key milik sendiri:

```dotenv
GEMINI_API_KEY=YOUR_GEMINI_API_KEY
```

Tambahkan minimal baris berikut ke `.gitignore`:

```gitignore
.env
tugas2/
__pycache__/
.ipynb_checkpoints/
```

Jangan menampilkan key di output notebook atau mengunggah `.env` ke repository. `.env` adalah file teks biasa, bukan penyimpanan terenkripsi.

### 3. Gunakan path yang dapat dipindahkan

Notebook selama pengerjaan menggunakan path absolut pada drive `E:`. **Sebelum dibagikan, sesuaikan pemanggilan file di notebook** menggunakan folder project. Tambahkan kode berikut sebelum membaca CSV dan membuat client; bagian ini bisa dijalankan dari folder utama atau folder `notebook`:

```python
from pathlib import Path
from dotenv import load_dotenv

PROJECT_ROOT = Path.cwd()
if not (PROJECT_ROOT / "data").is_dir():
    PROJECT_ROOT = PROJECT_ROOT.parent

load_dotenv(PROJECT_ROOT / ".env", override=True)
client = genai.Client(api_key=os.environ.get("GEMINI_API_KEY"))

df = pd.read_csv(
    PROJECT_ROOT / "data" / "ecommerce_faq_knowledge_base.csv"
)
```

Simpan index di section 4 setelah `index.add(embeddings)`:

```python
index_path = PROJECT_ROOT / "data" / "ecommerce_faq.index"
faiss.write_index(index, str(index_path))

saved_index = faiss.read_index(str(index_path))
print("Jumlah vektor:", saved_index.ntotal)
print("Dimensi vektor:", saved_index.d)
```

Index tersimpan tidak menyimpan teks jawaban; CSV tetap diperlukan sebagai pasangan data FAQ.

### 4. Buka dan jalankan notebook

Pasang ekstensi Python dan Jupyter pada VS Code, lalu buka notebook:

```powershell
code .\notebook\starter_notebook_semantic_search.ipynb
```

Pilih kernel dengan interpreter `tugas2\Scripts\python.exe`. Periksa interpreter dari code cell:

```python
import sys
print(sys.executable)
```

Jalankan section 1 sampai 8 secara berurutan, lalu simpan notebook beserta outputnya. Pada pembuatan embedding, hasil yang diharapkan untuk 59 FAQ dengan dimensi default adalah `(59, 3072)` dan `index.ntotal` bernilai `59`. Eksekusi melibatkan request API dan memerlukan koneksi internet serta kuota yang tersedia.

Untuk memakai index tersimpan pada sesi lain, muat CSV dengan urutan yang sama dan gunakan `faiss.read_index(str(index_path))` sebagai pengganti pembuatan index. Jalankan juga definisi fungsi yang diperlukan. Memuat file index saja tidak otomatis melewati cell embedding dalam notebook.

## Hasil pengujian relevansi

Lima query berikut benar-benar diuji. Dua query menggunakan pertanyaan langsung, dua menggunakan parafrase, dan satu meminta detail yang tidak tersedia dalam knowledge base.

| No. | Jenis dan query | FAQ teratas | Evaluasi jawaban |
|---|---|---|---|
| 1 | Langsung: Metode pembayaran apa saja yang tersedia? | Metode pembayaran apa saja yang tersedia? | Menyebut transfer bank, kartu kredit/debit, e-wallet, dan minimarket rekanan sesuai FAQ. |
| 2 | Langsung: Bagaimana cara melacak paket yang sedang dikirim? | Bagaimana cara melacak paket yang sedang dikirim? | Menjelaskan lokasi nomor resi pada Pesanan Saya dan pelacakan di website kurir. |
| 3 | Parafrase: Duit gue udah kepotong, kok pesanan masih belum terbayar? | Kenapa pembayaran saya berhasil dipotong tapi status pesanan masih belum terbayar? | Menjelaskan keterlambatan sinkronisasi dan waktu tunggu 15–30 menit sesuai sumber. |
| 4 | Parafrase: Barang yang datang beda dari pesanan, harus ngapain? | Saya menerima barang yang salah, apa yang harus saya lakukan? | Menjelaskan komplain dan bukti foto. Batas 3 hari didukung FAQ kedua tentang pengembalian. |
| 5 | Tidak tersedia: Berapa bunga cicilan kartu kredit BCA untuk tenor 12 bulan? | Apakah saya bisa membayar dengan cicilan? | Mengakui belum memiliki informasi bunga dan menyarankan customer service, tanpa mengarang angka. |

### Pemeriksaan konteks query keempat

Pemeriksaan lanjutan pada query keempat menghasilkan tiga FAQ berikut. Nilai distance dibulatkan menjadi empat desimal:

| Peringkat | FAQ | Distance | Analisis |
|---|---|---:|---|
| 1 | Saya menerima barang yang salah, apa yang harus saya lakukan? | 0.2501 | Paling sesuai dengan barang yang berbeda dari pesanan. |
| 2 | Bagaimana prosedur pengembalian barang yang tidak sesuai? | 0.3809 | Memberikan informasi pendukung: alasan, bukti foto, dan batas pengembalian 3 hari. |
| 3 | Apa yang harus dilakukan jika paket diterima dalam kondisi rusak? | 0.4666 | Masih terkait keluhan barang, tetapi membahas kasus yang berbeda. |

Jawaban menggabungkan informasi dari dua FAQ yang relevan. Detail maksimal 3 hari mempunyai dukungan pada FAQ kedua. Agar lebih presisi, batas itu sebaiknya disebut khusus sebagai batas pengajuan pengembalian, bukan sebagai batas umum seluruh jenis komplain. Ketentuan 2×24 jam pada FAQ paket rusak tidak dicampurkan ke jawaban.

### Interpretasi hasil

Keempat query dengan jawaban tersedia menemukan FAQ teratas yang sesuai, termasuk dua query berbahasa percakapan. Pada satu query tanpa detail jawaban, model berhasil menyatakan keterbatasan informasi. Ini merupakan pengamatan pada sampel uji, bukan ukuran akurasi umum sistem.

Query bunga cicilan menunjukkan bahwa retrieval dapat menemukan FAQ yang dekat secara topik tanpa menyediakan jawaban spesifik. FAISS tetap mengembalikan tetangga terdekat; LLM perlu menilai apakah informasi konteks mencukupi sebelum menjawab. Output lengkap lima pengujian dan analisisnya disimpan pada section 8 notebook final.

## Keterbatasan dan pengembangan berikutnya

- Lima query belum cukup untuk mengukur kinerja secara umum. Perlu pengujian tambahan pada pertanyaan ambigu, beberapa kebutuhan sekaligus, negasi, dan variasi bahasa.
- Sistem belum memakai ambang distance untuk menolak hasil retrieval yang terlalu jauh. Memilih ambang memerlukan data evaluasi tambahan.
- Tiga FAQ yang diambil dapat mencakup sumber yang hanya berkaitan secara topik. Informasi dengan kondisi berbeda perlu dipisahkan secara jelas dalam jawaban.
- Instruksi prompt belum menjamin seluruh jawaban bebas dari informasi yang tidak didukung sumber.
- Sistem bergantung pada ketersediaan, kuota, latensi, dan biaya layanan API. Pengujian ini tidak menghitung biaya per query atau membandingkan latency secara terkontrol.
- Belum ada cache embedding query, retry terstruktur, dan mekanisme pembaruan index otomatis ketika CSV berubah.

## Referensi

- Brief dan starter notebook Assignment 3 dari Reworkacademy.
- Dataset FAQ: https://drive.google.com/file/d/1A1m685JChJQ3gvUDh-IBWdj-F2O7_mL_/view
- Gemini API: https://ai.google.dev/gemini-api/docs
- FAISS: https://github.com/facebookresearch/faiss
- python-dotenv: https://github.com/theskumar/python-dotenv
