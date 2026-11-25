# MTCSE — Panduan Implementasi dan Operasi

## Daftar Isi

- [Modul 1 - Konfigurasi Dasar](#1-modul-1-konfigurasi-dasar)
- [Modul 2 - Firewall](#2-modul-2-firewall)
- [Modul 3 - Serangan pada Lapisan OSI](#3-modul-3-serangan-pada-lapisan-osi)
- [Modul 4 - Kriptografi](#4-modul-4-kriptografi)
- [Modul 5 - Pengamanan Router](#5-modul-5-pengamanan-router)
- [Modul 6 - Tunnel](#6-modul-6-tunnel-aman)

## 1. Modul 1 - Konfigurasi Dasar

### Studi Kasus
Sebuah kantor kecil menggunakan `password` administrator bawaan dan seluruh layanan berada dalam keadaan aktif. Botnet melakukan pemindaian terhadap Internet, menemukan port WinBox 8291 yang terbuka, menguji `password` yang lemah, dan dalam kurun waktu satu hari router tersebut menjadi bagian dari botnet.

### Contoh Konfigurasi
```
/system identity set name=GW-JKT-01
/system clock set time-zone-name=Asia/Jakarta
/system ntp client set enabled=yes servers=pool.ntp.org
/user add name=netadmin group=full password=GantiPasswordKuat2026!
/user set admin password=GantiPasswordKuatJuga2026!
/ip service disable telnet,ftp,www,api,api-ssl
/ip service set ssh port=2222 address=192.168.88.0/24
/ip service set winbox port=8299 address=192.168.88.0/24
/ip service set www-ssl port=8443 address=192.168.88.0/24 certificate=none
/tool mac-server set allowed-interface-list=none
/tool mac-server mac-winbox set allowed-interface-list=none
/ip neighbor discovery-settings set discover-interface-list=none
```

### Verifikasi
```
/user print
/ip service print
/log print where topics~account
```

> Catatan: Perubahan nomor port saja tidak dapat dianggap sebagai pengamanan yang memadai. Port non-bawaan hanya mengurangi kebisingan hasil pemindaian, bukan pengganti firewall, `password` yang kuat, dan pembatasan alamat IP.

### Perbandingan RouterOS v6 dan v7 — Konfigurasi Dasar

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| NTP Client | `/system ntp client set enabled=yes primary-ntp=103.10.2.1 secondary-ntp=103.10.2.2` | `/system ntp client set enabled=yes servers=pool.ntp.org` | RouterOS v7 mendukung daftar server dan sinkronisasi yang lebih baik |
| Manajemen `User` | `/user add name=... group=full` | `/user add name=... group=full` | Sintaks identik. RouterOS v7 menerapkan kebijakan `password` yang lebih ketat |
| Layanan | `/ip service set ssh port=...` | `/ip service set ssh port=...` | Sintaks identik. RouterOS v7 mendukung WinBox 4 |
| Discovery | `/ip neighbor discovery-settings set discover-interface-list=...` | Identik dengan RouterOS v6 | Sintaks identik |
| MAC Server | `/tool mac-server set allowed-interface-list=...` | Identik dengan RouterOS v6 | Sintaks identik |

#### Contoh Konfigurasi v6 — Konfigurasi Dasar
```
# RouterOS v6 - NTP menggunakan primary dan secondary
/system identity set name=GW-JKT-01
/system clock set time-zone-name=Asia/Jakarta
/system ntp client set enabled=yes primary-ntp=103.10.2.1 secondary-ntp=103.10.2.2
/user add name=netadmin group=full password=GantiPasswordKuat2026!
/ip service disable telnet,ftp,www,api,api-ssl
/ip service set ssh port=2222 address=192.168.88.0/24
/ip service set winbox port=8299 address=192.168.88.0/24
/tool mac-server set allowed-interface-list=none
/tool mac-server mac-winbox set allowed-interface-list=none
/ip neighbor discovery-settings set discover-interface-list=none
```

#### Contoh Konfigurasi v7 — Konfigurasi Dasar
```
# RouterOS v7 - NTP menggunakan daftar servers
/system identity set name=GW-JKT-01
/system clock set time-zone-name=Asia/Jakarta
/system ntp client set enabled=yes servers=pool.ntp.org
/user add name=netadmin group=full password=GantiPasswordKuat2026!
/ip service disable telnet,ftp,www,api,api-ssl
/ip service set ssh port=2222 address=192.168.88.0/24
/ip service set winbox port=8299 address=192.168.88.0/24
/tool mac-server set allowed-interface-list=none
/tool mac-server mac-winbox set allowed-interface-list=none
/ip neighbor discovery-settings set discover-interface-list=none
```

> Catatan: Perbedaan utama pada modul ini terletak pada NTP. Selain itu, sintaks konfigurasi dasar pada RouterOS v6 dan RouterOS v7 sebagian besar identik, sehingga migrasi dapat dilakukan dengan penyesuaian minimal.

## 2. Modul 2 - Firewall

**Tujuan:** Menguasai alur paket, firewall stateful, tabel RAW, mitigasi SYN flood, praktik terbaik akses manajemen, deteksi serangan terhadap layanan kritis, filter bridge, dan penyaringan ICMP.

### Konsep
- **Input** merupakan paket yang ditujukan kepada router itu sendiri. **Forward** merupakan paket yang melewati router. **Output** merupakan paket yang berasal dari router.
- **Stateful firewall** mencatat status koneksi. Paket dengan status `established,related` merupakan balasan yang sah. Paket dengan status `invalid` wajib ditolak.
- **Tabel RAW** diproses sebelum pelacakan koneksi. Tabel tersebut sesuai untuk membuang paket berbahaya sejak awal agar CPU tidak terbebani pencatatan koneksi palsu.

### Studi Kasus
Sebuah server di LAN menyebabkan CPU router mencapai 100 persen, Internet melambat, dan log dipenuhi koneksi SYN. Setelah dilakukan pemeriksaan, terdapat SYN flood sebesar 50.000 paket per detik dari Internet. Firewall filter biasa masih mencatat seluruh koneksi, sehingga CPU tetap penuh. Solusi yang diterapkan adalah memindahkan mitigasi ke tabel RAW dan membatasi SYN baru per alamat IP.

### Contoh Konfigurasi — Firewall Filter
```
/interface list add name=WAN
/interface list add name=LAN
/interface list member add list=WAN interface=ether1
/interface list member add list=LAN interface=ether2
# --- INPUT ---
/ip firewall filter add chain=input action=accept connection-state=established,related,untracked comment="Menerima balasan yang sah"
/ip firewall filter add chain=input action=accept protocol=icmp limit=5,5:packet comment="ICMP terbatas"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299,2222 src-address-list=MGMT in-interface-list=LAN comment="Manajemen dari MGMT"
/ip firewall filter add chain=input action=drop connection-state=invalid comment="Menolak paket invalid"
/ip firewall filter add chain=input action=drop in-interface-list=WAN comment="Menolak sisanya dari WAN" log=yes log-prefix="DROP-INPUT-WAN"
/ip firewall filter add chain=input action=drop comment="Menolak sisanya ke router"
# --- FORWARD ---
/ip firewall filter add chain=forward action=accept connection-state=established,related,untracked comment="Menerima balasan"
/ip firewall filter add chain=forward action=accept in-interface-list=LAN out-interface-list=WAN comment="LAN diizinkan ke Internet"
/ip firewall filter add chain=forward action=drop connection-state=invalid comment="Menolak invalid forward"
/ip firewall filter add chain=forward action=drop in-interface-list=WAN comment="Menolak WAN ke LAN kecuali port-forward"
/ip firewall filter add chain=forward action=drop comment="Menolak sisanya forward"
```

### Contoh Konfigurasi — RAW dan SYN Flood
```
/ip firewall raw add chain=prerouting action=drop src-address-list=BOGON comment="Menolak IP bogon"
/ip firewall raw add chain=prerouting action=drop protocol=tcp tcp-flags=syn src-address-list=SYN-FLOOD comment="Menolak SYN dari pelaku flood"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp tcp-flags=syn connection-state=new limit=100,5:packet address-list=SYN-FLOOD address-list-timeout=1m comment="Deteksi SYN Flood" log=yes log-prefix="SYN-FLOOD"
```

### Contoh Konfigurasi — DNS dan ICMP
```
/ip firewall filter add chain=input action=accept protocol=udp dst-port=53 in-interface-list=LAN comment="DNS dari LAN"
/ip firewall filter add chain=input action=drop protocol=udp dst-port=53 in-interface-list=WAN comment="Menolak DNS dari WAN"
/interface bridge filter add chain=forward action=drop mac-protocol=ip ip-protocol=udp dst-port=67 in-interface=ether4 comment="Memblokir DHCP liar dari ether4"
/ip firewall filter add chain=input action=accept protocol=icmp icmp-options=8:0 limit=5,5:packet comment="Ping terbatas"
/ip firewall filter add chain=input action=accept protocol=icmp icmp-options=3:4 comment="Packet Too Big untuk PMTU"
/ip firewall filter add chain=input action=accept protocol=icmp icmp-options=11:0 comment="Time Exceeded"
```

### Verifikasi
```
/ip firewall filter print stats
/ip firewall raw print stats
/ip firewall connection print where tcp-state=syn-received
/ip firewall address-list print where list=SYN-FLOOD
/tool torch interface=ether1
```

> Catatan: Urutan `rule` memiliki peranan yang menentukan. `Rule` dibaca dari atas ke bawah, dan `rule` pertama yang cocok akan diterapkan. Administrator wajib menempatkan `rule` accept established,related pada posisi paling atas, dilanjutkan `rule` spesifik, kemudian `rule` `drop` pada bagian bawah.

### Perbandingan RouterOS v6 dan v7 — Firewall

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| Firewall Filter | `/ip firewall filter add chain=input ...` | Identik dengan RouterOS v6 | Sintaks filter identik |
| Tabel RAW | `/ip firewall raw add chain=prerouting ...` | Identik dengan RouterOS v6 | Sintaks RAW identik |
| Connection Tracking | `/ip firewall connection tracking set enabled=yes tcp-syncookie=yes` | `/ip firewall connection tracking set enabled=yes loose-tcp-tracking=yes` | RouterOS v7 menambahkan opsi loose-tcp-tracking |
| Daftar Alamat | `/ip firewall address-list add list=...` | Identik dengan RouterOS v6 | Sintaks identik |
| IPv6 Firewall | `/ipv6 firewall filter ...` | `/ipv6 firewall filter ...` | RouterOS v7 memiliki penanganan IPv6 yang lebih optimal |
| Bridge Filter | `/interface bridge filter ...` | Identik, dengan peningkatan penyaringan VLAN | RouterOS v7 lebih stabil untuk bridge VLAN |

#### Contoh Konfigurasi v6 — Firewall
```
# RouterOS v6 - Connection Tracking
/ip firewall connection tracking set enabled=yes tcp-syncookie=yes tcp-established-timeout=1d

# Filter Input - sintaks identik dengan RouterOS v7
/ip firewall filter add chain=input action=accept connection-state=established,related,untracked comment="Menerima balasan yang sah"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299,2222 src-address-list=MGMT in-interface-list=LAN comment="Manajemen dari MGMT"
/ip firewall filter add chain=input action=drop connection-state=invalid comment="Menolak paket invalid"

# RAW - SYN Flood
/ip firewall raw add chain=prerouting action=drop protocol=tcp tcp-flags=syn src-address-list=SYN-FLOOD comment="Menolak SYN flood"
```

#### Contoh Konfigurasi v7 — Firewall
```
# RouterOS v7 - Connection Tracking dengan loose-tcp-tracking
/ip firewall connection tracking set enabled=yes loose-tcp-tracking=yes tcp-established-timeout=1d

# Filter Input - sintaks identik dengan RouterOS v6
/ip firewall filter add chain=input action=accept connection-state=established,related,untracked comment="Menerima balasan yang sah"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299,2222 src-address-list=MGMT in-interface-list=LAN comment="Manajemen dari MGMT"
/ip firewall filter add chain=input action=drop connection-state=invalid comment="Menolak paket invalid"

# RAW - SYN Flood
/ip firewall raw add chain=prerouting action=drop protocol=tcp tcp-flags=syn src-address-list=SYN-FLOOD comment="Menolak SYN flood"
```

> Catatan: Untuk firewall, sebagian besar `rule` dapat disalin secara langsung dari RouterOS v6 ke RouterOS v7. Perbedaan utama hanya terdapat pada peng`rule` connection tracking dan kinerja IPv6.

## 3. Modul 3 - Serangan pada Lapisan OSI

**Tujuan:** Mengenali dan mencegah serangan per protokol: MNDP, DHCP, TCP, UDP, ICMP, brute-force, dan port scan.

| Serangan | Mitigasi Inti |
|---|---|
| MNDP | Pembatasan discovery hanya ke antarmuka tepercaya |
| DHCP Rogue | DHCP Alert dan pembatasan port |
| DHCP Starvation | Batas lease, static lease untuk perangkat penting, dan alert |
| TCP SYN Flood | RAW, limit, dan `blocklist` |
| UDP Flood | RAW drop dan limit |
| ICMP Smurf | `Drop` directed broadcast dan pembatasan ICMP |
| Brute-force | Penonaktifan FTP/Telnet, pembatasan SSH, dan `blocklist` |
| Port Scan | Deteksi PSD dan `blocklist` pemindai |

### Studi Kasus 1 — DHCP Rogue
Seluruh klien secara tiba-tiba tidak dapat mengakses Internet. Klien memperoleh alamat IP 192.168.1.x dengan gateway 192.168.1.99, padahal gateway resmi adalah 192.168.88.1. Setelah dilakukan pemeriksaan, terdapat karyawan yang menghubungkan router rumahan ke LAN.

### Studi Kasus 2 — Brute-Force SSH
Log dipenuhi kegagalan login untuk `user` admin, root, dan test dari alamat IP luar negeri.

### Contoh Konfigurasi
```
# MNDP
/ip neighbor discovery-settings set discover-interface-list=LAN
# DHCP Rogue Alert - alamat MAC wajib diganti dengan alamat MAC server resmi
/ip dhcp-server alert add interface=ether2 on-alert=":log warning \"ROGUE DHCP DETECTED\"" valid-server=AA:BB:CC:DD:EE:FF
# DHCP Starvation
/ip dhcp-server set [find] lease-time=1h
/ip dhcp-server lease add address=192.168.88.10 mac-address=AA:BB:CC:DD:EE:10 server=dhcp1 comment="Server Utama"
```

```
# Brute-force SSH: tiga kali kegagalan dalam satu menit akan mengakibatkan penambahan ke `blocklist` selama satu hari
/ip firewall filter add chain=input action=drop protocol=tcp dst-port=2222 src-address-list=SSH-BRUTE comment="Menolak SSH dari brute-forcer"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=2222 connection-state=new limit=3,1m address-list=SSH-BRUTE address-list-timeout=1d comment="Deteksi brute-force SSH" log=yes log-prefix="SSH-BRUTE"
/ip service disable ftp,telnet
# Port Scan
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp psd=21,3s,3,1 address-list=PORT-SCANNER address-list-timeout=1d comment="Deteksi Port Scan" log=yes log-prefix="PORT-SCAN"
/ip firewall filter add chain=input action=drop src-address-list=PORT-SCANNER comment="Menolak pemindai"
# UDP Flood dan Smurf
/ip firewall raw add chain=prerouting action=drop dst-address-type=broadcast comment="Menolak directed broadcast"
/ip firewall filter add chain=input action=accept protocol=icmp limit=5,5:packet comment="ICMP terbatas"
```

### Verifikasi
```
/ip dhcp-server lease print
/ip firewall address-list print where list=SSH-BRUTE
/ip firewall address-list print where list=PORT-SCANNER
/log print where message~"ROGUE" or message~"BRUTE"
```


### Perbandingan RouterOS v6 dan v7 — Serangan pada Lapisan OSI

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| DHCP Alert | `/ip dhcp-server alert add interface=... valid-server=...` | Identik dengan RouterOS v6 | Sintaks identik |
| DHCP Server | `/ip dhcp-server set lease-time=...` | Identik, dengan manajemen lease yang lebih baik | RouterOS v7 lebih stabil untuk pool besar |
| Brute-Force SSH | Firewall filter dengan `add-src-to-address-list` | Identik dengan RouterOS v6 | Sintaks firewall identik |
| Port Scan (PSD) | `psd=21,3s,3,1` | Identik dengan RouterOS v6 | Sintaks identik |
| MNDP / Neighbor | `/ip neighbor discovery-settings ...` | Identik dengan RouterOS v6 | Sintaks identik |
| Log | `/system logging add topics=...` | Identik, dengan performa log yang lebih baik | RouterOS v7 mendukung log yang lebih terstruktur |

#### Contoh Konfigurasi v6 — Serangan Lapisan OSI
```
# RouterOS v6
/ip neighbor discovery-settings set discover-interface-list=LAN
/ip dhcp-server alert add interface=ether2 on-alert=":log warning \"ROGUE DHCP DETECTED\"" valid-server=AA:BB:CC:DD:EE:FF
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=2222 connection-state=new limit=3,1m address-list=SSH-BRUTE address-list-timeout=1d comment="Deteksi brute-force SSH"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp psd=21,3s,3,1 address-list=PORT-SCANNER address-list-timeout=1d comment="Deteksi Port Scan"
```

#### Contoh Konfigurasi v7 — Serangan Lapisan OSI
```
# RouterOS v7 - sintaks identik, hanya peningkatan internal
/ip neighbor discovery-settings set discover-interface-list=LAN
/ip dhcp-server alert add interface=ether2 on-alert=":log warning \"ROGUE DHCP DETECTED\"" valid-server=AA:BB:CC:DD:EE:FF
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=2222 connection-state=new limit=3,1m address-list=SSH-BRUTE address-list-timeout=1d comment="Deteksi brute-force SSH"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp psd=21,3s,3,1 address-list=PORT-SCANNER address-list-timeout=1d comment="Deteksi Port Scan"
# Tambahan RouterOS v7: pemeriksaan lease dilakukan dengan lebih cepat
/ip dhcp-server lease print where status=bound
```

> Catatan: Pada modul ini, konfigurasi RouterOS v6 dan RouterOS v7 dapat dipertukarkan. Perbedaan terletak pada kinerja DHCP dan pen`log`, bukan pada sintaks.

## 4. Modul 4 - Kriptografi

**Tujuan:** Memahami enkripsi, algoritma simetris dan asimetris, PKI, dan sertifikat, kemudian menggunakannya pada RouterOS.

- **Simetris:** Menggunakan satu kunci untuk mengunci dan membuka. Metode tersebut cepat dan sesuai untuk data berukuran besar. Contoh: AES.
- **Asimetris:** Menggunakan dua kunci, yaitu kunci publik dan kunci privat. Contoh: RSA.
- **PKI:** Certificate Authority (CA) mengesahkan sertifikat dan menjamin identitas yang tercantum di dalamnya.
- **Sertifikat:** Kartu identitas digital. Apabila masa berlaku telah berakhir atau CA tidak dipercaya, peramban atau VPN akan menolaknya.

### Studi Kasus
WebFig masih menggunakan HTTP. Administrator masuk dari WiFi kantor yang sama dengan tamu. Seseorang menjalankan sniffing dan menangkap `password` administrator dalam bentuk plaintext.

### Contoh Konfigurasi — CA Self-Signed dan Sertifikat Server
```
/certificate add name=my-ca common-name="My Lab CA" key-usage=key-cert-sign,crl-sign key-size=2048 days-valid=3650
/certificate sign my-ca ca-crl-host=127.0.0.1 name=my-ca
/certificate add name=web-cert common-name="gw-jkt-01.lab.local" key-usage=digital-signature,key-encipherment,tls-server key-size=2048 days-valid=825
/certificate sign web-cert ca=my-ca name=web-cert
/certificate set my-ca trusted=yes
/ip service set www-ssl certificate=web-cert port=8443
/ip service disable www
/certificate print
```

### Let's Encrypt untuk Lingkungan Produksi
```
# Syarat: router memiliki DNS publik, misalnya gw.jkt-kantor.com -> IP Publik router
/certificate enable-ssl-certificate dns-name=gw.jkt-kantor.com
/ip service set www-ssl certificate=gw.jkt-kantor.com
/interface sstp-server server set certificate=gw.jkt-kantor.com
```

> Catatan: Kunci privat CA wajib disimpan di tempat yang aman. Pengelola wajib membuat pengingat 30 hari sebelum sertifikat kedaluwarsa.


### Perbandingan RouterOS v6 dan v7 — Kriptografi

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| Pembuatan CA | `/certificate add name=my-ca ...` kemudian `sign` | Identik, dengan opsi `key-usage` yang lebih lengkap | RouterOS v7 mendukung template yang lebih rinci |
| Sertifikat Server | `/certificate add name=web-cert ...` | Identik dengan RouterOS v6 | Sintaks dasar identik |
| Let's Encrypt | Tidak tersedia bawaan, wajib impor manual | `/certificate enable-ssl-certificate dns-name=...` | RouterOS v7 memiliki fitur bawaan Let's Encrypt |
| Algoritma | RSA, DSA | RSA, ECDSA, Ed25519 | RouterOS v7 mendukung algoritma modern |
| Penerapan ke www-ssl | `/ip service set www-ssl certificate=...` | Identik dengan RouterOS v6 | Sintaks identik |
| Validasi | `/certificate print` | Identik, dengan informasi masa berlaku yang lebih jelas | RouterOS v7 menampilkan `invalid-after` lebih akurat |

#### Contoh Konfigurasi v6 — Kriptografi
```
# RouterOS v6 - CA dan Sertifikat Manual, Let's Encrypt wajib impor manual
/certificate add name=my-ca common-name="My Lab CA" key-usage=key-cert-sign,crl-sign key-size=2048 days-valid=3650
/certificate sign my-ca ca-crl-host=127.0.0.1 name=my-ca
/certificate add name=web-cert common-name="gw-jkt-01.lab.local" key-usage=digital-signature,key-encipherment,tls-server key-size=2048 days-valid=825
/certificate sign web-cert ca=my-ca name=web-cert
/certificate set my-ca trusted=yes
/ip service set www-ssl certificate=web-cert port=8443
/ip service disable www
# Untuk Let's Encrypt di RouterOS v6: pembuatan CSR, pengunduhan sertifikat dari CA eksternal, kemudian import
/certificate import file-name=cert.crt passphrase=""
```

#### Contoh Konfigurasi v7 — Kriptografi
```
# RouterOS v7 - CA dan Sertifikat, Let's Encrypt bawaan
/certificate add name=my-ca common-name="My Lab CA" key-usage=key-cert-sign,crl-sign key-size=2048 days-valid=3650
/certificate sign my-ca ca-crl-host=127.0.0.1 name=my-ca
/certificate add name=web-cert common-name="gw-jkt-01.lab.local" key-usage=digital-signature,key-encipherment,tls-server key-size=2048 days-valid=825
/certificate sign web-cert ca=my-ca name=web-cert
/certificate set my-ca trusted=yes
/ip service set www-ssl certificate=web-cert port=8443
/ip service disable www
# Let's Encrypt bawaan RouterOS v7 - tidak diperlukan impor manual
/certificate enable-ssl-certificate dns-name=gw.jkt-kantor.com
/ip service set www-ssl certificate=gw.jkt-kantor.com
```

> Catatan: Apabila RouterOS v6 digunakan pada lingkungan produksi, perpanjangan sertifikat wajib dijadwalkan secara manual. Pada RouterOS v7, perpanjangan Let's Encrypt dapat dilakukan melalui perintah yang sama.

## 5. Modul 5 - Pengamanan Router

**Tujuan:** Mengamankan router itu sendiri: port knocking, koneksi aman, port bawaan, dan `tunneling` melalui SSH.

### Studi Kasus
Port WinBox 8291 dipindai secara terus-menerus dan log dipenuhi brute-force. Administrator memerlukan akses jarak jauh, tetapi tidak menghendaki pembukaan port ke Internet.

### Port Knocking
```
# Urutan 1001, kemudian 1002, kemudian 1003 akan memberikan izin akses WinBox selama 30 detik
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1001 address-list=KNOCK1 address-list-timeout=10s comment="Knock 1"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1002 src-address-list=KNOCK1 address-list=KNOCK2 address-list-timeout=10s comment="Knock 2"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1003 src-address-list=KNOCK2 address-list=KNOCK3 address-list-timeout=30s comment="Knock 3"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299 src-address-list=KNOCK3 comment="WinBox hanya setelah knock"
/ip firewall filter add chain=input action=drop protocol=tcp dst-port=8299 comment="Menolak WinBox tanpa knock"
# Prosedur pengetukan port adalah sebagai berikut: for p in 1001 1002 1003; do nc -z -w1 GW-IP $p; done
```

### Koneksi Aman
```
/ip service disable www
/ip service set www-ssl port=8443 certificate=web-cert
/ip service set ssh port=2222 address=192.168.88.0/24
/ip ssh set strong-crypto=yes forwarding-enabled=remote
/tool mac-server mac-winbox set allowed-interface-list=none
/ip service set winbox port=8299 address=192.168.88.0/24
```

### `Tunneling` Melalui SSH
```
# WinBox diterowongkan secara lokal 8291 ke router 127.0.0.1:8299
ssh -L 8291:127.0.0.1:8299 netadmin@IP-PUBLIK-ROUTER -p 2222
# Selanjutnya, WinBox dibuka ke 127.0.0.1:8291
# SOCKS proxy:
ssh -D 1080 netadmin@IP-PUBLIK-ROUTER -p 2222
```

### Perbandingan RouterOS v6 dan v7 — Pengamanan Router

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| Port Knocking | Firewall filter dengan `add-src-to-address-list` | Identik dengan RouterOS v6 | Sintaks identik |
| SSH | `/ip ssh set strong-crypto=yes forwarding-enabled=...` | `/ip ssh set strong-crypto=yes forwarding-enabled=... host-key-size=...` | RouterOS v7 menambahkan peng`rule` ukuran host key |
| WinBox | WinBox 3 | WinBox 4 | RouterOS v7 memiliki antarmuka baru, port tetap sama |
| MAC WinBox | `/tool mac-server mac-winbox set allowed-interface-list=...` | Identik dengan RouterOS v6 | Sintaks identik |
| HTTPS | `/ip service set www-ssl certificate=...` | Identik dengan RouterOS v6 | Sintaks identik |
| SSH Tunnel | `ssh -L ...` dari klien | Identik dengan RouterOS v6 | Tidak bergantung pada versi RouterOS |

#### Contoh Konfigurasi v6 — Pengamanan Router
```
# RouterOS v6
/ip service disable www
/ip service set www-ssl port=8443 certificate=web-cert
/ip service set ssh port=2222 address=192.168.88.0/24
/ip ssh set strong-crypto=yes forwarding-enabled=remote
/tool mac-server mac-winbox set allowed-interface-list=none
# Port Knocking - identik dengan RouterOS v7
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1001 address-list=KNOCK1 address-list-timeout=10s comment="Knock 1"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1002 src-address-list=KNOCK1 address-list=KNOCK2 address-list-timeout=10s comment="Knock 2"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1003 src-address-list=KNOCK2 address-list=KNOCK3 address-list-timeout=30s comment="Knock 3"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299 src-address-list=KNOCK3 comment="WinBox hanya setelah knock"
```

#### Contoh Konfigurasi v7 — Pengamanan Router
```
# RouterOS v7 - tambahan host-key-size
/ip service disable www
/ip service set www-ssl port=8443 certificate=web-cert
/ip service set ssh port=2222 address=192.168.88.0/24
/ip ssh set strong-crypto=yes forwarding-enabled=remote host-key-size=4096
/tool mac-server mac-winbox set allowed-interface-list=none
# Port Knocking - identik dengan RouterOS v6
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1001 address-list=KNOCK1 address-list-timeout=10s comment="Knock 1"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1002 src-address-list=KNOCK1 address-list=KNOCK2 address-list-timeout=10s comment="Knock 2"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1003 src-address-list=KNOCK2 address-list=KNOCK3 address-list-timeout=30s comment="Knock 3"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299 src-address-list=KNOCK3 comment="WinBox hanya setelah knock"
# SSH Tunnel dari klien - identik untuk RouterOS v6 dan RouterOS v7
# ssh -L 8291:127.0.0.1:8299 netadmin@IP-PUBLIK-ROUTER -p 2222
```

> Catatan: Port knocking dan `tunnel` SSH tidak bergantung pada versi RouterOS, sehingga konfigurasi dapat digunakan pada kedua versi tanpa perubahan.

## 6. Modul 6 - Tunnel

**Tujuan:** Membangun tunnel : IPsec Site-to-Site, L2TP+IPsec, dan SSTP dengan sertifikat.

| Kebutuhan | Pilihan | Alasan |
|---|---|---|
| Kantor Pusat - Cabang, selalu terhubung | IPsec Site-to-Site | Stabil dan routing antar LAN otomatis |
| Karyawan WFH / mobile | L2TP + IPsec | Didukung bawaan Windows / Android / iOS |
| Vendor / hotel / jaringan yang memblokir VPN | SSTP (port 443) | Berjalan pada port HTTPS dan menembus firewall ketat |

### Studi Kasus
Kantor pusat di Jakarta (192.168.88.0/24) dan cabang di Surabaya (192.168.99.0/24) perlu saling mengakses server tanpa melalui Internet terbuka. Karyawan WFH perlu mengakses file server. Vendor perlu mengakses dari hotel yang memblokir VPN biasa.

### A. IPsec Site-to-Site
```
# Di JAKARTA
/ip firewall filter add chain=input action=accept protocol=udp dst-port=500,4500 comment="IPsec IKE"
/ip firewall filter add chain=input action=accept protocol=ipsec-esp comment="IPsec ESP"
/ip firewall filter add chain=input action=accept protocol=ipsec-ah comment="IPsec AH"
/ip ipsec profile add name=ike2-profile hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048
/ip ipsec proposal add name=ike2-proposal auth-algorithms=sha256 enc-algorithms=aes-256-cbc lifetime=8h pfs-group=modp2048
/ip ipsec peer add name=peer-sby address=203.0.113.10/32 exchange-mode=ike2 profile=ike2-profile
/ip ipsec identity add peer=peer-sby auth-method=pre-shared-key secret=PSK-Kuat-Banget-2026! generate-policy=port-strict
/ip ipsec policy add src-address=192.168.88.0/24 dst-address=192.168.99.0/24 tunnel=yes peer=peer-sby proposal=ike2-proposal action=encrypt
# Di SURABAYA, mirror: src 192.168.99.0/24 dst 192.168.88.0/24, peer address = IP Publik Jakarta
# Verifikasi
/ip ipsec active-peers print
/ip ipsec installed-sa print
/ping 192.168.99.1 src-address=192.168.88.1
```

### B. L2TP + IPsec
```
/interface l2tp-server server set enabled=yes use-ipsec=yes ipsec-secret=PSK-L2TP-Kuat-2026! authentication=mschap2 default-profile=default-encryption
/ip pool add name=vpn-pool ranges=10.10.10.10-10.10.10.50
/ppp profile add name=vpn-profile local-address=10.10.10.1 remote-address=vpn-pool dns-server=192.168.88.1 use-encryption=yes
/ppp secret add name=andi service=l2tp password=PasswordVPN-Andi-2026! profile=vpn-profile
/ip firewall filter add chain=input action=accept protocol=udp dst-port=1701 in-interface-list=WAN comment="L2TP"
/ppp active print
```

### C. SSTP dengan Sertifikat
```
/interface sstp-server server set enabled=yes port=443 certificate=gw.jkt-kantor.com authentication=mschap2 default-profile=vpn-profile
/ppp secret add name=vendor1 service=sstp password=PasswordVendor-2026! profile=vpn-profile
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=443 comment="SSTP / HTTPS"
/interface sstp-server print
/ppp active print
```

### Pemecahan Masalah `Tunnel`
- **IPsec tidak terhubung:** Administrator wajib memeriksa waktu (NTP), kesamaan PSK, firewall 500/4500 UDP, dan log melalui perintah `/log print where topics~ipsec`.
- **L2TP terhubung tetapi tidak dapat ping LAN:** Administrator wajib memeriksa firewall forward dan NAT. Lalu lintas VPN ke LAN tidak boleh dilakukan NAT.
- **SSTP gagal:** Administrator wajib memeriksa masa berlaku sertifikat, kesesuaian hostname dengan CN, dan port 443.
- **`Tunnel` terputus-putus:** Administrator wajib memeriksa MTU / MSS. Perintah berikut dapat ditambahkan: `/ip firewall mangle add chain=forward protocol=tcp tcp-flags=syn action=change-mss new-mss=1360`

```
/tool netwatch add host=192.168.99.1 interval=30s up-script=":log info \"Tunnel SBY UP\"" down-script=":log warning \"Tunnel SBY DOWN\""
```


### Perbandingan RouterOS v6 dan v7 — Tunnel

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| IPsec Profile | Menggunakan `/ip ipsec peer` dengan proposal di dalam peer | Menggunakan `/ip ipsec profile` terpisah, kemudian `/ip ipsec peer` dan `/ip ipsec identity` | RouterOS v7 memisahkan profile, peer, dan identity sehingga lebih modular |
| IPsec Identity | Tergabung dalam peer (`auth-method`, `secret`) | Terpisah: `/ip ipsec identity add peer=... auth-method=...` | Migrasi dari RouterOS v6 ke RouterOS v7 memerlukan pemisahan identity |
| IKE | IKEv1 utama, IKEv2 terbatas | IKEv1 dan IKEv2 penuh, dengan `exchange-mode=ike2` | RouterOS v7 lebih stabil untuk IKEv2 |
| L2TP + IPsec | `/interface l2tp-server server set use-ipsec=yes ipsec-secret=...` | Identik dengan RouterOS v6 | Sintaks identik |
| SSTP | `/interface sstp-server server set certificate=...` | Identik, dengan dukungan sertifikat yang lebih baik | Sintaks dasar identik |
| WireGuard | Tidak tersedia bawaan | Tersedia bawaan `/interface wireguard` | RouterOS v7 menambahkan WireGuard sebagai alternatif modern |
| Algoritma IPsec | AES-CBC, SHA1/SHA256 | AES-CBC, AES-GCM, SHA256/SHA512, ChaCha20 | RouterOS v7 mendukung algoritma yang lebih modern |

#### Contoh Konfigurasi v6 — IPsec Site-to-Site
```
# RouterOS v6 - Peer dan Identity tergabung
/ip firewall filter add chain=input action=accept protocol=udp dst-port=500,4500 comment="IPsec IKE"
/ip firewall filter add chain=input action=accept protocol=ipsec-esp comment="IPsec ESP"
/ip ipsec proposal add name=ike2-proposal auth-algorithms=sha256 enc-algorithms=aes-256-cbc lifetime=8h pfs-group=modp2048
# Pada RouterOS v6, peer memuat secret dan proposal secara langsung
/ip ipsec peer add name=peer-sby address=203.0.113.10/32 auth-method=pre-shared-key secret=PSK-Kuat-Banget-2026! generate-policy=port-strict exchange-mode=main proposal-check=obey hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048
/ip ipsec policy add src-address=192.168.88.0/24 dst-address=192.168.99.0/24 tunnel=yes peer=peer-sby proposal=ike2-proposal action=encrypt
/ip ipsec active-peers print
```

#### Contoh Konfigurasi v7 — IPsec Site-to-Site
```
# RouterOS v7 - Profile, Peer, dan Identity terpisah
/ip firewall filter add chain=input action=accept protocol=udp dst-port=500,4500 comment="IPsec IKE"
/ip firewall filter add chain=input action=accept protocol=ipsec-esp comment="IPsec ESP"
/ip ipsec profile add name=ike2-profile hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048
/ip ipsec proposal add name=ike2-proposal auth-algorithms=sha256 enc-algorithms=aes-256-cbc lifetime=8h pfs-group=modp2048
/ip ipsec peer add name=peer-sby address=203.0.113.10/32 exchange-mode=ike2 profile=ike2-profile
/ip ipsec identity add peer=peer-sby auth-method=pre-shared-key secret=PSK-Kuat-Banget-2026! generate-policy=port-strict
/ip ipsec policy add src-address=192.168.88.0/24 dst-address=192.168.99.0/24 tunnel=yes peer=peer-sby proposal=ike2-proposal action=encrypt
/ip ipsec active-peers print
```

#### Contoh Konfigurasi v6 dan v7 — L2TP + IPsec dan SSTP
```
# L2TP + IPsec - Sintaks identik pada RouterOS v6 dan RouterOS v7
/interface l2tp-server server set enabled=yes use-ipsec=yes ipsec-secret=PSK-L2TP-Kuat-2026! authentication=mschap2 default-profile=default-encryption
/ip pool add name=vpn-pool ranges=10.10.10.10-10.10.10.50
/ppp profile add name=vpn-profile local-address=10.10.10.1 remote-address=vpn-pool dns-server=192.168.88.1 use-encryption=yes
/ppp secret add name=andi service=l2tp password=PasswordVPN-Andi-2026! profile=vpn-profile
/ip firewall filter add chain=input action=accept protocol=udp dst-port=1701 in-interface-list=WAN comment="L2TP"

# SSTP - Sintaks identik pada RouterOS v6 dan RouterOS v7, RouterOS v7 memiliki validasi sertifikat yang lebih baik
/interface sstp-server server set enabled=yes port=443 certificate=gw.jkt-kantor.com authentication=mschap2 default-profile=vpn-profile
/ppp secret add name=vendor1 service=sstp password=PasswordVendor-2026! profile=vpn-profile
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=443 comment="SSTP / HTTPS"
```

> Catatan: Perbedaan paling signifikan pada modul ini adalah struktur IPsec. Apabila dilakukan migrasi dari RouterOS v6 ke RouterOS v7, konfigurasi peer pada RouterOS v6 wajib dipisahkan menjadi profile, peer, dan identity pada RouterOS v7. Untuk L2TP dan SSTP, konfigurasi dapat digunakan kembali tanpa perubahan besar.

## 7. Prosedur Operasional

### Harian (10 Menit)
- Administrator wajib memeriksa `/log print` untuk kegagalan login, `blocklist` baru, dan `tunnel` yang terputus.
- Administrator wajib memeriksa `/system resource print`. CPU wajib berada di bawah 70 persen dan memori wajib dalam kondisi memadai.
- Administrator wajib memeriksa `/ip firewall address-list print` untuk memastikan `blocklist` tidak membengkak secara tidak wajar.
- Administrator wajib memeriksa `/ppp active print` dan `/ip ipsec active-peers print`.

### Mingguan
- Administrator wajib memeriksa `backup` melalui perintah `/export file=backup-mingguan` dan menyimpannya di luar router.
- Administrator wajib memeriksa pembaruan melalui perintah `/system package update check-for-updates`. Pengujian pembaruan wajib dilakukan di laboratorium sebelum diterapkan pada lingkungan produksi.
- Administrator wajib memeriksa sertifikat. Sertifikat dengan masa berlaku kurang dari 30 hari wajib diperpanjang.
- Administrator wajib memeriksa `user`. `User` lama atau mantan karyawan wajib dinonaktifkan apabila diperlukan.

### Bulanan
- Administrator wajib melakukan audit firewall, audit port melalui perintah `/ip service print`, penggantian `password` / PSK VPN yang dibagikan, dan pengujian restore `backup` di laboratorium.

### Template `Backup`
```
/export file=backup-2026-11-24-sebelum-ubah-firewall
/system backup save name=backup-2026-11-24-sebelum-ubah-firewall
```


## 8. Panduan Penanganan Insiden

1. **Deteksi:** Administrator wajib mencatat waktu, alamat IP sumber, dan jenis serangan.
2. **Isolasi:** Administrator wajib memblokir alamat IP pelaku melalui perintah: `/ip firewall address-list add list=BLACKLIST address=IP-PELAKU`
3. **Eradikasi:** Administrator wajib menutup celah keamanan dengan cara mengganti `password` yang bocor, menonaktifkan layanan yang dieksploitasi, menghapus `user` atau skrip asing, dan memperbarui firmware.
4. **Pemulihan:** Administrator wajib memulihkan dari `backup` yang bersih apabila diperlukan, mengaktifkan layanan satu per satu sambil memantau log.
5. **Pasca-insiden:** Administrator wajib menulis laporan yang memuat kronologi, akar masalah, dan tindakan perbaikan.

> Catatan: Log tidak boleh dihapus sebelum penyelidikan selesai. Log merupakan bukti.


## 9. Pemecahan Masalah

| Gejala | Perintah Pemeriksaan |
|---|---|
| Internet mati total | `/interface print`, `/ip dhcp-client print`, `/ip dns print`, `/ip firewall nat print` |
| WinBox tidak dapat login | `/ip firewall address-list print`, `/ip service print`, `/log print` |
| DHCP tidak membagikan IP | `/ip dhcp-server print`, `/ip pool print`, `/ip dhcp-server lease print` |
| CPU 100% | `/tool profile`, `/ip firewall connection print`, `/tool torch` |
| VPN tidak terhubung | `/system clock print`, `/log print where topics~ipsec` |
| Sertifikat error | `/certificate print`, `/system clock print` |

---

## Lampiran A — Daftar Port Penting
```
WinBox 8291 (bawaan) -> ubah ke 8299 | SSH 22 -> 2222 | Telnet 23 -> NONAKTIFKAN
FTP 21 -> NONAKTIFKAN | Web HTTP 80 -> NONAKTIFKAN | Web HTTPS 443 / 8443 -> gunakan sertifikat
DNS 53 UDP/TCP | DHCP 67/68 UDP | NTP 123 UDP
IPsec IKE 500 UDP, NAT-T 4500 UDP, ESP (IP) | L2TP 1701 UDP | SSTP 443 TCP
```

## Lampiran B — Address-List
```
MGMT, BOGON, BLACKLIST, SYN-FLOOD, SSH-BRUTE, PORT-SCANNER, KNOCK1/2/3
```
