# VSK Bahan Baku — Setup Guide

Web app PWA untuk pencatatan kedatangan bahan baku cocopeat, dengan flow penimbangan per-karung dan sinkronisasi otomatis ke modul VSK Produksi via Google Sheets.

---

## Arsitektur

```
┌─────────────────────────┐         ┌──────────────────────────┐
│  vsk-bahan-baku-staging │         │  vsk-bahan-baku          │
│  (CF Worker, static)    │         │  (CF Worker, static)     │
│  hostname: *staging*    │         │  hostname: production    │
└────────────┬────────────┘         └────────────┬─────────────┘
             │ POST/GET                           │ POST/GET
             ▼                                    ▼
┌─────────────────────────┐         ┌──────────────────────────┐
│  Apps Script STAGING    │         │  Apps Script PRODUCTION  │
│  (bound to Sheet A)     │         │  (bound to Sheet B —     │
│                         │         │   yang existing dengan   │
│                         │         │   tab Produksi)          │
└────────────┬────────────┘         └────────────┬─────────────┘
             ▼                                    ▼
   ┌──────────────────┐                ┌──────────────────────┐
   │ Spreadsheet      │                │ Spreadsheet PROD     │
   │ STAGING (baru)   │                │ 1evJ9Eo... (existing)│
   │ - Produksi       │                │ - Produksi (live)    │
   │ - Rawmat_Keda... │                │ - Rawmat_Kedatangan  │
   │ - Rawmat_Karung  │                │ - Rawmat_Karung      │
   │ - Suppliers      │                │ - Suppliers          │
   └──────────────────┘                └──────────────────────┘
```

**Frontend identik di kedua repo** — environment di-detect otomatis dari hostname. Kalau hostname mengandung `staging` atau `localhost`, app pakai URL Apps Script staging; selain itu pakai URL production.

---

## Langkah Setup

### 1. Buat Google Spreadsheet STAGING (baru)

1. Buat spreadsheet baru di Google Drive, kasih nama misalnya `VSK Sistem — Staging`
2. Bisa biarkan kosong, tab `Produksi`, `Rawmat_Kedatangan`, `Rawmat_Karung`, `Suppliers` akan auto-created saat script pertama kali dipanggil.
3. **Penting**: kalau mau test rekonsiliasi stock, isi tab `Produksi` dengan beberapa baris dummy (manual atau lewat aplikasi VSK Produksi yang di-point ke staging).

### 2. Pasang Apps Script v6 di KEDUA spreadsheet

Untuk **spreadsheet PRODUCTION** (yang existing, `1evJ9Eo3O9UxwFlQAd1TlDRobsCx5fx1VSNbJ1uVPNrA`):

1. Buka spreadsheet → **Extensions → Apps Script**
2. Hapus seluruh kode v5 yang ada
3. Paste seluruh isi `VSK_AppsScript_v6.js`
4. Save (Ctrl+S)
5. **Deploy → Manage Deployments → klik Edit (pensil) di deployment yang sudah ada → Version: New version → Deploy**
6. URL **tidak berubah** — VSK Produksi yang sudah live otomatis pakai script v6 (backward compatible, endpoint lama tetap jalan)

Untuk **spreadsheet STAGING**:

1. Buka spreadsheet staging → **Extensions → Apps Script**
2. Paste isi `VSK_AppsScript_v6.js`
3. Save
4. **Deploy → New deployment → Type: Web app → Execute as: Me → Who has access: Anyone → Deploy**
5. **Copy Web App URL yang dihasilkan** — ini yang akan dipakai untuk staging.

### 3. Update `index.html` dengan URL staging

Buka `index.html`, cari baris ini di awal `<script>`:

```js
var SHEETS_URL_STAGING = 'PASTE_STAGING_APPS_SCRIPT_URL_HERE';
var SHEETS_URL_PROD    = 'https://script.google.com/macros/s/AKfycbw3pWXUCvStevJLIA2x1r7n39Hb2UrCNYuGzwUbzLVpMtyd_ICBMS8RLR3ZqLo9_PqRaw/exec';
```

Replace `PASTE_STAGING_APPS_SCRIPT_URL_HERE` dengan URL yang baru kamu deploy di langkah 2.

URL prod sudah aku set dari script v5 yang existing — kalau setelah deploy v6 URL berubah (harusnya tidak), update juga di sini.

### 4. Setup repo & Cloudflare Worker

**Repo staging** (`vsk-bahan-baku-staging`):
1. Buat repo baru di GitHub
2. Push 3 file: `index.html`, `manifest.json`, `sw.js`
3. Setup Cloudflare Worker baru, deploy ke hostname yang mengandung `staging`, contoh: `vsk-bahan-baku-staging.variantony1.workers.dev`

**Repo production** (`vsk-bahan-baku`):
1. Buat repo baru, push 3 file yang **identik** dengan staging
2. Setup CF Worker → hostname tanpa "staging", contoh: `vsk-bahan-baku.variantony1.workers.dev`

Karena environment auto-detect dari hostname, tidak perlu file berbeda antara kedua repo. Kalau ada update, tinggal copy-paste antara kedua repo (atau pakai git-flow).

---

## Verifikasi Setup

Buka URL staging di browser. Kalau setup benar:
- Topbar nampak badge orange "STAGING" di tengah atas
- Tab "Kedatangan" load tanpa error → ada 2 kemungkinan:
  - Belum ada batch aktif → form mulai kedatangan baru muncul
  - Ada batch open → langsung mode timbang

Buka URL prod → tidak ada badge "STAGING".

Test flow lengkap di staging:
1. Mulai kedatangan baru (isi operator + supplier, klik Mulai Timbang)
2. Timbang 3-5 karung → pastikan running total update
3. Coba flag salah satu karung → pastikan total tidak menghitung yang flagged
4. Tutup batch
5. Cek tab Stock → saldo = total karung valid (- 0 pemakaian kalau Produksi staging kosong)
6. Cek tab Riwayat → batch tadi muncul dengan status `closed`
7. Klik baris di Riwayat → drawer detail muncul

Kalau semua hijau, baru promote ke production.

---

## Schema Google Sheets

Tab-tab yang auto-created di kedua spreadsheet:

**`Produksi`** (existing, schema dari v5 — tidak berubah):
| Timestamp Server | Tanggal | Shift | Operator | Karung | Basah (kg) | Kering (kg) | Block (pcs) | Susut (%) | Catatan | Input Time |

**`Rawmat_Kedatangan`** (header per batch):
| Timestamp Server | Batch ID | Tanggal | Supplier | Surat Jalan | Total Karung | Total Berat (kg) | Status | Operator | Catatan | Closed Time |

- `Batch ID` format: `B-YYYYMMDD-NNN` (auto-generated, sequence harian)
- `Status`: `open` / `closed` / `cancelled`
- `Total Karung` & `Total Berat` di-update otomatis oleh script saat ada karung baru atau flag

**`Rawmat_Karung`** (detail per karung):
| Timestamp Server | Batch ID | Urut | Berat (kg) | Flagged | Flag Reason | Operator | Time |

- 1 row = 1 karung. Kalau batch punya 300 karung, 300 row.
- `Flagged = TRUE` → tidak dihitung ke total/saldo, tapi tetap tersimpan untuk audit trail

**`Suppliers`** (untuk fase berikutnya, sekarang belum dipakai aktif):
| Timestamp Server | Nama | Kontak | Alamat | Harga/kg | Aktif | Catatan |

---

## Kalkulasi Stock

```
Total Masuk (kg)  = SUM(Rawmat_Karung.Berat) WHERE Flagged = FALSE
                    AND Batch.Status IN ('open', 'closed')
Total Keluar (kg) = SUM(Produksi.Basah)
Saldo Stock (kg)  = Total Masuk - Total Keluar
```

Asumsi: kolom `Basah (kg)` di tab `Produksi` adalah berat bahan baku cocopeat yang dipakai dari stock per shift. Kalau asumsi ini salah, beritahu — formula gampang diubah.

---

## API Endpoints

Semua via 1 URL Apps Script per environment.

**GET endpoints:**
- `?action=test` → ping
- `?action=rekap[&date=YYYY-MM-DD]` → rekap produksi (legacy)
- `?action=riwayat` → riwayat 30 hari produksi (legacy)
- `?action=debug` → debug produksi (legacy)
- `?action=rawmat-stock` → saldo stock + 10 mutasi terakhir
- `?action=rawmat-batches` → list batch (50 terbaru)
- `?action=rawmat-batch&id=B-...` → detail batch + karung
- `?action=rawmat-active` → batch status open (kalau ada)
- `?action=rawmat-debug` → debug rawmat

**POST endpoints** (body: JSON dengan field `type`):
- `{type: 'produksi', ...}` — atau tanpa `type` sama sekali → simpan shift produksi (legacy)
- `{type: 'rawmat-start', operator, supplier, suratJalan, notes}` → mulai batch
- `{type: 'rawmat-karung', batchId, urut, berat, operator}` → tambah karung
- `{type: 'rawmat-flag', batchId, urut, reason}` → toggle flag
- `{type: 'rawmat-close', batchId}` → tutup batch
- `{type: 'rawmat-cancel', batchId}` → batal batch (hanya kalau belum ada karung)

---

## Rules & Constraints

- **Hanya 1 batch open di waktu yang sama.** Kalau coba mulai batch baru sementara ada yang open, server tolak. Selesaikan/cancel batch yang ada dulu.
- **Karung tidak bisa dihapus.** Hanya bisa di-flag. Flag bisa dicabut. Audit trail tetap utuh.
- **Cancel** hanya bisa kalau batch belum ada karung. Kalau sudah ada karung tapi mau dianggap salah, flag semua karung lalu close.
- **Total di header** (`Total Karung`, `Total Berat`) auto-recompute setiap kali ada perubahan karung.

---

## Roadmap (improvements untuk fase berikutnya)

- Foto surat jalan (upload ke Drive, simpan link)
- Dropdown supplier dengan auto-fill alamat (sekarang text input bebas)
- Master supplier UI (sekarang tab `Suppliers` tersedia tapi tidak ada UI input)
- Edit metadata batch setelah close (catatan, supplier)
- Export Excel/PDF laporan kedatangan
- Bluetooth/serial scale integration (auto-input berat)
- Offline queue (simpan POST di IndexedDB kalau offline, retry saat online)
- Dashboard rekonsiliasi terdedikasi (stock vs produksi per periode)
