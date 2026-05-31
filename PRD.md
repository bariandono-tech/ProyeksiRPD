# SiP3A — Project Requirements Document (PRD)

**Sistem Peramalan dan Proyeksi Pelaksanaan Anggaran Satker**

| | |
|---|---|
| Versi | 1.0 (Draft) |
| Tanggal | 31 Mei 2026 |
| Pemilik | Satker (Rudenim Pontianak) |
| Stack | Next.js · Supabase · Vercel · Google Sheet (sumber realisasi) |

---

## 1. Latar belakang & masalah

Satker telah memiliki dashboard MONEV PA + IKPA/NKPA berbasis Google Apps Script yang membaca **Pagu** dan **realisasi aktual** dari Google Sheet. Namun belum ada alat untuk **merencanakan/meramalkan** penyerapan ke depan secara terstruktur per **RO** dan **Sub RO**, lengkap dengan rekap triwulan, akumulasi, dan persentase proyeksi — yang bisa langsung dibandingkan dengan **sisa anggaran aktual**.

SiP3A mengisi celah ini: lapisan **perencanaan/proyeksi** di atas data realisasi yang sudah ada, tanpa mengganggu sumber data aktual.

## 2. Tujuan

1. Memungkinkan operator mengisi proyeksi penyerapan bulanan per RO/Sub RO + keterangan.
2. Menyajikan rekap triwulan otomatis (total triwulan & akumulasi).
3. Menghitung total parsial bulanan, akumulasi s.d. bulan, dan persentase proyeksi akumulatif.
4. Menampilkan sisa anggaran aktual (dari Sheet) berdampingan dengan proyeksi.
5. Menjadi aplikasi full‑stack mandiri (Supabase + Vercel) yang mudah dipelihara.

### Non‑tujuan (out of scope v1)
- Tidak menghitung ulang/menulis ke Google Sheet.
- Tidak menggantikan perhitungan IKPA/NKPA pada dashboard lama.
- Tidak mengelola dokumen pengadaan atau SPM/SP2D.

## 3. Pengguna & peran

| Peran | Hak |
|---|---|
| Operator Satker | CRUD proyeksi & keterangan untuk satkernya |
| Verifikator | Lihat semua, beri catatan, tandai terverifikasi |
| Pimpinan | Lihat rekap & grafik (read‑only) |

## 4. Ruang lingkup fungsional

Hierarki anggaran mengikuti sheet `DATA`: **Program → Kegiatan → KRO → RO → Sub Output (Sub RO) → Komponen → Akun → Item**, dengan **Jenis Belanja** (51 Pegawai, 52 Barang, 53 Modal, 57 Bansos). Proyeksi diisi pada level **RO/Sub RO**.

---

## 5. Kebutuhan fungsional (Functional Requirements)

| ID | Kebutuhan | Prioritas |
|---|---|---|
| FR‑01 | User dapat login sebagai satker (Supabase Auth). | Must |
| FR‑02 | Sistem menampilkan daftar RO & Sub RO untuk tahun terpilih, di‑seed dari sheet. | Must |
| FR‑03 | User dapat mengisi nilai proyeksi **parsial per bulan** (Jan–Des) per Sub RO. | Must |
| FR‑04 | User dapat mengisi **Keterangan** per Sub RO (atau per bulan). | Must |
| FR‑05 | Sistem menghitung **total parsial per bulan**. | Must |
| FR‑06 | Sistem menghitung **akumulasi s.d. bulan tsb**. | Must |
| FR‑07 | Sistem menghitung **total per triwulan** (TW I–IV). | Must |
| FR‑08 | Sistem menghitung **akumulasi s.d. bulan terakhir di triwulan tsb**. | Must |
| FR‑09 | Sistem menghitung **% proyeksi akumulatif** = akumulasi ÷ pagu × 100. | Must |
| FR‑10 | Sistem menampilkan **sisa anggaran aktual** (pagu − realisasi) dari Sheet. | Must |
| FR‑11 | Halaman **rekap triwulan** menggabungkan seluruh RO/Sub RO. | Must |
| FR‑12 | Sinkronisasi data aktual otomatis (cron) + tombol manual. | Must |
| FR‑13 | Grafik S‑curve proyeksi akumulatif vs realisasi akumulatif. | Should |
| FR‑14 | Peringatan bila akumulasi proyeksi > pagu, dan bila proyeksi bulan lewat ≠ realisasi. | Should |
| FR‑15 | Ekspor rekap ke Excel/CSV. | Should |
| FR‑16 | Jejak audit (siapa/kapan mengubah proyeksi). | Should |
| FR‑17 | Verifikator menandai proyeksi triwulan "terverifikasi". | Could |
| FR‑18 | Multi‑satker dengan isolasi RLS. | Could |

---

## 6. Formula perhitungan (acuan baku)

Notasi: `P[m]` = proyeksi parsial bulan ke‑m (m = 1..12). `Pagu` = pagu Sub RO. `R[m]` = realisasi aktual akumulatif s.d. bulan m (dari Sheet).

**6.1 Total parsial bulan**
```
TotalParsial[m] = P[m]
```

**6.2 Akumulasi proyeksi s.d. bulan m**
```
Akum[m] = Σ (k=1..m) P[k]
```

**6.3 Total per triwulan** (TW1=Jan–Mar, TW2=Apr–Jun, TW3=Jul–Sep, TW4=Okt–Des)
```
TotalTW1 = P[1]+P[2]+P[3]
TotalTW2 = P[4]+P[5]+P[6]
TotalTW3 = P[7]+P[8]+P[9]
TotalTW4 = P[10]+P[11]+P[12]
```

**6.4 Akumulasi s.d. bulan terakhir triwulan**
```
AkumTW1 = Akum[3]   (s.d. Mar)
AkumTW2 = Akum[6]   (s.d. Jun)
AkumTW3 = Akum[9]   (s.d. Sep)
AkumTW4 = Akum[12]  (s.d. Des)
```

**6.5 Persentase proyeksi akumulatif**
```
PctAkum[m] = Pagu > 0 ? (Akum[m] / Pagu) × 100 : 0
```

**6.6 Sisa anggaran (aktual, dari Sheet)**
```
Sisa[m] = Pagu − R[m]
```

**6.7 Deviasi rencana vs realisasi** (untuk bulan yang sudah berjalan)
```
Deviasi[m] = Akum[m] (proyeksi) − R[m] (realisasi)
```

### Implementasi SQL (view ringkas)
```sql
create or replace view v_proyeksi_rekap as
select
  pr.ref_ro_id,
  rr.ro, rr.sub_ro, rr.jenis_belanja, rr.pagu,
  pr.tahun, pr.bulan,
  pr.nilai_proyeksi                                   as total_parsial,
  sum(pr.nilai_proyeksi) over (
       partition by pr.ref_ro_id, pr.tahun
       order by pr.bulan
       rows between unbounded preceding and current row
  )                                                   as akumulasi,
  ((pr.bulan - 1) / 3) + 1                             as triwulan,
  case when rr.pagu > 0 then
     round( (sum(pr.nilai_proyeksi) over (
        partition by pr.ref_ro_id, pr.tahun
        order by pr.bulan
        rows between unbounded preceding and current row) / rr.pagu) * 100, 2)
  else 0 end                                          as pct_akumulatif
from proyeksi pr
join ref_ro rr on rr.id = pr.ref_ro_id;
```

Total & akumulasi triwulan diturunkan dengan agregasi `group by triwulan` dan pengambilan `akumulasi` pada bulan terakhir tiap triwulan. Sisa anggaran di‑join dari `actual_snapshot`.

---

## 7. Kebutuhan non‑fungsional

| ID | Kebutuhan |
|---|---|
| NFR‑01 | **Kinerja:** halaman rekap < 2 detik untuk ≤ 500 baris Sub RO. |
| NFR‑02 | **Keamanan:** RLS isolasi per satker; secret hanya di server (Vercel env). |
| NFR‑03 | **Keandalan sync:** kegagalan sync tidak menghapus snapshot lama; tampilkan status. |
| NFR‑04 | **Konsistensi:** semua angka turunan dihitung di SQL view, bukan di klien. |
| NFR‑05 | **Auditability:** perubahan proyeksi tercatat (user + waktu). |
| NFR‑06 | **Responsif:** berfungsi di desktop & mobile (sesuai gaya dashboard lama). |
| NFR‑07 | **Portabilitas data aktual:** adapter Sheet dapat diganti (Apps Script ↔ Sheets API) tanpa ubah skema. |

---

## 8. Skema database (ringkas)

Tabel inti: `satker`, `ref_ro`, `proyeksi`, `actual_snapshot`, `app_user`, `role`. Detail kolom ada di `ARCHITECTURE.md §4`. Kunci penting:
- `proyeksi`: unique `(ref_ro_id, tahun, bulan)` — satu nilai parsial per bulan.
- `ref_ro`: unique `(satker_id, tahun, kro, ro, sub_ro, jenis_belanja)` — anti‑duplikat saat seeding dari sheet.
- `actual_snapshot`: cache pagu/realisasi/sisa dari sheet, `synced_at`.

## 9. Integrasi data aktual (Sheet)

- Endpoint: Apps Script Web App `?api=1` → `{records, summary}` (sudah ada).
- Mapping kunci: `(tahun, kro, ro, sub_ro, jenis_belanja)`.
- Field dipakai: `pagu`, `realisasi`, `bulanIni`, `sisa` (sisa dihitung ulang di server = pagu − realisasi).
- Frekuensi: cron tiap 30–60 menit + manual.

## 10. Antarmuka (halaman)

1. **Dashboard** — ringkasan pagu, realisasi, sisa, % proyeksi akumulatif satker.
2. **Input Proyeksi** — grid Jan–Des per Sub RO + keterangan + sisa aktual.
3. **Rekap Triwulan** — tabel TW I–IV + S‑curve proyeksi vs realisasi.
4. **Sinkronisasi** — status & tombol sinkron data aktual.
5. **Pengaturan** — tahun anggaran, user/peran.

## 11. Metrik keberhasilan

- Operator dapat menyelesaikan proyeksi satu Sub RO < 1 menit.
- 100% angka rekap triwulan konsisten dengan grid input (uji otomatis).
- Sisa anggaran yang tampil sama persis dengan Sheet (selisih 0).

## 12. Roadmap

| Fase | Isi |
|---|---|
| **MVP** | FR‑01..FR‑12 + skema DB + sync + rekap triwulan |
| **v1.1** | FR‑13..FR‑16 (grafik, peringatan, ekspor, audit) |
| **v1.2** | FR‑17..FR‑18 (verifikasi, multi‑satker) |

## 13. Risiko & mitigasi

| Risiko | Mitigasi |
|---|---|
| Mapping RO/Sub RO sheet tidak konsisten | Kunci gabungan + tahap review auto‑seed |
| Pagu berubah (revisi DIPA) | Sync memperbarui pagu; persentase otomatis menyesuaikan |
| Apps Script down saat sync | Pertahankan snapshot terakhir + tampilkan status gagal |
| Ketergantungan satu sumber (Sheet) | Adapter dapat diganti ke Google Sheets API |

---

## 14. Pertanyaan terbuka

1. Apakah proyeksi diisi sampai level **Sub RO** saja, atau perlu turun ke **Komponen/Akun**?
2. Keterangan diisi **per Sub RO** atau **per bulan per Sub RO**?
3. Berapa banyak satker yang akan memakai (menentukan perlu/tidaknya multi‑tenant penuh)?
4. Apakah perlu mekanisme **kunci/lock** proyeksi setelah triwulan berjalan?

---

*Dokumen pendamping: `ARCHITECTURE.md` (struktur teknis) dan `WORKFLOW.md` (alur kerja pengguna).*
