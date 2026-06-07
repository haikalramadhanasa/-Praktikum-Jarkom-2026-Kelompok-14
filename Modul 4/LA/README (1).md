# Laporan Tugas Modul 4 — DMZ & Firewall
**Mata Kuliah:** Praktikum Jaringan Komputer  
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

Ini adalah gambar topologi jaringan:
<img width="1166" height="686" alt="WhatsApp Image 2026-06-07 at 09 36 26 (1)-1-1" src="https://github.com/user-attachments/assets/8d805d74-d4a2-4ad3-8199-24476492cc59" />
*Screenshot topologi di PNETLab — semua node sudah nyala dan terhubung*
---

## 2. Penjelasan Perangkat

| No. | Perangkat | Fungsi |
|-----|-----------|--------|
| 1 | **Cloud / Jaringan Lab** | Ini adalah sumber internet asli dari jaringan lab. MikroTik konek ke sini untuk dapat akses internet |
| 2 | **MikroTik ISP** | Berperan jadi router ISP, tugasnya kasih koneksi internet ke jaringan simulasi kita |
| 3 | **FortiGate Firewall** | Ini yang paling penting. Dia yang jaga dan atur siapa boleh lewat ke mana — antara jaringan luar (WAN), jaringan dalam (LAN), dan server (DMZ) |
| 4 | **Cisco Router** | Router yang menghubungkan FortiGate ke jaringan LAN. Dia jadi "pintu masuk" ke jaringan internal |
| 5 | **Client LAN Tiny Core Linux** | Ini simulasi user yang ada di dalam kantor/jaringan internal |
| 6 | **Client WAN Tiny Core Linux** | Ini simulasi user dari luar/internet, buat ngetes apakah firewall beneran kerja |
| 7 | **Ubuntu Server DMZ** | Server web yang kita taruh di zona DMZ. Bisa diakses dari luar tapi tetap ada batasan dari firewall |

---

## 3. Segmentasi Jaringan

Di topologi ini ada beberapa jaringan yang dipisah-pisah:

| Segment | Network | Keterangan |
|---------|---------|------------|
| Jaringan Lab / Internet | DHCP dari lab | Sumber internet, MikroTik dapat IP dari sini |
| ISP ke FortiGate | `10.10.10.0/30` | Jalur antara MikroTik dan FortiGate |
| Client WAN | `172.16.100.0/24` | Jaringan tempat client luar berada |
| FortiGate ke Cisco | `10.20.20.0/30` | Jalur antara FortiGate dan Cisco Router |
| LAN | `192.168.10.0/24` | Jaringan internal (client dalam) |
| DMZ | `192.168.20.0/24` | Jaringan khusus server yang bisa diakses publik |

Kenapa harus dipisah? Karena kalau semua jadi satu jaringan, orang dari internet bisa langsung akses komputer internal kita — itu bahaya banget. Dengan dipisah pakai FortiGate, kita bisa atur siapa boleh akses apa:

- **WAN** = zona luar/internet, tidak dipercaya
- **LAN** = zona dalam, hanya orang internal yang bisa akses
- **DMZ** = zona tengah, boleh diakses dari luar tapi tetap dibatasi supaya kalau server kena hack, jaringan LAN tetap aman

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

MikroTik di sini berperan jadi ISP. Kita set IP di tiap interface-nya, aktifkan DHCP client supaya dapat internet dari lab, terus set NAT biar semua bisa keluar ke internet, dan tambah route supaya MikroTik tahu jalan ke LAN dan DMZ.

#### Set IP tiap interface

```bash
# ether1 - minta IP otomatis dari jaringan lab
/ip dhcp-client
add interface=ether1 disabled=no

# ether2 - ke arah FortiGate
/ip address
add address=10.10.10.1/30 interface=ether2 network=10.10.10.0

# ether3 - ke arah Client WAN
/ip address
add address=172.16.100.1/24 interface=ether3 network=172.16.100.0
```

#### Aktifkan NAT

```bash
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade
```

NAT masquerade ini fungsinya supaya traffic dari jaringan di belakang MikroTik bisa keluar ke internet. Cara kerjanya, IP asli dari dalam "disembunyiin" dan diganti pakai IP yang dapat dari lab waktu keluar ke internet.

#### Tambah route ke LAN dan DMZ

```bash
/ip route
add dst-address=192.168.10.0/24 gateway=10.10.10.2
add dst-address=192.168.20.0/24 gateway=10.10.10.2
```

Route ini penting supaya kalau Client WAN minta data dari server DMZ, paket balasan bisa balik ke jalur yang bener lewat FortiGate. Tanpa ini, paket balasannya nyasar dan koneksi gagal.

![Konfigurasi MikroTik](./screenshots/mikrotik_config.png)
*Hasil konfigurasi IP dan route di MikroTik*

---

### 5.2 Konfigurasi FortiGate Firewall

FortiGate ini yang paling banyak dikonfigurasi karena dia yang jaga semua lalu lintas antar jaringan.

#### 5.2.1 Set IP tiap interface

```bash
# port1 - WAN (ke MikroTik)
config system interface
  edit port1
    set ip 10.10.10.2 255.255.255.252
    set allowaccess ping
    set role wan
  next
end

# port2 - ke Cisco Router
config system interface
  edit port2
    set ip 10.20.20.1 255.255.255.252
    set allowaccess ping
    set role lan
  next
end

# port3 - DMZ (ke Ubuntu Server)
config system interface
  edit port3
    set ip 192.168.20.1 255.255.255.0
    set allowaccess ping
    set role dmz
  next
end
```

#### 5.2.2 Set routing

```bash
# Default route - semua traffic yang tidak dikenal, lempar ke MikroTik
config router static
  edit 1
    set dst 0.0.0.0 0.0.0.0
    set gateway 10.10.10.1
    set device port1
  next
end

# Route ke LAN - lewat Cisco Router
config router static
  edit 2
    set dst 192.168.10.0 255.255.255.0
    set gateway 10.20.20.2
    set device port2
  next
end
```

FortiGate perlu dikasih tahu secara manual bahwa jaringan LAN (192.168.10.0/24) ada di balik Cisco Router. Kalau tidak ada route ini, FortiGate tidak tahu harus kirim paket ke mana kalau ada yang mau akses LAN.

#### 5.2.3 Buat Address Object

Address object itu ibarat kita kasih nama pada sebuah IP atau range IP, supaya nanti waktu bikin policy lebih mudah dibaca.

```bash
config firewall address
  edit "LAN_Network"
    set subnet 192.168.10.0 255.255.255.0
  next
  edit "Server_DMZ"
    set subnet 192.168.20.10 255.255.255.255
  next
  edit "Client_WAN"
    set subnet 172.16.100.0 255.255.255.0
  next
end
```

#### 5.2.4 Buat VIP (Port Forwarding)

```bash
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

VIP ini fungsinya kayak "penerusan panggilan". Kalau ada yang akses IP WAN FortiGate (10.10.10.2) di port 80, FortiGate otomatis nerusin request itu ke server DMZ (192.168.20.10:80). Jadi dari luar, orang tidak perlu tahu IP asli server-nya.

#### 5.2.5 Buat Firewall Policy

Policy ini yang menentukan traffic mana yang boleh lewat dan mana yang diblokir.

```bash
# Policy 1: LAN boleh akses internet (pakai NAT)
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
end

# Policy 2: LAN boleh akses server DMZ (tanpa NAT)
config firewall policy
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
end

# Policy 3: Luar (WAN) hanya boleh akses DMZ lewat HTTP
config firewall policy
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
- **LAN_to_DMZ** — Client LAN boleh akses server DMZ langsung pakai IP aslinya, tidak perlu NAT karena masih di jaringan internal
- **WAN_to_DMZ_HTTP** — Orang dari luar cuma boleh akses server DMZ via HTTP saja, itu pun harus lewat VIP. Akses ke LAN atau langsung ke IP DMZ? Diblokir.

![FortiGate Interface](./screenshots/fortigate_interface.png)
*Konfigurasi interface FortiGate*

![FortiGate Policy](./screenshots/fortigate_policy.png)
*Daftar firewall policy FortiGate*

![FortiGate VIP](./screenshots/fortigate_vip.png)
*Konfigurasi VIP FortiGate*

---

### 5.3 Konfigurasi Cisco Router

Cisco Router tugasnya simpel: jadi gateway untuk jaringan LAN dan nerusin semua traffic ke FortiGate.

```cisco
Router> enable
Router# configure terminal

Router(config)# hostname Cisco-Router

! Interface ke FortiGate
Cisco-Router(config)# interface GigabitEthernet0/0
Cisco-Router(config-if)# ip address 10.20.20.2 255.255.255.252
Cisco-Router(config-if)# no shutdown
Cisco-Router(config-if)# exit

! Interface ke LAN
Cisco-Router(config)# interface GigabitEthernet0/1
Cisco-Router(config-if)# ip address 192.168.10.1 255.255.255.0
Cisco-Router(config-if)# no shutdown
Cisco-Router(config-if)# exit

! Default route ke FortiGate
Cisco-Router(config)# ip route 0.0.0.0 0.0.0.0 10.20.20.1

! Simpan supaya tidak hilang kalau restart
Cisco-Router# copy running-config startup-config
```

Hal penting yang perlu diperhatikan:
- `no shutdown` wajib diketik karena interface Cisco defaultnya mati. Kalau lupa ini, interface tidak akan aktif
- Default route ke FortiGate (10.20.20.1) artinya semua traffic yang tidak tahu harus ke mana, dikirim ke FortiGate dulu
- Jangan lupa `copy running-config startup-config` supaya konfigurasi tidak hilang kalau router di-restart

![Cisco Config](./screenshots/cisco_config.png)
*Konfigurasi interface dan routing Cisco Router*

![Cisco Show Route](./screenshots/cisco_show_route.png)
*Hasil show ip route di Cisco Router*

---

### 5.4 Konfigurasi Client LAN (Tiny Core Linux)

Set IP bisa lewat Control Panel > Network di GUI, atau pakai perintah ini di terminal:

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

![Client LAN Config](./screenshots/client_lan_config.png)
*Konfigurasi IP Client LAN*

---

### 5.5 Konfigurasi Client WAN (Tiny Core Linux)

Sama seperti Client LAN, tinggal beda IP dan gateway-nya:

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

![Client WAN Config](./screenshots/client_wan_config.png)
*Konfigurasi IP Client WAN*

---

### 5.6 Konfigurasi Ubuntu Server DMZ

#### Set IP Statis

Edit file netplan:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Isi seperti ini:

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

Terapkan:

```bash
sudo netplan apply
```

#### Install dan Jalankan Nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

#### Ganti Halaman Default

```bash
sudo nano /var/www/html/index.nginx-debian.html
```

Ganti isinya jadi:

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

`systemctl enable nginx` itu supaya Nginx otomatis nyala lagi kalau server direstart, jadi tidak perlu start manual tiap kali.

![Nginx Status](./screenshots/ubuntu_nginx_status.png)
*Nginx aktif di Ubuntu Server DMZ*

![Web Page DMZ](./screenshots/ubuntu_web_page.png)
*Tampilan halaman web server DMZ*

---

## 6. Hasil Pengujian

### Pengujian 1 — Client LAN ping ke Cisco Router

```bash
ping 192.168.10.1
```

Harusnya berhasil karena Client LAN (192.168.10.10) dan Cisco Router (192.168.10.1) berada di subnet yang sama, jadi bisa langsung komunikasi.

![Test 1](./screenshots/test_lan_to_cisco.png)
*Client LAN ping ke Cisco Router — berhasil*

---

### Pengujian 2 — Client LAN ping ke FortiGate

```bash
ping 10.20.20.1
```

Harusnya berhasil. Packet dari Client LAN diteruskan Cisco Router ke FortiGate.

![Test 2](./screenshots/test_lan_to_fortigate.png)
*Client LAN ping ke FortiGate port2 — berhasil*

---

### Pengujian 3 — Client LAN ping ke Server DMZ

```bash
ping 192.168.20.10
```

Harusnya berhasil karena ada policy `LAN_to_DMZ` yang mengizinkan.

![Test 3](./screenshots/test_lan_to_dmz.png)
*Client LAN ping ke Ubuntu Server DMZ — berhasil*

---

### Pengujian 4 — Client LAN akses web server DMZ

```bash
wget http://192.168.20.10 -O - | head
```

Harusnya muncul halaman web yang sudah kita buat tadi di Nginx.

![Test 4](./screenshots/test_lan_akses_web_dmz.png)
*Client LAN akses halaman web DMZ — berhasil*

---

### Pengujian 5 — Client WAN ping ke MikroTik ISP

```bash
ping 172.16.100.1
```

Berhasil karena keduanya di subnet yang sama (172.16.100.0/24).

![Test 5](./screenshots/test_wan_to_mikrotik.png)
*Client WAN ping ke MikroTik — berhasil*

---

### Pengujian 6 — Client WAN ping ke FortiGate

```bash
ping 10.10.10.2
```

Berhasil karena FortiGate mengizinkan ping di port WAN-nya (`allowaccess ping`).

![Test 6](./screenshots/test_wan_to_fortigate.png)
*Client WAN ping ke FortiGate WAN — berhasil*

---

### Pengujian 7 — Client WAN akses web via VIP

```bash
wget http://10.10.10.2 -O - | head
```

Harusnya muncul halaman web server DMZ. Di balik layar, FortiGate yang nerusin request ini ke 192.168.20.10 lewat VIP.

![Test 7](./screenshots/test_wan_akses_web_dmz.png)
*Client WAN akses http://10.10.10.2 — halaman web DMZ muncul*

---

### Pengujian 8 — Client WAN ping ke Client LAN (harus GAGAL)

```bash
ping 192.168.10.10
```

Harusnya tidak ada reply sama sekali. Tidak ada policy yang mengizinkan WAN akses langsung ke LAN, jadi FortiGate otomatis blokir.

![Test 8](./screenshots/test_wan_ping_lan_fail.png)
*Client WAN ping ke Client LAN — diblokir firewall (sesuai harapan)*

---

### Pengujian 9 — Client WAN ping ke IP asli DMZ (harus GAGAL)

```bash
ping 192.168.20.10
```

Harusnya gagal juga. Client WAN hanya boleh akses DMZ lewat VIP (10.10.10.2), bukan IP asli server-nya.

![Test 9](./screenshots/test_wan_ping_dmz_fail.png)
*Client WAN ping ke IP asli DMZ — diblokir firewall (sesuai harapan)*

---

### Pengujian 10 — Server DMZ ping ke Client LAN

```bash
ping 192.168.10.10
```

Harusnya berhasil supaya server DMZ bisa balas request dari Client LAN.

![Test 10](./screenshots/test_dmz_ping_lan.png)
*Server DMZ ping ke Client LAN — berhasil*

---

### Tabel Ringkasan Pengujian

| No. | Skenario | Hasil | Keterangan |
|-----|----------|-------|------------|
| 1 | Client LAN → Cisco Router | ✅ Berhasil | Satu subnet, langsung nyambung |
| 2 | Client LAN → FortiGate | ✅ Berhasil | Diteruskan Cisco ke FortiGate |
| 3 | Client LAN → Server DMZ (ping) | ✅ Berhasil | Policy LAN_to_DMZ aktif |
| 4 | Client LAN → Akses Web DMZ | ✅ Berhasil | HTTP ke 192.168.20.10 berhasil |
| 5 | Client WAN → MikroTik ISP | ✅ Berhasil | Satu subnet, langsung nyambung |
| 6 | Client WAN → FortiGate WAN | ✅ Berhasil | Ping diizinkan di port WAN |
| 7 | Client WAN → Web via VIP | ✅ Berhasil | VIP meneruskan ke server DMZ |
| 8 | Client WAN → Client LAN | ❌ Gagal (aman) | Tidak ada policy, diblokir |
| 9 | Client WAN → IP Asli DMZ | ❌ Gagal (aman) | Harus lewat VIP, akses langsung diblokir |
| 10 | Server DMZ → Client LAN | ✅ Berhasil | Ada route dari DMZ ke LAN |

---

## 7. Analisis dan Kesimpulan

### Analisis

**Kenapa perlu DMZ?**

Bayangin punya rumah. Tamu boleh masuk ke ruang tamu, tapi tidak boleh langsung masuk ke kamar tidur. Nah DMZ itu ibarat ruang tamu — orang dari luar boleh akses server web kita di sana, tapi tidak bisa seenaknya masuk ke jaringan internal (LAN). Kalau server di DMZ kena hack sekalipun, jaringan LAN tetap aman karena ada FortiGate yang memisahkan.

**Kenapa VIP diperlukan?**

Karena IP server kita (192.168.20.10) itu IP private, tidak bisa langsung diakses dari internet. VIP ini yang "nerjemahin" — orang dari luar akses IP WAN FortiGate, nanti FortiGate yang otomatis nerusin ke IP asli server. Selain lebih aman karena IP asli server tidak kelihatan, ini juga lebih fleksibel karena satu IP WAN bisa diarahkan ke banyak server di port berbeda.

**Kenapa pengujian 8 dan 9 harus gagal?**

Justru kalau gagal itu berarti firewall kita berhasil! Tidak ada policy yang mengizinkan WAN akses langsung ke LAN atau ke IP asli DMZ. FortiGate secara default akan blokir semua traffic yang tidak punya izin (ini namanya implicit deny). Kalau dua pengujian ini berhasil, berarti ada yang salah di konfigurasi firewall.

**Kenapa NAT dipakai di policy LAN_to_WAN tapi tidak di LAN_to_DMZ?**

Waktu Client LAN mau akses internet, IP private-nya (192.168.10.x) harus diubah jadi IP yang bisa dikenali di internet — makanya perlu NAT. Tapi kalau Client LAN mau akses server DMZ, keduanya masih di jaringan internal yang sama dan saling tahu IP masing-masing, jadi tidak perlu NAT.

### Kesimpulan

Dari praktikum ini kita berhasil bikin jaringan dengan tiga zona keamanan yang bekerja dengan baik. FortiGate berhasil jadi firewall yang mengatur lalu lintas antar zona. Client WAN bisa akses web server DMZ lewat VIP tapi tidak bisa tembus ke jaringan LAN — itu tandanya konfigurasi firewall kita sudah benar. Nginx di server DMZ juga berjalan normal dan bisa diakses dari dua arah (LAN dan WAN). Intinya, konsep DMZ ini sangat berguna di dunia nyata supaya server publik bisa diakses internet tanpa membahayakan jaringan internal.

---

*Laporan Tugas Modul 4 — Praktikum Jaringan Komputer*  
*Ganti semua gambar di folder `screenshots/` dengan screenshot dari hasil praktikum kalian sendiri*
