# AI Document Processor

**Otomasi pencatatan dokumen keuangan menggunakan AI — dibangun di atas n8n.**

---

## Gambaran Umum

AI Document Processor adalah workflow n8n yang mengotomasi proses pencatatan dokumen keuangan. Sistem ini memantau folder Google Drive secara real-time, mengekstrak informasi penting dari dokumen menggunakan GPT-4o Mini, lalu mencatatnya secara otomatis ke Google Sheets — menggantikan proses input manual yang rawan kesalahan dan memakan waktu.

### Masalah yang Diselesaikan

Di banyak bagian keuangan, alur kerja pencatatan dokumen masih bersifat manual: staff membuka dokumen satu per satu, membaca informasi seperti nomor invoice, nama vendor, nominal, tanggal jatuh tempo, lalu mengetiknya ke spreadsheet. Proses ini lambat, rentan human error, dan tidak scalable saat volume dokumen meningkat.

AI Document Processor menghilangkan bottleneck tersebut. Cukup upload dokumen ke Google Drive — sistem menangani sisanya.

---

## Tech Stack

| Komponen | Teknologi |
|----------|-----------|
| Workflow Engine | n8n (self-hosted) |
| AI Model | OpenAI GPT-4o Mini |
| File Storage | Google Drive |
| Database/Log | Google Sheets |
| Notifikasi | Gmail |
| Bahasa Scripting | JavaScript (n8n Code Node) |

---

## Arsitektur Workflow

```
Google Drive (Upload)
        │
        ▼
  ┌─────────────┐
  │ Drive Trigger│ ◄── Polling setiap 1 menit
  └──────┬──────┘
         ▼
  ┌─────────────┐     ┌───────────────────┐
  │ Cek Format  │────►│ Notifikasi Error  │ (format tidak didukung)
  │ (PDF only)  │     │ via Gmail         │
  └──────┬──────┘     └───────────────────┘
         ▼
  ┌─────────────┐     ┌───────────────────┐
  │ Cek Size    │────►│ Notifikasi Error  │ (file > 10MB)
  │ (< 10MB)    │     │ via Gmail         │
  └──────┬──────┘     └───────────────────┘
         ▼
  ┌─────────────┐
  │ Download    │
  │ File        │
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │ Extract     │
  │ Metadata    │ (Code Node - JavaScript)
  └──────┬──────┘
         ▼
  ┌──────────────┐     ┌───────────────────┐
  │ Cek Duplikat │────►│ Notifikasi        │ (file sudah pernah diproses)
  │ (via Sheets) │     │ Duplikat via Gmail│
  └──────┬───────┘     └───────────────────┘
         ▼
  ┌──────────────┐
  │ GPT-4o Mini  │ ◄── Ekstraksi data via OpenAI API
  └──────┬───────┘
         ▼
  ┌──────────────┐
  │ Parsing JSON │ (Code Node - JavaScript)
  └──────┬───────┘
         ▼
  ┌──────────────────┐
  │ Confidence Score │
  │    >= 0.7 ?      │
  └──┬───────────┬───┘
     ▼           ▼
  ┌──────┐   ┌────────────┐
  │ YES  │   │ NO         │
  └──┬───┘   └─────┬──────┘
     ▼              ▼
 Input ke       Pindah ke folder
 Google Sheets  "Needs Review"
     ▼              ▼
 Pindah ke      Kirim email
 folder         notifikasi
 "Processed"    review manual
     ▼
 Kirim email
 sukses
     ▼
 Input ke
 Dashboard Log
```

---

## Alur Proses Detail

### 1. Trigger — Google Drive Watcher

Workflow dimulai dari node `Google Drive Trigger` yang melakukan polling setiap **1 menit** pada folder tertentu di Google Drive. Setiap kali ada file baru yang di-upload ke folder tersebut, workflow otomatis berjalan.

### 2. Validasi File

Sebelum diproses, setiap file melewati dua tahap validasi:

**Validasi Format** — Sistem memeriksa `mimeType` file. Hanya file PDF yang diterima. Jika format tidak sesuai, workflow mengirim notifikasi email via Gmail yang menjelaskan format apa saja yang diterima.

**Validasi Ukuran** — File harus berukuran di bawah **10MB** (`10.000.000 bytes`). File yang melebihi batas akan ditolak dengan notifikasi email yang menyarankan user untuk mengompres atau memecah dokumen.

### 3. Download & Ekstraksi Metadata

File yang lolos validasi didownload dari Google Drive melalui HTTP Request langsung ke Google Drive API (`googleapis.com/drive/v3/files/{id}?alt=media`). Setelah itu, Code Node JavaScript mengekstrak metadata dasar:

```javascript
{
  fileName: "invoice_001.pdf",
  mimeType: "application/pdf",
  fileId: "abc123...",
  uploadedAt: "2025-01-15T10:30:00.000Z"
}
```

### 4. Deteksi Duplikat

Sebelum mengkonsumsi kredit OpenAI, sistem memeriksa apakah file dengan nama yang sama sudah pernah diproses. Pengecekan dilakukan dengan query ke Google Sheets berdasarkan kolom `File Name`. Jika ditemukan duplikat, workflow berhenti dan mengirim notifikasi email.

### 5. Ekstraksi Data dengan AI

Inti dari sistem ini adalah node `Message a model` yang mengirim dokumen ke **GPT-4o Mini** dengan prompt terstruktur. AI diminta mengembalikan data dalam format JSON ketat:

```json
{
  "document_type": "invoice|receipt|report|other",
  "vendor_name": "PT Contoh Sejahtera",
  "document_number": "INV-2025-001",
  "document_date": "2025-01-15",
  "due_date": "2025-02-15",
  "total_amount": 15000000,
  "currency": "IDR",
  "tax_amount": 1500000,
  "line_items": [
    {
      "description": "Jasa Konsultasi IT",
      "quantity": 1,
      "unit_price": 15000000,
      "total": 15000000
    }
  ],
  "notes": "",
  "confidence_score": 0.95
}
```

`confidence_score` (0.0–1.0) menunjukkan seberapa yakin AI dalam membaca dokumen. Nilai ini menjadi penentu apakah data langsung dicatat atau perlu review manual.

### 6. Parsing & Enrichment

Output mentah dari OpenAI di-parse oleh Code Node. Proses ini membersihkan markdown fence (` ```json `) dari response, mem-parse JSON, lalu menambahkan metadata tambahan:

```javascript
{
  ...parsedAIResponse,
  fileName: "invoice_001.pdf",
  fileId: "abc123...",
  uploadedAt: "2025-01-15T10:30:00.000Z",
  processedAt: "2025-01-15T10:30:45.000Z",
  status: "processed"  // atau "needs_review" jika confidence < 0.7
}
```

### 7. Routing Berdasarkan Confidence Score

Sistem menggunakan threshold **confidence score ≥ 0.7** untuk menentukan alur selanjutnya:

**Score ≥ 0.7 → Processed:**
1. Data ditulis ke sheet `Log` di Google Sheets dengan 13 kolom lengkap
2. File dipindahkan ke folder `Processed` di Google Drive
3. Email notifikasi sukses dikirim berisi ringkasan data yang diekstrak
4. Data ringkasan ditulis ke sheet `Dashboard Log` termasuk `Processing Time` dalam detik

**Score < 0.7 → Needs Review:**
1. File dipindahkan ke folder `Needs Review` di Google Drive
2. Email notifikasi dikirim ke tim keuangan berisi data parsial dan instruksi untuk review manual

---

## Struktur Data Google Sheets

### Sheet: Log

Sheet utama yang menyimpan seluruh data hasil ekstraksi.

| Kolom | Tipe | Deskripsi |
|-------|------|-----------|
| Timestamp | String | Waktu file selesai diproses (ISO 8601) |
| File Name | String | Nama file asli dari Google Drive |
| File ID | String | Google Drive file ID |
| Document Type | String | Jenis dokumen: `invoice`, `receipt`, `report`, `other` |
| Vendor Name | String | Nama vendor/supplier |
| Document Number | String | Nomor dokumen (nomor invoice, nota, dll.) |
| Document Date | String | Tanggal dokumen |
| Due Date | String | Tanggal jatuh tempo pembayaran |
| Total Amount | Number | Nominal total |
| Currency | String | Mata uang (default: `IDR`) |
| Tax Amount | Number | Nominal pajak/PPN |
| Confidence Score | Number | Skor keyakinan AI (0.0–1.0) |
| Status | String | `processed` atau `needs_review` |

### Sheet: Dashboard Log

Sheet ringkasan untuk keperluan dashboard dan monitoring performa.

| Kolom | Tipe | Deskripsi |
|-------|------|-----------|
| Date | String | Tanggal dokumen |
| File Name | String | Nama file |
| Document Type | String | Jenis dokumen |
| Vendor Name | String | Nama vendor |
| Total Amount | Number | Nominal total |
| Confidence Score | Number | Skor keyakinan AI |
| Status | String | Status pemrosesan |
| Processing Time | Number | Durasi pemrosesan dalam detik (dihitung otomatis dari selisih `uploadedAt` dan `processedAt`) |

---

## Struktur Folder Google Drive

```
AI Document Processor/          ← Folder utama (trigger memantau folder ini)
├── Processed/                  ← File yang berhasil diproses (confidence ≥ 0.7)
└── Needs Review/               ← File yang butuh review manual (confidence < 0.7)
```

---

## Sistem Notifikasi Email

Workflow mengirim notifikasi Gmail untuk setiap skenario:

| Skenario | Subject | Keterangan |
|----------|---------|------------|
| Format salah | ⚠️ File Ditolak: {nama} | Menginfokan hanya PDF, JPG, PNG yang diterima |
| Ukuran berlebih | ⚠️ File Ditolak: {nama} | Menyarankan kompres/pecah file |
| File duplikat | ⚠️ Dokumen Duplikat: {nama} | Menginfokan file sudah pernah diproses |
| Confidence rendah | ⚠️ Perlu Review Manual: {nama} | Menampilkan data parsial + instruksi review |
| Berhasil diproses | ✅ Dokumen Berhasil Diproses: {vendor} — {nomor} | Ringkasan lengkap data yang diekstrak |

---

## Node Workflow Reference

Daftar seluruh node yang digunakan beserta fungsinya:

| # | Node | Tipe | Fungsi |
|---|------|------|--------|
| 1 | Drive Trigger | Google Drive Trigger | Memantau folder untuk file baru (polling 1 menit) |
| 2 | Chek File Format | IF | Memvalidasi mimeType mengandung "pdf" |
| 3 | Notifikasi Error Format | Gmail | Kirim email jika format tidak didukung |
| 4 | Chek Size | IF | Memvalidasi ukuran file < 10MB |
| 5 | Notifikasi Error Size | Gmail | Kirim email jika file terlalu besar |
| 6 | Download file 1 | HTTP Request | Download file dari Google Drive API |
| 7 | Code in JavaScript | Code | Ekstrak metadata file (nama, mime, id) |
| 8 | Check Duplicate | Google Sheets (Read) | Cek apakah file name sudah ada di sheet Log |
| 9 | Duplicate Condition | IF | Routing berdasarkan hasil cek duplikat |
| 10 | Send a message | Gmail | Notifikasi jika file duplikat |
| 11 | Message a model | OpenAI | Kirim dokumen ke GPT-4o Mini untuk ekstraksi |
| 12 | Parsing Value | Code | Parse JSON response + enrichment metadata |
| 13 | If | IF | Routing berdasarkan confidence score ≥ 0.7 |
| 14 | Input Data Log | Google Sheets (Append) | Tulis data lengkap ke sheet Log |
| 15 | Move to Need Processed | Google Drive (Move) | Pindahkan file ke folder Processed |
| 16 | Send Success Message | Gmail | Kirim email notifikasi sukses |
| 17 | Input Data Dashboard Log | Google Sheets (Append) | Tulis ringkasan ke sheet Dashboard Log |
| 18 | Move to Need Review | Google Drive (Move) | Pindahkan file ke folder Needs Review |
| 19 | Need Review | Gmail | Kirim email notifikasi review manual |

---

## Prasyarat

1. **n8n** — Instance n8n yang sudah berjalan (self-hosted atau cloud)
2. **Google Cloud Project** — Dengan OAuth 2.0 credentials yang mencakup scope untuk Drive, Sheets, dan Gmail
3. **OpenAI API Key** — Dengan akses ke model `gpt-4o-mini`
4. **Google Sheets** — Spreadsheet dengan dua sheet: `Log` dan `Dashboard Log` sesuai struktur kolom di atas
5. **Google Drive** — Tiga folder: folder utama untuk upload, `Processed`, dan `Needs Review`

---

## Cara Setup

1. **Import workflow** ke n8n melalui menu *Import from File*
2. **Konfigurasi credentials** untuk setiap service:
   - Google Drive OAuth2
   - Google Sheets OAuth2
   - Gmail OAuth2
   - OpenAI API
3. **Sesuaikan ID folder** Google Drive pada node `Drive Trigger`, `Move to Need Processed`, dan `Move to Need Review`
4. **Sesuaikan ID spreadsheet** pada node `Input Data Log`, `Check Duplicate`, dan `Input Data Dashboard Log`
5. **Sesuaikan email penerima** notifikasi pada seluruh node Gmail (default: satu alamat email untuk semua notifikasi)
6. **Buat kolom header** di Google Sheets sesuai tabel struktur data di atas
7. **Aktifkan workflow** — sistem mulai memantau folder Google Drive

---

## Limitasi & Catatan

- **Format file** — Saat ini hanya mendukung file PDF. File gambar (JPG/PNG) disebutkan di notifikasi error tapi belum dihandle di validasi format.
- **Deteksi duplikat** — Berbasis nama file, bukan hash konten. File yang di-rename tidak akan terdeteksi sebagai duplikat.
- **Polling interval** — Trigger menggunakan polling 1 menit, bukan webhook real-time. Ada delay hingga 60 detik antara upload dan pemrosesan.
- **Confidence threshold** — Nilai 0.7 bersifat hardcoded. Dokumen dengan scan buruk atau tulisan tangan kemungkinan akan selalu masuk ke review manual.
- **Single recipient** — Semua notifikasi email dikirim ke satu alamat yang sama.
- **Error handling** — Belum ada retry mechanism jika OpenAI API gagal atau rate-limited.

---

## Pengembangan Selanjutnya

- [ ] Dukungan format gambar (JPG/PNG) dengan OCR preprocessing
- [ ] Deteksi duplikat berbasis content hash (bukan nama file)
- [ ] Webhook trigger untuk pemrosesan real-time
- [ ] Multi-recipient notifikasi berdasarkan jenis dokumen
- [ ] Retry mechanism untuk API failure
- [ ] Dashboard monitoring via Google Data Studio / Looker
- [ ] Batch processing untuk volume tinggi
- [ ] Configurable confidence threshold

---

## Lisensi

Proyek ini bersifat internal. Silakan sesuaikan dengan kebutuhan organisasi Anda.
#   A i - D o c u m e n t - P r o c e s s o r  
 