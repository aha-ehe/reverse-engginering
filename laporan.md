# Laporan Analisis Komunikasi KCP dan Payload Game

## Pendahuluan
Berdasarkan hasil *reverse engineering* pada biner aplikasi yang dibagikan (`libmoba.so`), telah ditemukan implementasi protokol komunikasi server dan aplikasi. Game ini menggunakan protokol **KCP** yang dienkapsulasi menggunakan protokol UDP. Komunikasi paket dikelola menggunakan **UdpPipeManager**, sementara serialisasi/deserialisasi (packing/unpacking) data dilakukan menggunakan **SDP (Service Data Protocol)**.

## Alur Komunikasi Jaringan
Komunikasi utama antara klien dan server dikendalikan secara asinkron (atau *event-driven*) yang melibatkan *timer* dan mekanisme *polling*. Beberapa fungsi utama yang mengatur koneksi adalah:

1. **Inisialisasi & Koneksi**
   - `_ZN14UdpPipeManager10initializeERK13UdpPipeConfig` (0x0aa2ac): Digunakan untuk inisialisasi pengaturan UdpPipeManager dari konfigurasi internal.
   - `_ZN14UdpPipeManager10createPipeERK18UdpPipeCreateParamRj` (0x0acb74): Membuat *pipe* (saluran UDP baru) antar aplikasi dan server.
   - `_ZN14UdpPipeManager15startConnectTcpENSt6__ndk110shared_ptrI14PipeConnectionEE` (0x0b6e40): Terlihat ada *fallback* TCP jika koneksi UDP bermasalah (Dual Channel didukung via `EnableDualChannelEb`).

2. **Loop Utama / Penerimaan Paket KCP**
   - `ikcp_recv` (0x0ce7a0): Fungsi bawaan *skywind3000 KCP* untuk menerima data dari lapisan KCP.
   - `KCP_GetReceiveData` / `KCP_ReceiveCycleWithHandle` (0x0ca1e8): Pembungkus untuk mengambil data yang masuk dan menjadwalkan siklus penerimaan KCP.
   - `_ZN14UdpPipeManager9recvEventERNSt6__ndk16vectorINS0_10shared_ptrI8UdpEventEENS0_9allocatorIS4_EEEE` (0x0ac2a0): Membaca *event* UDP/TCP masuk.

3. **Pengiriman Paket KCP**
   - `KCP_SendMsg` (0x0ca540): Dipanggil oleh game saat ada pesan yang akan dikirim (contoh: payload pergerakan karakter).
   - `ikcp_send` (0x0ceb0c): Menempatkan paket ke dalam antrian *send* di algoritma KCP.
   - `_ZN14UdpPipeManager14processSendMsgEv` (0x0b92a8): Memproses pesan dari antrian *send* untuk dikirim ke *remote socket*.

## Struktur Payload (Pergerakan Karakter)
Proses (packing/unpacking) payload pergerakan ditangani oleh struktur *class* C++ seperti `mfw::OperType_Battle_Move`. Hasil analisis menggunakan `radare2` dari fungsi de-serialisasinya (`mfw::OperType_Battle_Move::visit(mfw::SdpUnpacker, bool)`) pada offset `0x0e1bb8` menghasilkan format payload *Tag-Length-Value* (TLV) yang dimodelkan sebagai berikut:

Struktur koordinat/pergerakan karakter dibaca berdasarkan tag identifier numerik spesifik:
- **Tag 2**: Membaca/Menulis Nilai **Posisi / Arah X (`fPosX`, `fDirX`)**. Disimpan dan diinterpretasikan sebagai **32-bit Float**.
- **Tag 3**: Membaca/Menulis Nilai **Posisi / Arah Y (`fPosY`, `fDirY`)**. Disimpan dan diinterpretasikan sebagai **32-bit Float**.

*Data float ini (SdpPacker/SdpUnpacker menggunakan instruksi `fcmp s0, 0.0` pada register FPU) di-*pack* ke dalam *stream byte* dan kemudian ditransmisikan via KCP*.

## Enkripsi / Kompresi
Proses komunikasi KCP pada sistem ini mendukung kompresi (Zlib) secara *native*, dan diaktifkan melalui fungsi *hook* berikut:
- `KCP_EnableZip` / `KCP_IsZipEnabled`
- `KCP_SetZipControlCode` / `KCP_GetZipControlCode`
- Terdapat fungsi zlib (`inflateSync`) yang dikonfigurasi di dalam `libmoba.so`.

Selain itu, ada indikasi enkripsi menggunakan library pihak ketiga `libEncryptor.so` yang dipanggil kemungkinan saat proses *login/handshake* pertama. Namun, aliran *in-game* (pergerakan karakter) yang dikemas oleh KCP diproses secara langsung oleh UdpPipeManager menggunakan enkripsi yang mungkin bergantung pada zip / algoritma *custom XOR* (bila `zipControlCode` bukan murni deflate).

## Referensi Fungsi Esensial KCP

Berikut daftar *offset* absolut fungsi penting dari file ELF `libmoba.so`:

| Nama Fungsi | Offset | Keterangan |
| --- | --- | --- |
| `ikcp_create` | `0x000ce300` | Inisialisasi control block KCP baru. |
| `ikcp_recv` | `0x000ce7a0` | Mengekstrak data stream dari KCP. |
| `ikcp_send` | `0x000ceb0c` | Memasukkan byte raw ke dalam send buffer KCP. |
| `KCP_SendMsg` | `0x000ca540` | Wrapper khusus C++ dari Engine untuk mengirim ke KCP. |
| `KCP_GetReceiveData` | `0x000ca310` | Wrapper Engine untuk *fetching* buffer KCP. |
| `UdpPipeManager::initialize` | `0x000aa2ac` | Inisialisasi Pipe UDP KCP. |
| `UdpPipeManager::createPipe` | `0x000acb74` | Connect Pipe. |
| `mfw::OperType_Battle_Move::visit` | `0x000e180c` | Serialisasi Data (SdpPacker). |

---
**Catatan:** Analisis lebih dinamis atau *real-time patch* (contoh dengan Frida) dapat dilakukan dengan melakukan *hooking* ke `KCP_SendMsg` (0x000ca540) untuk menyadap dan mengubah *byte* paket mentah sebelum dikirim ke `ikcp_send`, atau di `KCP_GetReceiveData` untuk *cheat/radar-hack*.