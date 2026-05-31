# SiP3A — Roadmap Implementasi

> Stack: **Next.js (App Router) · Supabase · Vercel**  
> Rudenim Pontianak · Ditjen Imigrasi  
> Diperbarui: 31 Mei 2026

---

## Fase 0 — Persiapan (Hari 1)

### T-01 · Setup Repository GitHub

- [ ] Buat repo GitHub: `sip3a` (public atau private)
- [ ] Upload file dokumentasi: `ARCHITECTURE.md`, `PRD.md`, `WORKFLOW.md`, `TASKS.md`
- [ ] Upload prototype: `sip3a-v2.html` (untuk referensi UI)
- [ ] Buat `.gitignore`:
  ```
  node_modules/
  .env
  .env.local
  .next/
  ```
- [ ] Buat struktur folder awal:
  ```
  sip3a/
  ├── app/              ← Next.js app
  ├── supabase/
  │   └── migrations/   ← SQL schema
  ├── docs/             ← ARCHITECTURE.md, PRD.md, dll
  └── sip3a-v2.html     ← Prototype referensi UI
  ```

---

### T-02 · Buat Project Supabase

- [ ] Daftar / login di [supabase.com](https://supabase.com)
- [ ] Buat project baru: nama `sip3a`, region `Southeast Asia (Singapore)`
- [ ] Catat:
  - `Project URL` → `NEXT_PUBLIC_SUPABASE_URL`
  - `anon public key` → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
  - `service_role key` → `SUPABASE_SERVICE_ROLE_KEY` (⚠️ jangan commit ke GitHub)
- [ ] Aktifkan **Auth → Email** (untuk login operator)
- [ ] Opsional: matikan "Confirm email" dulu untuk kemudahan testing awal

---

### T-03 · Setup Next.js Project

```bash
npx create-next-app@latest sip3a-app --typescript --tailwind --app
cd sip3a-app
npm install @supabase/supabase-js @supabase/ssr
```

- [ ] Buat `.env.local`:
  ```env
  NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
  NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
  SUPABASE_SERVICE_ROLE_KEY=eyJ...
  APPS_SCRIPT_URL=https://script.google.com/macros/s/...  # untuk sync realisasi
  ```
- [ ] Push ke GitHub
- [ ] Connect repo ke [vercel.com](https://vercel.com) → Import Project
- [ ] Set Environment Variables di Vercel (sama seperti `.env.local`)
- [ ] Verifikasi: Vercel auto-deploy dari `main` branch

---

## Fase 1 — Database Schema (Hari 2–3)

### T-04 · Buat Migrasi SQL di Supabase

Buat file `supabase/migrations/001_initial_schema.sql`:

```sql
-- Satker
CREATE TABLE satker (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  kode_satker text UNIQUE NOT NULL,
  nama_satker text NOT NULL,
  tahun_aktif int NOT NULL DEFAULT 2026
);

-- Master RO (di-seed dari DIPA)
CREATE TABLE ref_ro (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  satker_id uuid REFERENCES satker(id),
  tahun int NOT NULL,
  ro text NOT NULL,
  ro_name text,
  program text,
  program_name text,
  sub_out text,
  komp text,
  akun text NOT NULL,
  ur text,
  item text NOT NULL,
  pagu numeric DEFAULT 0,
  UNIQUE(satker_id, tahun, akun, item)
);

-- Proyeksi bulanan (inti)
CREATE TABLE proyeksi (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  ref_ro_id uuid REFERENCES ref_ro(id) ON DELETE CASCADE,
  tahun int NOT NULL,
  bulan int NOT NULL CHECK (bulan BETWEEN 1 AND 12),
  nilai numeric DEFAULT 0,
  keterangan text,
  updated_by uuid REFERENCES auth.users(id),
  updated_at timestamptz DEFAULT now(),
  UNIQUE(ref_ro_id, tahun, bulan)
);

-- Cache realisasi aktual (dari Google Sheet / OMSPAN)
CREATE TABLE actual_snapshot (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  ref_ro_id uuid REFERENCES ref_ro(id),
  tahun int NOT NULL,
  bulan int NOT NULL,
  realisasi numeric DEFAULT 0,
  synced_at timestamptz DEFAULT now(),
  UNIQUE(ref_ro_id, tahun, bulan)
);
```

- [ ] Jalankan di Supabase → SQL Editor
- [ ] Verifikasi tabel terbuat di Table Editor

---

### T-05 · Buat View Rekap di Supabase

```sql
-- View: gabungan pagu + proyeksi + realisasi per item per bulan
CREATE VIEW v_proyeksi_rekap AS
SELECT
  r.id AS ref_ro_id,
  r.satker_id,
  r.tahun,
  r.ro, r.ro_name, r.program, r.program_name,
  r.sub_out, r.komp, r.akun, r.ur, r.item,
  r.pagu,
  p.bulan,
  COALESCE(p.nilai, 0) AS proyeksi,
  COALESCE(a.realisasi, 0) AS realisasi,
  r.pagu - COALESCE(a.realisasi, 0) AS sisa,
  p.keterangan,
  p.updated_at
FROM ref_ro r
LEFT JOIN proyeksi p ON p.ref_ro_id = r.id AND p.tahun = r.tahun
LEFT JOIN actual_snapshot a ON a.ref_ro_id = r.id 
  AND a.tahun = r.tahun AND a.bulan = p.bulan;
```

- [ ] Jalankan di SQL Editor

---

### T-06 · Enable Row Level Security (RLS)

```sql
ALTER TABLE ref_ro ENABLE ROW LEVEL SECURITY;
ALTER TABLE proyeksi ENABLE ROW LEVEL SECURITY;
ALTER TABLE actual_snapshot ENABLE ROW LEVEL SECURITY;

-- Policy: user hanya bisa lihat/edit data satker-nya sendiri
-- (implementasi lengkap: tambahkan tabel user_satker untuk mapping)
-- Untuk single-satker, bisa pakai policy sederhana:
CREATE POLICY "Allow authenticated" ON proyeksi
  FOR ALL TO authenticated USING (true);

CREATE POLICY "Allow authenticated" ON ref_ro
  FOR ALL TO authenticated USING (true);

CREATE POLICY "Allow read" ON actual_snapshot
  FOR SELECT TO authenticated USING (true);
```

- [ ] Aktifkan dan test RLS di Supabase

---

### T-07 · Seed Data Master dari DIPA

- [ ] Buat script seed: `scripts/seed-data-master.ts`
- [ ] Isi dengan 9 baris data dari `sip3a-v2.html` (DATA array)
- [ ] Jalankan: `npx ts-node scripts/seed-data-master.ts`
- [ ] Verifikasi di Supabase Table Editor → tabel `ref_ro` terisi

---

## Fase 2 — Backend API (Hari 3–5)

### T-08 · Setup Supabase Client di Next.js

Buat `lib/supabase/server.ts` dan `lib/supabase/client.ts` sesuai dokumentasi `@supabase/ssr`.

- [ ] Server client (untuk Server Components & Route Handlers)
- [ ] Browser client (untuk Client Components)
- [ ] Middleware untuk refresh token: `middleware.ts`

---

### T-09 · Route Handler: Sync Realisasi dari Google Sheet

`app/api/actual/sync/route.ts`

- [ ] Fetch data dari Apps Script Web App (Google Sheet sumber realisasi)
- [ ] Normalisasi data (mapping kolom sheet → skema `actual_snapshot`)
- [ ] Upsert ke tabel `actual_snapshot` di Supabase
- [ ] Return: `{ synced: N, at: timestamp }`
- [ ] Protect dengan header `Authorization: Bearer SERVICE_ROLE_KEY`

Opsional: Setup Vercel Cron (`vercel.json`):
```json
{
  "crons": [{
    "path": "/api/actual/sync",
    "schedule": "0 7 * * 1-5"
  }]
}
```
(Jalankan otomatis setiap pagi hari kerja jam 07.00)

---

### T-10 · Server Actions: Simpan Proyeksi

`app/actions/proyeksi.ts`

```typescript
'use server'
export async function upsertProyeksi(
  ref_ro_id: string,
  tahun: number,
  bulan: number,
  nilai: number,
  keterangan: string
) {
  const supabase = createServerClient();
  const { data: { user } } = await supabase.auth.getUser();
  
  return supabase.from('proyeksi').upsert({
    ref_ro_id, tahun, bulan, nilai, keterangan,
    updated_by: user?.id,
    updated_at: new Date().toISOString()
  }, { onConflict: 'ref_ro_id,tahun,bulan' });
}
```

- [ ] Validasi: nilai tidak boleh negatif, tidak boleh melebihi sisa pagu
- [ ] Return error yang informatif ke client

---

## Fase 3 — Frontend Next.js (Hari 5–10)

### T-11 · Halaman Auth (Login)

`app/login/page.tsx`

- [ ] Form email + password
- [ ] Gunakan `supabase.auth.signInWithPassword()`
- [ ] Redirect ke `/` setelah login berhasil
- [ ] Middleware proteksi: redirect ke `/login` jika belum auth

---

### T-12 · Halaman Utama — Grid Pengisian

`app/page.tsx` (atau `app/pengisian/page.tsx`)

- [ ] Fetch data dari `v_proyeksi_rekap` via Supabase
- [ ] Render tabel dengan struktur sama seperti `sip3a-v2.html`:
  - Kolom: Item | Komponen/Akun | Uraian | Pagu | Realisasi | Sisa | [Bulan TW] | Total | Ket
  - Group by: RO → Sub Output
- [ ] Filter: Triwulan, Jenis Belanja, Search (state di URL params)
- [ ] Cell input editable → panggil Server Action `upsertProyeksi`
- [ ] Optimistic update agar UI tidak "jump" saat simpan
- [ ] Pertahankan desain dark/light dari prototype (port CSS ke Tailwind)

---

### T-13 · Halaman Rekapitulasi

`app/rekap/page.tsx`

- [ ] Fetch agregasi dari Supabase (query GROUP BY jenis_belanja, ro)
- [ ] Tabel Per Jenis Belanja: 51/52/53 + Total
- [ ] Tabel Per Program/RO: hierarki program → RO
- [ ] Kolom: Pagu | Realisasi | Sisa | [Bulan TW] | Total TW | % Proyeksi
- [ ] Tombol "← Pengisian"

---

### T-14 · Komponen Sidebar & Layout

`app/layout.tsx`

- [ ] Sidebar dengan navigasi: Pengisian, Rekapitulasi, Grafik (roadmap)
- [ ] Info filter aktif (Triwulan, JB)
- [ ] Aksi: Simpan Semua, Export CSV, Sinkron Realisasi
- [ ] Badge status: dirty/saved
- [ ] Toggle dark/light theme (simpan preferensi di localStorage)
- [ ] Responsive mobile: bottom navigation

---

### T-15 · Export CSV

`app/actions/export.ts`

- [ ] Fetch semua data proyeksi TW yang dipilih
- [ ] Generate CSV dengan header: RO, Sub Output, Komp, Akun, Uraian, Item, JB, Pagu, Realisasi, Sisa, [Bulan], Total TW, Ket
- [ ] Return sebagai file download (Route Handler dengan `Content-Type: text/csv`)

---

## Fase 4 — Fitur Tambahan (Minggu 2–3)

### T-16 · Grafik Proyeksi vs Realisasi

- [ ] Install: `npm install recharts`
- [ ] Halaman `/grafik`:
  - Line chart: Kumulatif Proyeksi vs Kumulatif Realisasi (Jan–Des)
  - Bar chart: Proyeksi per bulan vs Realisasi per bulan
- [ ] Filter: per RO, per Jenis Belanja

---

### T-17 · Export Excel (.xlsx)

- [ ] Install: `npm install xlsx`
- [ ] Satu workbook, multiple sheet per Jenis Belanja + sheet Rekap
- [ ] Header: logo satker, tahun anggaran, tanggal cetak
- [ ] Tombol di footer dan sidebar

---

### T-18 · Validasi & Alert

- [ ] Warning jika proyeksi + realisasi > pagu (sudah ada di prototype, port ke Next.js)
- [ ] Warning jika proyeksi TW II < 45% target kumulatif pagu
- [ ] Highlight baris proyeksi nol dengan sisa pagu besar
- [ ] Summary card di atas grid: total pagu, total proyeksi TW, % proyeksi

---

### T-19 · Audit Log

- [ ] Tampilkan di sidebar: "Terakhir diubah: [nama] · [tanggal jam]"
- [ ] Halaman `/log`: riwayat perubahan proyeksi (ref_ro, bulan, nilai lama → baru, operator, waktu)

---

## Checklist Launch

```
[ ] T-01: GitHub repo dibuat
[ ] T-02: Supabase project aktif
[ ] T-03: Next.js project di Vercel (auto-deploy)
[ ] T-04: Schema SQL terbuat
[ ] T-05: View rekap terbuat
[ ] T-06: RLS aktif
[ ] T-07: Data DIPA di-seed ke ref_ro
[ ] T-08: Supabase client terkonfigurasi
[ ] T-09: API sync realisasi jalan
[ ] T-10: Server Action simpan proyeksi jalan
[ ] T-11: Login/auth berfungsi
[ ] T-12: Grid pengisian tampil & bisa edit
[ ] T-13: Rekap tampil dengan angka benar
[ ] T-14: Sidebar & layout responsif
[ ] T-15: Export CSV jalan
[ ] UAT: Test bersama operator — input proyeksi, simpan, reload, cek data tetap ada
[ ] Go Live: Share URL Vercel ke tim
```

---

## Estimasi Waktu

| Fase | Durasi | Keterangan |
|---|---|---|
| Fase 0 (T-01–T-03) | 1 hari | Setup awal |
| Fase 1 (T-04–T-07) | 1–2 hari | Database schema + seed |
| Fase 2 (T-08–T-10) | 2–3 hari | Backend API |
| Fase 3 (T-11–T-15) | 3–5 hari | Frontend utama |
| Fase 4 (T-16–T-19) | 1–2 minggu | Fitur tambahan |

**Total ke production (Fase 0–3):** ±7–11 hari kerja

---

*Lihat `ARCHITECTURE.md` untuk detail teknis. Lihat `sip3a-v2.html` untuk referensi UI/UX.*
