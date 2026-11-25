# MTCSE — Panduan Implementasi dan Operasi

## 1. Konfigurasi Dasar — Wajib Sebelum Seluruh Modul

### Perbandingan RouterOS v6 dan v7 — Konfigurasi Dasar

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| NTP Client | `/system ntp client set enabled=yes primary-ntp=103.10.2.1 secondary-ntp=103.10.2.2` | `/system ntp client set enabled=yes servers=pool.ntp.org` | v7 mendukung daftar server dan sinkronisasi yang lebih baik |
| Manajemen Pengguna | `/user add name=... group=full` | `/user add name=... group=full` | Sintaks sama, v7 menambahkan kebijakan kata sandi yang lebih ketat |
| Layanan | `/ip service set ssh port=...` | `/ip service set ssh port=...` | Sintaks sama, v7 mendukung WinBox 4 |
| Discovery | `/ip neighbor discovery-settings set discover-interface-list=...` | Sama | Sintaks sama |
| MAC Server | `/tool mac-server set allowed-interface-list=...` | Sama | Sintaks sama |

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

## 2. Modul 2 — Firewall

**Tujuan:** Menguasai alur paket, firewall stateful, tabel RAW, mitigasi SYN flood, praktik terbaik akses manajemen, deteksi serangan terhadap layanan kritis, filter bridge, dan penyaringan ICMP.

### Konsep
- **Input** merupakan paket yang ditujukan kepada router itu sendiri. **Forward** merupakan paket yang melewati router. **Output** merupakan paket yang berasal dari router.
- **Stateful firewall** mengingat status koneksi. Paket dengan status `established,related` merupakan balasan yang sah. Paket dengan status `invalid` sebaiknya ditolak.
- **Tabel RAW** diproses sebelum pelacakan koneksi. Tabel ini sesuai untuk membuang paket berbahaya sejak awal agar CPU tidak terbebani pencatatan koneksi palsu.

### Studi Kasus
Sebuah server di LAN tiba-tiba menyebabkan CPU router mencapai 100%, internet melambat, dan log dipenuhi koneksi SYN. Setelah diperiksa, terdapat SYN flood sebesar 50.000 paket per detik dari internet. Firewall filter biasa masih mencatat seluruh koneksi, sehingga CPU tetap penuh. Solusinya adalah memindahkan mitigasi ke tabel RAW dan membatasi SYN baru per alamat IP.

### Contoh Konfigurasi — Firewall Filter
```
/interface list add name=WAN
/interface list add name=LAN
/interface list member add list=WAN interface=ether1
/interface list member add list=LAN interface=ether2
# --- INPUT ---
/ip firewall filter add chain=input action=accept connection-state=established,related,untracked comment="Terima balasan yang sah"
/ip firewall filter add chain=input action=accept protocol=icmp limit=5,5:packet comment="ICMP terbatas"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299,2222 src-address-list=MGMT in-interface-list=LAN comment="Manajemen dari MGMT"
/ip firewall filter add chain=input action=drop connection-state=invalid comment="Tolak paket invalid"
/ip firewall filter add chain=input action=drop in-interface-list=WAN comment="Tolak sisanya dari WAN" log=yes log-prefix="DROP-INPUT-WAN"
/ip firewall filter add chain=input action=drop comment="Tolak sisanya ke router"
# --- FORWARD ---
/ip firewall filter add chain=forward action=accept connection-state=established,related,untracked comment="Terima balasan"
/ip firewall filter add chain=forward action=accept in-interface-list=LAN out-interface-list=WAN comment="LAN boleh ke internet"
/ip firewall filter add chain=forward action=drop connection-state=invalid comment="Tolak invalid forward"
/ip firewall filter add chain=forward action=drop in-interface-list=WAN comment="Tolak WAN ke LAN kecuali port-forward"
/ip firewall filter add chain=forward action=drop comment="Tolak sisanya forward"
```

### Contoh Konfigurasi — RAW dan SYN Flood
```
/ip firewall raw add chain=prerouting action=drop src-address-list=BOGON comment="Tolak IP bogon"
/ip firewall raw add chain=prerouting action=drop protocol=tcp tcp-flags=syn src-address-list=SYN-FLOOD comment="Tolak SYN dari pelaku flood"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp tcp-flags=syn connection-state=new limit=100,5:packet address-list=SYN-FLOOD address-list-timeout=1m comment="Deteksi SYN Flood" log=yes log-prefix="SYN-FLOOD"
```

### Contoh Konfigurasi — DNS dan ICMP
```
/ip firewall filter add chain=input action=accept protocol=udp dst-port=53 in-interface-list=LAN comment="DNS dari LAN"
/ip firewall filter add chain=input action=drop protocol=udp dst-port=53 in-interface-list=WAN comment="Tolak DNS dari WAN"
/interface bridge filter add chain=forward action=drop mac-protocol=ip ip-protocol=udp dst-port=67 in-interface=ether4 comment="Blokir DHCP liar dari ether4"
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

> Catatan: Urutan rule sangat penting. Rule dibaca dari atas ke bawah, dan rule pertama yang cocok akan diterapkan. Letakkan rule accept established,related pada posisi paling atas, dilanjutkan rule spesifik, kemudian rule penolakan pada bagian bawah.


### Perbandingan RouterOS v6 dan v7 — Firewall

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| Firewall Filter | `/ip firewall filter add chain=input ...` | Sama | Sintaks filter identik |
| Tabel RAW | `/ip firewall raw add chain=prerouting ...` | Sama | Sintaks RAW identik |
| Connection Tracking | `/ip firewall connection tracking set enabled=yes tcp-syncookie=yes` | `/ip firewall connection tracking set enabled=yes loose-tcp-tracking=yes` | v7 menambahkan opsi loose-tcp-tracking |
| Daftar Alamat | `/ip firewall address-list add list=...` | Sama | Sintaks sama |
| IPv6 Firewall | `/ipv6 firewall filter ...` | `/ipv6 firewall filter ...` | v7 memiliki penanganan IPv6 yang lebih optimal |
| Bridge Filter | `/interface bridge filter ...` | Sama, dengan peningkatan VLAN filtering | v7 lebih stabil untuk bridge VLAN |

#### Contoh Konfigurasi v6 — Firewall
```
# RouterOS v6 - Connection Tracking
/ip firewall connection tracking set enabled=yes tcp-syncookie=yes tcp-established-timeout=1d

# Filter Input - sintaks sama dengan v7
/ip firewall filter add chain=input action=accept connection-state=established,related,untracked comment="Terima balasan yang sah"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299,2222 src-address-list=MGMT in-interface-list=LAN comment="Manajemen dari MGMT"
/ip firewall filter add chain=input action=drop connection-state=invalid comment="Tolak paket invalid"

# RAW - SYN Flood
/ip firewall raw add chain=prerouting action=drop protocol=tcp tcp-flags=syn src-address-list=SYN-FLOOD comment="Tolak SYN flood"
```

#### Contoh Konfigurasi v7 — Firewall
```
# RouterOS v7 - Connection Tracking dengan loose-tcp-tracking
/ip firewall connection tracking set enabled=yes loose-tcp-tracking=yes tcp-established-timeout=1d

# Filter Input - sintaks sama dengan v6
/ip firewall filter add chain=input action=accept connection-state=established,related,untracked comment="Terima balasan yang sah"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299,2222 src-address-list=MGMT in-interface-list=LAN comment="Manajemen dari MGMT"
/ip firewall filter add chain=input action=drop connection-state=invalid comment="Tolak paket invalid"

# RAW - SYN Flood
/ip firewall raw add chain=prerouting action=drop protocol=tcp tcp-flags=syn src-address-list=SYN-FLOOD comment="Tolak SYN flood"
```

> Catatan: Untuk firewall, sebagian besar rule dapat disalin langsung dari v6 ke v7. Perbedaan utama hanya pada pengaturan connection tracking dan kinerja IPv6.

## 3. Modul 3 — Serangan pada Lapisan OSI

**Tujuan:** Mengenali dan mencegah serangan per protokol: MNDP, DHCP, TCP, UDP, ICMP, brute-force, dan port scan.

| Serangan | Mitigasi Inti |
|---|---|
| MNDP | Batasi discovery hanya ke antarmuka tepercaya |
| DHCP Rogue | DHCP Alert dan pembatasan port |
| DHCP Starvation | Batas lease, static lease untuk perangkat penting, dan alert |
| TCP SYN Flood | RAW, limit, dan daftar blokir |
| UDP Flood | RAW drop dan limit |
| ICMP Smurf | Tolak directed broadcast dan batasi ICMP |
| Brute-force | Nonaktifkan FTP/Telnet, batasi SSH, dan daftar blokir |
| Port Scan | Deteksi PSD dan daftar blokir pemindai |

### Studi Kasus 1 — DHCP Rogue
Seluruh klien tiba-tiba tidak dapat mengakses internet. Klien memperoleh alamat IP 192.168.1.x dengan gateway 192.168.1.99, padahal gateway resmi adalah 192.168.88.1. Setelah diperiksa, terdapat karyawan yang menghubungkan router rumahan ke LAN.

### Studi Kasus 2 — Brute-Force SSH
Log dipenuhi kegagalan login untuk pengguna admin, root, dan test dari alamat IP luar negeri.

### Contoh Konfigurasi
```
# MNDP
/ip neighbor discovery-settings set discover-interface-list=LAN
# DHCP Rogue Alert - ganti MAC dengan MAC server resmi
/ip dhcp-server alert add interface=ether2 on-alert=":log warning \"ROGUE DHCP DETECTED\"" valid-server=AA:BB:CC:DD:EE:FF
# DHCP Starvation
/ip dhcp-server set [find] lease-time=1h
/ip dhcp-server lease add address=192.168.88.10 mac-address=AA:BB:CC:DD:EE:10 server=dhcp1 comment="Server Utama"
```

```
# Brute-force SSH: 3x gagal dalam 1 menit -> daftar blokir 1 hari
/ip firewall filter add chain=input action=drop protocol=tcp dst-port=2222 src-address-list=SSH-BRUTE comment="Tolak SSH dari brute-forcer"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=2222 connection-state=new limit=3,1m address-list=SSH-BRUTE address-list-timeout=1d comment="Deteksi brute-force SSH" log=yes log-prefix="SSH-BRUTE"
/ip service disable ftp,telnet
# Port Scan
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp psd=21,3s,3,1 address-list=PORT-SCANNER address-list-timeout=1d comment="Deteksi Port Scan" log=yes log-prefix="PORT-SCAN"
/ip firewall filter add chain=input action=drop src-address-list=PORT-SCANNER comment="Tolak pemindai"
# UDP Flood dan Smurf
/ip firewall raw add chain=prerouting action=drop dst-address-type=broadcast comment="Tolak directed broadcast"
/ip firewall filter add chain=input action=accept protocol=icmp limit=5,5:packet comment="ICMP terbatas"
```

### Verifikasi
```
/ip dhcp-server lease print
/ip firewall address-list print where list=SSH-BRUTE
/ip firewall address-list print where list=PORT-SCANNER
/log print where message~"ROGUE" or message~"BRUTE"
```

---

### Perbandingan RouterOS v6 dan v7 — Serangan pada Lapisan OSI

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| DHCP Alert | `/ip dhcp-server alert add interface=... valid-server=...` | Sama | Sintaks sama |
| DHCP Server | `/ip dhcp-server set lease-time=...` | Sama, dengan manajemen lease yang lebih baik | v7 lebih stabil untuk pool besar |
| Brute-Force SSH | Firewall filter dengan `add-src-to-address-list` | Sama | Sintaks firewall identik |
| Port Scan (PSD) | `psd=21,3s,3,1` | Sama | Sintaks sama |
| MNDP / Neighbor | `/ip neighbor discovery-settings ...` | Sama | Sintaks sama |
| Log | `/system logging add topics=...` | Sama, dengan performa log yang lebih baik | v7 mendukung log yang lebih terstruktur |

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
# Tambahan v7: periksa lease dengan lebih cepat
/ip dhcp-server lease print where status=bound
```

> Catatan: Pada modul ini, konfigurasi v6 dan v7 dapat dipertukarkan. Perbedaan terletak pada kinerja DHCP dan pencatatan log, bukan pada sintaks.

## 4. Modul 4 — Kriptografi

**Tujuan:** Memahami enkripsi, algoritma simetris dan asimetris, PKI, dan sertifikat, kemudian menggunakannya pada RouterOS.

- **Simetris:** Menggunakan satu kunci untuk mengunci dan membuka. Cepat dan sesuai untuk data berukuran besar. Contoh: AES.
- **Asimetris:** Menggunakan dua kunci, yaitu kunci publik dan kunci privat. Contoh: RSA.
- **PKI:** Certificate Authority (CA) menandatangani sertifikat dan menjamin identitas yang tercantum di dalamnya.
- **Sertifikat:** Kartu identitas digital. Apabila masa berlaku telah berakhir atau CA tidak dipercaya, peramban atau VPN akan menolaknya.

### Studi Kasus
WebFig masih menggunakan HTTP. Administrator masuk dari WiFi kantor yang sama dengan tamu. Seseorang menjalankan sniffing dan menangkap kata sandi administrator dalam bentuk plaintext.

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

> Catatan: Simpan kunci privat CA di tempat yang aman. Buat pengingat 30 hari sebelum sertifikat kedaluwarsa.


### Perbandingan RouterOS v6 dan v7 — Kriptografi

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| Pembuatan CA | `/certificate add name=my-ca ...` kemudian `sign` | Sama, dengan opsi `key-usage` yang lebih lengkap | v7 mendukung template yang lebih rinci |
| Sertifikat Server | `/certificate add name=web-cert ...` | Sama | Sintaks dasar sama |
| Let's Encrypt | Tidak tersedia bawaan, harus impor manual | `/certificate enable-ssl-certificate dns-name=...` | v7 memiliki fitur bawaan Let's Encrypt |
| Algoritma | RSA, DSA | RSA, ECDSA, Ed25519 | v7 mendukung algoritma modern |
| Penerapan ke www-ssl | `/ip service set www-ssl certificate=...` | Sama | Sintaks sama |
| Validasi | `/certificate print` | Sama, dengan informasi masa berlaku yang lebih jelas | v7 menampilkan `invalid-after` lebih akurat |

#### Contoh Konfigurasi v6 — Kriptografi
```
# RouterOS v6 - CA dan Sertifikat Manual, Let's Encrypt harus impor manual
/certificate add name=my-ca common-name="My Lab CA" key-usage=key-cert-sign,crl-sign key-size=2048 days-valid=3650
/certificate sign my-ca ca-crl-host=127.0.0.1 name=my-ca
/certificate add name=web-cert common-name="gw-jkt-01.lab.local" key-usage=digital-signature,key-encipherment,tls-server key-size=2048 days-valid=825
/certificate sign web-cert ca=my-ca name=web-cert
/certificate set my-ca trusted=yes
/ip service set www-ssl certificate=web-cert port=8443
/ip service disable www
# Untuk Let's Encrypt di v6: buat CSR, unduh sertifikat dari CA eksternal, lalu import
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
# Let's Encrypt bawaan v7 - tidak perlu impor manual
/certificate enable-ssl-certificate dns-name=gw.jkt-kantor.com
/ip service set www-ssl certificate=gw.jkt-kantor.com
```

> Catatan: Apabila menggunakan v6 pada produksi, jadwalkan perpanjangan sertifikat secara manual. Pada v7, perpanjangan Let's Encrypt dapat dilakukan melalui perintah yang sama.

## 5. Modul 5 — Pengamanan Router

**Tujuan:** Mengamankan router itu sendiri: port knocking, koneksi aman, port bawaan, dan tunneling melalui SSH.

### Studi Kasus
Port WinBox 8291 dipindai secara terus-menerus dan log dipenuhi brute-force. Administrator memerlukan akses jarak jauh, tetapi tidak ingin membuka port ke internet.

### Port Knocking
```
# 1001 -> 1002 -> 1003 -> boleh WinBox 30 detik
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1001 address-list=KNOCK1 address-list-timeout=10s comment="Knock 1"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1002 src-address-list=KNOCK1 address-list=KNOCK2 address-list-timeout=10s comment="Knock 2"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1003 src-address-list=KNOCK2 address-list=KNOCK3 address-list-timeout=30s comment="Knock 3"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299 src-address-list=KNOCK3 comment="WinBox hanya setelah knock"
/ip firewall filter add chain=input action=drop protocol=tcp dst-port=8299 comment="Tolak WinBox tanpa knock"
# Cara mengetuk: for p in 1001 1002 1003; do nc -z -w1 GW-IP $p; done
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

### Tunneling Melalui SSH
```
# Terowongkan WinBox lokal 8291 -> router 127.0.0.1:8299
ssh -L 8291:127.0.0.1:8299 netadmin@IP-PUBLIK-ROUTER -p 2222
# Kemudian buka WinBox ke 127.0.0.1:8291
# SOCKS proxy:
ssh -D 1080 netadmin@IP-PUBLIK-ROUTER -p 2222
```

### Perbandingan RouterOS v6 dan v7 — Pengamanan Router

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| Port Knocking | Firewall filter dengan `add-src-to-address-list` | Sama | Sintaks identik |
| SSH | `/ip ssh set strong-crypto=yes forwarding-enabled=...` | `/ip ssh set strong-crypto=yes forwarding-enabled=... host-key-size=...` | v7 menambahkan pengaturan ukuran host key |
| WinBox | WinBox 3 | WinBox 4 | v7 memiliki antarmuka baru, port tetap sama |
| MAC WinBox | `/tool mac-server mac-winbox set allowed-interface-list=...` | Sama | Sintaks sama |
| HTTPS | `/ip service set www-ssl certificate=...` | Sama | Sintaks sama |
| SSH Tunnel | `ssh -L ...` dari klien | Sama | Tidak bergantung pada versi RouterOS |

#### Contoh Konfigurasi v6 — Pengamanan Router
```
# RouterOS v6
/ip service disable www
/ip service set www-ssl port=8443 certificate=web-cert
/ip service set ssh port=2222 address=192.168.88.0/24
/ip ssh set strong-crypto=yes forwarding-enabled=remote
/tool mac-server mac-winbox set allowed-interface-list=none
# Port Knocking - sama dengan v7
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
# Port Knocking - identik dengan v6
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1001 address-list=KNOCK1 address-list-timeout=10s comment="Knock 1"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1002 src-address-list=KNOCK1 address-list=KNOCK2 address-list-timeout=10s comment="Knock 2"
/ip firewall filter add chain=input action=add-src-to-address-list protocol=tcp dst-port=1003 src-address-list=KNOCK2 address-list=KNOCK3 address-list-timeout=30s comment="Knock 3"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8299 src-address-list=KNOCK3 comment="WinBox hanya setelah knock"
# SSH Tunnel dari klien - sama untuk v6 dan v7
# ssh -L 8291:127.0.0.1:8299 netadmin@IP-PUBLIK-ROUTER -p 2222
```

> Catatan: Port knocking dan SSH tunnel tidak bergantung pada versi RouterOS, sehingga konfigurasi dapat digunakan pada kedua versi tanpa perubahan.

## 6. Modul 6 — Tunnel Aman

**Tujuan:** Membangun Tunnel aman: IPsec Site-to-Site, L2TP+IPsec, dan SSTP dengan sertifikat.

| Kebutuhan | Pilihan | Alasan |
|---|---|---|
| Kantor Pusat - Cabang, selalu terhubung | IPsec Site-to-Site | Stabil dan routing antar LAN otomatis |
| Karyawan WFH / mobile | L2TP + IPsec | Didukung bawaan Windows / Android / iOS |
| Vendor / hotel / jaringan yang memblokir VPN | SSTP (port 443) | Berjalan pada port HTTPS dan menembus firewall ketat |

### Studi Kasus
Kantor pusat Jakarta (192.168.88.0/24) dan cabang Surabaya (192.168.99.0/24) perlu saling mengakses server tanpa melalui internet terbuka. Karyawan WFH perlu mengakses file server. Vendor perlu mengakses dari hotel yang memblokir VPN biasa.

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

### Pemecahan Masalah Tunnel
- **IPsec tidak terhubung:** Periksa waktu (NTP), periksa kesamaan PSK, periksa firewall 500/4500 UDP, dan periksa log `/log print where topics~ipsec`.
- **L2TP terhubung tetapi tidak dapat ping LAN:** Periksa firewall forward dan NAT — jangan melakukan NAT terhadap lalu lintas VPN ke LAN.
- **SSTP gagal:** Periksa masa berlaku sertifikat, periksa kesesuaian hostname dengan CN, dan periksa port 443.
- **Tunnel terputus-putus:** Periksa MTU / MSS. Tambahkan `/ip firewall mangle add chain=forward protocol=tcp tcp-flags=syn action=change-mss new-mss=1360`

```
/tool netwatch add host=192.168.99.1 interval=30s up-script=":log info \"Tunnel SBY UP\"" down-script=":log warning \"Tunnel SBY DOWN\""
```

### Perbandingan RouterOS v6 dan v7 — Tunnel Aman

| Aspek | RouterOS v6 | RouterOS v7 | Catatan |
|---|---|---|---|
| IPsec Profile | Menggunakan `/ip ipsec peer` dengan proposal di dalam peer | Menggunakan `/ip ipsec profile` terpisah, kemudian `/ip ipsec peer` dan `/ip ipsec identity` | v7 memisahkan profile, peer, dan identity sehingga lebih modular |
| IPsec Identity | Tergabung dalam peer (`auth-method`, `secret`) | Terpisah: `/ip ipsec identity add peer=... auth-method=...` | Migrasi v6 ke v7 memerlukan pemisahan identity |
| IKE | IKEv1 utama, IKEv2 terbatas | IKEv1 dan IKEv2 penuh, dengan `exchange-mode=ike2` | v7 lebih stabil untuk IKEv2 |
| L2TP + IPsec | `/interface l2tp-server server set use-ipsec=yes ipsec-secret=...` | Sama | Sintaks sama |
| SSTP | `/interface sstp-server server set certificate=...` | Sama, dengan dukungan sertifikat yang lebih baik | Sintaks dasar sama |
| WireGuard | Tidak tersedia bawaan | Tersedia bawaan `/interface wireguard` | v7 menambahkan WireGuard sebagai alternatif modern |
| Algoritma IPsec | AES-CBC, SHA1/SHA256 | AES-CBC, AES-GCM, SHA256/SHA512, ChaCha20 | v7 mendukung algoritma yang lebih modern |

#### Contoh Konfigurasi v6 — IPsec Site-to-Site
```
# RouterOS v6 - Peer dan Identity tergabung
/ip firewall filter add chain=input action=accept protocol=udp dst-port=500,4500 comment="IPsec IKE"
/ip firewall filter add chain=input action=accept protocol=ipsec-esp comment="IPsec ESP"
/ip ipsec proposal add name=ike2-proposal auth-algorithms=sha256 enc-algorithms=aes-256-cbc lifetime=8h pfs-group=modp2048
# Pada v6, peer langsung memuat secret dan proposal
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
# L2TP + IPsec - Sintaks sama pada v6 dan v7
/interface l2tp-server server set enabled=yes use-ipsec=yes ipsec-secret=PSK-L2TP-Kuat-2026! authentication=mschap2 default-profile=default-encryption
/ip pool add name=vpn-pool ranges=10.10.10.10-10.10.10.50
/ppp profile add name=vpn-profile local-address=10.10.10.1 remote-address=vpn-pool dns-server=192.168.88.1 use-encryption=yes
/ppp secret add name=andi service=l2tp password=PasswordVPN-Andi-2026! profile=vpn-profile
/ip firewall filter add chain=input action=accept protocol=udp dst-port=1701 in-interface-list=WAN comment="L2TP"

# SSTP - Sintaks sama pada v6 dan v7, v7 memiliki validasi sertifikat yang lebih baik
/interface sstp-server server set enabled=yes port=443 certificate=gw.jkt-kantor.com authentication=mschap2 default-profile=vpn-profile
/ppp secret add name=vendor1 service=sstp password=PasswordVendor-2026! profile=vpn-profile
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=443 comment="SSTP / HTTPS"
```

> Catatan: Perbedaan paling signifikan pada modul ini adalah struktur IPsec. Apabila melakukan migrasi dari v6 ke v7, pisahkan konfigurasi peer v6 menjadi profile, peer, dan identity pada v7. Untuk L2TP dan SSTP, konfigurasi dapat digunakan kembali tanpa perubahan besar.

### Template Pencadangan
```
/export file=backup-2026-11-24-sebelum-ubah-firewall
/system backup save name=backup-2026-11-24-sebelum-ubah-firewall
```

## 7. Panduan Penanganan Insiden

1. **Deteksi:** Catat waktu, alamat IP sumber, dan jenis serangan.
2. **Isolasi:** Blokir alamat IP pelaku: `/ip firewall address-list add list=BLACKLIST address=IP-PELAKU`
3. **Eradikasi:** Tutup celah — ganti kata sandi yang bocor, nonaktifkan layanan yang dieksploitasi, hapus pengguna atau skrip asing, dan perbarui firmware.
4. **Pemulihan:** Pulihkan dari pencadangan yang bersih apabila diperlukan, aktifkan layanan satu per satu sambil memantau log.
5. **Pasca-insiden:** Tulis laporan: kronologi, akar masalah, dan tindakan perbaikan.

> Catatan: Jangan menghapus log sebelum penyelidikan selesai. Log merupakan bukti.


## 8. Pemecahan Masalah

| Gejala | Perintah Pemeriksaan |
|---|---|
| Internet mati total | `/interface print`, `/ip dhcp-client print`, `/ip dns print`, `/ip firewall nat print` |
| WinBox tidak dapat login | `/ip firewall address-list print`, `/ip service print`, `/log print` |
| DHCP tidak membagikan IP | `/ip dhcp-server print`, `/ip pool print`, `/ip dhcp-server lease print` |
| CPU 100% | `/tool profile`, `/ip firewall connection print`, `/tool torch` |
| VPN tidak terhubung | `/system clock print`, `/log print where topics~ipsec` |
| Sertifikat error | `/certificate print`, `/system clock print` |


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
