# PrintBridge

Aplikasi Android yang menjembatani **halaman web → printer struk ESC/POS** (printer jaringan port 9100 & Bluetooth).
Web cukup memanggil `fetch("http://127.0.0.1:18080/print")`; PrintBridge yang meneruskan ke printer.

- Multi printer bernama (Utama, Dapur, Bar, … nama bebas), IP atau Bluetooth, kertas 58/80 mm, auto cut
- Cari & pairing printer Bluetooth dari dalam aplikasi
- Port server bisa diubah, akses dari jaringan lokal opsional
- Tanpa akun, tanpa cloud, tanpa library tambahan

**Cara pakai, API, contoh kode:** [docs.md](docs.md)

Unduh APK: halaman **Releases**. Dikembangkan oleh **@zonegoh**.
