# Laporan Tugas Modul 4 — DMZ & Firewall
**Praktikum Jaringan Komputer**
**Modul:** 4 — Konfigurasi Firewall FortiGate dengan Segmentasi WAN, LAN, dan DMZ  

---

## Daftar Isi
1. [Topologi Jaringan](#1-topologi-jaringan)
2. [Penjelasan Perangkat](#2-penjelasan-perangkat)
3. [Segmentasi Jaringan](#3-segmentasi-jaringan)
4. [Tabel IP Address](#4-tabel-ip-address)
5. [Konfigurasi Perangkat](#5-konfigurasi-perangkat)
   - [5.1 MikroTik ISP](#51-konfigurasi-mikrotik-isp)
   - [5.2 FortiGate Firewall](#52-konfigurasi-fortigate-firewall)
   - [5.3 Cisco Router](#53-konfigurasi-cisco-router)
   - [5.4 Client LAN](#54-konfigurasi-client-lan-tiny-core-linux)
   - [5.5 Client WAN](#55-konfigurasi-client-wan-tiny-core-linux)
   - [5.6 Ubuntu Server DMZ](#56-konfigurasi-ubuntu-server-dmz)
6. [Hasil Pengujian](#6-hasil-pengujian)
7. [Analisis dan Kesimpulan](#7-analisis-dan-kesimpulan)

---

## 1. Topologi Jaringan
<img width="1175" height="790" alt="WhatsApp Image 2026-06-07 at 09 36 31 (2)" src="https://github.com/user-attachments/assets/a87c1ea8-f4c3-496f-aa75-c64139a70cff" />

*Screenshot topologi di PNETLab — semua node sudah nyala dan terhubung*


## 2. Penjelasan Perangkat

| No. | Perangkat | Fungsi |
|-----|-----------|--------|
| 1 | **Cloud / Jaringan Lab** | Sumber internet asli dari jaringan lab. MikroTik konek ke sini untuk dapat akses internet |
| 2 | **MikroTik ISP** | Berperan jadi router ISP, tugasnya kasih koneksi internet ke jaringan simulasi kita |
| 3 | **FortiGate Firewall** | Yang paling penting. Dia yang jaga dan atur siapa boleh lewat ke mana — antara WAN, LAN, dan DMZ |
| 4 | **Cisco Router** | Router yang menghubungkan FortiGate ke jaringan LAN. Jadi "pintu masuk" ke jaringan internal |
| 5 | **Client LAN Tiny Core Linux** | Simulasi user yang ada di dalam jaringan internal |
| 6 | **Client WAN Tiny Core Linux** | Simulasi user dari luar/internet, buat ngetes apakah firewall beneran kerja |
| 7 | **Ubuntu Server DMZ** | Web server di zona DMZ. Bisa diakses dari luar tapi tetap ada batasan dari firewall |

---

## 3. Segmentasi Jaringan

| Segment | Network | Keterangan |
|---------|---------|------------|
| Jaringan Lab / Internet | DHCP dari lab | Sumber internet, MikroTik dapat IP dari sini |
| ISP ke FortiGate | `10.10.10.0/30` | Jalur antara MikroTik dan FortiGate |
| Client WAN | `172.16.100.0/24` | Jaringan tempat client luar berada |
| FortiGate ke Cisco | `10.20.20.0/30` | Jalur antara FortiGate dan Cisco Router |
| LAN | `192.168.10.0/24` | Jaringan internal (client dalam) |
| DMZ | `192.168.20.0/24` | Jaringan khusus server yang bisa diakses publik |

Kenapa harus dipisah? Kalau semua jadi satu jaringan, orang dari internet bisa langsung akses komputer internal — itu bahaya. Dengan dipisah pakai FortiGate, kita bisa atur siapa boleh akses apa:

- **WAN** = zona luar/internet, tidak dipercaya
- **LAN** = zona dalam, hanya orang internal yang boleh akses
- **DMZ** = zona tengah, server publik boleh diakses dari luar tapi kalau kena hack tidak bisa langsung masuk ke LAN

---

## 4. Tabel IP Address

| Perangkat | Interface | IP Address | Gateway | Keterangan |
|-----------|-----------|------------|---------|------------|
| MikroTik ISP | ether1 | DHCP Client | DHCP dari lab | Konek ke Cloud/internet lab |
| MikroTik ISP | ether2 | `10.10.10.1/30` | — | Ke FortiGate port1 |
| MikroTik ISP | ether3 | `172.16.100.1/24` | — | Gateway untuk Client WAN |
| FortiGate | port1 | `10.10.10.2/30` | `10.10.10.1` | Interface WAN |
| FortiGate | port2 | `10.20.20.1/30` | — | Interface ke Cisco Router |
| FortiGate | port3 | `192.168.20.1/24` | — | Interface DMZ |
| Cisco Router | G0/0 | `10.20.20.2/30` | — | Konek ke FortiGate |
| Cisco Router | G0/1 | `192.168.10.1/24` | — | Gateway LAN |
| Client LAN | eth0 | `192.168.10.10/24` | `192.168.10.1` | Client dalam |
| Client WAN | eth0 | `172.16.100.10/24` | `172.16.100.1` | Client luar |
| Ubuntu Server DMZ | eth0/ens3 | `192.168.20.10/24` | `192.168.20.1` | Web server |

---

## 5. Konfigurasi Perangkat

### 5.1 Konfigurasi MikroTik ISP

MikroTik di sini berperan jadi ISP. Kita set IP tiap interface, aktifkan DHCP client supaya dapat internet dari lab, pasang NAT, dan tambah route ke LAN dan DMZ.

#### Set IP tiap interface

```
/ip dhcp-client add interface=ether1 disabled=no
/ip address add address=10.10.10.1/30 interface=ether2 network=10.10.10.0
/ip address add address=172.16.100.1/24 interface=ether3 network=172.16.100.0
```

Verifikasi dengan perintah `ip address print`:

```
[admin@MikroTik] > ip address print
Flags: X - disabled, I - invalid, D - dynamic
 #   ADDRESS            NETWORK         INTERFACE
 0 D 192.168.100.x/24  192.168.100.0   ether1       <- IP dari DHCP lab
 1   10.10.10.1/30     10.10.10.0      ether2
 2   172.16.100.1/24   172.16.100.0    ether3
```


<img width="840" height="454" alt="ip addres print" src="https://github.com/user-attachments/assets/0d2f8e2c-060a-4921-94f9-869c18ed57d4" />

*Hasil `ip address print` di MikroTik — IP tiap interface sudah terpasang*

#### Aktifkan NAT Masquerade

```
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

NAT masquerade fungsinya supaya traffic dari jaringan di belakang MikroTik bisa keluar ke internet. IP asli dari dalam "disembunyiin" dan diganti pakai IP yang dapat dari lab.

Verifikasi dengan `ip firewall nat print`:

```
[admin@MikroTik] > ip firewall nat print
Flags: X - disabled, I - invalid, D - dynamic
 0   chain=srcnat action=masquerade out-interface=ether1
```

<img width="852" height="483" alt="WhatsApp Image 2026-06-07 at 09 36 22 (1)" src="https://github.com/user-attachments/assets/d532511f-90b7-45fa-8939-b4cccdea6a1e" />

*Hasil `ip firewall nat print` — NAT masquerade aktif di ether1*

#### Tambah Route ke LAN dan DMZ

```
/ip route add dst-address=192.168.10.0/24 gateway=10.10.10.2
/ip route add dst-address=192.168.20.0/24 gateway=10.10.10.2
```

Route ini penting supaya kalau Client WAN minta data dari server DMZ, paket balasannya bisa balik ke jalur yang bener. Tanpa ini paket balasannya nyasar dan koneksi gagal.

Verifikasi dengan `ip route print`:

```
[admin@MikroTik] > ip route print
Flags: X - disabled, A - active, D - dynamic, C - connect, S - static, r - rip
 #      DST-ADDRESS        PREF-SRC        GATEWAY            DISTANCE
 0 ADS  0.0.0.0/0                          192.168.100.1             1  <- default dari DHCP
 1 ADC  10.10.10.0/30     10.10.10.1       ether2                    0
 2 ADC  172.16.100.0/24   172.16.100.1     ether3                    0
 3 A S  192.168.10.0/24                    10.10.10.2                1
 4 A S  192.168.20.0/24                    10.10.10.2                1
```

<img width="1092" height="460" alt="WhatsApp Image 2026-06-07 at 09 36 25 (1)" src="https://github.com/user-attachments/assets/35d70ff6-4f3f-45ec-893b-882ae74460e2" />

*Hasil `ip route print` — route ke LAN dan DMZ sudah ada*

---

### 5.2 Konfigurasi FortiGate Firewall

FortiGate ini yang paling banyak dikonfigurasi karena dia yang jaga semua lalu lintas antar jaringan.

#### 5.2.1 Set IP tiap interface

```
config system interface
  edit port1
    set ip 10.10.10.2 255.255.255.252
    set allowaccess ping
    set role wan
  next
  edit port2
    set ip 10.20.20.1 255.255.255.252
    set allowaccess ping
    set role lan
  next
  edit port3
    set ip 192.168.20.1 255.255.255.0
    set allowaccess ping
    set role dmz
  next
end
```

Verifikasi dengan `show system interface`:

```
FGT # show system interface
config system interface
    edit "port1"
        set ip 10.10.10.2 255.255.255.252
        set allowaccess ping
        set role wan
    next
    edit "port2"
        set ip 10.20.20.1 255.255.255.252
        set allowaccess ping
        set role lan
    next
    edit "port3"
        set ip 192.168.20.1 255.255.255.0
        set allowaccess ping
        set role dmz
    next
end
```

<img width="853" height="728" alt="WhatsApp Image 2026-06-07 at 09 36 32 (1)" src="https://github.com/user-attachments/assets/9aa3d260-c9d4-4224-8b88-37af55bc08ca" />

*Hasil `show system interface` di FortiGate — IP tiap port sudah terpasang*

#### 5.2.2 Set Routing

```
config router static
  edit 1
    set dst 0.0.0.0 0.0.0.0
    set gateway 10.10.10.1
    set device port1
  next
  edit 2
    set dst 192.168.10.0 255.255.255.0
    set gateway 10.20.20.2
    set device port2
  next
end
```

FortiGate perlu dikasih tahu secara manual bahwa jaringan LAN ada di balik Cisco Router (10.20.20.2). Kalau tidak ada route ini, FortiGate tidak tahu harus kirim paket ke mana kalau ada yang mau akses LAN.

Verifikasi dengan `get router info routing-table all`:

```
FGT # get router info routing-table all
Codes: K - kernel, C - connected, S - static, R - RIP, B - BGP
       O - OSPF, IA - OSPF inter area

S*      0.0.0.0/0 [10/0] via 10.10.10.1, port1
C       10.10.10.0/30 is directly connected, port1
C       10.20.20.0/30 is directly connected, port2
S       192.168.10.0/24 [10/0] via 10.20.20.2, port2
C       192.168.20.0/24 is directly connected, port3
```

<img width="838" height="333" alt="WhatsApp Image 2026-06-07 at 09 36 33" src="https://github.com/user-attachments/assets/1a468f29-c7de-479f-9248-6aacfac61ce9" />

*Hasil `get router info routing-table all` — semua route sudah terpasang*


#### 5.2.3 Buat VIP (Port Forwarding)

```
config firewall vip
  edit "VIP_WAN_to_DMZ"
    set extip 10.10.10.2
    set extintf port1
    set mappedip 192.168.20.10
    set portforward enable
    set protocol tcp
    set extport 80
    set mappedport 80
  next
end
```

VIP ini kayak "penerusan panggilan". Kalau ada yang akses IP WAN FortiGate (10.10.10.2) di port 80, FortiGate otomatis nerusin ke server DMZ (192.168.20.10:80). Jadi dari luar orang tidak perlu tahu IP asli server-nya.

Verifikasi dengan `show firewall vip`:

```
FGT # show firewall vip
config firewall vip
    edit "VIP_WAN_to_DMZ"
        set extip 10.10.10.2
        set extintf "port1"
        set mappedip "192.168.20.10"
        set portforward enable
        set protocol tcp
        set extport 80
        set mappedport 80
    next
end
```

<img width="875" height="207" alt="WhatsApp Image 2026-06-07 at 09 36 33 (2)" src="https://github.com/user-attachments/assets/5a6a9e0b-2030-4088-bd09-dd69213c5c38" />

*Hasil `show firewall vip` — VIP sudah terkonfigurasi*

#### 5.2.4 Buat Firewall Policy

```
config firewall policy
  edit 1
    set name "LAN_to_WAN"
    set srcintf port2
    set dstintf port1
    set srcaddr "LAN_Network"
    set dstaddr "all"
    set action accept
    set schedule always
    set service ALL
    set nat enable
  next
  edit 2
    set name "LAN_to_DMZ"
    set srcintf port2
    set dstintf port3
    set srcaddr "LAN_Network"
    set dstaddr "Server_DMZ"
    set action accept
    set schedule always
    set service HTTP
    set nat disable
  next
  edit 3
    set name "WAN_to_DMZ_HTTP"
    set srcintf port1
    set dstintf port3
    set srcaddr "all"
    set dstaddr "VIP_WAN_to_DMZ"
    set action accept
    set schedule always
    set service HTTP
  next
end
```

Penjelasan singkat tiap policy:
- **LAN_to_WAN** — Client LAN boleh akses internet, pakai NAT supaya IP private-nya tidak kelihatan dari luar
- **LAN_to_DMZ** — Client LAN boleh akses server DMZ langsung lewat IP aslinya, tidak perlu NAT
- **WAN_to_DMZ_HTTP** — Orang dari luar cuma boleh akses server DMZ via HTTP saja lewat VIP. Akses langsung ke LAN atau IP asli DMZ? Diblokir.

Verifikasi dengan `show firewall policy`:

```
FGT # show firewall policy
config firewall policy
    edit 1
        set name "LAN_to_WAN"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "LAN_Network"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
    edit 2
        set name "LAN_to_DMZ"
        set srcintf "port2"
        set dstintf "port3"
        set srcaddr "LAN_Network"
        set dstaddr "Server_DMZ"
        set action accept
        set schedule "always"
        set service "HTTP"
    next
    edit 3
        set name "WAN_to_DMZ_HTTP"
        set srcintf "port1"
        set dstintf "port3"
        set srcaddr "all"
        set dstaddr "VIP_WAN_to_DMZ"
        set action accept
        set schedule "always"
        set service "HTTP"
    next
end
```



<img width="846" height="723" alt="WhatsApp Image 2026-06-07 at 09 36 33 (1)" src="https://github.com/user-attachments/assets/ab168c72-64ed-4252-a7b4-df6d712214ba" />
<img width="845" height="718" alt="WhatsApp Image 2026-06-07 at 09 36 34" src="https://github.com/user-attachments/assets/def7fa27-8bf9-41f3-a4c0-a8d1e88003cd" />

*Hasil `show firewall policy` — semua policy sudah terdaftar*

---

### 5.3 Konfigurasi Cisco Router

Cisco Router tugasnya jadi gateway untuk jaringan LAN dan nerusin semua traffic ke FortiGate.

```
Router> enable
Router# configure terminal
Router(config)# hostname Cisco-Router
Cisco-Router(config)# interface GigabitEthernet0/0
Cisco-Router(config-if)# ip address 10.20.20.2 255.255.255.252
Cisco-Router(config-if)# no shutdown
Cisco-Router(config-if)# exit
Cisco-Router(config)# interface GigabitEthernet0/1
Cisco-Router(config-if)# ip address 192.168.10.1 255.255.255.0
Cisco-Router(config-if)# no shutdown
Cisco-Router(config-if)# exit
Cisco-Router(config)# ip route 0.0.0.0 0.0.0.0 10.20.20.1
Cisco-Router# copy running-config startup-config
```

`no shutdown` wajib diketik karena interface Cisco defaultnya mati. Kalau lupa ini, interface tidak akan aktif meskipun IP sudah dipasang.

Verifikasi dengan `show ip interface brief`:

```
Cisco-Router# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0    10.20.20.2      YES manual up                    up
GigabitEthernet0/1    192.168.10.1    YES manual up                    up
```

<img width="847" height="718" alt="WhatsApp Image 2026-06-07 at 09 36 34 (1)" src="https://github.com/user-attachments/assets/b380e806-c18a-48e7-8eb9-a8053ecfe3b1" />
*Hasil `show ip interface brief` — kedua interface aktif (up/up)*

Verifikasi routing dengan `show ip route`:

```
Cisco-Router# show ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP

Gateway of last resort is 10.20.20.1 to network 0.0.0.0

      10.0.0.0/30 is subnetted, 1 subnets
C        10.20.20.0 is directly connected, GigabitEthernet0/0
      192.168.10.0/24 is directly connected
C        192.168.10.0 is directly connected, GigabitEthernet0/1
S*    0.0.0.0/0 [1/0] via 10.20.20.1
```

![Cisco Show IP Route](./screenshots/cisco/cisco_show_ip_route.png)
*Hasil `show ip route` — default route ke FortiGate (10.20.20.1) sudah ada*

---

### 5.4 Konfigurasi Client LAN (Tiny Core Linux)

Set IP bisa lewat **Control Panel > Network** di GUI, atau pakai perintah ini di terminal:

```bash
sudo ifconfig eth0 192.168.10.10 netmask 255.255.255.0
sudo route add default gw 192.168.10.1
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

| Parameter | Nilai |
|-----------|-------|
| IP Address | `192.168.10.10` |
| Subnet Mask | `255.255.255.0` |
| Gateway | `192.168.10.1` |
| DNS | `8.8.8.8` |

Verifikasi dengan `ifconfig` dan `route -n`:

```
tc@box:~$ ifconfig eth0
eth0      Link encap:Ethernet  HWaddr xx:xx:xx:xx:xx:xx
          inet addr:192.168.10.10  Bcast:192.168.10.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

tc@box:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.10.1    0.0.0.0         UG    0      0        0 eth0
192.168.10.0    0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

<img width="1474" height="819" alt="WhatsApp Image 2026-06-07 at 09 36 35" src="https://github.com/user-attachments/assets/2f1ac74c-4270-4eb8-a68e-8fc0500139b9" />
*Hasil `ifconfig` di Client LAN — IP sudah terpasang*

---

### 5.5 Konfigurasi Client WAN (Tiny Core Linux)

```bash
sudo ifconfig eth0 172.16.100.10 netmask 255.255.255.0
sudo route add default gw 172.16.100.1
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

| Parameter | Nilai |
|-----------|-------|
| IP Address | `172.16.100.10` |
| Subnet Mask | `255.255.255.0` |
| Gateway | `172.16.100.1` |
| DNS | `8.8.8.8` |

Verifikasi dengan `ifconfig`:

```
tc@box:~$ ifconfig eth0
eth0      Link encap:Ethernet  HWaddr xx:xx:xx:xx:xx:xx
          inet addr:172.16.100.10  Bcast:172.16.100.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
```


<img width="1463" height="807" alt="WhatsApp Image 2026-06-07 at 09 36 35 (1)" src="https://github.com/user-attachments/assets/b9bdeb3e-2c3e-4f08-bc05-5340b6f9e6f1" />
*Hasil `ifconfig` di Client WAN — IP sudah terpasang*

---

### 5.6 Konfigurasi Ubuntu Server DMZ

#### Set IP Statis

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Isi filenya:

```yaml
network:
  version: 2
  ethernets:
    ens3:
      addresses:
        - 192.168.20.10/24
      routes:
        - to: default
          via: 192.168.20.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

```bash
sudo netplan apply
```

Verifikasi dengan `ip a` dan `ip route`:

```
root@ubuntu:~# ip a show ens3
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    inet 192.168.20.10/24 brd 192.168.20.255 scope global ens3

root@ubuntu:~# ip route
default via 192.168.20.1 dev ens3 proto static
192.168.20.0/24 dev ens3 proto kernel scope link src 192.168.20.10
```


<img width="751" height="340" alt="WhatsApp Image 2026-06-07 at 09 36 24 (1)" src="https://github.com/user-attachments/assets/d96ff407-8cb8-4fb1-aff9-7ed881580a18" />
*Hasil `ip a` di Ubuntu Server — IP 192.168.20.10 sudah terpasang*

#### Install dan Jalankan Nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

Output `systemctl status nginx` kalau berhasil:

```
root@ubuntu:~# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since ...
```


<img width="822" height="507" alt="WhatsApp Image 2026-06-07 at 09 36 27 (1)" src="https://github.com/user-attachments/assets/94a617bb-417f-46eb-95d3-a4d5363c781a" />
*Hasil `systemctl status nginx` — Nginx aktif (active running)*

#### Ganti Halaman Default Nginx

```bash
sudo nano /var/www/html/index.nginx-debian.html
```

Isi file:

```html
<!DOCTYPE html>
<html>
<head>
    <title>DMZ Web Server</title>
</head>
<body>
    <h1>Tumod_4_DMZ_Firewall_[No.Kel]-[NamaKelompok]</h1>
    <p>Web Server DMZ - Modul 4 Praktikum Jaringan Komputer</p>
</body>
</html>
```

Verifikasi dari server sendiri pakai `curl`:

```
root@ubuntu:~# curl http://localhost
<!DOCTYPE html>
<html>
<head>
    <title>DMZ Web Server</title>
</head>
<body>
    <h1>Tumod_4_DMZ_Firewall_03-Kelompok3</h1>
    <p>Web Server DMZ - Modul 4 Praktikum Jaringan Komputer</p>
</body>
</html>

## 6. Hasil Pengujian

### Pengujian 1 — Client LAN ping ke Cisco Router

```
tc@box:~$ ping 192.168.10.1
PING 192.168.10.1 (192.168.10.1): 56 data bytes
64 bytes from 192.168.10.1: seq=0 ttl=255 time=1.2 ms
64 bytes from 192.168.10.1: seq=1 ttl=255 time=0.8 ms
64 bytes from 192.168.10.1: seq=2 ttl=255 time=0.9 ms
```

Berhasil karena Client LAN dan Cisco Router berada di subnet yang sama (192.168.10.0/24).


<img width="903" height="566" alt="WhatsApp Image 2026-06-07 at 09 36 29 (1)" src="https://github.com/user-attachments/assets/4fc5f512-7db7-4415-86e6-28bb955d58db" />
*Ping dari Client LAN ke Cisco Router (192.168.10.1) — berhasil*

---

### Pengujian 2 — Client LAN ping ke FortiGate

```
tc@box:~$ ping 10.20.20.1
PING 10.20.20.1 (10.20.20.1): 56 data bytes
64 bytes from 10.20.20.1: seq=0 ttl=255 time=1.5 ms
64 bytes from 10.20.20.1: seq=1 ttl=255 time=1.1 ms
64 bytes from 10.20.20.1: seq=2 ttl=255 time=1.2 ms
```

Berhasil. Packet dari Client LAN diteruskan Cisco Router ke FortiGate.


<img width="903" height="566" alt="WhatsApp Image 2026-06-07 at 09 36 29 (1)" src="https://github.com/user-attachments/assets/3ecc5fd3-67fb-4e24-8258-f8296fb0d771" />
*Ping dari Client LAN ke FortiGate port2 (10.20.20.1) — berhasil*

---

### Pengujian 3 — Client LAN ping ke Server DMZ

```
tc@box:~$ ping 192.168.20.10
PING 192.168.20.10 (192.168.20.10): 56 data bytes
64 bytes from 192.168.20.10: seq=0 ttl=63 time=2.1 ms
64 bytes from 192.168.20.10: seq=1 ttl=63 time=1.8 ms
64 bytes from 192.168.20.10: seq=2 ttl=63 time=1.9 ms
```

Berhasil karena ada policy `LAN_to_DMZ` yang mengizinkan traffic dari LAN ke DMZ.

<img width="801" height="523" alt="WhatsApp Image 2026-06-07 at 09 36 29 (2)" src="https://github.com/user-attachments/assets/4fdd9ce7-d42f-4ef6-91f3-9c9333a40729" />
*Ping dari Client LAN ke Ubuntu Server DMZ (192.168.20.10) — berhasil*

---

### Pengujian 4 — Client LAN akses web server DMZ

```
tc@box:~$ wget http://192.168.20.10 -O - 2>/dev/null | head -10
<!DOCTYPE html>
<html>
<head>
    <title>DMZ Web Server</title>
</head>
<body>
    <h1>Tumod_4_DMZ_Firewall_03-Kelompok3</h1>
    <p>Web Server DMZ - Modul 4 Praktikum Jaringan Komputer</p>
</body>
</html>
```

Halaman web server DMZ berhasil diakses dari Client LAN.

<img width="990" height="723" alt="WhatsApp Image 2026-06-07 at 09 36 30" src="https://github.com/user-attachments/assets/cf015909-e39c-47c5-823a-7f7ce56b069a" />
*Client LAN akses web server DMZ via http://192.168.20.10 — berhasil*

---

### Pengujian 5 — Client WAN ping ke MikroTik ISP

```
tc@box:~$ ping 172.16.100.1
PING 172.16.100.1 (172.16.100.1): 56 data bytes
64 bytes from 172.16.100.1: seq=0 ttl=64 time=0.9 ms
64 bytes from 172.16.100.1: seq=1 ttl=64 time=0.7 ms
64 bytes from 172.16.100.1: seq=2 ttl=64 time=0.8 ms
```

Berhasil karena Client WAN dan MikroTik ether3 berada di subnet yang sama.

<img width="883" height="540" alt="WhatsApp Image 2026-06-07 at 09 36 31" src="https://github.com/user-attachments/assets/67154cba-f1c1-4386-8ee6-58a5f6b4b93b" />
*Ping dari Client WAN ke MikroTik ISP (172.16.100.1) — berhasil*

---

### Pengujian 6 — Client WAN ping ke FortiGate WAN

```
tc@box:~$ ping 10.10.10.2
PING 10.10.10.2 (10.10.10.2): 56 data bytes
64 bytes from 10.10.10.2: seq=0 ttl=64 time=1.6 ms
64 bytes from 10.10.10.2: seq=1 ttl=64 time=1.3 ms
64 bytes from 10.10.10.2: seq=2 ttl=64 time=1.4 ms
```

Berhasil. FortiGate mengizinkan ping di port WAN karena kita set `allowaccess ping` di port1.

<img width="883" height="540" alt="WhatsApp Image 2026-06-07 at 09 36 31" src="https://github.com/user-attachments/assets/ec03a59c-c9f7-4d62-a3d9-ff2d2752b2e8" />
*Ping dari Client WAN ke FortiGate WAN (10.10.10.2) — berhasil*

---

### Pengujian 7 — Client WAN akses web via VIP

```
tc@box:~$ wget http://10.10.10.2 -O - 2>/dev/null | head -10
<!DOCTYPE html>
<html>
<head>
    <title>DMZ Web Server</title>
</head>
<body>
    <h1>Tumod_4_DMZ_Firewall_03-Kelompok3</h1>
    <p>Web Server DMZ - Modul 4 Praktikum Jaringan Komputer</p>
</body>
</html>
```

Halaman web server DMZ muncul. Di balik layar, FortiGate yang nerusin request ini ke 192.168.20.10 lewat VIP.

<img width="998" height="714" alt="WhatsApp Image 2026-06-07 at 09 36 30 (1)" src="https://github.com/user-attachments/assets/e94a9bc1-2717-4422-89f2-f95142f3043a" />
*Client WAN akses http://10.10.10.2 — halaman web DMZ muncul (VIP bekerja)*

---

### Pengujian 8 — Client WAN ping ke Client LAN (HARUS GAGAL)

```
tc@box:~$ ping 192.168.10.10
PING 192.168.10.10 (192.168.10.10): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
```

Tidak ada reply sama sekali — FortiGate memblokir karena tidak ada policy yang mengizinkan WAN akses langsung ke LAN. Ini hasil yang diharapkan, artinya firewall bekerja dengan benar.

<<img width="1174" height="646" alt="WhatsApp Image 2026-06-07 at 09 36 36" src="https://github.com/user-attachments/assets/01e35ea9-dc87-4a2d-b867-152817618a06" />
*Ping dari Client WAN ke Client LAN (192.168.10.10) — diblokir firewall ✓*




### Pengujian 9 — Client WAN ping ke IP asli DMZ (HARUS GAGAL)

```
tc@box:~$ ping 192.168.20.10
PING 192.168.20.10 (192.168.20.10): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
```

Gagal juga. Client WAN hanya boleh akses DMZ lewat VIP (10.10.10.2), bukan IP asli server-nya. Ini juga hasil yang diharapkan.


<img width="653" height="175" alt="WhatsApp Image 2026-06-07 at 09 36 29" src="https://github.com/user-attachments/assets/61bcf2fe-38d5-4962-8d94-061fe2e4f2dc" />
*Ping dari Client WAN ke IP asli DMZ (192.168.20.10) — diblokir firewall ✓*

---

### Pengujian 10 — Server DMZ ping ke Client LAN

```
root@ubuntu:~# ping 192.168.10.10
PING 192.168.10.10 (192.168.10.10) 56(84) bytes of data.
64 bytes from 192.168.10.10: icmp_seq=1 ttl=62 time=2.3 ms
64 bytes from 192.168.10.10: icmp_seq=2 ttl=62 time=2.0 ms
64 bytes from 192.168.10.10: icmp_seq=3 ttl=62 time=2.1 ms
```

Berhasil. Route dari DMZ ke LAN sudah ada lewat FortiGate, dan tidak ada policy yang memblokir traffic ini.


<img width="1262" height="232" alt="WhatsApp Image 2026-06-07 at 09 36 31 (1)" src="https://github.com/user-attachments/assets/d3d36997-ca18-4b55-a0c2-d6d396fdd197" />
*Ping dari Ubuntu Server DMZ ke Client LAN (192.168.10.10) — berhasil*

---

### Ringkasan Hasil Pengujian

| No. | Skenario | Hasil | Keterangan |
|-----|----------|-------|------------|
| 1 | Client LAN → Cisco Router | ✅ Berhasil | Satu subnet, langsung nyambung |
| 2 | Client LAN → FortiGate | ✅ Berhasil | Diteruskan Cisco ke FortiGate |
| 3 | Client LAN → Server DMZ (ping) | ✅ Berhasil | Policy LAN_to_DMZ aktif |
| 4 | Client LAN → Akses Web DMZ | ✅ Berhasil | HTTP ke 192.168.20.10 berhasil |
| 5 | Client WAN → MikroTik ISP | ✅ Berhasil | Satu subnet, langsung nyambung |
| 6 | Client WAN → FortiGate WAN | ✅ Berhasil | Ping diizinkan di port WAN |
| 7 | Client WAN → Web via VIP | ✅ Berhasil | VIP meneruskan ke server DMZ |
| 8 | Client WAN → Client LAN | ❌ Gagal (aman ✓) | Tidak ada policy, diblokir firewall |
| 9 | Client WAN → IP Asli DMZ | ❌ Gagal (aman ✓) | Akses langsung diblokir, harus lewat VIP |
| 10 | Server DMZ → Client LAN | ✅ Berhasil | Route dari DMZ ke LAN via FortiGate |

---

## 7. Analisis dan Kesimpulan

### Analisis

**Kenapa perlu DMZ?**

Bayangin punya rumah. Tamu boleh masuk ke ruang tamu, tapi tidak boleh langsung masuk ke kamar tidur. Nah DMZ itu ibarat ruang tamu — orang dari luar boleh akses server web kita di sana, tapi tidak bisa seenaknya masuk ke jaringan LAN. Kalau server di DMZ kena hack sekalipun, jaringan LAN tetap aman karena ada FortiGate yang memisahkan.

**Kenapa VIP diperlukan?**

Karena IP server kita (192.168.20.10) itu IP private, tidak bisa langsung diakses dari internet. VIP ini yang "nerjemahin" — orang dari luar akses IP WAN FortiGate, nanti FortiGate yang otomatis nerusin ke IP asli server. Selain lebih aman karena IP asli server tidak kelihatan, ini juga lebih fleksibel karena satu IP WAN bisa diarahkan ke banyak server di port berbeda.

**Kenapa pengujian 8 dan 9 harus gagal?**

Justru kalau gagal itu berarti firewall kita berhasil! Tidak ada policy yang mengizinkan WAN akses langsung ke LAN atau ke IP asli DMZ. FortiGate secara default blokir semua traffic yang tidak punya izin (namanya *implicit deny*). Kalau dua pengujian ini berhasil, berarti ada yang salah di konfigurasi firewall.

**Kenapa NAT dipakai di policy LAN_to_WAN tapi tidak di LAN_to_DMZ?**

Waktu Client LAN mau akses internet, IP private-nya (192.168.10.x) harus diubah jadi IP yang bisa dikenali di internet — makanya perlu NAT. Tapi kalau Client LAN mau akses server DMZ, keduanya masih di jaringan internal yang saling tahu IP masing-masing, jadi tidak perlu NAT.
---
