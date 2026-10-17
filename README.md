# Kantor Kecil — Small Office Network Lab (Cisco Packet Tracer)

Simulasi jaringan kantor kecil berbasis VLAN segmentation menggunakan Cisco Packet Tracer.

## Deskripsi

Project ini mensimulasikan jaringan LAN kantor kecil dengan 3 VLAN (Staff, Admin/HR, Guest WiFi) di atas satu blok network `192.168.10.0/24` yang di-subnetting pakai VLSM. Inter-VLAN routing ditangani oleh SVI di Multilayer Switch. Untuk menjaga desain tetap simpel dan gampang di-debug, **DHCP hanya dipakai di VLAN Guest WiFi** — VLAN internal (Staff, Admin/HR) pakai static IP.

## Topologi

- **R1 (Cisco 2911)** — router uplink/gateway keluar jaringan kantor, terhubung ke Multilayer Switch lewat link point-to-point `192.168.120.0/30`
- **Multilayer Switch (3560-24PS)** — inter-VLAN routing via SVI, trunk ke dua access switch
- **2x Access Switch (2960-24TT)** — menghubungkan end device per VLAN ke Multilayer Switch lewat trunk
- **Access Point** — melayani VLAN 30 Guest WiFi
- **Server** — berada di segmen Guest WiFi

## VLAN & Subnetting (VLSM dari 192.168.10.0/24)

| VLAN | Nama | Subnet | Mask | Range Usable | Broadcast |
| --- | --- | --- | --- | --- | --- |
| 20 | STAFF | 192.168.10.0/26 | 255.255.255.192 | .1 – .62 | .63 |
| 30 | GUEST WIFI | 192.168.10.64/27 | 255.255.255.224 | .65 – .94 | .95 |
| 10 | ADMIN/HR | 192.168.10.96/28 | 255.255.255.240 | .97 – .110 | .111 |

## Device & IP Assignment

### VLAN 20 — STAFF (Static IP)

| Device | IP |
| --- | --- |
| PC0 | 192.168.10.2 |
| PC1 | 192.168.10.3 |
| PC2 | 192.168.10.4 |
| PC3 | 192.168.10.5 |
| Printer0 | static, sesuai kebutuhan |
| Gateway (SVI Vlan20) | 192.168.10.1 (sesuaikan dengan konfigurasi aktual) |

### VLAN 10 — ADMIN/HR (Static IP)

| Device | IP |
| --- | --- |
| PC5 | 192.168.10.97 |
| PC4 | 192.168.10.98 |
| PC6 | 192.168.10.100 |
| PC7 | 192.168.10.101 |
| Printer1 | 192.168.10.102 |
| PC11, PC12, PC13, Printer2 | lanjutkan sequence .103–.109 (belum tercantum di diagram) |
| Gateway (SVI Vlan10) | 192.168.10.96 + 1 (sesuaikan dengan konfigurasi aktual) |

### VLAN 30 — GUEST WIFI (DHCP)

Smartphone0–3 mendapat IP otomatis dari DHCP pool di bawah. Server di segmen ini disarankan pakai static IP dalam excluded range.

```
ip dhcp excluded-address 192.168.10.65 192.168.10.70

ip dhcp pool GUEST-WIFI
 network 192.168.10.64 255.255.255.224
 default-router 192.168.10.65
 dns-server 8.8.8.8
```

## Keputusan Desain Utama

- **DHCP hanya di Guest WiFi** — jumlah device tamu (smartphone) gak menentu, jadi lebih praktis otomatis. VLAN Staff & Admin/HR jumlah host sedikit dan stabil, jadi static IP lebih gampang di-debug dan menghindari resiko rogue DHCP server.
- **Inter-VLAN routing via SVI** di Multilayer Switch, bukan ROAS, karena switch yang dipakai sudah mendukung Layer 3 (`ip routing`).
- **R1 sebagai gateway/uplink** keluar jaringan kantor, terhubung ke Multilayer Switch lewat link point-to-point terpisah (`192.168.120.0/30`).
- Dua access switch (2960) menghubungkan end device ke Multilayer Switch lewat trunk, supaya VLAN yang sama (mis. Admin/HR) bisa tersebar di lebih dari satu switch fisik.

## Konfigurasi Kunci (contoh)

### VLAN & Trunk (Multilayer Switch)

```
vlan 10
 name ADMIN-HR
vlan 20
 name STAFF
vlan 30
 name GUEST-WIFI

interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

### SVI (Multilayer Switch)

```
ip routing

interface Vlan20
 ip address 192.168.10.1 255.255.255.192
 no shutdown

interface Vlan30
 ip address 192.168.10.65 255.255.255.224
 no shutdown

interface Vlan10
 ip address 192.168.10.97 255.255.255.240
 no shutdown
```

> Catatan: sesuaikan IP gateway di atas dengan yang benar-benar dipakai di topologi kamu — nilai di sini cuma contoh dari ujung awal range usable tiap subnet.

## Cara Test

1. Ping antar VLAN dari PC — pastikan default gateway PC sesuai SVI VLAN-nya masing-masing.
2. Cek Smartphone0–3 di Guest WiFi dapat IP otomatis (buka Desktop → IP Configuration, mode DHCP).
3. `show ip interface brief` di Multilayer Switch — pastikan semua SVI status `up/up`.
4. `show interfaces trunk` di kedua access switch — pastikan VLAN 10/20/30 ter-allow dan native VLAN sama di kedua ujung trunk.
5. `show vlan brief` di tiap switch — pastikan port ke PC/printer sudah jadi access port di VLAN yang benar.
