# Engine Dashboard — GitHub Pages

Folder ini siap di-upload ke GitHub Pages. Isinya:

- `index.html` — dashboard statis satu file (HTML + CSS + JS)
- `.nojekyll` — agar GitHub Pages tidak memproses file dengan Jekyll
- `README.md` — panduan ini

Tidak ada backend, tidak ada password, tidak perlu domain sendiri.
Dashboard membaca langsung sheet `SCRAPING`, `MISI HARIAN`, dan `REKAP SC` dari
browser. Grup menu **Analisa** berisi halaman **Penjualan**, **Analisa Performa
Depo/Cabang**, dan **Analisa Stok Gudang**; semua file diunggah di satu menu
Setting dan diproses di browser. Menu **Performance** selalu memakai sheet
`REKAP SC`; kartu unggah **Performance (REKAP SC)** sudah dihapus dari Setting.

Navigasi bawah membagi tampilan menjadi **Omset**, **Performance**, **Analisa**,
**JKS Editor**, **Logistik**, dan **Setting** (paling kanan). Label **Analisa**
membuka halaman **Penjualan**. Sidebar otomatis menampilkan menu sesuai grup yang
dipilih: **Analisa Performa Depo**, **Analisa Performa Cabang**, dan **Analisa
Stok Gudang** ada di grup **Analisa**; **JKS Editor** dan **Logistik** ada di
grup **Operational**. Sidebar Logistik hanya berisi **Route Armada**, **Rekap
Supir**, **Detail Supir**, dan **Upload Data**. **Master Data** dan **Setting
ROA** ada di grup **Setting**.
Leaflet dan SheetJS dimuat dari CDN, jadi koneksi internet diperlukan saat
halaman pertama kali dibuka.

## Performance (Google Sheets `REKAP SC`)

Menu **Performance** membaca tab **`REKAP SC`** langsung dari Google Sheets dan
menampilkannya seperti Score Card lama. Kartu unggah **Performance (REKAP SC)**
sudah dihapus dari Setting, sehingga menu Performance selalu memakai sheet.

- Semua kolom sheet dengan header berkelompok (mis. OMSET, ROA), baris ringkasan
  tim, dan total.
- Pencarian cepat, filter **AREA**, tombol sumber **Google Sheets**, dan
  **auto-refresh 60 detik** selama tampilan terbuka.
- Tidak ada panel atau tombol pengaturan ROA di halaman ini. Rekap ROA dan
  pengaturannya hanya ada di **Setting → Setting ROA**.
- Bila sheet tidak bisa dibaca, cache terakhir di browser tetap dipakai dan
  status berubah menjadi data cache.

## Setting (satu tempat untuk semua unggahan)

Menu **Setting** adalah satu-satunya tempat mengunggah file. Delapan kategori,
masing-masing **maksimal 15 file** (`.xlsx`, `.xls`, `.xlsb`, `.csv`, `.json`).
Tiap kartu punya tombol **Template**, **Hapus**, daftar file, dan catatan status.

1. **Dbase MTD** — data penjualan **bulan berjalan**, dipakai **Penjualan** dan
   sebagai sumber **omset gross MTD** Analisa Performa Depo/Cabang. Template 17
   kolom (`TGL`, kode/nama outlet, `PMA`/`DEPO`, channel, `DIVISI`, nama + kode
   salesman, `KD_BRG`, `NM_BRG`, `MARK NEW`, `QTY`, `VALUE`, `VALUE NETT`,
   `NO FAKTUR`, …).
2. **Dbase 6 Bulan Terakhir** — data historis untuk **pembanding omset, EC, dan
   ROA pada HK yang sama** di Analisa Performa Depo/Cabang serta **Setting ROA**.
   Format kolom sama dengan Dbase MTD; bulan berjalan di file ini tidak dihitung
   ganda sebagai MTD.
3. **Master Item** — `KODE ITEM, ITEM PRODUK, KATEGORI (MARK NEW), SATUAN`.
4. **Master Outlet** — template 33 kolom (**header dibaca dari baris 4**, 3 baris
   pembuka diabaikan): kode outlet → nama/depo/channel/divisi/salesman/rute.
5. **Master Salesman** — `KODE SALESMAN, NAMA SALESMAN, DIVISI, DEPO/PMA, CHANNEL`.
6. **Stok Gudang** — template `template-stok-gudang.xlsx`, dipakai **Analisa Stok
   Gudang**.
7. **Target Salesman** — `KODE SALESMAN, NAMA SALESMAN, BULAN, TARGET`; kolom
   `BULAN` opsional (kosong = target bulanan sama untuk setiap bulan; format
   `2026-10`, `10/2026`, `OKT 2026`, `OCT-26`, atau tanggal).
8. **Master Armada** — `NAMA SUPIR, JENIS ARMADA, KUBIKASI` (alias `KODE
   ARMADA`/`PLAT NOMOR`). Satu baris = satu kendaraan; kapasitas kubikasi dipakai
   otomatis oleh **Logistik**.

Master outlet/item/salesman **melengkapi kolom yang kosong** pada data penjualan
dan diterapkan ulang ke file yang sudah diunggah. Target dipakai untuk kolom
**Target** & **Pencapaian** di tabel salesman menu Penjualan (hanya bulan
MTD/bulan berjalan) dan **Target MTD** di Analisa Performa Depo (mode
Cabang menyembunyikan kolom target karena dimensinya depo). Master Armada
dipakai menu Logistik.

**Outlet covered** dihitung dari kolom `FLAG` Master Outlet: nilai yang memuat
`COVER` dan tidak memuat `NOT`, `UNCOVER`, atau `NON`. Bila tidak ada nilai
`FLAG`, semua outlet dianggap covered. Outlet dihitung unik per kode/nama dan
dipetakan ke salesman memakai **kode salesman** (`KD_SLS`/alias kode), bukan
nama salesman. Divisi sales yang dipakai di Penjualan, Performance, dan ROA
adalah **AEP** dan **GDK**.

## Setting ROA (per divisi sales)

Menu **Setting ROA** (grup Setting) mengatur aturan ROA tiap divisi sales secara
terpisah. Pengaturan disimpan lokal di `localStorage` key `roa-settings-v1` dan
hanya dipakai pratinjau di halaman ini serta kolom TAT pada tabel salesman
Penjualan. Tidak ada panel pengaturan ROA di Performance atau Penjualan.

- **Daftar Aturan ROA** menampilkan satu baris per divisi aktif dengan kolom
  **Divisi**, **Mapping Type**, **MARK NEW**, **Status**, dan **Aksi**.
- **Tambah Divisi**: pilih AEP atau GDK lalu klik **Tambah Divisi**. Dropdown
  hanya berisi divisi yang belum punya aturan. **Atur** membuka popup mapping
  type outlet, **Hapus** menghapus aturan beserta pilihan SKU-nya. Menambah
  kembali **tidak** otomatis memilih semua MARK NEW; SKU Fokus mulai kosong.
- **Mapping type outlet**: popup membaca type outlet dari Master Outlet
  (`TYPE OUTLET`, `JENIS OUTLET`, atau `CHANNEL`). Satu type hanya boleh masuk
  **RTL UP** atau **RTL DOWN**; checkbox saling meniadakan. Popup tidak lagi
  berisi Min QTY, Min BOX, atau target.
- **SKU Fokus per divisi**: hanya MARK NEW yang dipilih yang tampil dan dihitung.
  Tidak ada mode "tampilkan semua": tambah SKU lewat dropdown **Tambah MARK NEW…**
  + **+ Tambah Eceran**, lalu hapus lewat **Hapus** pada baris. Pilihan disimpan
  per divisi pada `divSkus`.
- **Target (%) & dropsize per SKU**: kolom **TARGET (%)**, **CB / JUMLAH
  OUTLET**, **RTL UP**, **RTL DOWN**, dan **SATUAN** (`KRT`/`BOX`/`PCS`).
  Target menerima angka atau persentase (mis. `25` atau `25%`), rentang 0–100;
  kosong = otomatis **100% CB**. **CB / Jumlah Outlet** adalah outlet covered
  divisi dari Master Outlet (`FLAG COVERED`), bukan angka manual. Aturan
  disimpan per divisi + MARK NEW di `skuRules`; AEP dan GDK boleh berbeda untuk
  SKU yang sama. **Min. Biasa sudah dihapus**; kategori biasa cukup qty > 0.
  RTL UP/DOWN tetap menjadi ambang dropsize. Default SKU baru `100% CB, DS 1/1
  KRT`.
- **Target & capaian per SKU**: target tiap salesman = **Target (%) × CB
  salesman**, minimal 1 bila persentase > 0. Tidak ada lagi pembagian target
  total divisi. Target absolute lama dimigrasikan ke persentase (`target lama ÷
  CB divisi`). Capaian = outlet unik yang membeli SKU dan memenuhi ambang
  dropsize UP/DOWN; kategori biasa cukup qty > 0. Capaian bisa >100% dari outlet
  uncovered/di luar cakupan. Outlet dihitung sekali per salesman + outlet + SKU
  + bulan.
- **Rekap ROA**: per grup divisi salesman (mis. `GDKDA`) dengan kolom **KODE
  SALESMAN**, **NAMA SALESMAN**, **CB** (outlet covered salesman dari Master
  Outlet), lalu per SKU `SKU — target% · DS n/n UNIT` dan `% SKU`; total per grup
  dan grand total. **Total CB** memakai total outlet covered Master Outlet
  (`FLAG COVERED`) per divisi. Warna persentase: hijau ≥100%, kuning 80–99%,
  merah <80%.
- **Hari Kerja & TAT**: HK total dan HK berjalan diisi manual; TAT = target
  bulanan × HK berjalan / HK total. Jika HK berjalan 0, TAT memakai target penuh.
  TAT tetap dipakai di tabel salesman Penjualan, bukan di rekap ROA.
- **Pratinjau** di bawah halaman memakai aturan yang sama dengan rekap ROA.

## Penjualan (sumber: Setting)

Menu **Penjualan** membaca **Dbase MTD dari menu Setting** (tidak ada unggahan
terpisah). Data MTD ini dipakai Penjualan dan menjadi **sumber omset gross MTD**
Analisa Performa Depo/Cabang; Dbase 6 Bulan Terakhir dipakai untuk pembanding
dan ROA.

- **Hanya data MTD (bulan berjalan)** yang ditampilkan. Bulan berjalan diambil
  dari bulan tanggal terakhir di data (bulan acuan yang sama dengan menu
  Analisa), sehingga kartu **Periode MTD** dan seluruh tabel hanya berisi bulan
  tersebut.
- Menampilkan penjualan **gross (`VALUE`)** dan **bersih (`VALUE NETT`)**,
  diskon, qty, jumlah outlet/salesman, tren harian, **tren per Type Outlet**,
  peringkat salesman (dengan **Target** & **Pencapaian** bila file target
  diunggah)/outlet/kategori produk (MARK NEW), filter **Depo**, pencarian
  outlet, dan filter channel. Semua tabel dan peringkat memakai **gross
  (`VALUE`)**; `VALUE NETT` hanya untuk kartu Penjualan bersih.
- **Filter Depo** mempersempit KPI, tren, type outlet, salesman, outlet, dan
  kategori ke depo terpilih (sumber: kolom depo/PMA Dbase/Master Outlet).
  **Tren Type Outlet** berupa stacked bar harian dari `TYPE OUTLET`/`JENIS
  OUTLET`/`CHANNEL` Master Outlet (fallback channel Dbase), dengan enam type
  teratas ditampilkan terpisah dan sisanya digabung sebagai **Lainnya**.
- Hasil parsing disimpan di **IndexedDB** (`analisaPenjualanLocalDB`, key
  `sales-mtd-v1`), dengan `localStorage` (`penjualanMtdLocalData.v1`) sebagai
  cadangan; **Hapus/Reset** di kartu Dbase MTD menghapus salinan lokal.

## Analisa Performa Depo & Cabang (sumber: Setting)

Menu **Analisa Performa Depo** di sidebar membuka analisa dengan **omset MTD
dari Dbase MTD** dan **pembanding dari Dbase 6 Bulan Terakhir** yang diunggah di
menu Setting. Menu **Analisa Performa Cabang** memakai **sumber data yang sama**
tetapi panel **Omset/ROA/EC by Depo** menggantikan panel salesman.

- Semua file Dbase digabung; file duplikat (nama, ukuran, waktu ubah sama)
  dilewati.
- **Omset memakai gross** (`VALUE`, lalu alias gross; `VALUE NETT` hanya
  cadangan). Semua label menulis **Omset Gross**.
- Perbandingan **MTD vs Last 1–6 Months**: omset MTD dari **Dbase MTD** (bulan
  berjalan), pembanding dari **Dbase 6 Bulan Terakhir** pada **HK (hari kerja)
  yang sama**, default **6 bulan**. Bulan berjalan di file pembanding tidak
  dihitung ganda. Bila Dbase MTD belum ada, MTD memakai bulan terakhir Dbase 6
  Bulan Terakhir sebagai fallback berpenanda. Tabel omset salesman/depo memakai
  total omset seluruh bulan pembanding; tabel bulanan juga menampilkan EC dan
  ROA per bulan. Bulan tanpa data HK sama ditampilkan `—`.
- Basis HK: Senin–Sabtu (default) atau Senin–Jumat, hari libur nasional
  dikecualikan. Baris pada hari non-kerja **tetap dihitung** dengan HK hari kerja
  berikutnya di bulan yang sama (agar total omset bulan sama dengan total
  `VALUE` di file).
- Filter divisi dari kolom `DIVISI`/`DIV` dengan fallback prefix salesman
  (`AEPDA-` → AEP, `GDKDA-` → GDK).
- Filter depo dari kolom `PMA`/`DEPO` pada data penjualan yang diunggah.
- **Rekap Depo**: saat **Semua depo** dipilih, panel **Rekap Depo** menampilkan
  ringkasan KPI MTD tiap depo (Omset MTD, rata-rata/HK, vs bulan lalu, total
  pembanding, outlet aktif, share MTD) plus baris total. Saat satu depo dipilih,
  panel disembunyikan dan analisa fokus ke depo itu.
- **Setting (kategori unggahan)**: header file master dideteksi dengan
  mencocokkan nama kolom, jadi file dengan header di baris 1 tetap terbaca
  (khusus **Master Outlet**, header dibaca dari baris 4). Master hanya
  melengkapi kolom yang kosong dan diterapkan ulang ke file yang sudah diunggah.
- Menampilkan tren omset bulanan beserta EC dan ROA, KPI MTD, rincian per
  depo/toko/kategori produk (MARK NEW)/salesman/channel, ROA per kategori produk
  & salesman, EC per salesman, dan **Insight & Action Plan** otomatis.
- Kartu KPI, baris tren bulanan, dan baris tabel depo/outlet/kategori/salesman/
  channel/ROA/EC dapat diklik untuk membuka modal detail berisi ringkasan dan
  rincian omset per outlet.
- **ROA** = jumlah **outlet bertransaksi** (distinct count) dalam sebulan (bukan
  omset ÷ outlet). Rincian omset & ROA dikelompokkan per kolom `MARK NEW`.
- Hasil parsing disimpan lokal di **IndexedDB**
  (`analisaPenjualanLocalDB`, key `analysis-v1`; master data `analysis-master-v2`
  termasuk Master Armada; stok `stok-gudang-v1`; SO/DO Logistik
  `logistik-sodo-v1`; Dbase MTD `sales-mtd-v1`), dengan `localStorage`
  (`analisaPenjualanLocalData.v1`) sebagai cadangan, sehingga tetap ada setelah
  refresh dan tidak memengaruhi pengguna lain.

## Analisa Stok Gudang (sumber: Setting)

Menu **Analisa Stok Gudang** menganalisa **Days of Inventory (DOI)** per depo
dari **kategori Stok Gudang di menu Setting**.

- Sesuai template `template-stok-gudang.xlsx`. Kolom: `NO`,
  `KODE ITEM`, `ITEM PRODUK`, lalu tiap depo (stok, `SELLOUT`, `DOI`);
  `SELLOUT LDC`/`DOI LDC` tanpa stok; `GST` tanpa DOI; ditutup `TOTAL QTT`,
  `TOTAL SELLOUT`, `TOTAL DOI`, `KATEGORI ECERAN`.
- **DOI Critical < 14 hari** (merah) dan **DOI Aman > 30 hari** (hijau) per depo.
- Panel **Item DOI Critical**, **Item DOI Aman (Surplus)**, dan **Saran
  Intra-Barang** (pindahkan stok dari depo terdekat yang surplus ke depo
  critical). Wilayah: Medan Raya, Sumatera Timur, Dataran Tinggi, Pantai Barat.
- **Filter depo** untuk membatasi analisa ke satu depo.
- Barang dengan DOI 0, sellout 0, dan stok 0 tidak ditampilkan. Bila file punya
  kolom **BI**, item hanya ditampilkan saat nilai BI **> 1**.
- Angka stok/sellout rapi (tanpa desimal bila bulat, maks. 2 desimal bila
  pecahan), DOI maks. 1 desimal. Tiap panel detail punya dropdown **Tampil**
  (10/25/50/100/semua) agar tampilan lebih simetris.
- Kartu KPI **Total Item/Critical/Aman/Depo** serta baris tabel dan daftar item
  dapat diklik untuk membuka modal detail stok per depo atau rincian item-depo.
- Hasil parsing disimpan lokal di **IndexedDB** (`analisaPenjualanLocalDB`, key
  `stok-gudang-v1`).

## JKS Editor

Menu **JKS Editor** membuka peta outlet full-screen dengan panel data di kiri.
Data diunggah manual dan diproses di browser; file tidak dikirim ke server.

- Format utama **Excel** (`.xlsx` / `.xls` / `.xlsb`). Sheet yang dipakai: sheet pertama,
  atau sheet bernama `JKS`, `DATA`, `OUTLET`, atau `PETA`.
- CSV, JSON, dan GeoJSON tetap diterima sebagai format cadangan.
- Kolom: `KODE OUTLET`, `NAMA OUTLET`, `LATITUDE`, `LONGITUDE`, `SALESMAN`,
  `RUTE`, `CYCLE`, `OMSET`, `EC`, `DEPO`, `STATUS`, plus kolom opsional
  `LATITUDE DEPO` dan `LONGITUDE DEPO`.
- Klik pin toko untuk membuka panel **Edit**. Gunakan **Lasso pilih** untuk
  memilih banyak toko dengan bentuk bebas, lalu ubah salesman/rute/cycle
  sekaligus. **Lasso area** memfilter outlet dengan poligon bebas.
- **Cluster** punya dua metode: **Traffic** (rayon berdekatan dibagi ke armada)
  dan **Blocking** (blok salesman utuh, tidak dipecah antar-armada).
- Pembagian armada memakai jumlah armada, kapasitas m³/armada, dan kubikasi
  per outlet. Estimasi jarak dihitung dari Pin Location; pilihan **Semua depo**
  memakai koordinat depo pada data atau isian manual.
- Data unggahan, hasil edit, dan pengaturan JKS tersimpan di `localStorage`
  perangkat (`petaJksLocalData.v1`) dan bertahan setelah refresh. Data contoh
  tidak disimpan; tombol **Reset** menghapus salinan lokal.
- Tombol **Template Excel** mengunduh contoh workbook. Tombol **Export**
  menyediakan **Export Excel**, **Export Peta HTML**, dan
  **Export Titik Koordinat** (CSV).
- Fitur peta: pencarian, filter salesman/rute/cycle/depo, fokus, marker
  cluster, dan basemap satelit/jalan/gelap.
- Tombol **Muat contoh** mengisi peta dengan 18 outlet demo tanpa file.

## Logistik

Menu **Logistik** membagi baris SO/DO ke **Master Armada** memakai koordinat
toko dan traffic salesman dari **JKS Editor**, nama/kode dari **Master Outlet**,
data sales dari **Master Salesman**, serta kapasitas kubikasi kendaraan. Sidebar
hanya berisi **Route Armada**, **Rekap Supir**, **Detail Supir**, dan **Upload
Data**. Semua proses berjalan di browser.

- **Template SO/DO** berisi tepat: `Delivery Order ID`, `Quotation ID`, `Order
  ID`, `Document Date`, `Delivery Date`, `Doc Type ID`, `Plant`, `Remarks`,
  `Route`, `Status ID`, `Sold To Customer ID`, `Salesman ID`, `Item ID`,
  `Product ID`, `Qty`, `Uom ID`, `Price`, `Gross Value`, `Discount Value`,
  `Net Value`, `Total Cubication`, `Total Amount`.
- **Filter faktur**: hanya baris dengan `Doc Type ID` **ZOP1** dan `Status ID`
  **1 atau 8** yang dihitung. Faktur **cancel**, tipe dokumen lain, dan status
  lain diabaikan dari grouping outlet, kubikasi, omset, dan statistik rute.
- **Blocking area**: outlet dikelompokkan per **salesman**; fallback **rute**,
  lalu sel geografis. Setiap blok dipertahankan utuh dan ditempatkan ke satu
  armada, sehingga satu supir tidak mengambil outlet acak yang tersebar.
- Blok hanya dipecah bila melebihi kapasitas/omset: pertama per sub-rute, lalu
  belahan **median geografis**. Blok kecil diproses lebih dulu; seed wilayah
  dipilih dengan **farthest-point**, lalu blok ditempatkan ke armada terdekat
  yang masih muat. Sisa Tinjauan diisi ke armada terdekat yang masih muat.
- Setiap armada dibatasi **100% kapasitas** dan omset **Rp17.000.000**.
  Kelebihan dialihkan ke armada lain; jika tidak muat, masuk **Tinjauan
  Armada**. Outlet di atas **Rp18.000.000** otomatis masuk **Tinjauan Armada
  Sewa**, begitu juga orderan **AEP** yang tidak muat. Baris dapat dicentang
  untuk **armada sewa** dengan kapasitas yang bisa diatur (default 8 m³).
- Urutan kunjungan memakai nearest-neighbor dari depo, lalu dirapikan dengan
  **2-opt** (4–140 titik, maksimal 3 pass); jarak memakai haversine.
- Hasil menampilkan armada (supir, jenis, traffic salesman, kapasitas,
  terpakai, outlet, omset, jarak), **Pemetaan Sales → Armada**, **Tinjauan
  Armada**, **Rekap Supir**, dan **Detail Supir**.
- **Rekap Supir** menampilkan jumlah outlet, turnover, dan total jarak. Klik
  baris supir untuk membuka **Detail Supir** berisi daftar toko, omset toko,
  kubikasi, jarak per segmen, dan jarak kumulatif sampai kembali ke Pin
  Location.
- **Route Armada** menampilkan **marker Pin Location berbentuk pin**, titik toko
  bernomor urut per armada, garis rute, dan marker tinjauan. Panel kiri memuat
  daftar armada/outlet dengan dropdown untuk memindahkan toko antar armada;
  outlet yang belum terbagi muncul di bagian **Belum ada armada** dengan
  dropdown yang sama, dan pilihan **Belum ada armada / Tinjauan**
  mengembalikan toko ke Tinjauan. Kubikasi, omset, urutan rute, dan jarak
  langsung dihitung ulang tanpa memfit ulang peta. Layer default **Satelit** (Esri World Imagery), dengan
  pilihan **Jalan** (Esri World Street Map), **Gelap** (Carto), dan toggle
  **Titik Tinjauan**.
- Data SO/DO disimpan di IndexedDB `logistik-sodo-v1`, dengan cadangan
  `localStorage["logistik-sodo-fallback-v1"]`.

## Cara paling mudah (lewat web GitHub)

1. Buat repository baru di GitHub, misalnya `omset-dashboard`.
   Pilih **Public** (GitHub Pages gratis untuk repo public).
2. Di halaman repo, klik **Add file → Upload files**.
3. Upload `index.html` dan `.nojekyll` dari folder ini.
   (File `.nojekyll` tidak terlihat di Explorer; aktifkan “show hidden files”
   atau upload lewat drag-and-drop folder.)
4. Commit ke branch `main`.
5. Buka **Settings → Pages**.
6. Di **Build and deployment → Source**, pilih **Deploy from a branch**.
7. Branch: `main`, folder: `/ (root)`, lalu **Save**.
8. Tunggu 1–2 menit. URL muncul di halaman Pages, bentuknya:

   ```text
   https://<username>.github.io/omset-dashboard/
   ```

Kalau nama repo-nya `<username>.github.io`, URL-nya:

```text
https://<username>.github.io/
```

## Cara lewat GitHub Desktop

1. Buka GitHub Desktop → **File → Add local repository**.
2. Pilih folder `github-pages` ini.
3. Klik **create a repository** / **publish repository**.
4. Setelah ter-push, buka **Settings → Pages** di GitHub dan aktifkan
   branch `main` folder `/ (root)`.

## Cara lewat Git CLI

```bash
cd github-pages
git init
git add .
git commit -m "Engine Dashboard statis"
git branch -M main
git remote add origin https://github.com/<username>/<repo>.git
git push -u origin main
```

Lalu aktifkan GitHub Pages di **Settings → Pages**.

## Setelah deploy

Buka URL GitHub Pages, lalu cek:

- KPI Omset, Target, Capaian, DO, dan Customer terisi.
- Badge di kanan atas menunjukkan **LIVE**.
- Kalau muncul **DATA LAMA**, berarti sheet belum diperbarui bot > 25 menit.
- Kalau muncul pesan **Sheet tidak dapat dibaca publik**, ubah sharing sheet
  menjadi **Anyone with the link → Viewer**.

## Kalau ingin mengubah data/config

File `index.html` di folder ini adalah hasil build. Jangan edit langsung.
Edit sumbernya di folder induk:

- `ai_studio/data.js` — ID sheet, nama tab, master salesman, aturan.
- `static/js/app.js` — tampilan/interaksi.
- `static/css/style.css` — CSS.

Lalu jalankan dari folder induk:

```bat
python build_ai_studio.py
```

File `github-pages/index.html` akan ikut diperbarui. Commit dan push lagi.

## Catatan keamanan

- GitHub Pages **publik** dan dashboard **tanpa password**.
- Sheet juga dapat dibaca publik lewat ID sheet yang tertanam di HTML.
- Jangan taruh data customer sensitif di repo/sheet publik.
- Kalau butuh akses terbatas, gunakan versi Flask + password atau proxy
  Google Apps Script; versi statis ini tidak mendukung login.
