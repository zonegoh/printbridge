# PrintBridge — Dokumentasi

PrintBridge adalah aplikasi Android kecil yang menjadi **jembatan dari halaman web ke printer struk ESC/POS**.
Browser tidak bisa membuka koneksi TCP (printer jaringan port 9100) atau Bluetooth Classic, jadi web Anda mengirim
data cetak ke server lokal PrintBridge di HP/tablet, lalu PrintBridge meneruskannya ke printer.

```
Web (Chrome di HP/tablet)  ──fetch──▶  PrintBridge  http://127.0.0.1:18080
                                          ├─ TCP 9100 ──────────▶ printer jaringan (IP)
                                          └─ Bluetooth (SPP) ──▶ printer Bluetooth
```

- Gratis, tanpa akun, tanpa cloud. Semua setelan tersimpan di perangkat.
- Multi printer bernama: printer **default** (urutan pertama, bawaan "Utama") + printer lain ("Dapur", "Bar", …). Semua nama bisa diganti.
- Setiap printer: jenis IP atau Bluetooth, ukuran kertas 58/80 mm, auto cut.
- Android 8.0 ke atas.

---

## 1. Pasang & atur

1. Unduh `printbridge.apk` dari halaman **Releases**, lalu pasang (izinkan *instal dari sumber tidak dikenal*).
2. Buka **PrintBridge**. Server langsung menyala: kartu atas menampilkan **Ready · 127.0.0.1:18080**.
3. Di kartu **Printer**, ketuk nama printer untuk membuka setelannya:
   - **Nama printer**: bebas diganti (printer pertama = default, bawaan "Utama").
   - **IP (jaringan)**: isi **Alamat IP** printer dan **Port** (umumnya `9100`).
   - **Bluetooth**: ketuk **Cari printer**. Printer yang sudah di-pair langsung tampil, printer di sekitar menyusul.
     Memilih printer *baru* akan memasangkannya (PIN biasanya `0000` atau `1234`).
   - Pilih **Ukuran kertas** dan nyalakan **Auto cut** bila printer punya pemotong.
   - Ketuk **Cetak tes**.
4. **+ Tambah printer** untuk printer berikutnya (mis. *Dapur*, *Bar*). Nama inilah yang dipakai web.
5. **Pengaturan perangkat › Baterai: tidak dibatasi** agar Android tidak mematikan PrintBridge.

Semua setelan tersimpan otomatis.

### Pengaturan server

| Setelan | Default | Keterangan |
|---|---|---|
| Port | `18080` | 1024 – 65535. Web memanggil `http://127.0.0.1:<port>` |
| Akses dari jaringan lokal | Mati | Nyala: server juga terbuka di `192.168.x.x:<port>`, sehingga perangkat lain di WiFi yang sama (mis. PC) bisa mencetak lewat perangkat ini |

> **Keamanan:** PrintBridge menerima permintaan dari **situs mana pun** yang dibuka di perangkat itu, dan, bila akses
> jaringan lokal dinyalakan, dari perangkat mana pun di WiFi yang sama. Printer IP hanya boleh beralamat lokal
> (`192.168.x.x`, `10.x.x.x`, `172.16–31.x.x`), jadi PrintBridge tidak bisa dipakai mengirim data ke internet.

---

## 2. API

Semua jawaban `application/json`. CORS terbuka untuk semua origin, dan preflight Private/Local Network Access dijawab.

### `GET /status`

```json
{
  "ok": true, "app": "printbridge", "version": "1.0.0", "version_code": 1,
  "port": 18080, "lan": false,
  "printer_type": "bt", "paper_width": 58, "auto_cut": true,
  "ip_printer": "", "bt_printer": "RPP02",
  "printers": [ { "name": "Utama", "default": true, "type": "bt", "width": 58, "cut": true, "ready": true, "summary": "Bluetooth RPP02 · 58 mm · auto cut" } ]
}
```

`printer_type`, `paper_width`, `auto_cut`, `ip_printer`, `bt_printer` adalah ringkasan printer **default** (urutan pertama).

### `GET /printers`

```json
{ "ok": true, "printers": [ { "name": "Utama", "default": true, "type": "ip", "width": 80, "cut": true, "ready": true, "summary": "IP 192.168.1.50:9100 · 80 mm · auto cut" }, { "name": "Dapur", "...": "..." } ] }
```

### `POST /print`

```json
{ "printer": "default", "escpos": "<base64 byte ESC/POS>" }
```

| `printer` | Tujuan |
|---|---|
| `"default"` (atau kosong) | printer default (urutan pertama, apa pun namanya) |
| `"name:Dapur"` | printer bernama *Dapur* (huruf besar/kecil tidak berpengaruh) |

Jawaban: `{"ok": true}` atau `{"ok": false, "error": "pesan yang bisa langsung ditampilkan ke pengguna"}`.

- `escpos` = **byte ESC/POS mentah** dalam base64 (teks, logo/raster, QR, dll. dibuat oleh aplikasi web/server Anda).
- **Auto cut** ditambahkan PrintBridge di akhir data bila setelan printer itu menyalakannya, jadi jangan mengirim perintah potong sendiri.
- Satu printer menerima satu cetakan dalam satu waktu; printer lain tetap jalan. Pengiriman yang menggantung diputus setelah 30 detik.

---

## 3. Contoh dari web

```js
const BRIDGE = "http://127.0.0.1:18080";

// cek aplikasi (panggil dari klik tombol, lihat bagian Chrome di bawah)
const status = await fetch(BRIDGE + "/status").then((r) => r.json());

// struk ESC/POS sederhana -> base64
const text = "\x1b@" + "TOKO SAYA\n" + "Kopi Susu x2   36.000\n" + "TOTAL         36.000\n\n\n";
const escpos = btoa(text);

const res = await fetch(BRIDGE + "/print", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ printer: "default", escpos }),
}).then((r) => r.json());

if (!res.ok) alert(res.error);
```

Mengirim ke printer lain: `{ printer: "name:Dapur", escpos }`.
Lebar baris mengikuti kertas: 58 mm ≈ 32 karakter, 80 mm ≈ 48 karakter (`paper_width` di `/status` atau `width` di `/printers`).

### Chrome & izin "aplikasi di perangkat"

- `http://127.0.0.1` dianggap aman, jadi halaman **HTTPS** boleh memanggilnya.
- Chrome versi baru (Local Network Access) meminta izin sekali: *"… ingin mengakses aplikasi & layanan di perangkat ini"*. Pilih **Izinkan**.
- Permintaan **pertama** harus berasal dari **klik** (mis. tombol *Tes printer*). Permintaan tanpa klik saat halaman dibuka bisa ditolak permanen.
- Bila sudah terlanjur diblokir: ikon di kiri alamat › **Izin** › aktifkan **Aplikasi di perangkat**, atau **Reset izin**.

---

## 4. Masalah umum

| Gejala | Penyebab & solusi |
|---|---|
| Web: "aplikasi tidak terhubung" | PrintBridge belum dibuka / server mati (kartu atas harus **Ready**), port berbeda, atau izin Chrome ditolak |
| `Printer "Bar" belum ada di aplikasi` | Tambahkan printer bernama persis *Bar* di PrintBridge |
| Printer IP tidak bisa dihubungi | Printer & HP harus satu WiFi; cek IP printer (cetak self-test: tahan FEED saat menyalakan) dan port 9100 |
| Bluetooth: tidak muncul di Cari printer | Nyalakan printer & Bluetooth HP; beri izin **Perangkat di sekitar**; printer hanya-BLE tidak didukung (butuh Bluetooth Classic/SPP) |
| Printer tidak merespons (30 detik) | Kertas habis / tutup printer terbuka / printer macet |
| Huruf aneh | Printer ESC/POS hanya punya set karakter terbatas; hindari emoji dan simbol khusus |
| Server mati sendiri | Atur **Baterai: tidak dibatasi**; buka PrintBridge sekali setelah HP restart bila server tidak menyala otomatis |

---

## 5. Build dari kode

Butuh JDK 17 dan Android SDK (`local.properties` → `sdk.dir=…`).

```sh
./publish.sh                                   # -> dist/printbridge.apk + version.json
PUBLISH_TO=../web/public ./publish.sh           # juga salin printbridge.apk + printbridge.json ke situs Anda
```

Rilis: naikkan `versionCode` & `versionName` di `app/build.gradle`, jalankan `./publish.sh`, upload
`dist/printbridge.apk` ke GitHub Release, lalu commit `version.json` (dipakai tombol **Cek update** di aplikasi).

APK release saat ini ditandatangani *debug key* (lihat `app/build.gradle`). Untuk distribusi luas / Play Store,
buat keystore sendiri dan simpan baik-baik: update hanya bisa dipasang di atas versi lama bila ditandatangani key yang sama.

---

Dikembangkan oleh **@zonegoh**.
