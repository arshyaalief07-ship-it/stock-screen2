# STOCK RADAR

Screener teknikal saham BEI berbasis Next.js, FastAPI, Supabase, Yahoo Finance chart data, dan GitHub Actions.

## Catatan data
- Provider bawaan: Yahoo Finance melalui library `yfinance`, simbol BEI memakai akhiran `.JK`.
- Bukan feed resmi BEI, bukan real-time, dan tidak memiliki SLA. Data dapat tertunda, dibatasi, atau berubah.
- Jangan gunakan hasil screener sebagai satu-satunya dasar keputusan. Validasi harga dan keterbukaan informasi dari sumber resmi.
- Cakupan "all" berarti semua ticker aktif yang dimasukkan ke tabel `stocks`. Impor daftar emiten BEI terbaru melalui CSV dari direktori resmi BEI. Repo tidak mengklaim membawa daftar ticker lengkap yang selalu mutakhir.

## Stack gratis
- Frontend: Vercel Hobby untuk proyek personal/non-komersial.
- Database: Supabase Free.
- Backend API: Render Free (service dapat sleep saat idle).
- Pengambilan data terjadwal: GitHub Actions.
Free tier dapat berubah dan punya batas pemakaian. Cek halaman pricing resmi sebelum deploy.

## Struktur
```
stock-radar/
├── frontend/                 # Next.js dashboard
├── backend/                  # FastAPI + indikator + collector
├── supabase/schema.sql       # tabel, index, RLS
├── .github/workflows/        # update data harian
└── .env.example
```

## 1. Siapkan Supabase
1. Buat project baru di Supabase.
2. Buka SQL Editor, jalankan seluruh `supabase/schema.sql`.
3. Simpan `Project URL`, `anon key`, dan `service_role key`.
4. Jangan pernah memasukkan `service_role key` ke frontend atau commit ke GitHub.

## 2. Impor universe saham BEI
1. Unduh daftar saham/perusahaan tercatat terbaru dari situs resmi BEI.
2. Simpan sebagai CSV dengan kolom `Kode Saham` (atau `symbol`).
3. Jalankan:
```bash
cd backend
python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate
pip install -r requirements.txt
python scripts/import_symbols.py path/to/daftar_emiten.csv
```
Script mengubah `BBCA` menjadi `BBCA.JK`. Periksa CSV sumber, hapus instrumen yang bukan saham biasa bila tidak ingin memasukkannya.

## 3. Backend lokal
```bash
cd backend
cp .env.example .env
# isi SUPABASE_URL dan SUPABASE_SERVICE_ROLE_KEY
pip install -r requirements.txt
uvicorn app.main:app --reload
```
API: `http://localhost:8000/docs`

## 4. Frontend lokal
```bash
cd frontend
cp .env.example .env.local
# isi NEXT_PUBLIC_API_URL=http://localhost:8000
npm install
npm run dev
```
Buka `http://localhost:3000`.

## 5. Deploy backend ke Render
1. Push repo ke GitHub.
2. Render > New > Web Service > pilih repo.
3. Root Directory: `backend`
4. Build Command: `pip install -r requirements.txt`
5. Start Command: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`
6. Tambahkan environment:
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_ROLE_KEY`
   - `CORS_ORIGINS` = URL Vercel frontend, mis. `https://stock-radar.vercel.app`
7. Deploy. Simpan URL service Render.

## 6. Deploy frontend ke Vercel
1. Vercel > Add New > Project > import repo.
2. Root Directory: `frontend`.
3. Environment Variables:
   - `NEXT_PUBLIC_API_URL` = URL Render, mis. `https://stock-radar-api.onrender.com`
4. Deploy.

## 7. Aktifkan update data otomatis
Tambahkan GitHub repository secrets:
- `SUPABASE_URL`
- `SUPABASE_SERVICE_ROLE_KEY`

Workflow `.github/workflows/daily-update.yml` berjalan Senin-Jumat sekitar 16:30 WIB (09:30 UTC). Waktu dapat bergeser karena antrean GitHub. Jalankan manual lewat Actions > Daily market data update > Run workflow.

## 8. Uji
- `GET /health`
- `GET /api/stocks`
- `GET /api/screener?limit=100`
- Jalankan collector manual:
```bash
cd backend
python -m app.collector
```

## Indikator & scoring
- Trend (30): posisi harga vs SMA20/50/200.
- Momentum (25): RSI dan MACD.
- Volume (20): volume vs rata-rata 20 hari.
- Volatilitas (10): ATR% relatif.
- Struktur harga (15): breakout 20 hari / posisi terhadap high-low 20 hari.
Skor adalah heuristik teknikal, bukan prediksi atau rekomendasi beli/jual. ATR rendah diberi nilai lebih tinggi pada komponen volatilitas; parameter dapat disesuaikan.

## Troubleshooting
- `No data`: cek apakah simbol memiliki akhiran `.JK`, ticker aktif, dan Yahoo Finance mengembalikan data.
- `429/rate limited`: kurangi frekuensi, jalankan collector sekali sehari, jangan paralelkan banyak ticker.
- Backend sleep: request pertama di Render Free bisa lambat.
- Cakupan belum lengkap: pastikan CSV emiten terbaru sudah diimpor dan `is_active=true`.
- Data basi: cek `updated_at` pada tabel `daily_prices` dan log GitHub Actions.

## Keamanan
- Jangan expose service role key.
- RLS membatasi anon hanya SELECT pada data pasar.
- Endpoint backend read-only untuk publik. Tambahkan autentikasi/rate limit sebelum membuka akses luas.
