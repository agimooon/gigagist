# TryHackMe — Hacker Holidays 2026: Management Wants a Word
# Day 14

| | |
|---|---|
| **Kategori** | Digital Forensics |
| **Tipe** | Forensic Investigation |

## Pendahuluan — Jejak yang Tersembunyi

Challenge **Management Wants a Word** ini kelihatannya sederhana di awal, tapi ternyata butuh ketelitian yang lebih dari sekadar "cari file, buka, baca isinya".

Yang tersedia di awal investigasi cuma sebuah hasil backup dari KAPE. Di dalamnya ada satu file bernama `backup` yang sepintas nggak kelihatan seperti file biasa. Setelah ditelusuri lebih jauh, ternyata itu adalah sebuah container yang cuma bisa dibuka lewat VeraCrypt.

Dari titik itu, saya coba perlakukan setiap artefak sebagai bagian dari satu cerita yang berurutan: temukan file, verifikasi integritasnya, buka container-nya, inventarisasi isinya, lalu cari hubungan antar dokumen yang ditemukan.

---

## 1. Menemukan Artefak Backup

Langkah pertama adalah memastikan lokasi backup yang sebenarnya. Percobaan awal sempat gagal karena variabel `ROOT` belum diarahkan ke direktori KAPE yang benar.

Setelah struktur foldernya diperiksa ulang, lokasi backup-nya ketemu di:

```text
KAPE/C/Users/vera/Documents/backup
```

Cek dulu tipe file-nya:

```bash
file "$ROOT/C/Users/vera/Documents/backup"
```

Karena file ini akan jadi objek analisis, saya buat dulu checksum SHA-256-nya sebagai baseline:

```bash
shasum -a 256 "$ROOT/C/Users/vera/Documents/backup"
```

Hash yang didapat:

```text
<SHA256_HASH>
```

Dalam konteks forensik, langkah ini penting buat mencatat identitas artefak dan memastikan file yang dianalisis nggak berubah selama proses investigasi berlangsung.

---

## 2. Kesalahan Kecil yang Ternyata Memberi Pelajaran

Ada satu kesalahan kecil yang sempat kejadian. Saya coba jalanin file `backup` langsung dari shell:

```bash
~/Downloads/.../backup
```

Shell-nya balas dengan:

```text
zsh: permission denied
```

Awalnya kelihatan kayak masalah permission, tapi ternyata bukan. File-nya memang bukan executable — dia artefak yang harus dibuka pakai aplikasi yang sesuai, bukan dijalankan langsung.

Setelah sadar soal ini, saya lanjut prosesnya pakai VeraCrypt.

---

## 3. Membuka Container VeraCrypt

Container-nya kemudian dibuka dan di-mount lewat VeraCrypt.

Setelah berhasil, volume-nya muncul di macOS sebagai:

```text
/Volumes/NO NAME
```

Enumerasi sederhana buat lihat isi volume-nya:

```bash
ls -la "/Volumes/NO NAME"
```

Lalu cari file dan folder yang tersedia di dalamnya:

```bash
find "/Volumes/NO NAME" -type f -print
```

Ada satu folder yang langsung menarik perhatian:

```text
secret_financial_documents
```

Cek isinya:

```bash
find "/Volumes/NO NAME/secret_financial_documents" -type f -print
```

Di dalamnya ada dua file utama:

```text
important_invoice_byte_lotus.pdf
transactions_q3.csv
```

Di titik ini konteks investigasinya mulai kelihatan: ada dokumen keuangan dan sebuah invoice yang berhubungan dengan Byte Lotus.

---

## 4. Membaca Jejak dari CSV

File `transactions_q3.csv` bisa langsung dibaca:

```bash
cat "/Volumes/NO NAME/secret_financial_documents/transactions_q3.csv"
```

Buat lihat bagian awalnya aja:

```bash
head "/Volumes/NO NAME/secret_financial_documents/transactions_q3.csv"
```

Isinya berupa daftar transaksi lengkap dengan tanggal, reference, vendor, deskripsi, nominal, dan status.

Salah satu transaksi yang relevan:

```text
2026-07-15, TXN-10547, Byte Lotus Resorts, Guest accommodation, 3840.00, Approved
```

Ada juga transaksi lain yang menyebut Byte Lotus Catering dan Lotus Printworks.

CSV ini nggak ngasih flag secara langsung. Tapi dari sisi investigasi, file ini tetap penting karena kasih konteks dan memperkuat hubungan antara dokumen invoice dengan entitas Byte Lotus.

---

## 5. Invoice yang Nggak Mau Bercerita

Selanjutnya, fokus pindah ke invoice PDF.

Cek dulu tipe file-nya:

```bash
file "/Volumes/NO NAME/secret_financial_documents/important_invoice_byte_lotus.pdf"
```

Lalu metadata-nya:

```bash
mdls "/Volumes/NO NAME/secret_financial_documents/important_invoice_byte_lotus.pdf"
```

Hasilnya menunjukkan dokumen ini PDF 1.7, satu halaman, ukuran sekitar 26 KB.

Coba cari string yang berhubungan dengan beberapa keyword:

```bash
strings "/Volumes/NO NAME/secret_financial_documents/important_invoice_byte_lotus.pdf" | \
grep -Ei 'lotus|password|credential|flag|http|https|url'
```

Nihil, nggak ada informasi yang berarti.

Coba juga ekstraksi teks pakai `pdftotext`:

```bash
pdftotext \
"/Volumes/NO NAME/secret_financial_documents/important_invoice_byte_lotus.pdf" \
-
```

Hasilnya juga nggak kasih teks yang berguna.

Dari sini muncul dugaan: mungkin PDF-nya nggak nyimpen isi invoice sebagai text layer, tapi sebagai gambar.

---

## 6. Membongkar Isi PDF

Buat memastikan dugaan itu, saya pakai `pdfimages` dari paket Poppler.

Pertama, lihat dulu daftar image yang ada di dalam PDF:

```bash
pdfimages -list \
"/Volumes/NO NAME/secret_financial_documents/important_invoice_byte_lotus.pdf"
```

Hasilnya nunjukkin halaman PDF ini punya satu image berukuran:

```text
636 × 724 pixels
```

plus sebuah soft mask.

Buat folder buat hasil ekstraksi:

```bash
mkdir -p "$HOME/Desktop/byte-lotus-images"
```

Ekstrak semua image-nya:

```bash
pdfimages -all \
"/Volumes/NO NAME/secret_financial_documents/important_invoice_byte_lotus.pdf" \
"$HOME/Desktop/byte-lotus-images/img"
```

Hasil utamanya:

```text
~/Desktop/byte-lotus-images/img-000.png
```

Cek hasil ekstraksinya:

```bash
file "$HOME/Desktop/byte-lotus-images/img-000.png"
```

File-nya teridentifikasi sebagai PNG RGB berukuran 636 × 724.

Nah, arah investigasi berubah di sini. Kalau teksnya nggak tersedia sebagai text layer, informasi yang dicari kemungkinan besar ada di dalam gambar itu sendiri.

---

## 7. Saat OCR Jadi Kunci

Karena PDF-nya ternyata berbasis gambar, langkah berikutnya jalan Optical Character Recognition (OCR).

Kalau Tesseract belum terpasang di macOS, install dulu lewat Homebrew:

```bash
brew install tesseract
```

Lalu jalankan Tesseract terhadap image yang sudah diekstrak:

```bash
tesseract \
"$HOME/Desktop/byte-lotus-images/img-000.png" \
stdout
```

Kali ini hasilnya beda — Tesseract berhasil membaca isi invoice-nya.

Beberapa bagian yang kebaca:

```text
INVOICE
BYTELOTUS RESORTS
INVOICE NR: 2122/9090/5050
DUE DATE: 8/09/2026
```

Tapi ada satu baris yang langsung jadi titik akhir investigasi — flag-nya ternyata nyempil di situ.

---

## 8. Flag

Flag yang berhasil ditemukan:

```text
THM[1t_w4s...AlOng?!}
```

*(Sengaja disensor sebagian ya gess.)*

Yang menarik, flag ini nggak ketemu lewat `strings`, metadata, text extraction, attachment, maupun JavaScript. Flag-nya justru bersembunyi di dalam gambar invoice.

Inilah kenapa pendekatan bertahap itu penting. Kalau investigasi berhenti begitu `pdftotext` atau `strings` nggak nemu apa-apa, flag-nya bakal kelewat gitu aja.

---

## 9. Lessons Learned

Challenge ini ngasih beberapa pelajaran yang relevan buat kerjaan digital forensics.

Pertama, jangan anggap ekstensi file sebagai gambaran lengkap dari isinya. Sebuah PDF bisa aja isinya cuma gambar, tanpa text layer yang berguna sama sekali.

Kedua, kegagalan sebuah tool bukan berarti artefaknya kosong. Waktu `pdftotext` nggak menghasilkan teks apa pun, langkah selanjutnya bukan nyerah, tapi memahami struktur PDF-nya dan cari objek lain di dalamnya.

Ketiga, konteks antar-artefak itu penting. CSV-nya sendiri nggak berisi flag, tapi dia bantu mengonfirmasi hubungan dengan Byte Lotus Resorts, yang jadi petunjuk buat fokus ke invoice.

Terakhir, investigasi forensik yang baik jalan kayak proses eliminasi. Setiap pemeriksaan yang nggak nemu apa-apa tetap ngasih informasi, karena dia mempersempit kemungkinan yang tersisa.

---

## 10. Alur Investigasi

Secara keseluruhan, alurnya kira-kira begini:

1. Menemukan backup di struktur KAPE.
2. Mengidentifikasi tipe file backup.
3. Menghitung SHA-256 buat mencatat integritas artefak.
4. Membuka container lewat VeraCrypt.
5. Enumerasi terhadap volume yang sudah di-mount.
6. Menemukan folder `secret_financial_documents`.
7. Menganalisis `transactions_q3.csv` buat dapetin konteks.
8. Memeriksa struktur dan metadata invoice PDF.
9. Mencoba `strings` dan `pdftotext`.
10. Memastikan PDF mengandung image lewat `pdfimages -list`.
11. Mengekstrak image dari dalam PDF.
12. Menjalankan Tesseract OCR terhadap image tersebut.
13. Menemukan flag di dalam invoice.

---

## 11. Kesimpulan

Pada akhirnya, challenge ini bukan soal satu perintah ajaib. Penyelesaiannya datang dari rangkaian pemeriksaan kecil yang dilakukan berurutan, satu per satu.

Sebuah file backup membawa saya ke container VeraCrypt. Container itu membawa ke dokumen keuangan. CSV-nya kasih konteks, sementara invoice-nya nyimpen informasi yang sebenarnya dicari.

Ketika PDF nggak ngasih teks lewat metode konvensional, struktur file-nya diperiksa dan image-nya diekstrak. Baru setelah OCR dijalankan, pesan tersembunyi itu akhirnya muncul.

Bagian paling menarik dari challenge ini, buat saya, adalah kenyataan bahwa jawabannya sebenarnya udah ada di depan mata sejak awal. Tantangannya cuma soal tahu di mana harus melihat.

---

## Tools yang Digunakan

- **VeraCrypt** — mounting encrypted container
- `find`, `ls`, `cat`, `head`, `grep` — enumerasi dan pemeriksaan artefak
- `file`, `mdls` — identifikasi tipe file dan metadata
- `shasum` — checksum SHA-256
- `strings` — pencarian string di dalam artefak
- **Poppler / `pdfimages`** — analisis dan ekstraksi image dari PDF
- `pdftotext` — percobaan ekstraksi text layer dari PDF
- **Tesseract OCR** — ekstraksi teks dari image
