# Laporan Analisis Lanjutan (Tahap 2): Struktur Raw Packet, Handshake & Kriptografi

## Pendahuluan
Berdasarkan hasil analisis *reverse engineering* yang lebih mendalam pada *binary* `libmoba.so` dan `libEncryptor.so`, kita telah mengidentifikasi urutan *byte* mentah (Raw Packet) dan proses pembentukan (Handshake) sesi komunikasi sebelum protokol KCP mengirimkan *payload* permainan (seperti pergerakan karakter).

## 1. Proses Handshake UDP KCP & Magic Bytes
Game ini menggunakan kelas `UdpPipeManager` (dengan sub-modul `mfw::ReliableUdp`) untuk mengelola siklus hidup koneksi. Sebelum KCP `Conv ID` (*Conversation ID*) diberikan untuk bertukar pesan, klien harus melakukan *handshake* awal menggunakan UDP reguler.

*Handshake* ini dibedakan menggunakan *Magic Byte / Control Character* di awal pesan (byte ke-0):

*   `makeCmdConnect`: Dikirim oleh klien untuk meminta inisialisasi. Paket diawali dengan karakter `'q'` (`0x71`).
*   `makeCmdEstablish`: Dikirim untuk mengonfirmasi / merespons *handshake* dengan server. Paket menggunakan karakter `'r'` (`0x72`).
*   Terdapat pula *command* identifikasi mesin yang menyematkan *string* tanda tangan khusus, yaitu karakter `'Q'` (`0x51`) diikuti dengan *string* internal `"F:/mlot"`.
*   `makeCmdDisconnect`: Digunakan untuk terminasi koneksi (*graceful disconnect*), diawali karakter `'s'` (`0x73`).

**Struktur Kasar Handshake Packet:**
`[ Control Char (1 byte, misal 'q') ] [ Version/Length (4 byte) ] [ Session Token (dinamis) ] [ Padding/Padding Size ] [ Checksum MAC ]`

## 2. Validasi KCP Protocol Header (Bukan "Custom KCP")
Ada anggapan bahwa game ini memodifikasi algoritma internal KCP secara *custom* (seperti *byte-swapping*). Namun, dengan melakukan *tracing* pada instruksi Assembly fungsi *encoder* KCP:
- `ikcp_encode8u` (Offset: `0x0ce18c`)
- `ikcp_encode16u` (Offset: `0x0ce1a0`)
- `ikcp_encode32u` (Offset: `0x0ce1b4`)
- `ikcp_encode64u` (Offset: `0x0ce1c8`)

Terbukti bahwa instruksi yang digunakan murni berupa `strb`, `strh`, `str` standar. Artinya, lapisan `ikcp_send` hingga `ikcp_output` benar-benar menghasilkan **24-byte KCP Header Standar** (terdiri dari `conv`, `cmd`, `frg`, `wnd`, `ts`, `sn`, `una`, `len`).

**Mengapa sering dianggap "Custom"?**
Yang membuat koneksinya sulit dibajak / dipalsukan *bukanlah* KCP-nya yang *custom*, melainkan apa yang dilakukan **sebelum** dan **sesudah** KCP memproses *buffer*:

1.  **Sebelum (Payload Data)**: *Payload* pergerakan (*Battle_Move*) diserialisasi secara *custom* oleh *Service Data Protocol* (SDP Packer) menggunakan urutan indeks *Tag-Length-Value* (Tag 2 untuk Posisi X, Tag 3 untuk Posisi Y dengan format 32-bit *Float*).
2.  **Sesudah (UDP Send Buffer)**: Begitu blok 24-byte KCP + SDP Payload selesai dibuat oleh `ikcp_output`, `UdpPipeManager` menambahkan Header *Routing* Kustom UDP milik Engine, mengompresinya (jika Zlib aktif via `KCP_EnableZip`), lalu menempelkan *MAC Checksum*.

## 3. Otorisasi Kriptografi (MAC Checksum) & `libEncryptor.so`
Fungsi kriptografi ditanam secara statis dalam `libmoba.so`:
- Memuat algoritma **MD5** (`MD5Init`, `MD5Update`, `MD5Final`).
- Memuat algoritma **SHA1** (`SHA1Init`, `SHA1Update`, `SHA1Final`).
- Memuat verifikasi tanda tangan digital `pkcs7::parse_signerInfo`.

Algoritma *hash* ini berfungsi sebagai **MAC (Message Authentication Code)**. Paket KCP + SDP yang terkompresi tidak langsung dikirim ke `sendto()`. Ia di-*hash* bersama dengan *Session Key* rahasia. MAC ini ditempatkan sebagai validitas di akhir atau awal paket UDP.

**Peran `libEncryptor.so`:**
Saat game dijalankan, `JNI_OnLoad` pada `libEncryptor.so` (Offset: `0x00000c38`) memanggil `GetSystemTime` (via instruksi `ldr x0, [x19]` dan manipulasi stack). *Timestamp* ini dijadikan *random seed* (`srand`) untuk menggeneralisasi atau mengacak *Session Key* yang nantinya dipakai di `libmoba.so`.

Akibatnya, kunci yang dipakai untuk *hash* MAC berubah di setiap sesi. Ini adalah *root cause* mutlak mengapa *Replay Packet* mentah via UDP akan selalu ditolak secara sepihak oleh server, meskipun struktur SDP TLV KCP di dalamnya benar.

## Kesimpulan Akhir
Protokol game ini adalah kombinasi dari **SDP (Layer Presentasi) + KCP Standar (Layer Transpor / Multiplexing) + UdpPipeManager Custom Header & Kriptografi MAC (Layer Keamanan / UDP Socket)**.

Bagi analis keamanan atau pengembang modifikasi, satu-satunya titik injeksi logis yang valid adalah *function hooking* langsung pada API `KCP_SendMsg` atau sub-rutin *SDP Packer* di memori (`libmoba.so`). Melakukan injeksi *raw packet* melalui antarmuka *network card/Wireshark* sepenuhnya mustahil karena MAC *Session Key* berubah per siklus peluncuran.