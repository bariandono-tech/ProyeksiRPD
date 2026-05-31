# SiP3A — Workflow / Alur Kerja

**Sistem Peramalan dan Proyeksi Pelaksanaan Anggaran Satker**

Versi 1.0 · 31 Mei 2026

---

## 1. Aktor

| Aktor | Peran |
|---|---|
| **Operator Satker** | Mengisi & memperbarui proyeksi per RO/Sub RO + keterangan |
| **Verifikator** | Memeriksa kewajaran proyeksi vs sisa anggaran, memberi catatan |
| **Pimpinan** | Melihat rekap triwulan, persentase akumulatif, dan deviasi |
| **Sistem (cron)** | Menyinkronkan Pagu/Realisasi/Sisa dari Google Sheet |

---

## 2. Alur utama (end‑to‑end)

```
(1) Login satker
        │
        ▼
(2) Sinkron data aktual ──────────────┐
   (Pagu, Realisasi, Sisa dari Sheet) │  otomatis (cron) atau tombol manual
        │                             │
        ▼                             ▼
(3) Pilih Tahun + RO / Sub RO   [actual_snapshot ter‑update]
        │
        ▼
(4) Isi proyeksi bulanan (Jan–Des) per Sub RO + Keterangan
        │
        ▼
(5) Sistem hitung otomatis:
      • total parsial per bulan
      • akumulasi s.d. bulan tsb
      • total per triwulan
      • akumulasi s.d. bulan terakhir triwulan
      • % proyeksi akumulatif (akumulasi ÷ pagu)
        │
        ▼
(6) Tampilkan berdampingan dengan SISA ANGGARAN aktual (dari Sheet)
        │
        ▼
(7) Rekap Triwulan + grafik proyeksi vs realisasi
        │
        ▼
(8) Verifikator/pimpinan review → catatan → finalisasi
```

---

## 3. Alur rinci per langkah

### Langkah 1 — Login
Operator masuk via Supabase Auth. RLS memastikan ia hanya melihat data satkernya.

### Langkah 2 — Sinkronisasi data aktual (Pagu, Realisasi, Sisa)
- **Otomatis:** Vercel Cron memanggil `/api/actual/sync` (mis. tiap jam). Route handler memanggil Apps Script `?api=1`, mengambil `records`, memetakan ke `ref_ro` berdasarkan `(tahun, kro, ro, sub_ro, jenis_belanja)`, dan meng‑upsert `actual_snapshot` (pagu, realisasi, bulan_ini, sisa = pagu − realisasi).
- **Manual:** tombol "Sinkron Data Aktual" memicu route yang sama; status & waktu sinkron ditampilkan ("terakhir disinkron: …").

### Langkah 3 — Pilih konteks
User memilih **Tahun Anggaran** dan menelusuri pohon **Program → Kegiatan → KRO → RO → Sub RO**. Grid proyeksi dibuka pada level **RO/Sub RO** (level terendah yang diramalkan).

### Langkah 4 — Input proyeksi + keterangan
Untuk setiap baris Sub RO, operator mengisi **nilai proyeksi parsial per bulan** (Jan–Des) pada kolom yang dapat di‑edit, plus kolom **Keterangan** (mis. "lelang selesai Maret, pembayaran termin Mei–Agustus"). Penyimpanan via Server Action → tabel `proyeksi` (upsert per `ref_ro_id, tahun, bulan`).

Aturan input:
- Nilai bersifat **parsial** (rencana penyerapan pada bulan itu saja), bukan kumulatif.
- Sistem memberi peringatan lunak bila **akumulasi proyeksi > pagu**.
- Sistem menandai bila **proyeksi bulan yang sudah lewat ≠ realisasi aktual** (deviasi rencana).

### Langkah 5 — Perhitungan otomatis
Begitu nilai disimpan, view `v_proyeksi_rekap` menghitung ulang (lihat formula di `PRD.md`):

| Kolom hasil | Definisi singkat |
|---|---|
| Total parsial bulan | nilai proyeksi bulan tsb |
| Akumulasi s.d. bulan | Σ proyeksi Jan…bulan tsb |
| Total triwulan | Σ proyeksi 3 bulan dalam triwulan |
| Akumulasi s.d. akhir triwulan | akumulasi pada bulan terakhir triwulan |
| % proyeksi akumulatif | akumulasi ÷ pagu × 100 |

### Langkah 6 — Tampilan sisa anggaran aktual
Di sisi/baris yang sama, sistem menampilkan **Pagu**, **Realisasi (akumulatif)**, dan **Sisa = Pagu − Realisasi** yang **bersumber dari `actual_snapshot`** (Sheet). Ini memberi operator konteks: "berapa yang masih bisa direncanakan".

### Langkah 7 — Rekap triwulan & grafik
Halaman rekap menyajikan tabel ringkas per triwulan (TW I–IV) untuk seluruh RO/Sub RO, beserta:
- Kurva **S‑curve**: proyeksi akumulatif vs realisasi akumulatif per bulan.
- Bar per jenis belanja (51/52/53/57).
- Indikator deviasi proyeksi vs realisasi.

### Langkah 8 — Review & finalisasi
Verifikator meninjau, menambahkan catatan pada keterangan, dan menandai proyeksi triwulan sebagai "terverifikasi". Pimpinan melihat ringkasan akhir.

---

## 4. Tabel perhitungan (contoh tata letak untuk satu Sub RO)

| Bulan | Proyeksi (parsial) | Akumulasi proyeksi | % akum. (÷pagu) | Realisasi aktual | Sisa anggaran |
|---|---|---|---|---|---|
| Jan | 10 | 10 | 5% | 8 | 192 |
| Feb | 15 | 25 | 12,5% | 20 | 180 |
| **TW I (Mar)** | 20 | **45** | **22,5%** | 40 | 160 |
| Apr | 25 | 70 | 35% | … | … |
| … | … | … | … | … | … |
| **TW II (Jun)** | … | **akum. s.d. Jun** | … | … | … |

> Baris triwulan menampilkan **total triwulan** (Σ 3 bulan) sekaligus **akumulasi s.d. bulan terakhir triwulan**. Pagu = 200 pada contoh ini.

---

## 5. Alur sinkronisasi (sequence)

```
Vercel Cron / Tombol
      │  POST /api/actual/sync
      ▼
Route Handler (server) ──GET?api=1──▶ Apps Script Web App ──read──▶ Google Sheet DATA
      │  ◀──── JSON {records, summary} ───────────────────────────────
      │
      ├─ map record → ref_ro (tahun,kro,ro,sub_ro,jenis_belanja)
      ├─ upsert actual_snapshot (pagu, realisasi, bulan_ini, sisa)
      └─ catat synced_at
      ▼
Supabase ◀── upsert ──  (RLS service role)
      ▼
UI menampilkan "Terakhir disinkron: hh:mm"
```

---

## 6. Penanganan kasus khusus

- **RO/Sub RO baru di sheet** yang belum ada di `ref_ro` → proses sync membuat baris master baru (auto‑seed) dan menandainya untuk ditinjau.
- **Pagu berubah (revisi DIPA)** → sync memperbarui `pagu`; persentase akumulatif otomatis menyesuaikan.
- **Proyeksi melebihi pagu** → diizinkan disimpan tapi diberi badge merah; tidak memblokir agar fleksibel saat perencanaan awal.
- **Bulan terlewat tanpa proyeksi** → dianggap 0 pada akumulasi, ditandai agar operator melengkapi.

---

*Lihat `ARCHITECTURE.md` untuk struktur teknis dan `PRD.md` untuk kebutuhan & formula lengkap.*
