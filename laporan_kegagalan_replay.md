# Laporan Analisis Kegagalan *Replay Packet* UDP/KCP

## Latar Belakang Masalah
Pengguna mencoba melakukan pengiriman (menembak) *raw packet* UDP/KCP ke server dari hasil rekaman aplikasi eksternal (PCAP Droid). Namun, server game ini selalu mengabaikan (drop) paket tersebut tanpa ada respons balik. Tujuan dari dokumen ini adalah untuk merinci mekanisme proteksi yang diimplementasikan oleh game dalam menolak *Replay Attack*.

## Alasan Teknis Penolakan Paket

Berdasarkan analisis *reverse engineering* menggunakan `radare2` pada file `libmoba.so` dan `libEncryptor.so`, berikut adalah alasan teknis yang mencegah injeksi paket mentah:

### 1. Conversation ID (`Conv ID`) yang Unik per Sesi
Protokol KCP sangat bergantung pada identifikasi sesi yang disebut `Conv ID` (`ikcp_getconv`). Ketika klien (`UdpPipeManager`) pertama kali terhubung ke server (melalui pemanggilan fungsi `parseCmdConnect` / `makeCmdConnect`), klien dan server menyepakati *Conversation ID* unik ini.

Paket hasil *dump* dari PCAP menyimpan `Conv ID` masa lalu yang sudah kedaluwarsa. Begitu server menerima paket KCP Anda saat ini, ia melihat `Conv ID` tersebut tidak cocok dengan status koneksi memori server untuk pemain Anda pada detik itu, sehingga KCP lapisan server akan secara otomatis mengeksekusi instruksi *drop packet*.

### 2. Time-Based Seed / Session Key (`JNI_OnLoad` & Kriptografi)
Game ini mengimplementasikan `libEncryptor.so` yang akan dipanggil saat inisiasi game berjalan (`JNI_OnLoad` memanggil fungsi `srand` dengan `time`). Artinya, setiap kali aplikasi berjalan atau melakukan *handshake* ke server, akan dibentuk kunci simetris (*Session Key*).
Kunci ini kemungkinan digunakan untuk mendekripsi struktur *payload* SDP atau untuk menyusun token *handshake*. Karena paket yang ditangkap (dari PCAP) memuat enkripsi dari *Session Key* masa lalu, dekripsi di sisi server gagal sepenuhnya, sehingga KCP Message dianggap *corrupt/invalid*.

### 3. KCP Message Authentication Code (MAC) / Hashing
Analisis *binary* terhadap `libmoba.so` menunjukkan keberadaan fungsi otentikasi kriptografi yang diekspor, yaitu:
- `MD5Init`, `MD5Update`, `MD5Final`
- `SHA1Init`, `SHA1Update`, `SHA1Final`
- *Parser Signer Info* (`_ZN5pkcs716parse_signerInfoEi`)

Hal ini mengindikasikan kuat bahwa game ini menerapkan **MAC (Message Authentication Code)**. Paket KCP kemungkinan memiliki *hash/checksum* di dalam struktur headernya (sebelum `ikcp_send` atau pada *layer* `UdpPipeManager::processSendMsg`). Saat sebuah paket direkam lalu dikirimkan ulang, jika ada satu *byte* (seperti `Timestamp KCP` atau *Sequence Number* `sn`) yang berubah, maka *hash* yang direkam di PCAP menjadi tidak sesuai (*invalid*).

### 4. Kompresi Zlib Dinamis
Game menggunakan kompresi kustom dengan fitur `KCP_EnableZip` dan `KCP_SetZipControlCode`. Terkadang, nilai kompresi (*Zip Control Code*) ini bisa berubah, atau struktur blok paket tidak dapat dibongkar secara terisolasi tanpa *state* kompresi yang dijaga oleh *session stream* KCP asli (`inflateSync`). Menembak paket mentah (yang terkompresi dari masa lalu) akan merusak *sliding window* Zlib di memori server.

## Kesimpulan dan Strategi *Workaround*
Keempat faktor di atas (*Conv ID*, *Session Key*, *Checksum MD5/SHA1*, dan *Zlib Window*) bekerja secara bersinergi sehingga **Replay Attack UDP (PCAP-based injection)** tidak mungkin dilakukan dari eksternal aplikasi.

**Strategi *Workaround*:**
Jika ingin mengirim/menginjeksi paket secara *custom* ke server, metode yang direkomendasikan adalah melakukan *Memory Hooking* (misal: menggunakan *Frida* atau *GameGuardian* yang dijalankan satu proses dengan aplikasi).
- **Target Hook**: *Hook* pada fungsi `KCP_SendMsg` (Offset `0x000ca540` di `libmoba.so`).
- **Alasan**: Dengan melakukan injeksi pada fungsi `KCP_SendMsg`, Anda menyerahkan tugas penambahan `Conv ID`, penyesuaian *Timestamp*, perhitungan *Checksum MD5/SHA1*, dan Kompresi langsung kepada *Engine* KCP game itu sendiri. Dengan demikian, *Engine* akan membungkus modifikasi *payload* buatan Anda (misal modifikasi Posisi X/Y) seolah-olah itu adalah paket yang sah dari game di sesi aktif saat ini.
