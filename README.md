# Panduan Implementasi dan Operasi MTCINE

## Daftar Isi

[Modul 1 - BGP](#1-modul-1---bgp)
[Modul 2 - MPLS](#2-modul-2---mpls)
[Modul 3 - Traffic Engineering](#3-modul-3---traffic-engineering)
[Lampiran A - Matriks Penanganan Masalah dan Perintah Cepat](#4-lampiran-a---matriks-penanganan-masalah-dan-perintah-cepat)
[Lampiran B - Templat Pencadangan (Backup), Rollback (Pengembalian), dan Permintaan Perubahan](#5-lampiran-b---templat-pencadangan-rollback-dan-permintaan-perubahan)


## 1. Modul 1 - BGP

Border Gateway Protocol (BGP) merupakan protokol yang menghubungkan Autonomous System (AS). Pada modul ini, pembahasan tidak hanya mencakup cara kerja BGP, tetapi juga cara mengendalikannya agar jaringan tetap stabil, aman, dan tidak menjadi transit yang tidak dikehendaki.

### 1.1 Studi Kasus — Perusahaan Dual-Homed (Terhubung ke Dua ISP)

**Skenario:** PT Nusantara Retail (AS 65001) memiliki dua kantor pusat di Surabaya dan Jakarta yang terhubung ke dua ISP yang berbeda agar koneksi internet tetap tersedia apabila salah satu ISP mengalami gangguan. Perusahaan tersebut memiliki dua prefix, yaitu `203.0.113.0/24` untuk layanan publik dan `198.51.100.0/24` untuk layanan internal. Di dalam AS 65001 terdapat tiga router, yaitu R1 menuju ISP-A, R2 menuju ISP-B, dan R3 sebagai core sekaligus Route Reflector (RR). Tujuan yang hendak dicapai adalah trafik internet keluar melalui ISP-A sebagai jalur utama dan ISP-B sebagai jalur cadangan, sedangkan trafik masuk dapat diatur. Sesi iBGP harus stabil tanpa menggunakan full-mesh.

**Permasalahan yang umum terjadi tanpa desain tersebut:** Apabila R1 dan R2 tidak saling mempertukarkan rute iBGP secara benar, trafik dari Jakarta dapat keluar melalui jalur yang tidak optimal, atau prefix perusahaan tidak diumumkan ke salah satu ISP sehingga sebagian pengguna tidak dapat dijangkau.

#### Topologi Modul 1

R1 (10.0.0.1) melakukan eBGP ke ISP-A (AS 65002, 192.168.12.2) melalui ether1 192.168.12.1/30. R2 (10.0.0.2) melakukan eBGP ke ISP-B (AS 65003, 192.168.23.2) melalui ether1 192.168.23.1/30. R1—R3—R2 terhubung melalui tautan internal 10.10.13.0/30 dan 10.10.23.0/30. R3 (10.0.0.3) berperan sebagai Route Reflector. Jaringan LAN perusahaan 10.10.0.0/16 berada di belakang R3.

#### Rencana Pengalamatan IP Modul 1

| Perangkat / Tautan | IP / Prefix | Keterangan |
|---|---|---|
| R1 Loopback | 10.0.0.1/32 | Router-ID BGP |
| R2 Loopback | 10.0.0.2/32 | Router-ID BGP |
| R3 Loopback | 10.0.0.3/32 | Router-ID + RR |
| R1 — ISP-A | 192.168.12.1/30 — 192.168.12.2/30 | eBGP AS 65001 — 65002 |
| R2 — ISP-B | 192.168.23.1/30 — 192.168.23.2/30 | eBGP AS 65001 — 65003 |
| R1 — R3 | 10.10.13.0/30 | iBGP + OSPF internal |
| R2 — R3 | 10.10.23.0/30 | iBGP + OSPF internal |
| Prefix advertise | 203.0.113.0/24, 198.51.100.0/24 | Prefix milik AS 65001 |

### 1.2 Implementasi Bertahap — BGP

#### Tahap 1 — Konektivitas Dasar dan Loopback

Wajib dipastikan bahwa antarrouter internal dapat melakukan ping ke loopback satu sama lain. Pada laboratorium kecil, OSPF atau static route dapat digunakan untuk keterjangkauan (reachability) loopback. Pada jaringan produksi, Open Shortest Path First (OSPF) merupakan Interior Gateway Protocol (IGP) yang umum digunakan.

```routeros
/ip address
add address=10.0.0.1/32 interface=lo comment="Loopback R1"
add address=192.168.12.1/30 interface=ether1 comment="to ISP-A"
add address=10.10.13.1/30 interface=ether2 comment="to R3"

/routing ospf instance
add name=ospf1 router-id=10.0.0.1
/routing ospf area
add name=backbone instance=ospf1
/routing ospf interface-template
add interfaces=ether2 area=backbone
```

#### Tahap 2 — eBGP ke ISP-A dan ISP-B (R1 dan R2)

Pada RouterOS v7, template BGP dibuat satu kali, kemudian connection dibuat untuk setiap peer. Cara tersebut membuat konfigurasi menjadi lebih rapi dan mudah digunakan kembali.

```routeros
# --- Di R1 (AS 65001) ---
/routing bgp template
add name=bgp-65001 as=65001 router-id=10.0.0.1 \
  address-families=ip \
  output.network=bgp-networks

/routing bgp network
add network=203.0.113.0/24 synchronize=no
add network=198.51.100.0/24 synchronize=no

/routing bgp connection
add name=to-ISP-A templates=bgp-65001 \
  remote.address=192.168.12.2 remote.as=65002 \
  local.address=192.168.12.1 \
  output.filter-chain=to-ISP-A-out \
  input.filter-chain=from-ISP-A-in

# --- Di R2 (AS 65001) ---
/routing bgp template
add name=bgp-65001 as=65001 router-id=10.0.0.2 \
  address-families=ip \
  output.network=bgp-networks
/routing bgp network
add network=203.0.113.0/24 synchronize=no
add network=198.51.100.0/24 synchronize=no
/routing bgp connection
add name=to-ISP-B templates=bgp-65001 \
  remote.address=192.168.23.2 remote.as=65003 \
  local.address=192.168.23.1 \
  output.filter-chain=to-ISP-B-out \
  input.filter-chain=from-ISP-B-in
```

#### Tahap 3 — Filter: Pencegahan Transit yang Tidak Dikehendaki

Tahap ini merupakan tahap yang paling penting. Tanpa filter, router dapat mengumumkan (advertise) rute dari ISP-A ke ISP-B sehingga trafik pihak lain melewati jaringan perusahaan. Filter outbound hanya diperkenankan mengirimkan prefix milik perusahaan. Filter inbound menolak bogon, default yang tidak dikehendaki, dan prefix milik perusahaan yang kembali (antispoof).

```routeros
# Filter OUT: hanya prefix milik perusahaan yang diperkenankan keluar
/routing filter rule
add chain=to-ISP-A-out rule="if (dst in 203.0.113.0/24) { accept; }"
add chain=to-ISP-A-out rule="if (dst in 198.51.100.0/24) { accept; }"
add chain=to-ISP-A-out rule="reject;"

# Filter IN dari ISP-A: menolak prefix privat dan prefix milik perusahaan
/routing filter rule
add chain=from-ISP-A-in rule="if (dst in 10.0.0.0/8) { reject; }"
add chain=from-ISP-A-in rule="if (dst in 203.0.113.0/24) { reject; }"
add chain=from-ISP-A-in rule="accept;"

# Berlaku pula untuk ISP-B, dengan penyesuaian nama chain
/routing filter rule
add chain=to-ISP-B-out rule="if (dst in 203.0.113.0/24) { accept; }"
add chain=to-ISP-B-out rule="if (dst in 198.51.100.0/24) { accept; }"
add chain=to-ISP-B-out rule="reject;"
```

#### Tahap 4 — iBGP dan Route Reflector (R3)

Tanpa Route Reflector, R1, R2, dan R3 harus menggunakan full-mesh (mesh penuh) iBGP. Dengan RR pada R3, R1 dan R2 cukup melakukan peering ke R3, kemudian R3 memantulkan rute tersebut. Cara tersebut mengurangi jumlah sesi dan lebih mudah ditingkatkan skalanya.

```routeros
# --- Di R3 (Route Reflector) ---
/routing bgp template
add name=bgp-65001 as=65001 router-id=10.0.0.3 address-families=ip
/routing bgp connection
add name=to-R1 templates=bgp-65001 remote.address=10.0.0.1 \
  remote.as=65001 local.address=10.0.0.3 \
  address-families=ip route-reflector=yes
add name=to-R2 templates=bgp-65001 remote.address=10.0.0.2 \
  remote.as=65001 local.address=10.0.0.3 \
  address-families=ip route-reflector=yes

# --- Di R1: iBGP ke R3 melalui loopback ---
/routing bgp connection
add name=to-RR templates=bgp-65001 remote.address=10.0.0.3 \
  remote.as=65001 local.address=10.0.0.1 \
  output.filter-chain=ibgp-out input.filter-chain=ibgp-in
# Loopback digunakan agar sesi tidak terputus apabila satu tautan terputus
```

#### Tahap 5 — Pengaturan Best Path: Local Preference (Local-Pref), Multi-Exit Discriminator (MED), dan AS Path Prepend

BGP tidak memilih rute hanya berdasarkan hop terpendek. Pengelola jaringan dapat mengendalikan trafik keluar dan trafik masuk. **Local-Pref** mengendalikan trafik keluar (nilai yang lebih besar lebih diutamakan dan hanya berlaku di dalam AS). **AS Path Prepend** mengendalikan trafik masuk (prepend membuat jalur tampak lebih panjang sehingga ISP lain cenderung tidak memilihnya).

```routeros
# Contoh: ISP-A dijadikan jalur utama untuk trafik keluar
# Di R1, ketika menerima default dari ISP-A, nilai Local-Pref ditingkatkan
/routing filter rule
add chain=from-ISP-A-in rule="if (dst == 0.0.0.0/0) { set bgp-local-pref 200; accept; }"

# Di R2, Local-Pref dibuat lebih kecil untuk ISP-B (jalur cadangan)
# chain from-ISP-B-in: set bgp-local-pref 100 untuk default

# Contoh: Prefix 198.51.100.0/24 dibuat kurang menarik melalui ISP-B
# agar trafik masuk lebih banyak melalui ISP-A
# Di R2, prepend dilakukan 2 kali pada outbound ke ISP-B
/routing filter rule
add chain=to-ISP-B-out rule="if (dst in 198.51.100.0/24) { set bgp-as-path 65001 65001; accept; }"

# Private AS Removal: apabila AS privat 64512 digunakan pada laboratorium
# dan dilakukan peering ke ISP, AS privat dihapus pada saat advertise
# Pada connection: output.remove-private-as=yes
```

#### Tahap 6 — Stub, Non-Stub, Multihop (Multi-Hop), dan Loopback

- **Stub (Ujung):** Hanya memiliki satu jalur keluar (egress). Hanya perlu menerima default route dari ISP, tidak menerima full table (tabel rute lengkap), dan hanya mengumumkan (advertise) prefix milik perusahaan.
- **Non-Stub:** Sebagaimana studi kasus ini, memiliki dua ISP, harus menerima rute dari kedua ISP dan mengatur Local-Pref.
- **Multihop dan Loopback:** Untuk iBGP dan eBGP yang tidak terhubung secara langsung (misalnya peer melalui firewall atau loopback), multihop ditetapkan dan loopback wajib dapat dijangkau melalui IGP. Apabila loopback tidak dapat dijangkau, sesi iBGP akan mengalami flapping.

> **Catatan untuk RouterOS v6:** Pada RouterOS v6, perintah yang digunakan adalah `/routing bgp peer add remote-address=... remote-as=... in-filter=... out-filter=... route-reflect=yes`. Konsep filter dan Local-Pref adalah sama, hanya sintaks yang berbeda.

### 1.3 Perbandingan RouterOS v6 dan v7 — BGP

Contoh konfigurasi pada Tahap 1 hingga Tahap 6 menggunakan RouterOS v7. Pada subbab ini, perbandingan dan contoh setara untuk RouterOS v6 disediakan agar panduan ini dapat digunakan pada kedua versi tersebut.

#### Tabel Perbandingan

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| Struktur konfigurasi BGP | Instance + Peer | Template + Connection | Pada RouterOS v7, template dipisahkan dari connection agar dapat digunakan kembali |
| Menu utama BGP | `/routing bgp peer`, `/routing bgp instance` | `/routing bgp template`, `/routing bgp connection` | Nama menu berbeda, prinsip peering sama |
| Advertise prefix | `/routing bgp network` | `/routing bgp network` | Pada RouterOS v7, network terikat pada template melalui `output.network` |
| Filter | `/routing filter` | `/routing filter rule` | Pada RouterOS v7, filter ditulis sebagai `rule` |
| Route Reflector | `route-reflect=yes` pada peer | `route-reflector=yes` pada connection | Fungsi tersebut identik, penamaan berbeda |
| Address Family | `address-families` pada peer (terbatas) | `address-families=ip, vpnv4, l2vpn` pada template | Pada RouterOS v7, address family dinyatakan secara eksplisit |
| Private AS Removal | `remove-private-as=yes` pada peer | `output.remove-private-as=yes` pada connection | Berlaku pada saat advertise ke eBGP |
| Pemeriksaan status | `/routing bgp peer print status` | `/routing bgp peer print status` | Perintah pemeriksaan hampir identik |
| Pemeriksaan rute | `/routing bgp route print` | `/routing bgp route print` | Output v7 lebih rinci pada kolom Local-Pref dan AS Path |

#### Konfigurasi RouterOS v6 — Setara dengan Tahap 2 hingga Tahap 4

```routeros
# --- Di R1 (AS 65001) - RouterOS v6 ---
/routing bgp instance
add name=bgp-65001 as=65001 router-id=10.0.0.1 redistribute-connected=no

/routing bgp network
add network=203.0.113.0/24 synchronize=no
add network=198.51.100.0/24 synchronize=no

/routing bgp peer
add name=to-ISP-A instance=bgp-65001 remote-address=192.168.12.2 remote-as=65002 \
  in-filter=from-ISP-A-in out-filter=to-ISP-A-out

# --- Di R3 (Route Reflector) - RouterOS v6 ---
/routing bgp instance
add name=bgp-65001 as=65001 router-id=10.0.0.3

/routing bgp peer
add name=to-R1 instance=bgp-65001 remote-address=10.0.0.1 remote-as=65001 \
  route-reflect=yes in-filter=ibgp-in out-filter=ibgp-out
add name=to-R2 instance=bgp-65001 remote-address=10.0.0.2 remote-as=65001 \
  route-reflect=yes in-filter=ibgp-in out-filter=ibgp-out

# --- Filter - RouterOS v6 ---
/routing filter
add chain=to-ISP-A-out rule="if (dst in 203.0.113.0/24) { accept; }"
add chain=to-ISP-A-out rule="if (dst in 198.51.100.0/24) { accept; }"
add chain=to-ISP-A-out rule="reject;"
add chain=from-ISP-A-in rule="if (dst in 10.0.0.0/8) { reject; }"
add chain=from-ISP-A-in rule="if (dst in 203.0.113.0/24) { reject; }"
add chain=from-ISP-A-in rule="accept;"
```

#### Konfigurasi RouterOS v7 — Ringkasan Setara

```routeros
# --- Di R1 (AS 65001) - RouterOS v7 ---
/routing bgp template
add name=bgp-65001 as=65001 router-id=10.0.0.1 address-families=ip output.network=bgp-networks
/routing bgp network
add network=203.0.113.0/24 synchronize=no
add network=198.51.100.0/24 synchronize=no
/routing bgp connection
add name=to-ISP-A templates=bgp-65001 remote.address=192.168.12.2 remote.as=65002 \
  local.address=192.168.12.1 output.filter-chain=to-ISP-A-out input.filter-chain=from-ISP-A-in

# --- Di R3 (Route Reflector) - RouterOS v7 ---
/routing bgp template
add name=bgp-65001 as=65001 router-id=10.0.0.3 address-families=ip
/routing bgp connection
add name=to-R1 templates=bgp-65001 remote.address=10.0.0.1 remote.as=65001 \
  local.address=10.0.0.3 address-families=ip route-reflector=yes
```

#### Panduan Pemilihan Versi

- Apabila jaringan masih menggunakan RouterOS v6, konfigurasi v6 tersebut dapat digunakan. Perintah `instance` dan `peer` tidak tersedia pada RouterOS v7.
- Apabila jaringan menggunakan RouterOS v7, konfigurasi v7 dapat digunakan. Konfigurasi RouterOS v7 tidak dapat disalin ke RouterOS v6 tanpa penyesuaian.
- Filter dan prinsip Local-Pref, MED, dan AS Path Prepend berlaku identik pada kedua versi. Perbedaan hanya terdapat pada sintaks dan nama menu.

### 1.4 Verifikasi BGP

```routeros
/routing bgp peer print status
/routing bgp session print
/routing bgp advertisements print where peer=to-ISP-A
/routing bgp route print where dst in 203.0.113.0/24
/ip route print where bgp
# Pemeriksaan best path: perhatikan active, bgp-local-pref, bgp-as-path
/routing bgp route print detail where dst=0.0.0.0/0
```

- **Sesi Established?** Apabila status berhenti pada Active / Connect (tahap pembentukan sesi), periksa alamat IP, nomor AS, firewall TCP 179, dan keterjangkauan (reachability).
- **Rute diterima tetapi tidak aktif?** Perlu diperiksa apakah next-hop dapat dijangkau melalui IGP. BGP tidak akan menginstal rute apabila next-hop tidak terdapat pada tabel routing.
- **Prefix tidak diumumkan?** Perlu diperiksa `/routing bgp network` dan filter outbound. Perlu dipastikan bahwa prefix terdapat pada IP route (atau synchronize=no pada laboratorium).

### 1.5 Operasi BGP — Prosedur Operasional

- **Pemantauan:** Peringatan (alert) dikirimkan apabila sesi BGP terputus lebih dari 2 menit, jumlah prefix yang diterima menurun secara drastis, atau terdapat prefix baru yang tidak dikenal. Pemantauan dapat menggunakan `/tool e-mail` atau `/system script` yang terintegrasi dengan The Dude, Zabbix, atau Grafana.
- **Perubahan:** Setiap perubahan filter atau Local-Pref dilaksanakan pada jendela pemeliharaan, dengan rollback (pengembalian) yang telah disiapkan: ekspor konfigurasi disimpan sebelum perubahan dan perintah untuk mengembalikan filter lama telah disiapkan.
- **Pencadangan:** Konfigurasi BGP diekspor setiap minggu dengan perintah `/export file=bgp-backup` dan disimpan di luar router.
- **Audit:** Setiap bulan, dilakukan pemeriksaan apakah terdapat prefix bogon yang lolos, apakah community masih sesuai dengan dokumentasi, dan apakah RR masih menjadi single point of failure (RR kedua perlu disiapkan).

#### Penanganan Masalah BGP — Cepat

| Gejala | Penyebab Umum | Tindakan |
|---|---|---|
| Session Idle / Active | IP / AS tidak sesuai, firewall memblokir 179 | Periksa ping, periksa /ip firewall, periksa remote-as |
| Established tetapi tidak ada rute | Filter inbound menolak seluruh rute | Periksa filter chain, periksa /routing bgp route |
| Rute tersedia tetapi tidak digunakan | Next-hop tidak dapat dijangkau | Periksa OSPF / static menuju next-hop |
| Trafik hanya melalui satu ISP | Local-Pref / prepend tidak sesuai | Periksa filter, periksa bgp-local-pref pada route print |
| Loop / transit tidak disengaja | Filter outbound terlalu longgar | Perketat filter, hanya prefix milik perusahaan |

## 2. Modul 2 - MPLS

Multiprotocol Label Switching (MPLS) membuat forwarding menjadi lebih cepat dan memungkinkan VPN yang terisolasi pada infrastruktur yang sama. Pada modul ini, pembahasan dimulai dari dasar label hingga pembangunan Virtual Private LAN Service (VPLS) atau VPN Layer 2 dan L3VPN atau VPN Layer 3 sebagaimana yang digunakan oleh ISP.

### 2.1 Studi Kasus — ISP Membangun Backbone MPLS untuk Dua Pelanggan

**Skenario:** ISP Nusantara Net memiliki backbone yang terdiri dari tiga router, yaitu PE1 (Surabaya), P1 (Madiun, transit), dan PE2 (Jakarta). Terdapat dua pelanggan: **Pelanggan A (Toko Retail)** memerlukan VPLS agar dua cabangnya terasa seperti terhubung ke satu switch yang sama (memerlukan Layer 2 yang sama, misalnya untuk POS dan CCTV yang menggunakan broadcast). **Pelanggan B (Kantor Akuntan)** memerlukan L3VPN agar dua cabangnya dapat melakukan routing antar-site tetapi terisolasi dari pelanggan lain. Backbone harus menggunakan Open Shortest Path First (OSPF) sebagai Interior Gateway Protocol (IGP) dan Label Distribution Protocol (LDP) untuk distribusi label.

**Tantangan:** Tanpa MPLS, ISP harus membuat tunnel untuk setiap pelanggan yang rumit dan tidak skalabel. Dengan MPLS, satu backbone dapat melayani banyak pelanggan dengan isolasi Virtual Routing and Forwarding (VRF) dan Virtual Private LAN Service (VPLS) yang rapi.

#### Topologi Modul 2

PE1 (10.0.0.1) — P1 (10.0.0.3) — PE2 (10.0.0.2). CE1 berada di belakang PE1 (192.168.10.1/30), CE2 berada di belakang PE2 (192.168.20.1/30). Untuk VPLS, CE1 dan CE2 harus berada pada subnet yang sama, yaitu 192.168.100.0/24. Untuk L3VPN, CE1 menggunakan 172.16.10.0/24, CE2 menggunakan 172.16.20.0/24, dan routing dilakukan melalui OSPF CE-PE.

#### Rencana Pengalamatan IP Modul 2

| Perangkat | Loopback / IP | Peran |
|---|---|---|
| PE1 | 10.0.0.1/32 | PE Surabaya |
| P1 | 10.0.0.3/32 | P transit |
| PE2 | 10.0.0.2/32 | PE Jakarta |
| PE1 — P1 | 10.10.13.0/30 | OSPF + LDP |
| P1 — PE2 | 10.10.32.0/30 | OSPF + LDP |
| CE1 — PE1 (VPLS) | 192.168.100.11/24 (bridge) | Pelanggan A Site 1 |
| CE2 — PE2 (VPLS) | 192.168.100.12/24 (bridge) | Pelanggan A Site 2 |
| CE1 — PE1 (L3VPN) | 172.16.10.0/30 | Pelanggan B Site 1 |
| CE2 — PE2 (L3VPN) | 172.16.20.0/30 | Pelanggan B Site 2 |

### 2.2 Implementasi Bertahap — MPLS

#### Tahap 1 — IGP OSPF dan MTU

MPLS memerlukan IGP yang stabil. OSPF wajib mengumumkan (advertise) loopback. MTU pada seluruh interface backbone wajib ditingkatkan sebelum MPLS diaktifkan karena label menambah 4 byte per label, dan VPLS / VPN dapat menambah lebih banyak. Apabila MTU terlalu kecil, paket yang berukuran besar akan mengalami fragmentasi atau terbuang (drop).

```routeros
# Pada SELURUH router backbone (PE1, P1, PE2)
/interface ethernet
set [find name=ether1] l2mtu=1600 mtu=1600
set [find name=ether2] l2mtu=1600 mtu=1600

/routing ospf instance
add name=ospf1 router-id=10.0.0.1
/routing ospf area
add name=backbone instance=ospf1
/routing ospf interface-template
add interfaces=ether1,ether2 area=backbone
# Pastikan loopback diumumkan (advertised) melalui OSPF
/routing ospf interface-template
add interfaces=lo area=backbone
```

#### Tahap 2 — MPLS dan LDP Dasar

Label Distribution Protocol (LDP) akan membagikan label secara otomatis untuk setiap Forwarding Equivalence Class (FEC) pada tabel routing, sehingga terbentuk Label Switched Path (LSP). Perlu diperiksa apakah LDP neighbor telah terbentuk dan label telah terdistribusi.

```routeros
# Di PE1, P1, PE2
/mpls ldp
set enabled=yes lsr-id=10.0.0.1 transport-address=10.0.0.1
/mpls ldp interface
add interface=ether1
add interface=ether2
# Verifikasi awal
# /mpls ldp neighbor print
# /mpls forwarding-table print
```

#### Tahap 3 — Static Label Mapping (Konsep Laboratorium)

Untuk memahami label, buat static mapping pada laboratorium: label 100 untuk prefix 10.0.0.2/32. Hal tersebut membantu memahami bagaimana label ditempelkan dan dilepas. Pada jaringan produksi, LDP digunakan, bukan static, karena lebih skalabel.

```routeros
# Contoh ini hanya untuk laboratorium, pada PE1
/mpls static-label
add label=100 nexthop=10.10.13.2 out-label=200
# Pemeriksaan: /mpls forwarding-table print
```

#### Tahap 4 — Penultimate Hop Popping (PHP) dan Traceroute

PHP menyebabkan router kedua terakhir melepas label terakhir sehingga PE tujuan hanya menerima IP biasa. Dampaknya, traceroute pada MPLS akan menampilkan hop MPLS. Pada RouterOS, `/tool traceroute` akan menampilkan label MPLS. Apabila traceroute dari CE tidak menampilkan label, periksa apakah LDP berjalan dan apakah MPLS telah diaktifkan pada interface.

#### Tahap 5 — VPLS Berbasis LDP (Pelanggan A)

VPLS menyebabkan dua site terasa seperti terhubung ke satu switch yang sama. PE1 dan PE2 membuat tunnel VPLS dengan ID yang sama (misalnya 100). Bridge pada PE akan menggabungkan tunnel VPLS dengan interface menuju CE.

```routeros
# --- Di PE1 ---
/interface vpls
add name=vpls-A peer=10.0.0.2 vpls-id=100:100 \
  cisco-style=no disabled=no
/interface bridge
add name=bridge-VPLS-A
/interface bridge port
add bridge=bridge-VPLS-A interface=vpls-A
add bridge=bridge-VPLS-A interface=ether3 comment="to CE1"

# --- Di PE2 ---
/interface vpls
add name=vpls-A peer=10.0.0.1 vpls-id=100:100
/interface bridge
add name=bridge-VPLS-A
/interface bridge port
add bridge=bridge-VPLS-A interface=vpls-A
add bridge=bridge-VPLS-A interface=ether3 comment="to CE2"

# Pemeriksaan: CE1 melakukan ping ke CE2 pada 192.168.100.0/24 harus berhasil
# /interface vpls print
# /interface bridge host print where bridge=bridge-VPLS-A
```

**Bridge Split Horizon:** Apabila terdapat lebih dari dua PE dalam satu VPLS (full-mesh atau mesh penuh), split-horizon diaktifkan pada bridge port VPLS agar tidak terjadi loop broadcast. Pada contoh dua PE, hal tersebut tidak wajib, tetapi wajib pada topologi hub-and-spoke atau full-mesh VPLS.

```routeros
/interface bridge port
set [find interface=vpls-A] horizon=1
# Seluruh port VPLS pada bridge yang sama wajib memiliki horizon yang identik
```

**VPLS Control Word (CW) atau kata kontrol VPLS:** Diaktifkan apabila terdapat perangkat yang memerlukan deteksi mis-order atau apabila melewati jaringan yang melakukan ECMP berdasarkan payload. Pada RouterOS v7, CW diatur pada VPLS. Apabila salah satu sisi CW aktif dan sisi lain tidak aktif, VPLS dapat tidak aktif (down). Nilai CW harus disamakan pada kedua PE.

```routeros
/interface vpls
set [find name=vpls-A] use-control-word=yes
```

#### Tahap 6 — VPLS Berbasis BGP (Alternatif yang Skalabel)

Untuk ISP besar dengan banyak PE, VPLS berbasis BGP lebih mudah karena auto-discovery melalui BGP, tanpa konfigurasi peer manual satu per satu. Prinsipnya, BGP mengumumkan (advertise) VPLS instance, sedangkan LDP tetap digunakan untuk label transport.

```routeros
# Konsep BGP-VPLS di PE1 (v7)
/routing bgp template
add name=bgp-mpls as=65000 router-id=10.0.0.1 address-families=l2vpn
/routing bgp connection
add name=to-PE2 templates=bgp-mpls remote.address=10.0.0.2 remote.as=65000
# VPLS instance pada bridge / BGP VPLS
# Pada laboratorium kecil, LDP-VPLS lebih mudah dipahami, BGP-VPLS digunakan apabila PE > 5
```

#### Tahap 7 — L3VPN: VRF, BGP VPNv4, dan Route Leaking

L3VPN menggunakan VRF untuk memisahkan tabel routing pelanggan. PE1 dan PE2 saling mempertukarkan rute VPN melalui Multi-Protocol BGP (MP-BGP) address-family VPNv4. Route Distinguisher (RD) membedakan VRF antar pelanggan, sedangkan Route Target (RT) menentukan VRF mana yang diperkenankan mengimpor rute.

```routeros
# --- Di PE1 dan PE2: Pembuatan VRF Pelanggan B ---
/ip route vrf
add routing-mark=CUST-B interfaces=ether4 \
  route-distinguisher=65000:100 \
  import-route-targets=65000:100 \
  export-route-targets=65000:100

/ip address
add address=172.16.10.1/30 interface=ether4 routing-mark=CUST-B
# Di PE2: 172.16.20.1/30

# --- BGP VPNv4 antar PE ---
/routing bgp template
add name=bgp-vpn as=65000 router-id=10.0.0.1 address-families=vpnv4
/routing bgp connection
add name=to-PE2-vpn templates=bgp-vpn remote.address=10.0.0.2 remote.as=65000

# Route Leaking: Apabila Pelanggan B memerlukan akses ke layanan bersama
# (misalnya server DNS 10.99.0.0/24 pada VRF lain), leaking dibuat
# dengan import-route-targets tambahan atau static route pada VRF
/ip route
add dst-address=10.99.0.0/24 gateway=10.0.0.3 routing-mark=CUST-B
```

#### Tahap 8 — OSPF sebagai Protokol CE-PE (Pelanggan B)

Di sisi pelanggan, CE dan PE dapat berkomunikasi melalui OSPF agar rute LAN pelanggan secara otomatis masuk ke VPN. PE wajib melakukan redistribusi OSPF ke BGP VPNv4 dan sebaliknya, dengan domain-id yang sama agar tampak sebagai satu area OSPF.

```routeros
# Di PE1 untuk VRF CUST-B
/routing ospf instance
add name=ospf-CUST-B router-id=172.16.10.1 vrf=CUST-B
/routing ospf area
add name=area-CUST-B instance=ospf-CUST-B
/routing ospf interface-template
add interfaces=ether4 area=area-CUST-B
# Di CE1, OSPF biasa menuju PE1, advertise LAN 172.16.10.0/24
# Pemeriksaan: /ip route print where routing-mark=CUST-B
```

### 2.3 Perbandingan RouterOS v6 dan v7 — MPLS

Contoh konfigurasi pada Tahap 1 hingga Tahap 8 menggunakan RouterOS v7. Pada subbab ini, perbandingan dan contoh setara untuk RouterOS v6 disediakan agar implementasi MPLS dapat dilakukan pada kedua versi tersebut.

#### Tabel Perbandingan

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| OSPF | `/routing ospf instance`, `/routing ospf network` | `/routing ospf instance`, `/routing ospf area`, `/routing ospf interface-template` | Pada RouterOS v7, interface dinyatakan melalui `interface-template` |
| MPLS LDP | `/mpls ldp`, `/mpls ldp interface` | `/mpls ldp`, `/mpls ldp interface` | Perintah hampir identik, perlu penyesuaian `lsr-id` dan `transport-address` |
| Static Label | `/mpls static-label` | `/mpls static-label` | Hanya untuk laboratorium, tidak digunakan pada produksi |
| VPLS | `/interface vpls` | `/interface vpls` | Parameter `vpls-id`, `peer`, dan `use-control-word` identik |
| Bridge dan Split Horizon | `/interface bridge port` dengan `horizon` | `/interface bridge port` dengan `horizon` | Konsep split horizon identik pada kedua versi |
| VRF | `/ip route vrf` | `/ip route vrf` | Pada RouterOS v7, VRF tetap menggunakan `routing-mark` |
| BGP VPNv4 | `/routing bgp peer` dengan address family VPNv4 | `/routing bgp template` dengan `address-families=vpnv4` | Perbedaan utama pada struktur template dan connection |
| BGP VPLS (L2VPN) | `/routing bgp peer` dengan L2VPN | `/routing bgp template` dengan `address-families=l2vpn` | Pada RouterOS v7, L2VPN dinyatakan secara eksplisit |
| MTU | `/interface ethernet` dengan `l2mtu` dan `mtu` | `/interface ethernet` dengan `l2mtu` dan `mtu` | Nilai 1600 disarankan pada kedua versi untuk VPLS |
| Pemeriksaan MPLS | `/mpls forwarding-table print`, `/mpls ldp neighbor print` | `/mpls forwarding-table print`, `/mpls ldp neighbor print` | Perintah pemeriksaan identik |

#### Konfigurasi RouterOS v6 — Setara dengan Tahap 1, 2, 5, dan 7

```routeros
# --- OSPF + MTU - RouterOS v6, di seluruh router backbone ---
/interface ethernet
set [find name=ether1] l2mtu=1600 mtu=1600
set [find name=ether2] l2mtu=1600 mtu=1600
/routing ospf instance
add name=ospf1 router-id=10.0.0.1
/routing ospf network
add network=10.10.13.0/30 area=backbone
add network=10.0.0.1/32 area=backbone

# --- MPLS + LDP - RouterOS v6 ---
/mpls ldp
set enabled=yes lsr-id=10.0.0.1 transport-address=10.0.0.1
/mpls ldp interface
add interface=ether1
add interface=ether2

# --- VPLS LDP - RouterOS v6, di PE1 ---
/interface vpls
add name=vpls-A peer=10.0.0.2 vpls-id=100:100 cisco-style=no disabled=no
/interface bridge
add name=bridge-VPLS-A
/interface bridge port
add bridge=bridge-VPLS-A interface=vpls-A
add bridge=bridge-VPLS-A interface=ether3 comment="to CE1"

# --- L3VPN VRF - RouterOS v6, di PE1 ---
/ip route vrf
add routing-mark=CUST-B interfaces=ether4 route-distinguisher=65000:100 \
  import-route-targets=65000:100 export-route-targets=65000:100
/ip address
add address=172.16.10.1/30 interface=ether4 routing-mark=CUST-B

# --- BGP VPNv4 - RouterOS v6, di PE1 ---
/routing bgp instance
add name=bgp-vpn as=65000 router-id=10.0.0.1
/routing bgp peer
add name=to-PE2-vpn instance=bgp-vpn remote-address=10.0.0.2 remote-as=65000 \
  address-families=vpnv4
```

#### Konfigurasi RouterOS v7 — Ringkasan Setara

```routeros
# --- OSPF + MTU - RouterOS v7 ---
/interface ethernet
set [find name=ether1] l2mtu=1600 mtu=1600
/routing ospf instance
add name=ospf1 router-id=10.0.0.1
/routing ospf area
add name=backbone instance=ospf1
/routing ospf interface-template
add interfaces=ether1,ether2 area=backbone
add interfaces=lo area=backbone

# --- MPLS + LDP - RouterOS v7 ---
/mpls ldp
set enabled=yes lsr-id=10.0.0.1 transport-address=10.0.0.1
/mpls ldp interface
add interface=ether1
add interface=ether2

# --- VPLS LDP - RouterOS v7, di PE1 ---
/interface vpls
add name=vpls-A peer=10.0.0.2 vpls-id=100:100 cisco-style=no disabled=no
/interface bridge
add name=bridge-VPLS-A
/interface bridge port
add bridge=bridge-VPLS-A interface=vpls-A
add bridge=bridge-VPLS-A interface=ether3 comment="to CE1"

# --- L3VPN VRF - RouterOS v7 ---
/ip route vrf
add routing-mark=CUST-B interfaces=ether4 route-distinguisher=65000:100 \
  import-route-targets=65000:100 export-route-targets=65000:100

# --- BGP VPNv4 - RouterOS v7 ---
/routing bgp template
add name=bgp-vpn as=65000 router-id=10.0.0.1 address-families=vpnv4
/routing bgp connection
add name=to-PE2-vpn templates=bgp-vpn remote.address=10.0.0.2 remote.as=65000
```

#### Panduan Pemilihan Versi

- Perbedaan yang paling signifikan terdapat pada OSPF dan BGP. VPLS, LDP, VRF, dan MTU hampir identik.
- Apabila migrasi dari v6 ke v7 dilakukan, OSPF dan BGP harus ditulis ulang. VPLS dan VRF dapat dipertahankan dengan penyesuaian kecil.
- Nilai `vpls-id`, RD, RT, dan `routing-mark` harus identik pada kedua versi agar pelanggan tidak terputus pada saat migrasi.

### 2.4 Verifikasi MPLS

```routeros
/mpls ldp neighbor print
/mpls forwarding-table print
/interface vpls print
/ip route vrf print
/routing bgp peer print status
# Dari CE1
/ping 192.168.100.12  # VPLS Pelanggan A
/ping 172.16.20.2 routing-mark=CUST-B  # L3VPN Pelanggan B
/tool traceroute 10.0.0.2
```

- **VPLS tidak aktif?** Perlu diperiksa LDP neighbor, kesamaan vpls-id, MTU, kesamaan CW, dan bridge port.
- **L3VPN tidak memiliki rute?** Perlu diperiksa kesamaan RD/RT, status BGP VPNv4 Established (terbentuk), dan routing-mark VRF pada interface.
- **Paket berukuran besar terbuang?** Perlu diperiksa L2MTU pada seluruh tautan, minimal 1600 untuk VPLS dan label MPLS.

### 2.5 Operasi MPLS — Prosedur Operasional

- **Pemantauan:** LDP neighbor, status tunnel VPLS, jumlah label, dan trafik per VRF dipantau. Apabila satu VPLS terputus, LDP dan OSPF diperiksa terlebih dahulu, bukan bridge.
- **Penambahan pelanggan baru:** VRF / VPLS ID baru dibuat, ID lama tidak digunakan kembali. RD/RT dan VPLS-ID didokumentasikan pada spreadsheet.
- **Audit MTU:** Setiap penambahan tautan baru, MTU dipastikan telah 1600. Satu tautan dengan MTU yang kecil dapat merusak seluruh VPLS.
- **Pencadangan:** Konfigurasi MPLS dan VRF diekspor. VPLS dan VRF sangat sensitif terhadap kesalahan penulisan ID.

## 3. Modul 3 - Traffic Engineering

Traffic Engineering (TE) menjawab permasalahan klasik: tautan utama penuh, sedangkan tautan cadangan kosong. Tanpa TE, routing selalu memilih jalur terpendek sehingga tautan utama mengalami overload. Dengan TE, sebagian trafik dapat dipaksa melewati jalur lain yang lebih lengang.

### 3.1 Studi Kasus — Kongesti (Congestion) Tautan Surabaya—Jakarta

**Skenario:** ISP memiliki dua jalur Surabaya (PE1) menuju Jakarta (PE2): **Jalur Utama** PE1—P1—PE2 melalui Madiun dengan kapasitas 20 Mbps yang telah terpakai 95% pada jam sibuk karena trafik video. **Jalur Alternatif** PE1—P2—PE2 melalui Malang dengan kapasitas 50 Mbps yang baru terpakai 20%. Tanpa TE, seluruh trafik tetap melewati jalur utama karena jalur tersebut lebih pendek. Tujuan yang hendak dicapai adalah memindahkan trafik video (misalnya 10 Mbps) ke jalur alternatif dan menyisakan jalur utama untuk trafik prioritas. Jalur TE harus melakukan reservasi bandwidth agar tidak berebut dengan trafik lain.

**Target:** Setelah TE aktif, utilisasi jalur utama menurun menjadi 60%, jalur alternatif meningkat menjadi 40%, dan tidak terjadi packet loss pada jam sibuk.

#### Topologi Modul 3

PE1 (10.0.0.1) — P1 (10.0.0.3) — PE2 (10.0.0.2) merupakan jalur utama (cost OSPF rendah). PE1 — P2 (10.0.0.4) — PE2 merupakan jalur alternatif (cost OSPF lebih tinggi). TE Tunnel dibuat dari PE1 menuju PE2 dengan explicit path melalui P2.

| Tautan | Kapasitas | Utilisasi Awal |
|---|---|---|
| PE1 — P1 — PE2 (Utama) | 20 Mbps | 95% (kongesti / congestion) |
| PE1 — P2 — PE2 (Alternatif) | 50 Mbps | 20% (lengang) |
| TE Tunnel PE1 ke PE2 | Reservasi 10 Mbps | Untuk trafik video |

### 3.2 Implementasi Bertahap — Traffic Engineering

#### Tahap 1 — Memastikan MPLS, OSPF, dan LDP Berjalan

Traffic Engineering (TE) tidak dapat berdiri sendiri. TE memerlukan OSPF dengan TE Extensions (ekstensi Traffic Engineering) dan MPLS. Perlu dipastikan bahwa Modul 2 telah berjalan sebelum melanjutkan.

```routeros
# Pada seluruh router backbone, pastikan OSPF telah mengumumkan (advertise)
# TE extensions dan MPLS telah aktif
/routing ospf instance
set [find name=ospf1] mpls-te=yes
/mpls traffic-eng interface
add interface=ether1 bandwidth=20M
add interface=ether2 bandwidth=50M
```

#### Tahap 2 — Resource Reservation Protocol with Traffic Engineering (RSVP-TE)

RSVP (Resource Reservation Protocol) digunakan untuk reservasi bandwidth di sepanjang jalur TE. Tanpa RSVP, router tidak mengetahui berapa bandwidth yang telah terpakai pada setiap tautan.

```routeros
# Mengaktifkan RSVP-TE pada seluruh router backbone
/mpls traffic-eng rsvp
set enabled=yes
# Pemeriksaan: /mpls traffic-eng rsvp session print
```

#### Tahap 3 — TE Tunnel: Static Path dan Dynamic Path (Constrained Shortest Path First / CSPF)

**Dynamic Path (CSPF):** Router menghitung secara mandiri best path (jalur terbaik) yang memenuhi constraint (batasan) bandwidth dengan menggunakan database OSPF-TE. Metode ini sesuai apabila topologi sering berubah. **Static Path / Explicit Path:** Pengelola jaringan menentukan secara manual router mana hanya yang harus dilalui oleh tunnel. Metode ini sesuai untuk studi kasus ini karena jalur dipaksa melalui P2.

```routeros
# --- Pembuatan Explicit Path melalui P2 (10.0.0.4) ---
/mpls traffic-eng path
add name=via-P2 use-cspf=no
/mpls traffic-eng path hop
add path=via-P2 address=10.0.0.4 strict=yes
add path=via-P2 address=10.0.0.2 strict=yes

# --- Pembuatan TE Tunnel dari PE1 menuju PE2 ---
/mpls traffic-eng tunnel
add name=te-PE1-to-PE2 to-address=10.0.0.2 \
  bandwidth=10M \
  primary-path=via-P2 \
  record-route=yes

# Alternatif: Dynamic CSPF (tanpa explicit path)
# /mpls traffic-eng tunnel add name=te-dynamic to-address=10.0.0.2 \
#   bandwidth=10M primary-path="" use-cspf=yes
```

#### Tahap 4 — Bandwidth Allocation dan Bandwidth Limitation

**Bandwidth Allocation (pada TE):** Bandwidth yang direservasi untuk tunnel. Apabila reservasi 10 Mbps dilakukan, OSPF-TE akan mengurangi bandwidth yang tersedia pada tautan tersebut untuk tunnel lain. **Bandwidth Limitation (pada Queue / Interface):** Batas keras trafik yang diperkenankan lewat. TE tidak membatasi trafik secara otomatis, TE hanya melakukan reservasi. Untuk benar-benar membatasi, Queue atau policer perlu ditambahkan. Kedua konsep tersebut tidak dapat dicampur.

```routeros
# Contoh: Penandaan trafik video yang masuk ke TE Tunnel
# Trafik video ditandai dengan Mangle, kemudian diarahkan ke TE Tunnel
/ip firewall mangle
add chain=forward dst-port=443 protocol=tcp \
  action=mark-routing new-routing-mark=via-TE \
  comment="Video traffic to TE"

/ip route
add dst-address=0.0.0.0/0 gateway=te-PE1-to-PE2 \
  routing-mark=via-TE comment="Steer video via TE"

# Atau menggunakan policy routing
# /mpls traffic-eng tunnel set [find name=te-PE1-to-PE2] auto-bandwidth=yes
```

#### Tahap 5 — Steering (Pengarahan Trafik) dan Verifikasi

TE tunnel yang telah aktif belum tentu digunakan. Pengelola jaringan wajib melakukan steering (pengarahan trafik) ke tunnel, dapat melalui static route, policy routing, atau BGP next-hop. Pada laboratorium, static route dengan routing-mark sebagaimana contoh di atas digunakan, kemudian diuji.

```routeros
# Verifikasi TE
/mpls traffic-eng tunnel print
/mpls traffic-eng tunnel path print
/mpls traffic-eng rsvp session print
# Pemeriksaan bandwidth yang yang telah dipesan (reserved) pada tautan
/mpls traffic-eng interface print detail
# Dari PE1, traceroute harus melalui P2 (10.0.0.4)
/tool traceroute 10.0.0.2 src-address=10.0.0.1
```

### 3.3 Perbandingan RouterOS v6 dan v7 — Traffic Engineering

Contoh konfigurasi pada Tahap 1 hingga Tahap 5 menggunakan RouterOS v7. Pada subbab ini, perbandingan dan contoh setara untuk RouterOS v6 disediakan agar Traffic Engineering dapat diterapkan pada kedua versi tersebut.

#### Tabel Perbandingan

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| OSPF-TE | `/routing ospf instance` dengan `mpls-te` | `/routing ospf instance` dengan `mpls-te=yes` | Pada kedua versi, OSPF-TE harus diaktifkan agar CSPF berfungsi |
| Interface TE | `/mpls traffic-eng interface` | `/mpls traffic-eng interface` | Parameter `bandwidth` dinyatakan pada kedua versi |
| RSVP-TE | `/mpls traffic-eng rsvp` | `/mpls traffic-eng rsvp` | Perintah pengaktifan hampir identik |
| TE Path | `/mpls traffic-eng path` | `/mpls traffic-eng path` | Explicit path dan `strict` identik pada kedua versi |
| TE Tunnel | `/mpls traffic-eng tunnel` | `/mpls traffic-eng tunnel` | Parameter `bandwidth`, `primary-path`, dan `to-address` identik |
| CSPF | `use-cspf=yes` pada path / tunnel | `use-cspf=yes` pada path / tunnel | CSPF menggunakan database OSPF-TE |
| Steering | Static route, policy routing, mangle | Static route, policy routing, mangle | Mekanisme steering identik, hanya menu routing yang berbeda pada v7 |
| Pemeriksaan TE | `/mpls traffic-eng tunnel print`, `/mpls traffic-eng rsvp session print` | `/mpls traffic-eng tunnel print`, `/mpls traffic-eng rsvp session print` | Perintah pemeriksaan identik |
| Bandwidth Allocation | Reservasi pada TE Tunnel | Reservasi pada TE Tunnel | Tidak membatasi trafik secara otomatis pada kedua versi |
| Bandwidth Limitation | Queue / policer pada interface | Queue / policer pada interface | Diperlukan untuk batas keras trafik |

#### Konfigurasi RouterOS v6 — Setara dengan Tahap 1 hingga Tahap 3

```routeros
# --- OSPF-TE + Interface - RouterOS v6, di seluruh router backbone ---
/routing ospf instance
set [find name=ospf1] mpls-te=yes
/mpls traffic-eng interface
add interface=ether1 bandwidth=20M
add interface=ether2 bandwidth=50M

# --- RSVP-TE - RouterOS v6 ---
/mpls traffic-eng rsvp
set enabled=yes

# --- Explicit Path via P2 - RouterOS v6 ---
/mpls traffic-eng path
add name=via-P2 use-cspf=no
/mpls traffic-eng path hop
add path=via-P2 address=10.0.0.4 strict=yes
add path=via-P2 address=10.0.0.2 strict=yes

# --- TE Tunnel - RouterOS v6 ---
/mpls traffic-eng tunnel
add name=te-PE1-to-PE2 to-address=10.0.0.2 bandwidth=10M \
  primary-path=via-P2 record-route=yes

# --- Steering - RouterOS v6 ---
/ip firewall mangle
add chain=forward dst-port=443 protocol=tcp \
  action=mark-routing new-routing-mark=via-TE comment="Video traffic to TE"
/ip route
add dst-address=0.0.0.0/0 gateway=te-PE1-to-PE2 routing-mark=via-TE
```

#### Konfigurasi RouterOS v7 — Ringkasan Setara

```routeros
# --- OSPF-TE + Interface - RouterOS v7 ---
/routing ospf instance
set [find name=ospf1] mpls-te=yes
/mpls traffic-eng interface
add interface=ether1 bandwidth=20M
add interface=ether2 bandwidth=50M

# --- RSVP-TE - RouterOS v7 ---
/mpls traffic-eng rsvp
set enabled=yes

# --- Explicit Path via P2 - RouterOS v7 ---
/mpls traffic-eng path
add name=via-P2 use-cspf=no
/mpls traffic-eng path hop
add path=via-P2 address=10.0.0.4 strict=yes
add path=via-P2 address=10.0.0.2 strict=yes

# --- TE Tunnel - RouterOS v7 ---
/mpls traffic-eng tunnel
add name=te-PE1-to-PE2 to-address=10.0.0.2 bandwidth=10M \
  primary-path=via-P2 record-route=yes

# --- Steering - RouterOS v7 ---
/ip firewall mangle
add chain=forward dst-port=443 protocol=tcp \
  action=mark-routing new-routing-mark=via-TE comment="Video traffic to TE"
/ip route
add dst-address=0.0.0.0/0 gateway=te-PE1-to-PE2 routing-mark=via-TE
```

#### Panduan Pemilihan Versi

- Pada Traffic Engineering, perbedaan v6 dan v7 relatif kecil. Perintah TE, RSVP, path, dan tunnel hampir identik.
- Perbedaan utama terdapat pada OSPF. Apabila OSPF telah dimigrasikan ke v7 dengan `interface-template`, konfigurasi TE dapat dipertahankan.
- Apabila tunnel tidak aktif setelah migrasi, periksa terlebih dahulu OSPF-TE dan RSVP, bukan tunnel. Kedua protokol tersebut merupakan prasyarat TE pada kedua versi.

### 3.4 Operasi TE — Prosedur Operasional

- **Pemantauan:** Status tunnel diperiksa setiap hari. Tunnel TE dapat terputus apabila RSVP gagal melakukan reservasi karena bandwidth tautan telah habis. Peringatan dikirimkan apabila tunnel terputus lebih dari 1 menit.
- **Perencanaan Kapasitas:** Total bandwidth TE yang direservasi tidak melebihi 80% kapasitas tautan fisik. Sisa kapasitas disediakan untuk trafik best-effort dan failover.
- **Failover:** Secondary path disiapkan. Apabila primary path melalui P2 terputus, tunnel harus dapat berpindah ke jalur lain atau kembali ke IGP. Failover diuji pada jendela pemeliharaan dengan menonaktifkan tautan P2.
- **Dokumentasi:** Setiap tunnel wajib memiliki nama yang jelas (dari mana menuju ke mana, untuk trafik apa, dan berapa bandwidth). Tanpa dokumentasi, TE akan menjadi tunnel yang tidak terdokumentasi dan tidak terstruktur yang tidak ada yang berani menghapus.

#### Penanganan Masalah TE

| Gejala | Penyebab | Tindakan |
|---|---|---|
| Tunnel Down / No Path | Bandwidth tidak mencukupi / CSPF tidak menemukan jalur | Kurangi bandwidth, periksa OSPF-TE DB, periksa explicit path |
| Tunnel Up tetapi trafik tidak melewati | Steering belum diatur | Periksa routing-mark / static route menuju tunnel |
| RSVP session gagal | RSVP tidak aktif pada salah satu hop | Aktifkan RSVP pada seluruh router pada jalur |
| Utilisasi tidak berubah | Trafik yang diarahkan (steered) tidak sesuai / mangle (penandaan paket) tidak sesuai | Periksa mangle (penandaan paket) counter, periksa /ip firewall mangle print stats |


## 4. Lampiran A - Matriks Penanganan Masalah dan Perintah Cepat

| Domain | Perintah Pemeriksaan Cepat | Hal yang Dicari |
|---|---|---|
| BGP | `/routing bgp peer print status` | Established, prefix count |
| BGP Route | `/routing bgp route print` | Active, Local-Pref, AS Path |
| OSPF | `/routing ospf neighbor print` | Full (terbentuk penuh), Router-ID |
| LDP | `/mpls ldp neighbor print` | Established |
| MPLS Forwarding | `/mpls forwarding-table print` | Label tersedia untuk loopback PE |
| VPLS | `/interface vpls print` | Running, peer reachable |
| VRF / L3VPN | `/ip route print where routing-mark=CUST-B` | Rute pelanggan tersedia |
| TE Tunnel | `/mpls traffic-eng tunnel print` | Up, bandwidth, path |
| RSVP | `/mpls traffic-eng rsvp session print` | Session Up |
| Interface | `/interface print where running` | MTU, error, drop |

**Prinsip penanganan masalah:** Pemeriksaan selalu dilakukan dari bawah ke atas — Fisik (tautan aktif?) — IGP (OSPF Full (terbentuk penuh)?) — LDP (neighbor?) — BGP / VPN (session?) — Layanan (VPLS / TE aktif?) — Trafik (mangle / route?). Pemeriksaan tidak dilakukan dengan melompat langsung ke BGP apabila OSPF belum Full.

## 5. Lampiran B - Templat Pencadangan (Backup), Rollback (Pengembalian), dan Permintaan Perubahan

### Backup Sebelum Ubah Konfigurasi

```routeros
# Pada setiap router, sebelum jendela pemeliharaan
/export file=backup-YYYYMMDD-HHMM
/system backup save name=backup-YYYYMMDD-HHMM
# Berkas cadangan diunduh ke server, tidak hanya disimpan pada router
```

### Rollback

```routeros
# Apabila perubahan filter BGP bermasalah, filter lama dikembalikan
/routing filter rule remove [find chain=to-ISP-A-out]
# Kemudian impor ekspor lama atau tempel konfigurasi cadangan
/import file=backup-YYYYMMDD-HHMM.rsc
```
