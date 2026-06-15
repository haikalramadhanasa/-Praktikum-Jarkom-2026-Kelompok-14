# Laporan Tugas Modul 5 - VLAN, Trunk, VRRP, GRE, dan OSPF

**Praktikum Jaringan Komputer**

**Kelompok:** 14

**Modul:** 5 - Implementasi Jaringan Enterprise HQ-Branch

## Anggota Kelompok

| No. | Nama |
| ---: | ---- |
| 1 | Danendra Nusi Putra Rahanda |
| 2 | Muhammad Syaiful Kalam |
| 3 | Rozaq Nafi'ul Hafidz |

---

## Daftar Isi

1. [Deskripsi Jaringan](#1-deskripsi-jaringan)
2. [Topologi Jaringan](#2-topologi-jaringan)
3. [Tabel Addressing](#3-tabel-addressing)
4. [Konfigurasi Cisco Switch Jakarta](#4-konfigurasi-cisco-switch-jakarta)
5. [Konfigurasi Cisco Router Jakarta](#5-konfigurasi-cisco-router-jakarta)
6. [Konfigurasi MikroTik Router Jakarta](#6-konfigurasi-mikrotik-router-jakarta)
7. [Konfigurasi Ubuntu Server Jakarta](#7-konfigurasi-ubuntu-server-jakarta)
8. [Konfigurasi FortiGate Jakarta](#8-konfigurasi-fortigate-jakarta)
9. [Konfigurasi MikroTik ISP](#9-konfigurasi-mikrotik-isp)
10. [Konfigurasi Switch dan MikroTik Surabaya](#10-konfigurasi-switch-dan-mikrotik-surabaya)
11. [Konfigurasi FortiGate Surabaya](#11-konfigurasi-fortigate-surabaya)
12. [Konfigurasi GRE dan OSPF](#12-konfigurasi-gre-dan-ospf)
13. [Pengujian Akhir](#13-pengujian-akhir)
14. [Analisis](#14-analisis)
15. [Kesimpulan](#15-kesimpulan)

---

## 1. Deskripsi Jaringan

Topologi pada Modul 5 mensimulasikan jaringan enterprise yang memiliki kantor pusat
di Jakarta dan kantor cabang di Surabaya. Kedua lokasi terhubung melalui jaringan
ISP dan GRE tunnel pada FortiGate.

Teknologi yang digunakan pada percobaan ini meliputi:

- VLAN dan trunk IEEE 802.1Q untuk segmentasi jaringan.
- VRRP antara Cisco Router dan MikroTik Router di Jakarta.
- DHCP relay menuju Ubuntu Server Jakarta.
- ISC-DHCP Server dan Nginx pada Ubuntu Server.
- Firewall policy dan NAT pada FortiGate.
- GRE tunnel antara Jakarta dan Surabaya.
- OSPF di atas GRE untuk pertukaran route antarlokasi.
- DHCP Server lokal pada MikroTik Surabaya.

---

## 2. Topologi Jaringan

> **Bukti 01:** Tambahkan screenshot topologi lengkap PNETLab ke bagian ini.

Topologi terbagi menjadi tiga bagian:

1. **Jakarta/HQ**, berisi Cisco Switch, Cisco Router, MikroTik Router, Ubuntu
   Server, FortiGate, serta client VLAN 10 dan VLAN 20.
2. **ISP**, menggunakan MikroTik sebagai penghubung WAN Jakarta dan Surabaya.
3. **Surabaya/Branch**, berisi Cisco Switch, MikroTik Router, FortiGate, serta
   client VLAN 30 dan VLAN 40.

---

## 3. Tabel Addressing

### 3.1 VLAN Jakarta

| VLAN | Nama | Network | Virtual Gateway | Keterangan |
| ---: | ---- | ------- | --------------- | ---------- |
| 10 | FINANCE | `192.168.10.0/24` | `192.168.10.1` | DHCP dari Ubuntu Server |
| 20 | IT | `192.168.20.0/24` | `192.168.20.1` | DHCP dari Ubuntu Server |
| 60 | SERVER-HQ | `192.168.60.0/24` | `192.168.60.1` | Jaringan server Jakarta |

### 3.2 Router Jakarta

| Perangkat | Interface | IP Address | Keterangan |
| --------- | --------- | ---------- | ---------- |
| Cisco Jakarta | `Gi0/1.10` | `192.168.10.2/24` | IP fisik VLAN 10 |
| Cisco Jakarta | `Gi0/1.20` | `192.168.20.2/24` | IP fisik VLAN 20 |
| Cisco Jakarta | `Gi0/1.60` | `192.168.60.2/24` | IP fisik VLAN 60 |
| Cisco Jakarta | `Gi0/0` | `10.10.100.2/30` | Ke FortiGate Jakarta |
| MikroTik Jakarta | `vlan10-finance` | `192.168.10.3/24` | IP fisik VLAN 10 |
| MikroTik Jakarta | `vlan20-it` | `192.168.20.3/24` | IP fisik VLAN 20 |
| MikroTik Jakarta | `vlan60-server` | `192.168.60.3/24` | IP fisik VLAN 60 |
| MikroTik Jakarta | `ether1` | `10.10.101.2/30` | Ke FortiGate Jakarta |

### 3.3 VRRP Jakarta

| VLAN | Virtual IP | Master | Backup |
| ---: | ---------- | ------ | ------ |
| 10 | `192.168.10.1` | Cisco Jakarta | MikroTik Jakarta |
| 20 | `192.168.20.1` | MikroTik Jakarta | Cisco Jakarta |
| 60 | `192.168.60.1` | Cisco Jakarta | MikroTik Jakarta |

### 3.4 Server dan DHCP Jakarta

| Perangkat/Pool | IP atau Range | Gateway |
| -------------- | ------------- | ------- |
| Ubuntu Server | `192.168.60.10/24` | `192.168.60.1` |
| DHCP VLAN 10 | `192.168.10.100-192.168.10.200` | `192.168.10.1` |
| DHCP VLAN 20 | `192.168.20.100-192.168.20.200` | `192.168.20.1` |

### 3.5 WAN, FortiGate, dan ISP

| Perangkat | Interface | IP Address | Terhubung ke |
| --------- | --------- | ---------- | ------------ |
| FortiGate Jakarta | `port1` | `10.10.100.1/30` | Cisco Jakarta |
| FortiGate Jakarta | `port2` | `10.10.101.1/30` | MikroTik Jakarta |
| FortiGate Jakarta | `port3` | `10.0.12.2/30` | MikroTik ISP |
| MikroTik ISP | `ether2` | `10.0.12.1/30` | FortiGate Jakarta |
| MikroTik ISP | `ether3` | `10.0.13.1/30` | FortiGate Surabaya |
| FortiGate Surabaya | `port1` | `10.0.13.2/30` | MikroTik ISP |
| FortiGate Surabaya | `port2` | `10.10.200.1/30` | MikroTik Surabaya |

### 3.6 VLAN Surabaya

| VLAN | Nama | Network | Gateway | Keterangan |
| ---: | ---- | ------- | ------- | ---------- |
| 30 | SALES | `192.168.30.0/24` | `192.168.30.1` | DHCP MikroTik Surabaya |
| 40 | OPERATIONS | `192.168.40.0/24` | `192.168.40.1` | IP static |

| Perangkat | Interface/IP | Keterangan |
| --------- | ------------ | ---------- |
| MikroTik Surabaya | `ether1 - 10.10.200.2/30` | Ke FortiGate Surabaya |
| MikroTik Surabaya | `vlan30-sales - 192.168.30.1/24` | Gateway VLAN 30 |
| MikroTik Surabaya | `vlan40-operations - 192.168.40.1/24` | Gateway VLAN 40 |
| PC Sales | DHCP | Client VLAN 30 |
| PC Operations | `192.168.40.10/24` | Client VLAN 40 |

### 3.7 GRE Tunnel

| Perangkat | Local WAN | Remote WAN | Tunnel IP |
| --------- | --------- | ---------- | --------- |
| FortiGate Jakarta | `10.0.12.2` | `10.0.13.2` | `172.16.0.1/32` |
| FortiGate Surabaya | `10.0.13.2` | `10.0.12.2` | `172.16.0.2/32` |

---

## 4. Konfigurasi Cisco Switch Jakarta

Cisco Switch Jakarta digunakan untuk membuat VLAN 10, 20, dan 60. Port menuju
client dan server dibuat sebagai access port, sedangkan port menuju Cisco Router
dan MikroTik Router dibuat sebagai trunk.

> Nomor interface harus disesuaikan dengan koneksi pada topologi kelompok.

### 4.1 Membuat VLAN

```cisco
enable
configure terminal

vlan 10
 name FINANCE
exit

vlan 20
 name IT
exit

vlan 60
 name SERVER-HQ
exit
```

### 4.2 Mengatur Access Port

```cisco
interface gi0/1
 description CLIENT-VLAN10
 switchport mode access
 switchport access vlan 10
 no shutdown
exit

interface gi0/2
 description CLIENT-VLAN20
 switchport mode access
 switchport access vlan 20
 no shutdown
exit

interface gi0/3
 description UBUNTU-SERVER-VLAN60
 switchport mode access
 switchport access vlan 60
 no shutdown
exit
```

### 4.3 Mengatur Trunk

```cisco
interface gi1/0
 description TRUNK-TO-CISCO-ROUTER
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,60
 no shutdown
exit

interface gi1/1
 description TRUNK-TO-MIKROTIK-JAKARTA
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,60
 no shutdown
exit

end
write memory
```

### 4.4 Verifikasi

```cisco
show vlan brief
show interfaces trunk
```

> **Bukti 02:** Screenshot output `show vlan brief`.
>
> **Bukti 03:** Screenshot output `show interfaces trunk`.

Hasil yang diharapkan adalah VLAN 10, 20, dan 60 berstatus aktif, serta kedua
link router berstatus trunk dan membawa ketiga VLAN tersebut.

---

## 5. Konfigurasi Cisco Router Jakarta

Cisco Router Jakarta menjalankan router-on-a-stick, VRRP, DHCP relay, dan
menjadi salah satu jalur keluar jaringan Jakarta menuju FortiGate.

### 5.1 Subinterface VLAN

```cisco
enable
configure terminal

interface gi0/1
 description TRUNK-TO-SWITCH-JAKARTA
 no ip address
 no shutdown
exit

interface gi0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.2 255.255.255.0
 ip helper-address 192.168.60.10
 vrrp 10 ip 192.168.10.1
 vrrp 10 priority 120
 vrrp 10 preempt
exit

interface gi0/1.20
 encapsulation dot1Q 20
 ip address 192.168.20.2 255.255.255.0
 ip helper-address 192.168.60.10
 vrrp 20 ip 192.168.20.1
 vrrp 20 priority 90
 vrrp 20 preempt
exit

interface gi0/1.60
 encapsulation dot1Q 60
 ip address 192.168.60.2 255.255.255.0
 vrrp 60 ip 192.168.60.1
 vrrp 60 priority 120
 vrrp 60 preempt
exit
```

Priority Cisco dibuat lebih tinggi pada VLAN 10 dan VLAN 60 sehingga Cisco
menjadi master. Pada VLAN 20, priority Cisco dibuat lebih rendah karena
MikroTik Jakarta bertindak sebagai master.

### 5.2 Link ke FortiGate

```cisco
interface gi0/0
 description LINK-TO-FORTIGATE-JAKARTA
 ip address 10.10.100.2 255.255.255.252
 no shutdown
exit

ip route 0.0.0.0 0.0.0.0 10.10.100.1
end
write memory
```

### 5.3 Verifikasi

```cisco
show ip interface brief
show vrrp brief
show running-config interface gi0/1.10
show running-config interface gi0/1.20
show running-config interface gi0/1.60
ping 10.10.100.1
```

> **Bukti 04:** Screenshot `show ip interface brief`.
>
> **Bukti 05:** Screenshot `show vrrp brief`.
>
> **Bukti 06:** Screenshot konfigurasi subinterface.
>
> **Bukti 07:** Screenshot ping Cisco ke FortiGate Jakarta.

---

## 6. Konfigurasi MikroTik Router Jakarta

MikroTik Jakarta menjadi pasangan VRRP Cisco. MikroTik berperan sebagai master
VLAN 20 dan backup untuk VLAN 10 serta VLAN 60.

> Contoh berikut menggunakan `ether2` sebagai trunk ke switch dan `ether1`
> sebagai link ke FortiGate. Sesuaikan dengan topologi aktual.

### 6.1 VLAN Interface

```routeros
/interface vlan
add name=vlan10-finance interface=ether2 vlan-id=10
add name=vlan20-it interface=ether2 vlan-id=20
add name=vlan60-server interface=ether2 vlan-id=60
```

### 6.2 IP Address Fisik

```routeros
/ip address
add address=192.168.10.3/24 interface=vlan10-finance
add address=192.168.20.3/24 interface=vlan20-it
add address=192.168.60.3/24 interface=vlan60-server
add address=10.10.101.2/30 interface=ether1
```

### 6.3 VRRP

```routeros
/interface vrrp
add name=vrrp10 interface=vlan10-finance vrid=10 priority=90 preemption-mode=yes
add name=vrrp20 interface=vlan20-it vrid=20 priority=120 preemption-mode=yes
add name=vrrp60 interface=vlan60-server vrid=60 priority=90 preemption-mode=yes

/ip address
add address=192.168.10.1/32 interface=vrrp10
add address=192.168.20.1/32 interface=vrrp20
add address=192.168.60.1/32 interface=vrrp60
```

### 6.4 DHCP Relay dan Default Route

```routeros
/ip dhcp-relay
add name=relay-vlan10 interface=vlan10-finance dhcp-server=192.168.60.10 local-address=192.168.10.3
add name=relay-vlan20 interface=vlan20-it dhcp-server=192.168.60.10 local-address=192.168.20.3

/ip route
add dst-address=0.0.0.0/0 gateway=10.10.101.1
```

### 6.5 Verifikasi

```routeros
/interface vlan print
/ip address print
/interface vrrp print
/ip dhcp-relay print
/ip route print
/ping 10.10.101.1 count=5
```

> **Bukti 08:** Screenshot `/ip address print`.
>
> **Bukti 09:** Screenshot `/interface vrrp print`.
>
> **Bukti 10:** Screenshot `/ip dhcp-relay print`.
>
> **Bukti 11:** Screenshot `/ip route print`.
>
> **Bukti 12:** Screenshot ping MikroTik ke FortiGate Jakarta.

---

## 7. Konfigurasi Ubuntu Server Jakarta

Ubuntu Server berada pada VLAN 60 dan berfungsi sebagai DHCP Server untuk VLAN
10 dan VLAN 20, sekaligus web server Nginx.

### 7.1 Instalasi Service

Sebelum server dipindahkan ke VLAN 60, hubungkan Ubuntu ke management network
agar paket dapat diunduh.

```bash
sudo apt update
sudo apt install isc-dhcp-server nginx -y
```

### 7.2 IP Static

Contoh konfigurasi Netplan:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.60.10/24
      routes:
        - to: default
          via: 192.168.60.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

Terapkan konfigurasi:

```bash
sudo netplan apply
```

### 7.3 Konfigurasi ISC-DHCP Server

Isi utama `/etc/dhcp/dhcpd.conf`:

```conf
authoritative;

default-lease-time 600;
max-lease-time 7200;

option domain-name-servers 8.8.8.8, 1.1.1.1;

subnet 192.168.10.0 netmask 255.255.255.0 {
  range 192.168.10.100 192.168.10.200;
  option routers 192.168.10.1;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.10.255;
}

subnet 192.168.20.0 netmask 255.255.255.0 {
  range 192.168.20.100 192.168.20.200;
  option routers 192.168.20.1;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.20.255;
}

subnet 192.168.60.0 netmask 255.255.255.0 {
  option routers 192.168.60.1;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.60.255;
}
```

Tentukan interface DHCP pada `/etc/default/isc-dhcp-server`, kemudian restart:

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

### 7.4 Konfigurasi Nginx

```bash
echo '<h1>Web Server Jakarta - Kelompok 14</h1>' | sudo tee /var/www/html/index.html
sudo systemctl enable nginx
sudo systemctl restart nginx
sudo systemctl status nginx
```

### 7.5 Verifikasi

```bash
ip a
ip route
sudo cat /etc/dhcp/dhcpd.conf
ping -c 4 8.8.8.8
```

> **Bukti 13:** Screenshot `ip a`.
>
> **Bukti 14:** Screenshot `ip route`.
>
> **Bukti 15:** Screenshot `/etc/dhcp/dhcpd.conf`.
>
> **Bukti 16:** Screenshot status ISC-DHCP Server dan Nginx.
>
> **Bukti 17:** Screenshot ping Ubuntu ke `8.8.8.8`.

---

## 8. Konfigurasi FortiGate Jakarta

FortiGate Jakarta menghubungkan dua router internal Jakarta dengan ISP,
menjalankan NAT internet, dan menjadi endpoint GRE menuju Surabaya.

### 8.1 Interface

```fortios
config system interface
    edit "port1"
        set alias "TO-CISCO-JAKARTA"
        set ip 10.10.100.1 255.255.255.252
        set allowaccess ping https ssh
    next
    edit "port2"
        set alias "TO-MIKROTIK-JAKARTA"
        set ip 10.10.101.1 255.255.255.252
        set allowaccess ping https ssh
    next
    edit "port3"
        set alias "TO-MIKROTIK-ISP"
        set ip 10.0.12.2 255.255.255.252
        set allowaccess ping
    next
end
```

### 8.2 Static Route

```fortios
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 10.0.12.1
        set device "port3"
    next
    edit 2
        set dst 192.168.10.0 255.255.255.0
        set gateway 10.10.100.2
        set device "port1"
    next
    edit 3
        set dst 192.168.20.0 255.255.255.0
        set gateway 10.10.101.2
        set device "port2"
    next
    edit 4
        set dst 192.168.60.0 255.255.255.0
        set gateway 10.10.100.2
        set device "port1"
    next
    edit 5
        set dst 192.168.10.0 255.255.255.0
        set gateway 10.10.101.2
        set device "port2"
        set distance 20
    next
    edit 6
        set dst 192.168.20.0 255.255.255.0
        set gateway 10.10.100.2
        set device "port1"
        set distance 20
    next
    edit 7
        set dst 192.168.60.0 255.255.255.0
        set gateway 10.10.101.2
        set device "port2"
        set distance 20
    next
end
```

Route dengan distance 20 menjadi jalur cadangan. Route tersebut digunakan saat
router utama untuk VLAN terkait tidak dapat dijangkau.

### 8.3 Firewall Policy dan NAT

```fortios
config firewall policy
    edit 1
        set name "CISCO-JKT-TO-INTERNET"
        set srcintf "port1"
        set dstintf "port3"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
    edit 2
        set name "MIKROTIK-JKT-TO-INTERNET"
        set srcintf "port2"
        set dstintf "port3"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end
```

### 8.4 Verifikasi Awal

```fortios
get system interface physical
get router info routing-table all
show firewall policy
execute ping 10.0.12.1
execute ping 8.8.8.8
```

> **Bukti 18:** Screenshot `get system interface physical`.
>
> **Bukti 19:** Screenshot routing table FortiGate Jakarta.
>
> **Bukti 20:** Screenshot firewall policy Jakarta.
>
> **Bukti 21:** Screenshot ping FortiGate Jakarta ke `8.8.8.8`.

---

## 9. Konfigurasi MikroTik ISP

MikroTik ISP menghubungkan WAN FortiGate Jakarta dan FortiGate Surabaya dengan
Cloud NAT PNETLab. Router ini tidak menjalankan OSPF enterprise.

### 9.1 DHCP Client dan IP WAN

```routeros
/ip dhcp-client
add interface=ether1 disabled=no

/ip address
add address=10.0.12.1/30 interface=ether2 comment=TO-FORTIGATE-JAKARTA
add address=10.0.13.1/30 interface=ether3 comment=TO-FORTIGATE-SURABAYA
```

### 9.2 NAT

```routeros
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade
```

Default route umumnya diperoleh secara dinamis dari DHCP Client pada `ether1`.

### 9.3 Verifikasi

```routeros
/ip address print
/ip route print
/ip firewall nat print
/ping 8.8.8.8 count=5
/ping 10.0.12.2 count=5
/ping 10.0.13.2 count=5
```

> **Bukti 22:** Screenshot `/ip address print`.
>
> **Bukti 23:** Screenshot `/ip route print`.
>
> **Bukti 24:** Screenshot `/ip firewall nat print`.
>
> **Bukti 25:** Screenshot ping ISP ke `8.8.8.8`.
>
> **Bukti 26:** Screenshot pengujian WAN kedua FortiGate.

---

## 10. Konfigurasi Switch dan MikroTik Surabaya

### 10.1 Cisco Switch Surabaya

```cisco
enable
configure terminal

vlan 30
 name SALES
exit

vlan 40
 name OPERATIONS
exit

interface gi0/1
 description CLIENT-SALES
 switchport mode access
 switchport access vlan 30
 no shutdown
exit

interface gi0/2
 description CLIENT-OPERATIONS
 switchport mode access
 switchport access vlan 40
 no shutdown
exit

interface gi0/0
 description TRUNK-TO-MIKROTIK-SURABAYA
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 30,40
 no shutdown
exit

end
write memory
```

Verifikasi:

```cisco
show vlan brief
show interfaces trunk
```

### 10.2 MikroTik Surabaya

Contoh menggunakan `ether2` sebagai trunk ke switch dan `ether1` sebagai link
ke FortiGate Surabaya.

```routeros
/interface vlan
add name=vlan30-sales interface=ether2 vlan-id=30
add name=vlan40-operations interface=ether2 vlan-id=40

/ip address
add address=192.168.30.1/24 interface=vlan30-sales
add address=192.168.40.1/24 interface=vlan40-operations
add address=10.10.200.2/30 interface=ether1

/ip pool
add name=pool-vlan30 ranges=192.168.30.100-192.168.30.200

/ip dhcp-server
add name=dhcp-vlan30 interface=vlan30-sales address-pool=pool-vlan30 disabled=no

/ip dhcp-server network
add address=192.168.30.0/24 gateway=192.168.30.1 dns-server=8.8.8.8

/ip route
add dst-address=0.0.0.0/0 gateway=10.10.200.1
```

Client VLAN 40 menggunakan IP static:

```text
ip 192.168.40.10/24 192.168.40.1
```

### 10.3 Verifikasi

```routeros
/ip address print
/ip pool print
/ip dhcp-server print
/ip route print
```

Pada client VLAN 30:

```text
ip dhcp
show ip
ping 192.168.30.1
```

Pada client VLAN 40:

```text
show ip
ping 192.168.40.1
```

> **Bukti 27:** Screenshot `show vlan brief` Switch Surabaya.
>
> **Bukti 28:** Screenshot `show interfaces trunk` Switch Surabaya.
>
> **Bukti 29:** Screenshot `/ip address print` MikroTik Surabaya.
>
> **Bukti 30:** Screenshot DHCP Server dan pool VLAN 30.
>
> **Bukti 31:** Screenshot route MikroTik Surabaya.
>
> **Bukti 32:** Screenshot client VLAN 30 memperoleh IP DHCP.
>
> **Bukti 33:** Screenshot client VLAN 40 menggunakan IP static.

---

## 11. Konfigurasi FortiGate Surabaya

### 11.1 Interface

```fortios
config system interface
    edit "port1"
        set alias "TO-MIKROTIK-ISP"
        set ip 10.0.13.2 255.255.255.252
        set allowaccess ping https ssh
    next
    edit "port2"
        set alias "TO-MIKROTIK-SURABAYA"
        set ip 10.10.200.1 255.255.255.252
        set allowaccess ping https ssh
    next
end
```

### 11.2 Static Route

```fortios
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 10.0.13.1
        set device "port1"
    next
    edit 2
        set dst 192.168.30.0 255.255.255.0
        set gateway 10.10.200.2
        set device "port2"
    next
    edit 3
        set dst 192.168.40.0 255.255.255.0
        set gateway 10.10.200.2
        set device "port2"
    next
end
```

### 11.3 Firewall Policy dan NAT

```fortios
config firewall policy
    edit 1
        set name "SURABAYA-TO-INTERNET"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end
```

### 11.4 Verifikasi Awal

```fortios
get system interface physical
get router info routing-table all
show firewall policy
execute ping 10.0.13.1
execute ping 8.8.8.8
```

> **Bukti 34:** Screenshot interface FortiGate Surabaya.
>
> **Bukti 35:** Screenshot routing table FortiGate Surabaya.
>
> **Bukti 36:** Screenshot firewall policy Surabaya.
>
> **Bukti 37:** Screenshot ping FortiGate Surabaya ke `8.8.8.8`.

---

## 12. Konfigurasi GRE dan OSPF

GRE membuat jalur virtual Jakarta-Surabaya melalui WAN ISP. OSPF dijalankan di
atas interface GRE agar route jaringan internal dapat dipertukarkan secara
dinamis.

### 12.1 GRE FortiGate Jakarta

```fortios
config system gre-tunnel
    edit "GRE-JKT-SBY"
        set interface "port3"
        set local-gw 10.0.12.2
        set remote-gw 10.0.13.2
    next
end

config system interface
    edit "GRE-JKT-SBY"
        set ip 172.16.0.1 255.255.255.255
        set remote-ip 172.16.0.2 255.255.255.255
        set allowaccess ping
        set interface "port3"
    next
end
```

### 12.2 GRE FortiGate Surabaya

```fortios
config system gre-tunnel
    edit "GRE-SBY-JKT"
        set interface "port1"
        set local-gw 10.0.13.2
        set remote-gw 10.0.12.2
    next
end

config system interface
    edit "GRE-SBY-JKT"
        set ip 172.16.0.2 255.255.255.255
        set remote-ip 172.16.0.1 255.255.255.255
        set allowaccess ping
        set interface "port1"
    next
end
```

### 12.3 OSPF FortiGate Jakarta

```fortios
config router ospf
    set router-id 1.1.1.1
    config area
        edit 0.0.0.0
        next
    end
    config ospf-interface
        edit "OSPF-GRE-JKT"
            set interface "GRE-JKT-SBY"
            set network-type point-to-point
        next
    end
    config network
        edit 1
            set prefix 172.16.0.1 255.255.255.255
        next
    end
    config redistribute "static"
        set status enable
    end
end
```

### 12.4 OSPF FortiGate Surabaya

```fortios
config router ospf
    set router-id 2.2.2.2
    config area
        edit 0.0.0.0
        next
    end
    config ospf-interface
        edit "OSPF-GRE-SBY"
            set interface "GRE-SBY-JKT"
            set network-type point-to-point
        next
    end
    config network
        edit 1
            set prefix 172.16.0.2 255.255.255.255
        next
    end
    config redistribute "static"
        set status enable
    end
end
```

Route internal Jakarta dan Surabaya sudah dibuat sebagai static route pada
masing-masing FortiGate. Konfigurasi `redistribute "static"` menyebarkan route
tersebut melalui OSPF di atas GRE.

### 12.5 Policy Antar-Site

Policy dua arah diperlukan agar traffic internal dapat melewati GRE.

Contoh pada FortiGate Jakarta:

```fortios
config firewall policy
    edit 10
        set name "JAKARTA-TO-SURABAYA"
        set srcintf "port1" "port2"
        set dstintf "GRE-JKT-SBY"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
    edit 11
        set name "SURABAYA-TO-JAKARTA"
        set srcintf "GRE-JKT-SBY"
        set dstintf "port1" "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

Policy serupa dibuat pada FortiGate Surabaya antara `port2` dan
`GRE-SBY-JKT`.

### 12.6 Verifikasi

FortiGate Jakarta:

```fortios
execute ping 10.0.13.2
execute ping 172.16.0.2
get router info ospf neighbor
get router info routing-table ospf
```

FortiGate Surabaya:

```fortios
execute ping 10.0.12.2
execute ping 172.16.0.1
get router info ospf neighbor
get router info routing-table ospf
```

> **Bukti 38:** Screenshot ping WAN antar-FortiGate.
>
> **Bukti 39:** Screenshot ping antar-IP GRE.
>
> **Bukti 40:** Screenshot OSPF neighbor berstatus `Full`.
>
> **Bukti 41:** Screenshot route OSPF pada FortiGate Jakarta.
>
> **Bukti 42:** Screenshot route OSPF pada FortiGate Surabaya.

---

## 13. Pengujian Akhir

### 13.1 Pengujian DHCP

Pada client Jakarta VLAN 10 dan VLAN 20:

```text
ip dhcp
show ip
```

Pada client Surabaya VLAN 30:

```text
ip dhcp
show ip
```

Hasil yang diharapkan:

- VLAN 10 memperoleh alamat `192.168.10.100-192.168.10.200`.
- VLAN 20 memperoleh alamat `192.168.20.100-192.168.20.200`.
- VLAN 30 memperoleh alamat `192.168.30.100-192.168.30.200`.
- Gateway yang diperoleh sesuai tabel addressing.

> **Bukti 43:** Screenshot IP DHCP client VLAN 10 Jakarta.
>
> **Bukti 44:** Screenshot IP DHCP client VLAN 20 Jakarta.
>
> **Bukti 45:** Screenshot IP DHCP client VLAN 30 Surabaya.

### 13.2 Pengujian Internet

```text
ping 8.8.8.8
```

Perintah dilakukan dari client Jakarta dan Surabaya. Reply menunjukkan default
route, firewall policy, dan NAT telah bekerja.

> **Bukti 46:** Screenshot ping internet dari client Jakarta.
>
> **Bukti 47:** Screenshot ping internet dari client Surabaya.

### 13.3 Pengujian Antar-Site

Dari client Jakarta:

```text
ping 192.168.40.10
```

Dari client Surabaya, gunakan alamat DHCP aktual client Jakarta:

```text
ping 192.168.10.x
```

Reply menunjukkan GRE tunnel, OSPF, static route internal, dan policy
antar-site telah bekerja.

> **Bukti 48:** Screenshot ping Jakarta ke Surabaya.
>
> **Bukti 49:** Screenshot ping Surabaya ke Jakarta.

### 13.4 Pengujian Web Server Jakarta

Dari Tiny Core Linux Surabaya, akses:

```text
http://192.168.60.10
```

Halaman harus menampilkan identitas web server Jakarta.

> **Bukti 50:** Screenshot akses Nginx Jakarta dari Surabaya.

### 13.5 Pengujian Failover VRRP

1. Pastikan client VLAN 10 dapat ping `192.168.10.1` dan tujuan jaringan lain.
2. Matikan interface trunk atau router Cisco Jakarta.
3. Amati perubahan status VRRP pada MikroTik.
4. Ulangi ping dari client.
5. Nyalakan kembali Cisco dan amati proses preemption.

Pada kondisi normal, Cisco menjadi master VLAN 10 dan 60, sedangkan MikroTik
menjadi master VLAN 20. Ketika master gagal, router backup mengambil alih
virtual IP sehingga gateway client tidak perlu diubah.

> **Bukti 51:** Screenshot status VRRP sebelum failover.
>
> **Bukti 52:** Screenshot status VRRP setelah master dimatikan.
>
> **Bukti 53:** Screenshot ping tetap berjalan atau pulih setelah failover.

### 13.6 Ringkasan Pengujian

| No. | Pengujian | Hasil | Keterangan |
| ---: | ---------- | ----- | ---------- |
| 1 | DHCP VLAN 10 Jakarta | Isi setelah praktikum | DHCP dari Ubuntu |
| 2 | DHCP VLAN 20 Jakarta | Isi setelah praktikum | DHCP dari Ubuntu |
| 3 | DHCP VLAN 30 Surabaya | Isi setelah praktikum | DHCP dari MikroTik |
| 4 | Internet client Jakarta | Isi setelah praktikum | NAT FortiGate Jakarta |
| 5 | Internet client Surabaya | Isi setelah praktikum | NAT FortiGate Surabaya |
| 6 | GRE Jakarta-Surabaya | Isi setelah praktikum | Ping tunnel |
| 7 | OSPF neighbor | Isi setelah praktikum | Harus `Full` |
| 8 | Ping antarlokasi | Isi setelah praktikum | Route OSPF |
| 9 | Akses web Jakarta dari Surabaya | Isi setelah praktikum | HTTP ke Ubuntu |
| 10 | Failover VRRP | Isi setelah praktikum | Backup mengambil alih |

---

## 14. Analisis

### 14.1 Jalur Traffic Jakarta ke Internet

```text
Client Jakarta
-> Virtual Gateway VRRP
-> Cisco atau MikroTik Jakarta
-> FortiGate Jakarta
-> MikroTik ISP
-> Cloud NAT
-> Internet
```

Client selalu menggunakan virtual IP sebagai gateway. Router yang sedang
menjadi VRRP master menerima traffic dan meneruskannya ke FortiGate. FortiGate
melakukan pemeriksaan policy serta NAT sebelum paket menuju ISP.

### 14.2 Jalur Traffic Surabaya ke Internet

```text
Client Surabaya
-> MikroTik Surabaya
-> FortiGate Surabaya
-> MikroTik ISP
-> Cloud NAT
-> Internet
```

MikroTik Surabaya menjadi gateway VLAN 30 dan VLAN 40. Default route mengarah
ke FortiGate Surabaya, kemudian traffic di-NAT dan diteruskan melalui ISP.

### 14.3 Jalur Traffic Jakarta ke Surabaya

```text
Client Jakarta
-> Gateway Jakarta
-> FortiGate Jakarta
-> GRE-JKT-SBY
-> FortiGate Surabaya
-> MikroTik Surabaya
-> Client Surabaya
```

FortiGate Jakarta memperoleh network Surabaya dari OSPF. Paket dimasukkan ke
GRE tunnel, melewati jaringan ISP sebagai paket GRE, lalu dikeluarkan oleh
FortiGate Surabaya menuju jaringan internal cabang.

### 14.4 Peran VRRP

VRRP menyediakan virtual gateway yang tetap bagi client meskipun perangkat
router aktif berubah. Pembagian master antarsubnet juga membuat traffic normal
tidak hanya melewati satu router:

- Cisco menjadi master VLAN 10 dan VLAN 60.
- MikroTik menjadi master VLAN 20.
- Jika master gagal, backup mengambil alih virtual IP.

### 14.5 Peran GRE dan OSPF

GRE menyediakan interface point-to-point logis antara kedua FortiGate walaupun
keduanya dipisahkan oleh ISP. OSPF memanfaatkan interface tersebut untuk
membentuk adjacency dan menyebarkan route internal. GRE tidak mengenkripsi
traffic, sehingga pada implementasi produksi sebaiknya dikombinasikan dengan
IPsec.

---

## 15. Kesimpulan

Percobaan Modul 5 menggabungkan segmentasi VLAN, redundansi gateway, layanan
DHCP, firewall, NAT, tunnel, dan dynamic routing dalam satu topologi enterprise.
Jika seluruh pengujian berhasil, maka:

1. Client memperoleh konfigurasi IP sesuai VLAN masing-masing.
2. Gateway Jakarta tetap tersedia melalui VRRP.
3. Client Jakarta dan Surabaya dapat mengakses internet.
4. GRE tunnel Jakarta-Surabaya aktif.
5. OSPF neighbor antarkedua FortiGate berstatus `Full`.
6. Route antarlokasi dipertukarkan melalui OSPF.
7. Client Jakarta dan Surabaya dapat saling berkomunikasi.
8. Web server Jakarta dapat diakses dari jaringan Surabaya.

> **Catatan:** Semua hasil pada tabel pengujian dan bagian bukti harus
> disesuaikan dengan output serta screenshot percobaan kelompok, karena nomor
> interface dan IP DHCP dapat berbeda dari contoh modul.
