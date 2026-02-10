---

# 🛡️ Ultimate Privacy Server

## Pi-hole · Unbound · PiVPN (WireGuard) · DPI Bypass (Zapret)

A **self-hosted privacy infrastructure** that provides **network-wide ad blocking**, **fully recursive DNS**, **ISP DPI / censorship bypass**, and **secure VPN access** — designed to be **reboot-proof** and survive **power outages**.

---

## 🌍 Languages
- [English](#english)
- [Türkçe](#turkce)

---
<a name="english"></a>
# English Guide

## ✨ Features

* Network-wide ad & tracker blocking (Pi-hole)
* Fully recursive DNS (no Google / ISP DNS) using Unbound
* DPI & censorship bypass via Zapret (nfqws)
* Fast and secure VPN with WireGuard (PiVPN)
* Persistent NAT & IP forwarding (reboot-safe)
* Single DNS architecture for LAN + VPN

---

## 🧱 Architecture

Devices
→ Router (DNS → Pi-hole)
→ Pi-hole (port 53)
→ Unbound (127.0.0.1:5335)
→ Root DNS Servers

VPN Clients
→ WireGuard (PiVPN)
→ Same DNS filtering

Outbound traffic
→ Zapret (DPI desync)

---

## ⚙️ Requirements

* Debian / Ubuntu based system (Raspberry Pi recommended)
* Static local IP (example: 192.168.1.100)
* Router with LAN DNS configuration
* sudo / root access

---

## 🚀 Installation

### 1. System Update

sudo apt update && sudo apt upgrade -y

---

### 2. Pi-hole (Ad Blocking DNS)

sudo curl -sSL [https://install.pi-hole.net](https://install.pi-hole.net) | bash

During installation:

* Accept **Static IP**
* Do NOT select upstream DNS providers

Admin panel:
http://STATIC_IP/admin

---

### 3. Unbound (Recursive DNS Resolver)

sudo apt install unbound -y
wget [https://www.internic.net/domain/named.root](https://www.internic.net/domain/named.root) -qO- | sudo tee /var/lib/unbound/root.hints

Create config file:

sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf

Content:

server:
interface: 127.0.0.1
port: 5335
do-ip4: yes
do-ip6: no
do-udp: yes
do-tcp: yes
harden-glue: yes
harden-dnssec-stripped: yes
edns-buffer-size: 1472
prefetch: yes
private-address: 10.0.0.0/8
private-address: 172.16.0.0/12
private-address: 192.168.0.0/16

Restart service:

sudo service unbound restart

---

### 4. Pi-hole → Unbound Integration

Pi-hole Admin → Settings → DNS

* Disable all default DNS providers
* Custom 1 (IPv4): 127.0.0.1#5335
* Save

---

### 5. DPI Bypass (Zapret)

sudo apt install curl git ipset nftables -y
git clone [https://github.com/bol-van/zapret.git](https://github.com/bol-van/zapret.git)
cd zapret
sudo ./install_easy.sh

Edit config:

sudo nano /opt/zapret/config

Set:

MODE="nfqws"
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"

Restart service:

sudo service zapret restart

---

### 6. PiVPN (WireGuard)

curl -L [https://install.pivpn.io](https://install.pivpn.io) | bash

* Protocol: WireGuard
* Port: 51820 (default)

Add client:

pivpn add
pivpn -qr

---

### 7. 🔁 Make It Reboot-Proof (CRITICAL)

Linux clears NAT and IP forwarding rules on reboot.
These steps prevent internet loss after power outages.

Enable IP forwarding permanently:

echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

Auto-detect active interface and add NAT rule:

INTERFACE=$(ip route get 1.1.1.1 | awk '{print $5}')
sudo iptables -t nat -A POSTROUTING -o $INTERFACE -j MASQUERADE

Persist rules:

sudo apt install iptables-persistent -y
sudo netfilter-persistent save

---

## 🌐 Router Configuration (Mandatory)

Router → LAN / DHCP Settings

* Primary DNS: 192.168.1.100
* Secondary DNS: empty or same
* WAN DNS: DO NOT USE

Reboot router.

---

## 📱 Client Usage

Mobile:

* Install WireGuard
* Scan QR via pivpn -qr

Desktop:

* Import .conf files from ~/wireguard/configs/

---

## 🧪 Verification

dig google.com @127.0.0.1 -p 5335

* Must resolve
* Queries visible in Pi-hole
* Ads blocked network-wide

---

## ⚠️ Notes

* IPv6 intentionally disabled (better DPI bypass compatibility)
* Never configure public DNS anywhere
* Router must distribute DNS via LAN

---

## 📚 Credits

* Evrim Ağacı TeknoBilim – “Bu Kara Kutu İnternetinizi Düzeltiyor”
* bol-van/zapret
* Pi-hole, Unbound, PiVPN projects

---

## 🧠 Philosophy

If you don’t run your own DNS, you don’t own your internet.

---
<a name="turkce"></a>
# Türkçe Rehber

*(Aşağıdaki bölüm, yukarıdaki yapının birebir Türkçe karşılığıdır.)*

## ✨ Özellikler

* Ağ genelinde reklam ve takipçi engelleme (Pi-hole)
* Google / ISS’siz tam recursive DNS (Unbound)
* DPI ve sansür bypass (Zapret – nfqws)
* WireGuard tabanlı hızlı VPN (PiVPN)
* Elektrik kesintisine dayanıklı NAT & yönlendirme
* LAN + VPN tek DNS mimarisi

---

## 🧱 Mimari

Cihazlar
→ Router (DNS → Pi-hole)
→ Pi-hole (53)
→ Unbound (127.0.0.1:5335)
→ Root DNS Sunucuları

VPN istemcileri
→ WireGuard (PiVPN)
→ Aynı DNS filtreleme

Tüm çıkış trafiği
→ Zapret (DPI desync)

---

## ⚙️ Gereksinimler

* Debian / Ubuntu tabanlı sistem
* Statik IP (örn: 192.168.1.100)
* LAN DNS ayarı yapılabilen router
* sudo / root erişimi

---

## 🚀 Kurulum

### 1. Sistem Güncelleme

sudo apt update && sudo apt upgrade -y

---

### 2. Pi-hole

sudo curl -sSL [https://install.pi-hole.net](https://install.pi-hole.net) | bash

* Statik IP kabul et
* Upstream DNS seçme

Panel:
http://STATIK_IP/admin

---

### 3. Unbound

sudo apt install unbound -y
wget [https://www.internic.net/domain/named.root](https://www.internic.net/domain/named.root) -qO- | sudo tee /var/lib/unbound/root.hints

---

### 4. Pi-hole → Unbound

Custom DNS:
127.0.0.1#5335

---

### 5. Zapret

MODE="nfqws"
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"

---

### 6. PiVPN

pivpn add
pivpn -qr

---

### 7. 🔁 Kalıcılık (ÇOK ÖNEMLİ)

net.ipv4.ip_forward=1
iptables MASQUERADE
iptables-persistent save

---

## 🌐 Router Ayarları

Primary DNS: Pi IP
Secondary DNS: boş
WAN DNS: YOK

---

## 🧠 Felsefe

DNS senin değilse, internet de senin değildir.

---
