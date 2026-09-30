# Rencana Implementasi Monorepo RollTrack (Build Plan)

Dokumen ini menjadi acuan kerja end-to-end pembangunan sistem **Malilkids RollTrack** (Backend Node.js + Frontend Mobile Expo React Native + Integrasi Google Sheets API + MongoDB).

---

## 1. Arsitektur Proyek (Monorepo)

```text
Malilkids_RollTrack/
  ├── shared/
  │    └── domain/                     # Logika bisnis MURNI (satu sumber kebenaran)
  │         ├── cutting.ts             # calculateCutRemaining (sisa meter + status roll)
  │         ├── productionCode.ts      # formatProductionCode {M|A}{DDMMYY}-{SKU}-{NN}
  │         ├── inventory.ts           # aggregateSkuInventory (Summary by SKU)
  │         └── index.ts
  │
  ├── rolltrack-backend/               # Node.js TypeScript API (Express)
  │    ├── src/
  │    │    ├── adapters/
  │    │    │    └── sheetsAdapter.ts  # Google Sheets API v4: baca/append baris, ambil judul
  │    │    ├── mappers/
  │    │    │    └── sheetsRows.ts     # Baris Sheet -> record MongoDB (murni, tanpa I/O)
  │    │    ├── models/                # Mongoose Models (Technical_Pencocokan Kolom Data.xlsx)
  │    │    │    ├── FabricRoll.ts     # History Datang Kain
  │    │    │    ├── UsageLog.ts       # History Penggunaan Kain
  │    │    │    ├── LocationLog.ts    # Pindah Gudang (tanpa ruangan/rak)
  │    │    │    ├── FabricMaster.ts   # F. Database Kain
  │    │    │    ├── FabricProduct.ts  # F. Database Produk
  │    │    │    ├── Warehouse.ts      # Master Gudang (lintas merek)
  │    │    │    ├── User.ts           # Operator, Admin, Super Admin + PIN
  │    │    │    └── Settings.ts       # Kredensial Service Account terenkripsi (AES-256)
  │    │    ├── services/              # Deep module: satu use case per file
  │    │    │    ├── cuttingService.ts  # Validasi + orkestrasi potong kain
  │    │    │    ├── transferService.ts # Validasi + orkestrasi pindah gudang bulk
  │    │    │    ├── sheetsSync.ts      # Sinkron Database / Sinkron Penuh
  │    │    │    └── crypto.ts          # Enkripsi/dekripsi Service Account
  │    │    ├── routes/                 # Tipis: transport HTTP -> services
  │    │    └── server.ts
  │    ├── test/domain.test.ts          # Runnable check: shared/domain + mappers
  │    └── package.json
  │
  ├── rolltrack-mobile/                # Expo React Native App (TypeScript)
  │    ├── metro.config.js              # watchFolders repo root (agar shared/ ter-resolve)
  │    ├── src/
  │    │    ├── domain/index.ts         # Re-export shared/domain
  │    │    ├── components/ui.tsx       # Card, Button, BrandToggle, StatusBadge
  │    │    ├── screens/                # 22 Layar Presisi (1.1 s.d 5.6)
  │    │    ├── data/                   # Offline-first queue & AsyncStorage state
  │    │    ├── types.ts                # Tipe UI, melimpah ke shared/domain
  │    │    └── theme.ts                # Token Stitch Light
  │    └── App.tsx                      # Tab: Beranda, Pindah, Cutting, Kain, Profil
  │
  ├── Google Stitch UI (Light)/        # 22 folder layar referensi HTML & screenshot
  ├── Technical_Pencocokan Kolom Data.xlsx
  └── PRD_260926.md, ARCH_260926.md, DESIGN_260926.md, BUILD_PLAN.md
```

### Aturan Seam

| Lapis | Isi | Larangan |
|-------|-----|----------|
| `shared/domain` | Perhitungan murni, tanpa I/O, tanpa library eksternal | Tidak boleh menyentuh Mongoose / React / googleapis |
| `mappers`, `adapters` | Pemetaan baris & transport ke pihak luar | Tidak boleh memuat aturan bisnis |
| `services` | Use case: validasi -> hitung (`shared/domain`) -> orkestrasi persistensi | Tidak boleh menyentuh `req`/`res` |
| `routes` | Transport HTTP saja | Tidak boleh query langsung ke Mongoose |
| `mobile/src/screens` | Tampilan saja | Memanggil `shared/domain`, tidak menyalin logikanya |

---

## 2. Fase Build (wajib berurutan — tidak sekaligus)

> Catatan 29-09-2026: prototipe awal (`rolltrack-backend`, `rolltrack-mobile`, `shared`, `scripts`) **dihapus** dari repo agar build dimulai dari dokumen final. Riwayatnya tetap ada di git (`8d47a34`, `981cf22`).
>
> Aturan fase: fase N+1 **tidak dimulai** sebelum kriteria keluar fase N lulus. Setiap fase diakhiri commit + demo singkat. Acuan dokumen per fase ditulis eksplisit agar AI mana pun bisa mengerjakan satu fase tanpa membaca seluruh repo.

### P0 — Fondasi (skeleton + infra verifikasi)

- Monorepo: `rolltrack-backend` (TS, Express, Mongoose, googleapis, dotenv), `rolltrack-mobile` (Expo), `shared/domain` (murni, tanpa I/O).
- Target deploy: backend serverless di **Vercel gratis** (`vercel.json` + entry serverless), DB **MongoDB Atlas**, API URL publik HTTPS; mobile jadi **APK internal**.
- `.env.example` (`JWT_SECRET`, `ENCRYPTION_KEY`, `SUPERADMIN_PIN`, `GOOGLE_SA_JSON`, `MONGODB_URI`, `EXPO_PUBLIC_API_URL`); `.env` di-gitignore.
- **Keluar:** `npm run typecheck`, `npm test`, `npx tsc --noEmit` (mobile) semuanya jalan.

### P1 — Auth + Users backend (AUTH L1–L5, L7–L8)

- `User`: `pin_hash` (bcryptjs cost 10, `select: false`), `must_change_pin`; tanpa `User.brand`.
- `POST /auth/login` (pesan error identik), JWT 8 jam, `requireAuth` global, `requireRole` (sync/config/users), rate limit 5/15 mnt, hapus fallback `ENCRYPTION_KEY`.
- Seed Super Admin (`SUPERADMIN_PIN`, default `000000`, idempoten) + endpoint users + `change-pin`.
- **Keluar:** AUTH V1–V6, V10, V13, V14.

### P2 — Sinkron Database + Masters (ARCH §3/§5, PRD §3.2)

- `crypto.ts` (AES-256-CBC), `sheetsAdapter.ts` (baca A2:ZZ, tulis `USER_ENTERED`), `sheetsRows.ts` (**kolom ikut excel**, bukan tebakan).
- `POST /sync/database`: Kain, Produk (gabung relasi per SKU), Gudang, `Pengumuman` (Malilkids saja); gagal satu tab tidak hentikan lain.
- `GET /masters?brand=` (produk hanya `Active`, masters urut `no_index`) + `GET /announcement`.
- **Keluar:** masters + produk + pengumuman tampil benar per merek di Mongo.

### P3 — Pindah Gudang end-to-end (PRD §3.4, DESIGN 2.1–2.4)

- `transferService`: bulk 1 asal terkunci → 1 tujuan, lintas merek (`brand` dari roll), lewati tak dikenal/sudah-di-tujuan, auto-daftar gudang baru; `POST /rolls/transfer`.
- Mobile 2.1–2.4 (tanggal default hari ini bisa diubah, asal terkunci, tambah roll gudang sama) + antrean offline; append realtime ke sheet saat online (`is_synced` langsung true; gagal → antre, terkirim otomatis tanpa tombol).
- **Keluar:** AUTH V9, V11 (split 3:2 per spreadsheet).

### P4 — Cutting end-to-end (PRD §3.3, DATAFLOW §4, DESIGN 3.1–3.5)

- `cuttingService`: 1 transaksi = N roll = N baris 1 Kode Produksi; toggle merek → DB merek itu; meter diutamakan, Kg via `rasio_konversi_meter`; `tim_cutting` dari token; blokir roll `Habis`; pesan mismatch merek.
- Mobile 3.1 (produksi/non-produksi, tanggal+NN, cari SKU, auto nama+varian, kunci kode, Tambah Roll loop, VALID + jenis/motif/SKU/sisa, Sisa/Habis, waste/reject) + 3.2/3.3/3.4/3.5 + antrean offline.
- **Keluar:** AUTH V7, V8, V12 + skenario 1 kode 2 roll → 2 baris di sheet.

### P5 — Dashboard Kain + QR (PRD §3.5/§4, DESIGN 4.1–4.5)

- 4.1 SKU (group-by, KPI Roll/Roll Detail/Meter) + 4.2 filter + 4.3 Roll + 4.4 filter + 4.5 checklist; toggle merek = filter tampilan; baca operasional terkini.
- Download QR (**Admin/Super Admin saja**): 1 file PDF A4 landscape per unduhan, N/4 halaman grid 2×2; per kotak QR + Roll ID + tanggal/jenis/motif/ukuran/no.urut (ukuran dari `ukuran_asal`+`satuan_asal`).
- **Keluar:** angka dashboard = operasional terkini per merek; QR terunduh berisi Roll ID.

### P6 — Profil + Admin (PRD §3.6, DESIGN 5.1–5.6, AUTH L9)

- 5.1/5.2 (Keluar vs Log Out, kartu pengumuman dari `GET /announcement` + fallback mockup), 5.3/5.6 (tombol Sinkron Database/Penuh; 5.6 + ganti PIN di atas Daftar Admin), halaman Koneksi Spreadsheet terpisah (ID + Uji Koneksi), 5.4/5.5 (tambah/edit/hapus user, satu modal, role ikut pemanggil).
- **Keluar:** AUTH V13; admin kelola staff dari HP tanpa sentuh server.

### P7 — Hardening + UAT

- Offline end-to-end di perangkat nyata (antrean → backend → Sheets), AUTH V1–V14 penuh, DoD AUTH §8, kriteria PRD §6 (< 4 detik/scan, lokasi 100% cocok, angka identik saat normal). UAT pakai **data asli** (keputusan user) — baris uji akan masuk spreadsheet produksi.
- **Keluar:** semua checklist hijau → siap produksi.

---

## 3. Sub-fase hemat token (1 sub-fase = 1 sesi AI)

Fase P0–P7 di atas terlalu besar untuk satu sesi. Pecahannya di bawah — tiap baris dirancang
selesai dalam satu sesi tanpa kehabisan token. Aturan: **1 sesi = 1 sub-fase, tutup dengan commit.**
Prompt mulai sesi baru (copy-paste, ganti ID):
`Kerjakan <ID> di BUILD_PLAN §3. Acuan: <dokumen di kolom Acuan>. Jangan kerjakan sub-fase lain. Tutup dengan kriteria keluar + commit.`

| ID | Isi | Acuan | Keluar |
|---|---|---|---|
| P0a | Skeleton: folder monorepo, `package.json`, `tsconfig`, `.env.example`, `vercel.json`, `metro.config.js` | BUILD §1, ARCH §6 | Struktur ada |
| P0b | Skrip `typecheck`/`test` hijau di repo kosong | BUILD P0 | 3 perintah jalan |
| P1a | Model `User` (`pin_hash`, `must_change_pin`, tanpa `brand`/`username`) + `seed-superadmin.ts` | AUTH §5, L7 | Seed jalan, superadmin tercipta |
| P1b | `POST /auth/login` + JWT 8 jam + `requireAuth`/`requireRole` + rate limit | AUTH L1–L3, L5 | V1, V4–V6 |
| P1c | Guard endpoint + `users` CRUD + `DELETE` (hapus login, history tetap) + `change-pin` | AUTH L3, L8, D18 | V2, V3, V13, V14 |
| P2a | `crypto.ts` + `sheetsAdapter.ts` (baca A2:ZZ, tulis `USER_ENTERED`) | ARCH §3–§5 | Adapter baca 1 tab |
| P2b | `sheetsRows.ts` (**kolom ikut xlsx**) + test mapper | xlsx, ARCH §1.9 | Test mapper hijau |
| P2c | `POST /sync/database` + `GET /masters` + `GET /announcement` | ARCH §4–§5, PRD §3.2 | Master per merek tampil |
| P3a | `transferService` + `POST /rolls/transfer` + test | PRD §3.4, AUTH V9/V11 | Test split 3:2 hijau |
| P3b | Mobile 2.1–2.4 + antrean offline + append realtime (sukses → kembali ke 2.1 fresh + toast) | DESIGN 2.x | Pindah E2E jalan |
| P4a | `cuttingService` (N roll = N baris) + `POST /rolls/cut` + test | PRD §3.3, DATAFLOW §4 | Test multi-roll hijau |
| P4b | Mobile 3.1 (batch, kunci/buka gembok tanpa reset, Tambah Roll loop, dropdown `{SKU} - {Nama} - {Varian}` + search + urut — abaikan teks mockup tanpa Varian, sukses → kembali ke 3.1 fresh + toast) | DESIGN 3.1 | Batch + kunci + dropdown + kembali 3.1 jalan |
| P4c | Mobile 3.2/3.3/3.4/3.5 (kartu Roll ID + jenis•motif•SKU + Sisa Meter Terkini tanpa Kg + preview live, kunci kode via gembok, waste, non-produksi tanpa NN tulis Tanggal→A Nama→G; abaikan Sisa Kg di mockup — ikut dokumen; sukses 3.5 → kembali ke 3.1 fresh) + offline | DESIGN 3.x | Cutting E2E jalan |
| P5a | Mobile 4.1 + 4.2 (group-by SKU, KPI, filter) | PRD §3.5/§4 | Angka = operasional |
| P5b | Mobile 4.3 + 4.4 (daftar roll, filter) | DESIGN 4.3/4.4 | Daftar + filter jalan |
| P5c | Mobile 4.5 + QR PDF A4 landscape (acuan `contoh-cetak-qr-a4.html`) | DESIGN 4.5 | PDF N/4 halaman |
| P6a | Mobile 1.1 + 1.2 + 5.1 (login, beranda, profil + pengumuman) | DESIGN 1.x/5.1, AUTH L6 | Login E2E jalan |
| P6b | Mobile 5.3/5.6 + 5.4/5.5 + halaman Koneksi + ganti PIN | DESIGN 5.x, AUTH L9 | V13 |
| P7a | Verifikasi V1–V14 + DoD AUTH §8 | AUTH §7–§8 | Checklist hijau |
| P7b | Uji perangkat nyata + APK internal + UAT data asli | PRD §6 | Siap produksi |

## 4. Preview vs distribusi (keputusan user: Expo Go untuk coba, APK untuk pakai)

- **Preview harian (Expo Go):** `npx expo start` di laptop → scan QR-nya dari aplikasi Expo Go di HP (satu WiFi, atau mode tunnel). Bisa karena semua dependensi JS-only (tanpa modul native custom). `EXPO_PUBLIC_API_URL` arahkan ke URL Vercel (tembak production) atau IP LAN laptop (tembak backend lokal).
- **Distribusi (APK internal):** `eas build -p android --profile preview` → file APK dipasang langsung ke HP operator, tanpa Play Store. Hanya dibuat saat P7b atau rilis bertahap per fase yang butuh uji lapangan.
