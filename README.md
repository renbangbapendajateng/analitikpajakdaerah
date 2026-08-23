# Analitik Data Pajak Daerah — Panduan Deploy Komprehensif (GitHub + Firebase)

Status aplikasi saat ini: modul **PKB** (Capaian, Per Komponen, Opsen) dan **BBNKB** (Capaian,
Ranking) sudah jalan. Modul **PAP** menyusul setelah format datanya diupload.

---

## 0. Gambaran Umum Arsitektur

```
Browser (siapa pun yang buka link) 
   │
   ├─ index.html (HTML+CSS+JS, 1 file, di-hosting statis)
   │     dijalankan oleh:  Firebase Hosting  (gratis, CDN global)
   │
   └─ Data (upload Excel, target, dll)
         disimpan di:      Firestore  (database NoSQL, gratis s.d. kuota tertentu)
         akses dijaga oleh: Firebase Authentication (Google Sign-In)
                              + Firestore Security Rules

Kode disimpan & di-versi-kontrol di:  GitHub
Setiap `git push` ke branch `main`  →  GitHub Actions  →  otomatis deploy ke Firebase Hosting
```

Tidak ada server backend yang perlu Bapak kelola — semuanya "serverless" (Google yang urus
infrastrukturnya). Biaya: **Rp0** untuk skala pemakaian internal Bapenda (jauh di bawah kuota
gratis Firebase Spark plan).

---

## 1. Persiapan Akun & Tools

Yang dibutuhkan sebelum mulai:
- Akun Google (untuk Firebase Console — bisa pakai akun pribadi atau akun instansi).
- Akun GitHub (gratis, buat di github.com kalau belum punya).
- Node.js terpasang di komputer Bapak (untuk menjalankan `firebase-tools`). Cek dengan:
  ```bash
  node --version
  ```
  Kalau belum ada, unduh di nodejs.org (pilih versi LTS).

---

## 2. Buat & Konfigurasi Firebase Project

1. Buka console.firebase.google.com → **Add project**.
2. Nama project, misalnya `analitik-pajak-jateng`. Google Analytics boleh dimatikan (tidak perlu).
3. Setelah project dibuat, aktifkan dua layanan:
   - **Build -> Firestore Database -> Create database**
     - Mode: **Production** (bukan Test mode — supaya tidak terbuka bebas)
     - Lokasi: `asia-southeast2 (Jakarta)`
   - **Build -> Authentication -> Sign-in method -> aktifkan "Google"**
4. Daftarkan aplikasi web: **Project settings (ikon gerigi) -> General -> scroll ke "Your apps"
   -> klik ikon `</>` (Web)** -> beri nama (mis. "Analitik Pajak Web") -> **Register app**.
5. Firebase akan menampilkan blok kode `firebaseConfig` — **salin seluruh isinya**, akan
   dipakai di langkah 4.

---

## 3. Siapkan File di Komputer Bapak

1. Ekstrak `deploy-kit.zip` yang saya kirim ke folder kerja, misal `D:\analitik-pajak\`.
2. Isinya:
   ```
   analitik-pajak/
   |- public/index.html   <- aplikasi (sudah otomatis mendukung Firebase)
   |- firebase.json
   |- firestore.rules
   |- firestore.indexes.json
   |- .firebaserc
   |- .github/workflows/  <- untuk auto-deploy nanti
   ```

---

## 4. Isi Config Firebase ke Aplikasi

1. Buka `public/index.html` dengan text editor (Notepad++, VS Code, dll).
2. Cari blok ini (dekat baris atas, di dalam `<script type="module">`):
   ```js
   const firebaseConfig = {
     apiKey: "GANTI_DENGAN_API_KEY",
     authDomain: "GANTI.firebaseapp.com",
     projectId: "GANTI_PROJECT_ID",
     storageBucket: "GANTI.appspot.com",
     messagingSenderId: "GANTI",
     appId: "GANTI",
   };
   ```
3. Ganti seluruh isinya dengan config asli dari Langkah 2.5 tadi. Simpan file.
4. Buka `.firebaserc`, ganti `GANTI_DENGAN_PROJECT_ID` dengan Project ID Firebase Bapak
   (terlihat di Firebase Console, biasanya sama dengan nama project tapi huruf kecil semua).
5. Buka `.github/workflows/deploy.yml` dan `.github/workflows/deploy-preview.yml`, ganti
   `GANTI_DENGAN_PROJECT_ID` di masing-masing dengan Project ID yang sama.

> Config `apiKey` dkk. **aman** ditaruh di kode publik — ini bukan password. Proteksi data
> sesungguhnya ada di **Firestore Security Rules** (langkah 6), bukan di config ini.

---

## 5. Deploy Manual Pertama Kali

Buka terminal/Command Prompt di folder `analitik-pajak/`:

```bash
npm install -g firebase-tools      # sekali saja, install CLI Firebase
firebase login                     # akan buka browser, login pakai akun Google project ini
firebase deploy --only firestore:rules,hosting
```

Kalau berhasil, akan muncul URL seperti:
```
Hosting URL: https://analitik-pajak-jateng.web.app
```
Buka URL itu — akan muncul halaman **login dengan Google**. Ini tandanya sudah live.

---

## 6. Firestore Security Rules — Siapa yang Boleh Akses

File `firestore.rules` sudah saya siapkan dengan 2 mode:

**Mode dasar (aktif secara default):** siapa pun yang berhasil login dengan akun Google
(akun apa saja) boleh baca/tulis data.

**Mode lebih ketat (disarankan untuk produksi):** hanya email domain instansi
(`@jatengprov.go.id`) yang boleh akses. Untuk mengaktifkan:
1. Buka `firestore.rules`, cari baris:
   ```
   allow read, write: if isSignedIn();
   ```
2. Ganti jadi:
   ```
   allow read, write: if isAllowedDomain();
   ```
3. Deploy ulang rules-nya saja (cepat, tidak perlu deploy ulang semua):
   ```bash
   firebase deploy --only firestore:rules
   ```

---

## 7. Hubungkan ke GitHub (Supaya Auto-Deploy)

1. Buat repository baru di GitHub (bisa **Private**, supaya kode tidak terlihat publik —
   meski sebenarnya tidak masalah karena tidak ada rahasia di dalamnya).
2. Push isi folder ke repo:
   ```bash
   cd analitik-pajak
   git init
   git add .
   git commit -m "Setup awal: Analitik Pajak Daerah (PKB + BBNKB)"
   git branch -M main
   git remote add origin https://github.com/USERNAME/NAMA_REPO.git
   git push -u origin main
   ```
3. Hubungkan GitHub <-> Firebase (perintah ini otomatis membuat secret `FIREBASE_SERVICE_ACCOUNT`
   di repo GitHub dan menyiapkan workflow deploy):
   ```bash
   firebase init hosting:github
   ```
   Ikuti pertanyaannya (pilih repo GitHub yang baru dibuat, izinkan deploy saat push ke `main`).

Setelah ini: **setiap `git push` ke branch `main` otomatis ter-deploy ke Firebase Hosting**
dalam 1-2 menit, tanpa perlu jalankan `firebase deploy` manual lagi.

---

## 8. Alur Kerja Harian Setelah Live

**Untuk perubahan kecil (revisi tampilan, perbaikan bug):**
```bash
git add .
git commit -m "Perbaikan: <deskripsi singkat>"
git push
```
Selesai — otomatis ter-deploy.

**Untuk fitur besar (mis. modul PAP nanti), gunakan Pull Request supaya bisa dicek dulu:**
```bash
git checkout -b fitur-pap
# ...saya edit index.html untuk fitur PAP...
git add .
git commit -m "Tambah modul PAP"
git push origin fitur-pap
```
Lalu buka Pull Request di GitHub dari `fitur-pap` ke `main`. Workflow
`deploy-preview.yml` otomatis membuat **URL uji coba terpisah** (tidak menimpa situs yang
sedang dipakai staf lain) — cek dulu di situ, baru **merge** ke `main` kalau sudah oke.

---

## 9. Backup Data Berkala

Sebelum perubahan besar atau secara rutin (mis. tiap akhir bulan), backup Firestore:
```bash
gcloud firestore export gs://NAMA_BUCKET/backup-$(date +%Y%m%d) --project=NAMA_PROJECT_ID
```
Butuh `gcloud` CLI (Google Cloud SDK) dan sebuah Cloud Storage bucket — kalau belum ada,
tinggal buat sekali di console.cloud.google.com/storage.
Bisa dijadwalkan otomatis lewat Cloud Scheduler kalau nanti volumenya besar.

---

## 10. Siapa yang Bisa Akses Apa

| Peran | Akses |
|---|---|
| Staf biasa (login Google, domain diizinkan) | Buka aplikasi, upload data, atur target, kelola data lewat menu di aplikasi |
| Admin/developer (Bapak) | Semua di atas + akses Firebase Console (lihat data mentah, atur rules, lihat log, kelola siapa saja yang bisa login) |
| Publik / tidak login | **Tidak bisa akses apa pun** — halaman login akan menghalangi |

Kalau ingin membatasi siapa yang boleh login (bukan cuma domain email), bisa tambah
allowlist email spesifik di Firestore Rules — tinggal bilang kalau perlu, saya siapkan.

---

## 11. Troubleshooting Umum

| Gejala | Kemungkinan Penyebab | Solusi |
|---|---|---|
| Halaman putih / error di console browser | `firebaseConfig` belum diisi / salah salin | Cek ulang Langkah 4 |
| Login Google gagal / "unauthorized domain" | Domain hosting belum diizinkan | Firebase Console -> Authentication -> Settings -> Authorized domains -> tambahkan domain hosting Bapak |
| Data tidak tersimpan / error "permission denied" | Firestore Rules terlalu ketat atau belum login | Cek Langkah 6, pastikan email yang dipakai sesuai domain yang diizinkan |
| `firebase deploy` gagal "not logged in" | Sesi CLI habis | Jalankan `firebase login` ulang |
| GitHub Actions gagal (tanda silang merah) | Secret `FIREBASE_SERVICE_ACCOUNT` belum ada / salah | Ulangi `firebase init hosting:github` |

---

## 12. Ringkasan Perintah Paling Sering Dipakai

```bash
firebase login                                    # login sekali di awal / kalau sesi habis
firebase deploy --only hosting                    # deploy manual (biasanya tak perlu, sudah otomatis)
firebase deploy --only firestore:rules             # deploy ulang rules setelah diedit
git add . && git commit -m "pesan" && git push     # cara paling umum: commit lalu push, sisanya otomatis
```
