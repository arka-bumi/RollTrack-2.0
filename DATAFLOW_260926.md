# DATAFLOW_260926 — Alur Data Bisnis (RollTrack)
*Aturan main untuk semua pihak, 26-09-2026. Selaras `PRD_260926.md`. Detail mesin ada di `ARCH_260926.md`. Menggantikan `Data Flow_Alur Bisnis.md`.*

## 1. Latar Belakang
- Spreadsheet inventory ±16MB (tab History 22.395 baris × 25 kolom) lambat dibuka; tab master + summary hanya ratusan–5.000 baris.
- Dua file inventory (MALILKIDS / ANAKKECILKU); data antar merek tidak bercampur.
- HP harus real-time; pihak lain membaca via spreadsheet.

## 2. Peran Data
- **Operasional (cepat):** sisa meter, status (`1 Roll/Sisa/Habis`), lokasi gudang. HP baca/tulis di sini; dashboard menampilkan data ini. Awalnya mirror spreadsheet, lalu hitung mandiri poin UI (Jumlah Roll, Roll Detail, Meter, Lokasi Gudang); tiap Sinkron Database mirror ulang dan bila beda **spreadsheet menang**.
- **Spreadsheet (hitung + audit):** konversi pcs/estimasi + kebenaran audit. Hasil akhir konvergen dengan operasional (beda hanya latensi/offline/konflik).

## 3. Master (hidden, read-only)
F. Database Kain (lookup SKU Kain → Jenis Kain, untuk label) · F. Database Produk (dropdown cutting saja: filter Aktif per merek, dedup per SKU) · History Kain Datang (acuan roll saat scan/tulis) · **Master Gudang** (daftar gudang, identik di kedua merek — dipakai Pindah Gudang & filter lokasi) · **Pengumuman** (tab khusus, hanya di file Malilkids — kartu quotes 5.1/5.2 menampilkan satu isi + atribusi aktif). Tidak ada tambah/ubah dari HP.

## 4. Cutting (wajib pilih merek dulu)
Toggle merek → produksi/non-produksi → tanggal potong + NN → cari/pilih produk by SKU/nama (auto Nama Produk + Varian size) → Kode `{M|A}{DDMMYY}-{SKU_PRODUK}-{NN}` (`M`=Malilkids, `A`=Anakkecilku) → **kunci kode** → "Tambah Roll Kain" → scan/tulis Roll ID (cek VALID + kartu: Roll ID, Jenis•Motif•SKU, Sisa Meter Terkini saja — tanpa sisa Kg, tanpa gudang; bebas kain manapun milik merek aktif) → isi Panjang(m)/Estimasi(Kg) + radio Sisa/Habis + preview live + waste/reject → Potong (baris ke-1) → ulangi Tambah Roll untuk roll berikut → **Simpan Transaksi** = N baris 1 Kode Produksi. Duplikat (merek, hari, SKU, NN) ditolak; lebih dari sisa tetap tercatat. **Meter diutamakan**; hanya-Kg dikonversi via `rasio_konversi_meter` (DB Kain). `Tim Cutting` = nama orang. Append batch ke file merek terkait. Sisa berjalan meter saja; sisa Kg tidak dihitung/ditampilkan. Setelah Simpan berhasil (3.1/3.5), kembali ke 3.1 fresh (reset batch, tanggal default hari ini) + toast sukses; gagal tetap di layar.

## 5. Pindah Gudang (bulk, lintas merek, gudang bersama)
**Pindah massal:** satu eksekusi memindahkan banyak roll **dalam satu asal gudang → satu tujuan gudang**. Gudang asal terkunci. Tanpa ruangan. **Tanpa pemilihan merek** — daftar gudang SAMA untuk kedua merek (Master Gudang identik), roll dari Malilkids maupun Anakkecilku dapat ikut pindah, dan `brand` tiap roll diturunkan dari Roll ID-nya.

Konsekuensi teknis: `location_logs` mencatat **satu baris per roll** (bukan satu baris per batch pindah),share gudang yang sama;endpoint transfer menerima array roll. `tanggal_pindah` default hari ini, bisa diubah staff via tanggal di 2.1/2.4. Setelah Eksekusi berhasil, kembali ke 2.1 fresh (reset daftar, tanggal default hari ini) + toast sukses; gagal tetap di 2.4.

## 6. Tiga Jalur Sync (nama dikunci)
- **Selalu tersinkron** (otomatis, tanpa tombol, semua role): push log realtime + pull baris hari-ini dari sheet (termasuk kolom sheet-side seperti Pcs Datang) + retry antrean umur berapa pun sampai terkirim.
- **Sinkron Database** (tombol, Admin/Super Admin, ringan, boleh kapan pun): master + 2 tab Summary (kolom sempit).
- **Sinkron Penuh** (tombol, Admin/Super Admin, berat, jarang, malam hari): tarik full History 22rb baris bertahap untuk rekonsiliasi; **Sheets menang** bila selisih. Tidak mendorong data harian, tidak menyentuh antrean.

## 7. List Roll per SKU (layar 4.1)
Disintesis dengan group-by kolom SKU dari baris **Summary By Roll ID** (tab `By Code` di Sheets) di middleware — tanpa rumus/baca tambahan di Sheets: `{SKU, total_roll, total_sisa, rolls:[{roll_id, sisa, status, gudang}]}`.
