# SiP3A — Arsitektur Sistem

**Sistem Peramalan dan Proyeksi Pelaksanaan Anggaran Satker**  
Rudenim Pontianak · Ditjen Imigrasi

Versi 2.0 · 31 Mei 2026 · Status: Prototype selesai, siap implementasi backend

---

## Status Saat Ini — v2.0 (Prototype)

File `sip3a-v2.html` adalah prototype single-file yang sudah berfungsi penuh sebagai UI:
- ✅ Grid input proyeksi per item/akun/bulan
- ✅ Filter Triwulan, Jenis Belanja, pencarian
- ✅ Rekap otomatis (per JB dan per Program/RO)
- ✅ Ekspor CSV
- ✅ Dark/light theme, responsive mobile
- ❌ Data masih hard-coded (belum terhubung ke backend/Sheets)
- ❌ Simpan hanya in-memory (reset saat reload)

**Tujuan arsitektur ini:** mendeskripsikan jalan dari prototype → sistem produksi dengan Google Sheets / Supabase sebagai backend data.

---

## 1. Ringkasan

SiP3A adalah aplikasi web full‑stack yang memungkinkan satker mengisi **peramalan (proyeksi) rencana pelaksanaan anggaran** pada level **RO (Rincian Output)** dan **Sub RO**, lalu membandingkannya dengan **realisasi aktual** dan **sisa anggaran** yang bersumber dari Google Sheet yang sudah berjalan (database MONEV PA / IKPA).

Aplikasi dibangun sebagai aplikasi standalone dengan stack modern (Next.js + Supabase + Vercel), terpisah dari dashboard Apps Script yang ada, tetapi **tetap membaca data Pagu & realisasi dari Google Sheet yang sama** agar tidak ada duplikasi sumber kebenaran (single source of truth) untuk angka realisasi.

### Pembagian sumber data (prinsip inti)

| Jenis data | Sumber kebenaran | Sifat |
|---|---|---|
| Pagu, Realisasi, Sisa anggaran (aktual) | **Google Sheet** (via Apps Script API yang sudah ada) | Read‑only bagi SiP3A |
| Proyeksi / peramalan per RO & Sub RO per bulan | **Supabase (Postgres)** | Read‑write, dikelola SiP3A |
| Keterangan / catatan proyeksi | **Supabase** | Read‑write |
| User, peran, audit | **Supabase Auth + tabel** | Read‑write |

Prinsip: **angka aktual tidak pernah ditulis ulang oleh SiP3A**; SiP3A hanya menambah lapisan proyeksi di atasnya.

---

## 2. Diagram arsitektur tingkat tinggi

```
┌──────────────────────────────────────────────────────────────────┐
│                          PENGGUNA (Browser)                        │
│                Operator Satker · Verifikator · Pimpinan            │
└───────────────────────────────┬──────────────────────────────────┘
                                 │ HTTPS
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                  VERCEL  ·  Next.js (App Router)                    │
│                                                                    │
│  ┌──────────────┐   ┌───────────────┐   ┌──────────────────────┐  │
│  │  UI / Pages  │   │ Server Actions │   │  Route Handlers (API)│  │
│  │  (React/RSC) │   │  & RSC fetch   │   │  /api/actual/sync    │  │
│  └──────┬───────┘   └───────┬───────┘   └──────────┬───────────┘  │
│         │                   │                      │              │
│         └─────────┬─────────┴──────────┬───────────┘              │
│                   ▼                    ▼                          │
│        ┌────────────────────┐  ┌───────────────────────┐         │
│        │  Supabase JS client│  │ Google Sheet adapter  │         │
│        │  (anon + RLS)      │  │ (fetch Apps Script API)│         │
│        └─────────┬──────────┘  └───────────┬───────────┘         │
└──────────────────┼─────────────────────────┼─────────────────────┘
                   │                          │
                   ▼                          ▼
   ┌───────────────────────────┐   ┌──────────────────────────────┐
   │      SUPABASE             │   │   GOOGLE SHEET (existing)     │
   │  • Postgres (proyeksi)    │   │   • Sheet DATA (Pagu/Real.)   │
   │  • Auth (login satker)    │   │   • Apps Script Web App API   │
   │  • RLS (isolasi satker)   │   │     ?api=1  → records + summary│
   │  • Edge Function (opsional│   │                               │
   │    untuk cron sync)       │   │   Sumber kebenaran realisasi  │
   └───────────────────────────┘   └──────────────────────────────┘
```

---

## 3. Komponen

### 3.1 Frontend — Next.js di Vercel

- **Framework:** Next.js 14+ (App Router), React Server Components untuk data fetching, Client Components untuk grid input proyeksi.
- **Styling:** Tailwind CSS (mengikuti gaya dark/light dashboard lama agar konsisten secara visual).
- **State input:** form grid bulanan (Jan–Des) per baris RO/Sub RO; menyimpan perubahan via Server Action ke Supabase.
- **Charting:** Recharts atau Chart.js untuk kurva proyeksi vs realisasi.
- **Tabel:** komponen tabel dengan kolom bulan, total parsial, akumulasi, rekap triwulan, dan persentase.

### 3.2 Backend — Next.js Server (Vercel)

Tidak ada server terpisah; logika server berjalan sebagai **Server Actions** dan **Route Handlers** di Vercel:

- **Server Actions** — menyimpan/memperbarui proyeksi (mutasi tertulis ke Supabase dengan validasi).
- **Route Handler `/api/actual/sync`** — memanggil Apps Script Web App (`?api=1`), menormalisasi `records`, dan menyimpan snapshot Pagu/Realisasi/Sisa ke tabel cache di Supabase.
- **Perhitungan** — agregasi total parsial, akumulasi, triwulan, dan persentase dilakukan di SQL (Postgres view) dan/atau di server, bukan di sheet.

### 3.3 Database — Supabase (Postgres)

- Menyimpan master RO/Sub RO (di‑seed dari sheet), proyeksi bulanan, keterangan, snapshot aktual, user & peran.
- **Row Level Security (RLS)** untuk mengisolasi data antar‑satker bila multi‑satker.
- **Views/Functions** untuk perhitungan akumulasi & rekap triwulan agar konsisten dan cepat.

### 3.4 Integrasi Google Sheet (data aktual)

SiP3A **tidak** menulis ke sheet. Ia hanya membaca melalui salah satu cara:

1. **Apps Script Web App (direkomendasikan, sudah ada)** — `GET BASE_URL?api=1` mengembalikan `{records, summary}`. SiP3A memanggil endpoint ini lalu cache ke Supabase.
2. **Google Sheets API langsung** (service account) — alternatif bila ingin lepas dari Apps Script.

Sinkronisasi dijadwalkan (mis. tiap 30–60 menit via Vercel Cron atau Supabase Edge Function) dan bisa dipicu manual lewat tombol "Sinkron Data Aktual".

---

## 4. Model data (skema Supabase)

### 4.1 `satker`
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid PK | |
| kode_satker | text unique | mis. kode KPPN/satker |
| nama_satker | text | mis. "Rudenim Pontianak" |
| tahun_aktif | int | tahun anggaran berjalan |

### 4.2 `ref_ro` (master RO & Sub RO, di‑seed dari sheet DATA)
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid PK | |
| satker_id | uuid FK | |
| tahun | int | tahun anggaran |
| program | text | |
| kegiatan | text | |
| kro | text | Klasifikasi RO |
| ro | text | **Rincian Output** |
| sub_ro | text | **Sub Output** (kolom "SUB OUTOUT" di sheet) |
| jenis_belanja | text | 51/52/53/57 |
| pagu | numeric | pagu aktual (cache, dari sheet) |
| unique(satker, tahun, kro, ro, sub_ro) | | mencegah duplikat |

### 4.3 `proyeksi` (inti — peramalan per RO/Sub RO per bulan)
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid PK | |
| ref_ro_id | uuid FK → ref_ro | level RO/Sub RO |
| tahun | int | |
| bulan | int (1–12) | |
| nilai_proyeksi | numeric | rencana penyerapan bulan tsb (parsial) |
| keterangan | text | catatan kegiatan/alasan |
| updated_by | uuid FK → user | |
| updated_at | timestamptz | |
| unique(ref_ro_id, tahun, bulan) | | satu nilai per RO/Sub RO per bulan |

> Catatan desain: `nilai_proyeksi` disimpan sebagai **nilai parsial per bulan**. Semua akumulasi, total triwulan, dan persentase **dihitung** (bukan disimpan) lewat view/fungsi — sehingga selalu konsisten.

### 4.4 `actual_snapshot` (cache realisasi dari sheet)
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | uuid PK | |
| ref_ro_id | uuid FK | dipetakan dari record sheet |
| tahun | int | |
| periode | text | bulan (Jan…Des) |
| pagu | numeric | |
| realisasi | numeric | akumulatif s.d. periode |
| bulan_ini | numeric | realisasi parsial bulan |
| sisa | numeric | `pagu - realisasi` |
| synced_at | timestamptz | waktu sinkron terakhir |

### 4.5 `app_user` & `role`
Login satker, peran `operator` / `verifikator` / `pimpinan`, dan kolom audit. Bila single‑satker, peran cukup membatasi siapa boleh mengubah proyeksi vs hanya melihat.

---

## 5. Logika perhitungan (di mana dihitung)

Semua angka turunan dihitung di **Postgres view** `v_proyeksi_rekap` agar konsisten antara grid input, rekap triwulan, dan grafik. Rumus detail ada di PRD §Formula; ringkasnya:

- **Total parsial bulan** = `nilai_proyeksi` bulan tsb (langsung).
- **Akumulasi s.d. bulan** = `SUM(nilai_proyeksi)` dari Jan s.d. bulan tsb (window function `SUM() OVER (PARTITION BY ref_ro ORDER BY bulan)`).
- **Total triwulan** = jumlah 3 bulan dalam triwulan tsb.
- **Akumulasi s.d. akhir triwulan** = akumulasi pada bulan terakhir triwulan (Mar/Jun/Sep/Des).
- **% proyeksi akumulatif** = `akumulasi proyeksi / pagu × 100`.
- **Sisa anggaran** = `pagu - realisasi` (dari `actual_snapshot`, bukan dari proyeksi).

---

## 6. Keamanan

- **Auth:** Supabase Auth (email/password atau magic link untuk operator satker).
- **RLS:** setiap baris `proyeksi`/`ref_ro` terikat `satker_id`; policy hanya mengizinkan user pada satker yang sama.
- **Secret:** `SUPABASE_SERVICE_ROLE_KEY`, URL Apps Script, dan kunci Google disimpan sebagai **Environment Variables di Vercel**, tidak pernah di klien.
- **Read‑only ke sheet:** kredensial sheet hanya dipakai di server (Route Handler), tidak diekspos ke browser.
- **Audit:** kolom `updated_by`/`updated_at` pada proyeksi untuk jejak perubahan.

---

## 7. Deployment (GitHub → Vercel → Supabase)

```
Developer  ──push──▶  GitHub repo  ──auto deploy──▶  Vercel (Preview/Prod)
                                                          │
                                       Environment Vars   │  Supabase project
                                       (SUPABASE_URL,      │  (DB + Auth + RLS)
                                        ANON/SERVICE KEY,  │
                                        APPS_SCRIPT_URL)   ▼
                                                     Migrations (supabase/migrations)
```

- **GitHub:** repo monorepo (`/app`, `/supabase/migrations`, `/docs`).
- **Vercel:** terhubung ke GitHub; setiap PR → Preview Deployment; merge ke `main` → Production. Vercel Cron memicu `/api/actual/sync`.
- **Supabase:** migrasi SQL dikelola via folder `supabase/migrations` dan CLI; RLS policy versioned bersama kode.

---

## 8. Pertimbangan & alternatif

- **Mapping RO/Sub RO sheet → Supabase** adalah titik paling rawan. Disarankan kunci gabungan `(tahun, kro, ro, sub_ro, jenis_belanja)` dan proses seeding sekali + sinkron berkala untuk pagu.
- Bila ke depan ingin lepas dari Apps Script, ganti adapter §3.4 opsi 1 → opsi 2 (Google Sheets API service account) tanpa mengubah skema Supabase.
- Perhitungan di SQL view lebih disukai daripada di klien agar rekap triwulan & akumulasi tidak pernah "miring" antar halaman.

---

*Lihat `WORKFLOW.md` untuk alur kerja pengguna dan `PRD.md` untuk kebutuhan rinci + formula.*
