# Modul 05 — EtherChannel

## Deskripsi

Praktikum ini membahas **EtherChannel (link aggregation) dengan protokol LACP** menggunakan Cisco Packet Tracer. Dua switch 2960 (SW-REKTOR dan SW-LABKOM) dihubungkan dengan empat kabel paralel, lalu keempat port digabung menjadi satu interface logis **Port-channel 1** yang membawa trunk VLAN 10 dan 20. Dua PC di VLAN 10 dipakai untuk menguji konektivitas, termasuk uji redundansi saat sebagian link fisik dimatikan.

Penyesuaian terhadap modul asli, karena batasan switch 2960 di Packet Tracer:

| Di modul | Di praktikum ini | Alasan |
|---|---|---|
| Gi0/1 - Gi0/4 | **Fa0/1 - Fa0/4** | Switch 2960 hanya punya 2 port Gigabit |
| Port-channel 10 | **Port-channel 1** | Switch 2960 hanya menerima channel-group nomor 1 sampai 6 |
| PC di Fa0/10 | **PC di Fa0/5** | Pilihan konfigurasi, tidak bentrok dengan port bundel |
| Bandwidth teori 4 Gbps | **400 Mbps** | 4 port x 100 Mbps |

Konsep dan alur konfigurasi tidak berubah, hanya nomor port, nomor grup, dan bandwidth total.

## Tujuan

- Menjelaskan konsep link aggregation dan cara kerja EtherChannel dengan LACP.
- Menggabungkan beberapa port fisik menjadi satu Port-channel logis.
- Mengonfigurasi trunking VLAN di atas Port-channel.
- Memverifikasi status EtherChannel dengan perintah `show`.
- Menguji redundansi: koneksi tetap hidup saat sebagian link fisik mati.

## Dasar Teori

### Masalah yang diselesaikan

Jika dua switch dihubungkan dengan beberapa kabel paralel tanpa EtherChannel, Spanning Tree Protocol (STP) melihatnya sebagai loop dan memblokir semua kabel kecuali satu. Bandwidth tambahan terbuang dan kabel cadangan hanya menganggur.

### EtherChannel

EtherChannel menggabungkan beberapa link fisik menjadi **satu link logis** bernama **Port-channel**. Bagi STP, bundel ini dianggap satu kabel sehingga tidak ada port yang diblokir. Manfaatnya:

- **Bandwidth akumulasi:** 4 link x 100 Mbps menjadi sekitar 400 Mbps.
- **Redundansi:** jika satu link putus, trafik pindah otomatis ke link yang tersisa tanpa memutus koneksi.
- **Load balancing:** trafik dibagi ke beberapa link berdasarkan hash (misalnya MAC atau IP).

### LACP dan PAgP

| Aspek | LACP | PAgP |
|---|---|---|
| Standar | IEEE 802.3ad (terbuka) | Milik Cisco |
| Dukungan vendor | Multi-vendor | Hanya Cisco |
| Mode memulai negosiasi | `active` | `desirable` |
| Mode menunggu | `passive` | `auto` |
| Pasangan yang berhasil | active-active, active-passive | desirable-desirable, desirable-auto |
| Pasangan yang gagal | passive-passive | auto-auto |

LACP dan PAgP tidak bisa dicampur dalam satu channel.

### Syarat anggota bundel

Semua port dalam satu bundel harus seragam dalam speed dan duplex, mode switchport (access atau trunk), serta allowed VLAN dan native VLAN. Port yang tidak seragam dikeluarkan dari bundel (status `s` suspended atau `I` stand-alone).

### Jenis kabel

| Sambungan | Kabel | Alasan |
|---|---|---|
| Switch ke switch | Copper Cross-Over | Perangkat sejenis, pin TX/RX harus ditukar |
| PC ke switch | Copper Straight-Through | Perangkat beda jenis |

### Perintah dan flag penting

| Perintah | Fungsi |
|---|---|
| `channel-group 1 mode active` | Memasukkan port ke grup 1 dan memulai negosiasi LACP |
| `show etherchannel summary` | Menampilkan status bundel dan port anggota |
| `show interfaces port-channel 1` | Menampilkan detail interface logis, termasuk bandwidth |
| `show interfaces trunk` | Menampilkan status trunk dan VLAN yang diizinkan |

| Flag | Arti |
|---|---|
| `S` | Layer 2 (switching) |
| `U` | In use |
| `P` | Port bundled di port-channel |
| `D` | Down |
| `I` | Stand-alone |
| `s` | Suspended |
| `w` | Menunggu negosiasi |

## Topologi Jaringan

![Topologi jaringan](images/topologi.png)

**Keterangan:** Dua switch 2960 dihubungkan dengan 4 kabel cross-over pada Fa0/1 - Fa0/4 (membentuk Port-channel 1). PC-Rektorat dan PC-LabKom masing-masing tersambung ke Fa0/5 dengan kabel straight-through. Titik oranye sementara pada gambar adalah STP yang memblokir port sebelum bundel terbentuk.

### Alokasi port

| Perangkat | Port | Tipe | Channel | Mode LACP | Allowed VLAN |
|---|---|---|---|---|---|
| SW-REKTOR | Fa0/1 - Fa0/4 | Trunk | Port-channel 1 | Active | 10, 20 |
| SW-REKTOR | Fa0/5 | Access (VLAN 10) | - | - | 10 |
| SW-LABKOM | Fa0/1 - Fa0/4 | Trunk | Port-channel 1 | Active | 10, 20 |
| SW-LABKOM | Fa0/5 | Access (VLAN 10) | - | - | 10 |

### Alamat IP

| Perangkat | Interface | IP Address | Subnet Mask |
|---|---|---|---|
| PC-Rektorat | FastEthernet0 | 192.168.10.2 | 255.255.255.0 |
| PC-LabKom | FastEthernet0 | 192.168.10.3 | 255.255.255.0 |

## Konfigurasi

### Langkah 1: IP address PC

Di tiap PC: **Desktop**, **IP Configuration**, **Static**. Isi IP dan subnet mask sesuai tabel alamat IP. Gateway dikosongkan.

### Langkah 2: VLAN dan access port

Dijalankan di kedua switch (ganti hostname sesuai perangkat):

```text
Switch> enable
Switch# configure terminal
Switch(config)# hostname SW-REKTOR
SW-REKTOR(config)# vlan 10
SW-REKTOR(config-vlan)# name Manajemen
SW-REKTOR(config-vlan)# exit
SW-REKTOR(config)# interface FastEthernet0/5
SW-REKTOR(config-if)# switchport mode access
SW-REKTOR(config-if)# switchport access vlan 10
SW-REKTOR(config-if)# end
```

**Penjelasan:**
- `enable`: masuk ke mode Privileged EXEC.
- `configure terminal`: masuk ke mode konfigurasi global.
- `hostname SW-REKTOR`: mengubah nama perangkat (SW-LABKOM untuk switch kedua).
- `vlan 10` dan `name Manajemen`: membuat VLAN 10 bernama Manajemen.
- `switchport mode access`: menjadikan port untuk perangkat ujung (PC).
- `switchport access vlan 10`: memasukkan port ke VLAN 10.

### Langkah 3: EtherChannel LACP dan trunk

Dijalankan di **kedua switch** dengan mode `active`:

```text
SW-REKTOR# configure terminal
SW-REKTOR(config)# interface range FastEthernet0/1 - 4
SW-REKTOR(config-if-range)# shutdown
SW-REKTOR(config-if-range)# switchport mode trunk
SW-REKTOR(config-if-range)# switchport trunk allowed vlan 10,20
SW-REKTOR(config-if-range)# channel-group 1 mode active
SW-REKTOR(config-if-range)# no shutdown
SW-REKTOR(config-if-range)# exit
SW-REKTOR(config)# interface Port-channel 1
SW-REKTOR(config-if)# switchport mode trunk
SW-REKTOR(config-if)# switchport trunk allowed vlan 10,20
SW-REKTOR(config-if)# end
SW-REKTOR# write memory
```

**Penjelasan:**
- `interface range FastEthernet0/1 - 4`: memilih 4 port sekaligus.
- `shutdown` dan `no shutdown`: mematikan port sementara agar perubahan masuk tanpa konflik, lalu menyalakannya lagi.
- `switchport mode trunk` dan `allowed vlan 10,20`: menjadikan port trunk yang hanya membawa VLAN 10 dan 20.
- `channel-group 1 mode active`: memasukkan port ke grup 1 dan memulai negosiasi LACP. Interface `Port-channel 1` terbentuk otomatis di sini.
- `interface Port-channel 1`: konfigurasi interface logis, karena inilah yang dianggap switch sebagai satu link.
- `write memory`: menyimpan konfigurasi ke startup-config.

### Langkah 4: Uji failover

```text
SW-REKTOR# configure terminal
SW-REKTOR(config)# interface range FastEthernet0/1 - 2
SW-REKTOR(config-if-range)# shutdown
SW-REKTOR(config-if-range)# end
```

Untuk menyalakan kembali, ganti `shutdown` dengan `no shutdown`.

### Konfigurasi akhir per switch

```text
hostname SW-REKTOR            (SW-LABKOM untuk switch kedua)
!
vlan 10
 name Manajemen
!
interface range FastEthernet0/1 - 4
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 channel-group 1 mode active
!
interface Port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
!
interface FastEthernet0/5
 switchport mode access
 switchport access vlan 10
```

## Hasil Pengujian

### Pengujian 1: Ping sebelum EtherChannel

- **Metode pengujian:** dari PC-Rektorat jalankan `ping 192.168.10.3` setelah VLAN dan access port diatur, sebelum EtherChannel dikonfigurasi.
- **Hasil yang diharapkan:** ping gagal, karena port antar switch masih access VLAN 1 sehingga VLAN 10 tidak bisa melintas.
- **Hasil aktual:** _isi sesuai hasil (Request timed out / Destination host unreachable)._

![Ping sebelum EtherChannel](images/ping-sebelum.png)

### Pengujian 2: Status EtherChannel

- **Metode pengujian:** jalankan `show etherchannel summary` di kedua switch.
- **Hasil yang diharapkan:** `Po1(SU)` dengan keempat port anggota berstatus `(P)`.
- **Hasil aktual (SW-LABKOM):**

```text
SW-LABKOM#show etherchannel summary
Number of channel-groups in use: 1
Number of aggregators:           1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------------------------
1      Po1(SU)         LACP      Fa0/1(P) Fa0/2(P) Fa0/3(P) Fa0/4(P)
```

Port-channel 1 berstatus **SU** (Layer 2, in use) dan keempat port anggota berstatus **P** (bundled).

![show etherchannel summary SW-REKTOR](images/etherchannel-sw-rektor.png)
![show etherchannel summary SW-LABKOM](images/etherchannel-sw-labkom.png)

### Pengujian 3: Status trunk

- **Metode pengujian:** jalankan `show interfaces trunk`.
- **Hasil yang diharapkan:** trunk tampil sebagai `Po1` (satu interface logis), status `trunking`, VLAN allowed `10,20`.
- **Hasil aktual (SW-LABKOM):**

```text
SW-LABKOM#show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Po1         on           802.1q         trunking      1

Port        Vlans allowed on trunk
Po1         10,20

Port        Vlans allowed and active in management domain
Po1         10

Port        Vlans in spanning tree forwarding state and not pruned
Po1         10
```

![show interfaces trunk](images/trunk.png)

### Pengujian 4: Bandwidth Port-channel

- **Metode pengujian:** jalankan `show interfaces port-channel 1` dan lihat baris `BW`.
- **Hasil yang diharapkan:** secara teori 4 x 100 Mbps = 400.000 Kbps (400 Mbps).
- **Hasil aktual:** `BW ______ Kbit` _(isi dari layar)._

![Bandwidth Port-channel 1](images/bandwidth.png)

### Pengujian 5: Ping normal (4 link aktif)

- **Metode pengujian:** dari PC-Rektorat jalankan `ping 192.168.10.3`.
- **Hasil yang diharapkan:** semua paket mendapat balasan (0% loss).
- **Hasil aktual:**

```text
C:\>ping 192.168.10.3

Pinging 192.168.10.3 with 32 bytes of data:

Reply from 192.168.10.3: bytes=32 time<1ms TTL=128
Reply from 192.168.10.3: bytes=32 time=1ms TTL=128
Reply from 192.168.10.3: bytes=32 time<1ms TTL=128
Reply from 192.168.10.3: bytes=32 time<1ms TTL=128

Ping statistics for 192.168.10.3:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 1ms, Average = 0ms
```

![Ping normal](images/ping-normal.png)

### Pengujian 6: Failover (Fa0/1 dan Fa0/2 dimatikan)

- **Metode pengujian:** jalankan `ping -t 192.168.10.3` dari PC-Rektorat, matikan Fa0/1 dan Fa0/2 di SW-REKTOR, lalu jalankan `show etherchannel summary`. Hentikan ping dengan `Ctrl + C`.
- **Hasil yang diharapkan:** ping tetap berjalan (paling banyak 1 sampai 2 RTO sesaat), dan summary menampilkan `Po1(SU)` dengan `Fa0/1(D) Fa0/2(D) Fa0/3(P) Fa0/4(P)`.
- **Hasil aktual:** _(tempel output summary dan statistik ping dari layar)._

```text
(tempel output show etherchannel summary saat Fa0/1 dan Fa0/2 down)
```

Statistik ping: **____ paket terkirim, ____ diterima, ____ hilang (____ %)**.

![Ping -t saat port dimatikan](images/ping-t-failover.png)
![Summary saat Fa0/1 dan Fa0/2 down](images/summary-failover.png)
![Statistik ping setelah Ctrl+C](images/ping-statistik.png)

### Ringkasan hasil

| Parameter | Hasil Pengamatan | Catatan Analisis |
|---|---|---|
| Status EtherChannel | Po1( **SU** ) | S = Layer 2, U = in use. Keempat port anggota berstatus P (bundled). |
| Total Bandwidth Port-channel 1 | ______ Kbps (teori 400.000 Kbps) | Akumulasi 4 port FastEthernet x 100 Mbps. |
| Ping normal (4 link aktif) | Success / Reply (**100**%) | 4/4 reply, latensi 0 sampai 1 ms. |
| Ping saat Fa0/1 & Fa0/2 down | Success / RTO ( ____ %) | _isi sesuai hasil_ |

## Analisis

### Kesesuaian dengan tujuan

- Sebelum EtherChannel, ping gagal karena port antar switch masih access VLAN 1. Setelah EtherChannel dan trunk aktif, ping berhasil 4/4 dengan latensi 0 sampai 1 ms, sehingga VLAN 10 terbukti melewati Port-channel 1.
- `show interfaces trunk` menampilkan satu interface logis `Po1`, bukan empat port terpisah. Itu bukti keempat link sudah menyatu menjadi satu jalur logis.
- Titik oranye pada topologi sebelum bundel terbentuk adalah STP yang memblokir port karena kabel paralel dianggap loop. Setelah EtherChannel terbentuk, STP memperlakukan Po1 sebagai satu link.

### Kendala dan solusi

| Kendala | Penyebab | Solusi |
|---|---|---|
| Port Gi0/3 dan Gi0/4 tidak tersedia | Switch 2960 di Packet Tracer hanya punya 2 port Gigabit | Bundel memakai Fa0/1 - Fa0/4 |
| `Invalid input` pada `channel-group 10 mode active` | Switch 2960 hanya menerima grup nomor 1 sampai 6 | Memakai `channel-group 1` |
| Hostname salah ketik (`SW_LABKOM`) | Salah ketik saat konfigurasi awal | Menjalankan `hostname SW-LABKOM` kembali |
| `show etherchannel summary` kosong | Konfigurasi EtherChannel belum dimasukkan di SW-REKTOR | Mengulang konfigurasi Langkah 3 di kedua switch |

### Jawaban soal evaluasi

**1. Perbedaan mendasar LACP dan PAgP**

- LACP adalah standar terbuka IEEE 802.3ad dan bisa dipakai di switch berbagai vendor. Modenya `active` (memulai negosiasi) dan `passive` (hanya menjawab).
- PAgP milik Cisco dan hanya jalan di perangkat Cisco. Modenya `desirable` (memulai) dan `auto` (hanya menjawab).
- Channel terbentuk bila minimal satu sisi memulai: active-active, active-passive, desirable-desirable, desirable-auto. Passive-passive dan auto-auto gagal.
- LACP dan PAgP tidak bisa dicampur dalam satu channel.

**2. Gi0/1 di SW-REKTOR speed 100, SW-LABKOM tetap 1000 (auto)**

- Gi0/1 tidak menjadi anggota EtherChannel yang aktif. Semua port dalam satu bundel wajib punya speed dan duplex yang sama karena diperlakukan sebagai satu link logis.
- Gi0/1 berjalan di 100 Mbps sedangkan Gi0/2 sampai Gi0/4 di 1000 Mbps, sehingga konfigurasinya tidak konsisten. Port itu dikeluarkan dari bundel dan biasanya berstatus suspended (`s`) atau stand-alone (`I`).
- Port-channel tetap hidup dengan 3 port sisanya (kapasitas turun dari sekitar 4 Gbps ke sekitar 3 Gbps), jaringan tidak mati. Status persisnya bisa sedikit berbeda antar versi IOS.

**3. Kenapa `switchport mode trunk` juga diterapkan di interface Port-channel**

- Port-channel adalah interface logis yang dianggap switch sebagai satu link, dan konfigurasinya menjadi acuan bagi seluruh port anggota.
- Jika setting di Port-channel berbeda dengan port fisik, port anggota bisa gagal bergabung dan berstatus suspended.
- Dengan mengatur di Port-channel, konfigurasi tetap konsisten saat port anggota ditambah atau dicopot.

**4. Analisis troubleshooting: `channel-group` mode passive di Switch A dan Switch B**

> Soal di modul berhenti di kalimat ini tanpa pertanyaan lanjutan. Jawaban berikut membahas kejadian, penyebab, dan solusinya.

- **Kejadian:** EtherChannel tidak terbentuk. Port berjalan sebagai link terpisah (stand-alone, `I`) dan tidak ada anggota berflag `(P)` di `show etherchannel summary`.
- **Penyebab:** mode passive hanya menjawab paket LACP dan tidak memulai. Jika kedua sisi passive, tidak ada yang mengirim paket LACP pertama.
- **Dampak:** STP menganggap kabel paralel sebagai loop dan memblokir sebagian port. Hanya satu link yang efektif dan bandwidth tidak terakumulasi.
- **Solusi:** ubah minimal satu sisi, idealnya keduanya, ke mode active:

```text
interface range FastEthernet0/1 - 4
channel-group 1 mode active
```

- **Verifikasi:** `show etherchannel summary` menampilkan `Po1(SU)` dengan anggota `(P)`.

## Kesimpulan

1. EtherChannel menggabungkan beberapa link fisik menjadi satu Port-channel logis, sehingga STP tidak memblokir kabel paralel dan bandwidth bisa diakumulasi.
2. LACP (IEEE 802.3ad) membentuk bundel secara dinamis, dengan syarat minimal satu sisi berada di mode `active`.
3. Pada praktikum ini, 4 port FastEthernet (Fa0/1 - Fa0/4) berhasil digabung menjadi `Po1(SU)` dengan seluruh anggota berstatus `(P)`, dan trunk VLAN 10 melintasi Port-channel dengan ping 4/4 reply tanpa kehilangan paket.
4. Port anggota bundel harus seragam dalam speed, duplex, dan konfigurasi switchport. Pengaturan di interface Port-channel menjaga konsistensi itu.
5. _Tambahkan kesimpulan uji failover berdasarkan hasil Pengujian 6._
6. Pemahaman ini menjadi dasar untuk materi berikutnya seperti redundansi jaringan, STP, dan desain jaringan berskala lebih besar yang butuh bandwidth antar switch lebih tinggi.
