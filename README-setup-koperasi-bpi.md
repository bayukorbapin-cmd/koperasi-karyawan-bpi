# Koperasi BPI — Setup GitHub + Vercel + Supabase (Real-time, 10 pengguna)

File `index.html` di folder ini sudah dimodifikasi dari file asli Anda:
- Nama diubah dari **FA Tax and Accounting** menjadi **Koperasi BPI** (judul tab, layar login, sidebar).
- Ditambahkan lapisan sinkronisasi real-time ke **Supabase**, sehingga input dari 10 orang di
  perangkat berbeda bisa saling terlihat.
- Ditambahkan notifikasi **"Ada data terbaru dari perangkat lain"** di pojok kanan atas
  (tombol **Muat Sekarang** / **Nanti**), persis seperti contoh gambar yang Anda kirim.
- Login tetap memakai sistem akun yang **sudah ada** di aplikasi (menu *1f. Menu Akses/Login*).
  Akun default: **kode `123` / password `123`**. 9 akun lainnya silakan Anda tambahkan sendiri
  lewat menu tersebut (masing-masing bisa diberi kode & password `123` juga, atau berbeda-beda).

Tidak perlu mengubah kode apa pun untuk fitur di atas — yang perlu Anda lakukan hanya 3 langkah
setup di bawah ini (semuanya gratis untuk pemakaian skala kecil seperti ini).

---

## 1) Buat project Supabase (database + real-time)

1. Buka https://supabase.com → **Start your project** → daftar/masuk → **New project**.
2. Isi nama project (mis. `koperasi-bpi`), pilih password database (simpan baik-baik), pilih region
   terdekat (mis. Singapore), lalu **Create new project**. Tunggu ± 1-2 menit sampai siap.
3. Di sidebar kiri, buka **SQL Editor** → **New query**, tempel skrip berikut, lalu klik **Run**:

   ```sql
   create table if not exists app_data (
     key text primary key,
     value jsonb not null,
     updated_at timestamptz not null default now(),
     updated_by text
   );

   alter table app_data enable row level security;

   create policy "Allow anon read"   on app_data for select using (true);
   create policy "Allow anon insert" on app_data for insert with check (true);
   create policy "Allow anon update" on app_data for update using (true);

   alter publication supabase_realtime add table app_data;
   ```

   > Catatan keamanan: kebijakan (`policy`) di atas mengizinkan siapa pun yang memegang *anon key*
   > project ini untuk membaca/menulis tabel `app_data`. Kontrol akses tetap dilakukan oleh layar
   > login aplikasi (kode + password), bukan oleh Supabase. Ini wajar untuk aplikasi internal
   > koperasi dengan jumlah pengguna terbatas, tapi jangan sebarkan URL project & anon key ke
   > pihak luar.

4. Buka **Project Settings** (ikon gerigi) → **API**. Catat dua nilai ini:
   - **Project URL** (bentuknya `https://xxxxx.supabase.co`)
   - **anon public** key (kunci publik yang panjang, di bagian "Project API keys")

5. Buka `index.html`, cari (dekat baris ke-2576) bagian ini dan ganti dengan nilai Anda:

   ```js
   const SUPABASE_URL = 'GANTI-DENGAN-PROJECT-URL-SUPABASE';
   const SUPABASE_ANON_KEY = 'GANTI-DENGAN-ANON-PUBLIC-KEY-SUPABASE';
   ```

   Simpan file. (Jika kedua nilai ini belum diganti, aplikasi tetap berjalan normal seperti
   sebelumnya — hanya tersimpan lokal di satu perangkat, tanpa sinkron.)

---

## 2) Unggah ke GitHub

**Lewat browser (tanpa command line):**
1. Buka https://github.com → **New repository** → beri nama mis. `koperasi-bpi` → **Create repository**.
2. Di halaman repo kosong, klik **uploading an existing file**.
3. Seret file `index.html` (yang sudah diisi URL & key Supabase) ke area upload → **Commit changes**.

**Lewat command line** (jika Anda punya git terpasang):
```bash
cd koperasi-bpi
git init
git add index.html
git commit -m "Koperasi BPI - versi real-time Supabase"
git branch -M main
git remote add origin https://github.com/USERNAME/koperasi-bpi.git
git push -u origin main
```

---

## 3) Deploy ke Vercel

1. Buka https://vercel.com → masuk pakai akun GitHub Anda.
2. **Add New... → Project**, pilih repo `koperasi-bpi` yang tadi dibuat → **Import**.
3. Framework Preset biarkan **Other** (karena ini file statis) → langsung klik **Deploy**.
4. Setelah selesai (± 30 detik), Vercel memberi URL publik, mis.
   `https://koperasi-bpi.vercel.app` — bagikan URL ini ke 10 orang tersebut.

Setiap kali Anda meng-upload ulang `index.html` yang sudah diperbarui ke GitHub (commit baru),
Vercel otomatis men-deploy ulang.

---

## Cara kerja singkat fitur sinkron

- Setiap kali seseorang menyimpan data (tambah/edit/hapus transaksi, dsb.), datanya otomatis
  terkirim ke tabel `app_data` di Supabase beberapa saat kemudian (di-*debounce* 0.7 detik).
- Saat aplikasi pertama dibuka, ia menarik data terbaru dari Supabase; jika ada yang lebih baru
  dari cache di perangkat itu, halaman otomatis dimuat ulang sekali agar datanya sinkron.
- Selama aplikasi terbuka, jika ada perubahan dari perangkat lain, muncul notifikasi di pojok
  kanan atas: **Muat Sekarang** (langsung tarik & terapkan perubahan lalu reload) atau
  **Nanti** (tutup notifikasi, lanjut bekerja — perubahan tetap tersimpan di Supabase, tinggal
  reload manual kapan pun untuk melihatnya).
- Jika dua orang mengedit data yang sama nyaris bersamaan, yang tersimpan terakhir yang menang
  (*last write wins*) — wajar untuk skala 10 pengguna, tapi hindari mengedit transaksi yang
  sama persis di saat bersamaan.
