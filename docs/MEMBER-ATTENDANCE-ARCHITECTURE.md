# Presensi Member — Feature Architecture

> **Presensi Member** adalah fitur mandiri di bawah modul **Class & Product** pada arsitektur Dialogika Pilot.
> Fitur ini memfasilitasi pencatatan kehadiran (*attendance*) peserta/member kursus Dialogika secara cepat (*frictionless check-in*) berbasis nomor telepon/WhatsApp dengan pendeteksian otomatis identitas siswa, kelas aktif, serta frekuensi pengulangan sesi/remedial (*repeat retake attendance*).
> Dokumentasi ini disusun berdasarkan panduan `Refactoring Prompt.txt` dan `docs/ARCHITECTURE-FOUNDATION.md`.

---

## 1. Feature Purpose & Domain Boundary

- **Domain**: `Class & Product` / `Member Lifecycle` (merujuk ke `docs/team-internal-map.html`).
- **Tujuan**:
  1. Member hanya memasukkan nomor WhatsApp pada halaman presensi publik / kiosk.
  2. Sistem menstandarkan format nomor dan melakukan pencarian ke koleksi `data_member`.
  3. Sistem mencocokkan jadwal/kelas aktif dari koleksi `class_planning`.
  4. Sistem memeriksa histori kehadiran siswa di koleksi `member_attendance` untuk mendeteksi apakah kehadiran berstatus **Reguler** atau **Mengulang/Remedial ke-N**.
  5. Member memverifikasi kartu identitas dan konfirmasi hadir.
  6. Mencegah presensi ganda (*double submission*) pada tanggal/sesi yang sama.
  7. Menyediakan tab rekapitulasi kehadiran dengan pemilih tanggal (*date picker*) dan Universal Export Modal (Excel `.xlsx` & CSV `.csv`) berstandar Dialogika Pilot.
  8. Menyediakan tab khusus data mengulang & remedial lengkap dengan pemilih tanggal (*date picker*) dan Universal Export Modal.
  9. Menampilkan identitas visual resmi Dialogika (*white-logo.webp wordmark*).
- **Kemandirian Fitur (Preservation of Existing Features)**:
  - Fitur presensi tim internal (`presence.html` dan `pages/hr/presence-team/`) **tidak diubah**.
  - Koleksi `data_member` dan `class_planning` dibaca secara *read-only* tanpa merusak skema lama.

---

## 2. Struktur Modul & Pemisahan Tanggung Jawab

Mengikuti arsitektur Pilot (`pages/product/member-attendance/`):

```text
pages/product/member-attendance/
├── index.html                   # Struktur markup semantik, mount points
├── member-attendance.js          # Orchestrator (koordinasi alur, event binding, state)
├── member-attendance.repository.js  # Data Layer (Zero DOM: Firestore query & persistence)
├── member-attendance.ui.js       # Presentation Layer (Zero Firestore: DOM rendering, badges, alert)
└── member-attendance.css         # Scoped styling (Dialogika design tokens, responsive kiosk)
```

### Tanggung Jawab File:

| File | Tanggung Jawab Utama | Larangan Ketat |
| :--- | :--- | :--- |
| `member-attendance.repository.js` | Akses data Firestore (`data_member`, `class_planning`, `member_attendance`), normalisasi nomor HP, hitung statistik kehadiran member. | **Dilarang** menyentuh DOM (`document.*`, `window.*`). |
| `member-attendance.ui.js` | Rendering kartu identitas siswa, tiket konfirmasi sukses, badge status kehadiran, format tanggal/jam Indonesia. | **Dilarang** mengimpor atau memanggil Firestore SDK secara langsung. |
| `member-attendance.js` | Menghubungkan input pengguna dengan repository dan meneruskannya ke UI renderer, mengatur transisi langkah. | **Dilarang** menjadi god-file; tidak ada raw query atau raw HTML generator. |
| `index.html` | Struktur tata letak halaman, navigasi tab mode, dan wadah (*mount points*). | **Dilarang** menaruh logika bisnis inline. |
| `member-attendance.css` | Gaya visual bertema modern Dialogika (`#0B2B6A`, `#F9C72E`), mobile-friendly. | Menggunakan CSS variabel/tokens. |

---

## 3. Data & Skema Firestore

### Koleksi yang Digunakan:

1. **`data_member`** (Read-Only):
   - Dicari berdasarkan nomor WhatsApp siswa (dinormalisasi awalan `0` / `62`).
   - Fields: `nama`, `whatsapp`, `kota`, `produk_dibeli`.

2. **`class_planning`** (Read-Only):
   - Mengambil daftar kelas aktif / berjalan.
   - Fields: `className`, `productName`, `isRemedial`, `remedialLevel`, `type`.

3. **`member_attendance`** (Create & Read):
   - Dokumen log presensi yang tersimpan:
     ```json
     {
       "member_id": "docId_data_member",
       "member_name": "Budi Santoso",
       "member_phone": "628123456789",
       "member_city": "Jakarta",
       "class_id": "docId_class_planning",
       "class_name": "First Class Jakarta Batch #3",
       "product_name": "First Class",
       "is_repeat": false,
       "repeat_count": 0,
       "attendance_label": "Kehadiran Reguler (Pertemuan Pertama)",
       "attendance_date": "2026-10-06",
       "client_timestamp": "2026-10-06T14:30:00.000Z",
       "created_at": "serverTimestamp()",
       "status": "Hadir"
     }
     ```

4. **`certificate_logs`** (Read-Only):
   - Riwayat sertifikat kelulusan yang telah diterbitkan oleh tim Dialogika.
   - Digunakan oleh fitur **Cetak / Unduh Sertifikat** untuk pengecekan sertifikat siswa berdasarkan nama atau nomor invoice.
   - Fields: `recipientName`, `invoice`, `className`, `certificateDate`, `createdAt`, `createdAtMs`.

---

## 4. Menu & Aksi Dashboard Member (3 Menu Utama)

Pada tampilan setelah member terdeteksi, tersedia 3 kartu menu aksi utama:
1. **Presensi Hari Ini**: Konfirmasi kehadiran kelas rutin aktif sesuai jadwal sesi hari ini.
2. **Presensi Mengulang**: Konfirmasi kehadiran remedial / retake materi pertemuan tertentu.
3. **Cetak Sertifikat**: Akses mandiri untuk mengunduh kembali sertifikat kelulusan resmi Dialogika (resolusi cetak PNG 2000×1414 px & SVG vektor) jika sertifikat hilang atau membutuhkan salinan baru.

### Perubahan Visual yang Disengaja (UI Refresh — atas permintaan developer)

Tidak ada fitur yang ditambah atau dikurangi; hanya hierarki visual yang diubah (referensi: pola aplikasi presensi *mobile-first*):

- **Header profil ringkas**: avatar, nama, kota, dan WhatsApp dalam satu baris, plus tombol `Ganti Akun`.
- **Kartu hero (aksi utama)**: *Presensi Hari Ini* berwarna navy dengan tombol kuning (brand Dialogika). Tanggal dan program kini tampil di kartu ini.
- **Tile sekunder**: *Presensi Mengulang* dan *Cetak Sertifikat* menjadi tile ringkas (ikon, judul, subjudul) tanpa paragraf dan daftar *perks*.
- **Satu palet**: navy + kuning; amber hanya untuk konteks "mengulang". Aksen hijau/biru pada form & riwayat diseragamkan ke navy.
- Badge *Status Kehadiran* pada ringkasan form kini berbentuk *pill* (sebelumnya melebar seperti input).
- Seluruh ID DOM (`btnOpenRegularForm`, `btnOpenRepeatForm`, `btnOpenCertificateForm`, `btnLogoutMember`, `memberFormMount`, `personalHistoryMount`, `historyCountBadge`, `cardSelect*`) dipertahankan, sehingga orchestrator dan repository tidak berubah.

---

## 5. Rute Publik & URL Routing

Dikonfigurasi di `firebase.json`:
- **URL Publik Bersih**: `/presensi-member` & `/member-attendance`
- **Tujuan Rewrites**: `/pages/product/member-attendance/index.html`
- **Routing Protection**: Akses langsung ke `/presensi-member.html` dialihkan via 301 redirect ke `/presensi-member`, dan rute `/presensi-member` dilayani langsung via Firebase rewrites tanpa redirect perantara.
- **URL Auto-Normalization**: `member-attendance.js` otomatis merapikan URL address bar ke `/presensi-member` via `window.history.replaceState`.

---

## 6. Standar Layout & Navigasi (Dialogika Portal Alignment)

- **Header / Brand Navbar**:
  - Mengikuti standar layout portal Dialogika (`top-bar` / `element/topbar.js`).
  - Container `.attendance-navbar-content` menggunakan `width: 100%; max-width: 100%; display: flex; justify-content: space-between; align-items: center;`.
  - Brand Dialogika (`white-logo.webp` + Portal Presensi) berada di **sisi paling kiri**.
  - Tombol navigasi mode tab (`Presensi`, `Rekap Hari Ini`, `Data Mengulang`) berada di **sisi paling kanan**.
  - Responsif dengan padding fluida `clamp(1rem, 3.5vw, 2.5rem)` dan adaptif untuk perangkat mobile tanpa horizontal overflow.


