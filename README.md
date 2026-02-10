---

# 🛡️ Ultimate Privacy Server

## Pi-hole · Unbound · PiVPN (WireGuard) · DPI Bypass (Zapret)

A **self-hosted privacy infrastructure** that provides **network-wide ad blocking**, **fully recursive DNS**, **ISP DPI / censorship bypass**, and **secure VPN access** — designed to be **reboot-proof** and survive **power outages**.

---

## 🌍 Languages

* [English](#english)
* [Türkçe](#turkce)

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

```text
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

IMPORTANT (for DPI bypass):
→ LAN clients must route traffic through the server (the Zapret host).
   That means the server must be the default gateway for those clients
   (either directly, or via router policy-based routing).
```

---

## ⚙️ Requirements

* Debian / Ubuntu based system (Raspberry Pi recommended)
* Static local IP for the server (example: `192.168.1.100`)
* Router with LAN DNS configuration
* **(DPI bypass) Network routing that makes the server the gateway for clients**
* sudo / root access

---

## 🚀 Installation

### 1) System Update

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 2) Pi-hole (Ad Blocking DNS)

Install:

```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

During installation:

* Accept **Static IP**
* Do **NOT** select upstream DNS providers (you will use Unbound)

Admin panel:

```text
http://STATIC_IP/admin
```

Sanity check (optional):

```bash
pihole status
```

---

### 3) Unbound (Recursive DNS Resolver)

Install Unbound and fetch root hints:

```bash
sudo apt install unbound -y
wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints
```

Create the Unbound config file:

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

Paste the following content:

```conf
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
```

Save & exit nano:

```text
Ctrl + O  → Enter  → Ctrl + X
```

Check config syntax (recommended):

```bash
sudo unbound-checkconf
```

Restart and enable Unbound:

```bash
sudo service unbound restart
sudo service unbound status
```

Test Unbound directly:

```bash
dig google.com @127.0.0.1 -p 5335
```

---

### 4) Pi-hole → Unbound Integration

Open Pi-hole admin:

```text
http://STATIC_IP/admin
```

Go to:

```text
Settings → DNS
```

Do the following:

* Disable all default upstream DNS providers
* Set **Custom 1 (IPv4)** to:

```text
127.0.0.1#5335
```

Save.

Verify Pi-hole can resolve via Unbound (optional):

```bash
dig google.com @127.0.0.1
```

---

## 🌐 Gateway Requirement for DPI Bypass (CRITICAL)

Zapret `nfqws` only affects traffic that **passes through the server**.
If your LAN clients send traffic directly to the router/ISP, DPI bypass will not apply.

You must ensure **LAN clients use the server as their default gateway** (or the router routes selected traffic via the server).

### Option A — Recommended: Make the server the default gateway via DHCP (Router)

In your router’s DHCP/LAN settings, set:

```text
Default Gateway / Router / Gateway: 192.168.1.100   (server IP)
Primary DNS:                     192.168.1.100   (server IP)
Secondary DNS:                   empty (or same)
```

Then renew DHCP leases on clients (or reboot them).

Verify on a client:

* Windows:

```bat
ipconfig /all
```

* Linux/macOS:

```bash
ip route | head
```

Expected default route on clients:

```text
default via 192.168.1.100
```

### Option B — Advanced: Router Policy-Based Routing (PBR)

If you cannot change the default gateway for the entire LAN, configure your router to route specific clients or ports (e.g., `80/443`) via `192.168.1.100`.

(Exact UI/commands vary by router firmware.)

---

### 5) DPI Bypass (Zapret)

Install dependencies:

```bash
sudo apt update
sudo apt install curl git ipset nftables -y
```

Clone and install Zapret:

```bash
cd ~
git clone https://github.com/bol-van/zapret.git
cd zapret
sudo ./install_easy.sh
```

Edit Zapret config:

```bash
sudo nano /opt/zapret/config
```

Inside the file, find (nano search: `Ctrl + W`) and set these variables.
If they do not exist, add them near other config variables:

```conf
MODE="nfqws"
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"
```

Save & exit nano:

```text
Ctrl + O  → Enter  → Ctrl + X
```

Restart and check service:

```bash
sudo service zapret restart
sudo service zapret status
```

Confirm nfqws is running (optional):

```bash
ps aux | grep -E "nfqws|zapret" | grep -v grep
```

Check logs (optional):

```bash
sudo journalctl -u zapret -n 200 --no-pager
```

---

### 6) PiVPN (WireGuard)

Install PiVPN:

```bash
curl -L https://install.pivpn.io | bash
```

During installation:

* Protocol: **WireGuard**
* Port: `51820` (default)

Add a client:

```bash
pivpn add
```

Show QR code for the latest client:

```bash
pivpn -qr
```

List clients (optional):

```bash
pivpn list
```

---

### 7) 🔁 Make It Reboot-Proof (CRITICAL)

Enable IPv4 forwarding permanently:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Verify it applied:

```bash
sysctl net.ipv4.ip_forward
```

Auto-detect the active WAN interface and add NAT (MASQUERADE):

```bash
INTERFACE=$(ip route get 1.1.1.1 | awk '{print $5}')
echo "$INTERFACE"
sudo iptables -t nat -A POSTROUTING -o "$INTERFACE" -j MASQUERADE
```

Verify NAT rule exists (recommended):

```bash
sudo iptables -t nat -S POSTROUTING
```

Persist rules:

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

---

## 🌐 Router Configuration (Mandatory)

Router → LAN / DHCP Settings

### DNS (mandatory)

* Primary DNS:

```text
192.168.1.100
```

* Secondary DNS:

```text
(empty) or same as Primary
```

* WAN DNS:

```text
DO NOT USE
```

### Gateway (mandatory for DPI bypass)

If you want DPI bypass to apply to LAN clients, also set:

```text
Default Gateway / Router / Gateway: 192.168.1.100
```

Reboot router and renew client leases.

---

## 📱 Client Usage

Mobile:

```text
Install WireGuard → Scan QR using: pivpn -qr
```

Desktop:

```text
Import .conf files from: ~/wireguard/configs/
```

---

## 🧪 Verification

### DNS verification

Unbound test:

```bash
dig google.com @127.0.0.1 -p 5335
```

Pi-hole local test:

```bash
dig google.com @127.0.0.1
```

### Gateway verification (DPI bypass prerequisite)

On a LAN client, verify default route:

* Windows:

```bat
ipconfig /all
```

* Linux/macOS:

```bash
ip route | head
```

Expected:

```text
default via 192.168.1.100
```

### Zapret verification (server-side)

```bash
sudo service zapret status
ps aux | grep -E "nfqws|zapret" | grep -v grep
```

---

## ⚠️ Notes

* IPv6 intentionally disabled (better DPI bypass compatibility)
* Never configure public DNS anywhere (router, clients, VPN) if you want a single-DNS architecture
* **DPI bypass only works for traffic that flows through the Zapret host** (gateway/PBR requirement)

---

## 📚 Credits

```text
- Evrim Ağacı TeknoBilim – “Bu Kara Kutu İnternetinizi Düzeltiyor”
- bol-van/zapret
- Pi-hole, Unbound, PiVPN projects
```

---

## 🧠 Philosophy

```text
If you don’t run your own DNS, you don’t own your internet.
```

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

```text
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

ÖNEMLİ (DPI için):
→ LAN cihazlarının trafiği sunucudan geçmeli.
   Yani sunucu, istemciler için ağ geçidi (default gateway) olmalı
   (ya da router, seçili trafiği policy-based routing ile sunucuya yönlendirmeli).
```

---

## ⚙️ Gereksinimler

* Debian / Ubuntu tabanlı sistem (Raspberry Pi önerilir)
* Statik IP (örn: `192.168.1.100`)
* LAN DNS ayarı yapılabilen router
* **(DPI bypass) İstemcilerin trafiğini sunucudan geçirecek routing / gateway ayarı**
* sudo / root erişimi

---

## 🚀 Kurulum

### 1) Sistem Güncelleme

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 2) Pi-hole

Kurulum:

```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

Kurulum sırasında:

* Statik IP kabul et
* Upstream DNS seçme (Unbound kullanılacak)

Panel:

```text
http://STATIK_IP/admin
```

Durum kontrol (opsiyonel):

```bash
pihole status
```

---

### 3) Unbound (Recursive DNS)

Kurulum + root hints:

```bash
sudo apt install unbound -y
wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints
```

Config dosyasını oluştur:

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

İçeriği yapıştır:

```conf
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
```

Kaydet/çık:

```text
Ctrl + O  → Enter  → Ctrl + X
```

Config kontrolü (önerilir):

```bash
sudo unbound-checkconf
```

Servisi restart et ve kontrol et:

```bash
sudo service unbound restart
sudo service unbound status
```

Unbound test:

```bash
dig google.com @127.0.0.1 -p 5335
```

---

### 4) Pi-hole → Unbound

Pi-hole panel:

```text
http://STATIK_IP/admin
```

Menü:

```text
Settings → DNS
```

Yapılacaklar:

* Varsayılan upstream DNS sağlayıcılarının hepsini kapat
* **Custom 1 (IPv4)** alanına şunu gir:

```text
127.0.0.1#5335
```

Kaydet.

Pi-hole test (opsiyonel):

```bash
dig google.com @127.0.0.1
```

---

## 🌐 DPI İçin Gateway Şartı (KRİTİK)

Zapret `nfqws` sadece **sunucudan geçen** trafiğe etki eder.
LAN cihazları internete router üzerinden direkt çıkıyorsa DPI bypass uygulanmaz.

Bu yüzden LAN istemcilerinin **default gateway**’i sunucu olmalı **veya**
router policy-based routing (PBR) ile seçili trafiği sunucuya yönlendirmeli.

### Seçenek A — Önerilen: DHCP ile Gateway’i sunucu yap (Router)

Router’ın DHCP/LAN ayarlarında şunları ver:

```text
Default Gateway / Router / Gateway: 192.168.1.100   (sunucu IP)
Primary DNS:                     192.168.1.100   (sunucu IP)
Secondary DNS:                   boş (veya aynı)
```

Sonra istemciler DHCP yenilesin (veya cihazları yeniden başlat).

İstemci üstünde kontrol:

* Windows:

```bat
ipconfig /all
```

* Linux/macOS:

```bash
ip route | head
```

Beklenen:

```text
default via 192.168.1.100
```

### Seçenek B — Gelişmiş: Router Policy-Based Routing (PBR)

Tüm ağı gateway değiştirmek istemiyorsan router üzerinde belirli cihazları / portları (örn `80/443`) `192.168.1.100` üzerinden çıkacak şekilde yönlendir.

(Adımlar router firmware’ine göre değişir.)

---

### 5) Zapret (DPI Bypass)

Bağımlılıkları kur:

```bash
sudo apt update
sudo apt install curl git ipset nftables -y
```

Zapret’i indir ve kur:

```bash
cd ~
git clone https://github.com/bol-van/zapret.git
cd zapret
sudo ./install_easy.sh
```

Config aç:

```bash
sudo nano /opt/zapret/config
```

Dosyanın içinde (nano arama: `Ctrl + W`) şu değişkenleri bul ve bu hale getir.
Yoksa uygun bir yere ekle:

```conf
MODE="nfqws"
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"
```

Kaydet/çık:

```text
Ctrl + O  → Enter  → Ctrl + X
```

Servisi yeniden başlat ve kontrol et:

```bash
sudo service zapret restart
sudo service zapret status
```

nfqws çalışıyor mu kontrol (opsiyonel):

```bash
ps aux | grep -E "nfqws|zapret" | grep -v grep
```

Log kontrol (opsiyonel):

```bash
sudo journalctl -u zapret -n 200 --no-pager
```

---

### 6) PiVPN (WireGuard)

Kurulum:

```bash
curl -L https://install.pivpn.io | bash
```

İstemci ekleme:

```bash
pivpn add
pivpn -qr
```

İstemci listesi (opsiyonel):

```bash
pivpn list
```

---

### 7) 🔁 Kalıcılık (ÇOK ÖNEMLİ)

IPv4 yönlendirmeyi kalıcı aç:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Kontrol:

```bash
sysctl net.ipv4.ip_forward
```

Aktif arayüzü bul ve NAT kuralını ekle:

```bash
INTERFACE=$(ip route get 1.1.1.1 | awk '{print $5}')
echo "$INTERFACE"
sudo iptables -t nat -A POSTROUTING -o "$INTERFACE" -j MASQUERADE
```

Kural geldi mi kontrol (önerilir):

```bash
sudo iptables -t nat -S POSTROUTING
```

Kuralları kalıcı yap:

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

---

## 🌐 Router Ayarları

Router → LAN / DHCP Settings

### DNS (zorunlu)

* Primary DNS: Pi-hole IP (örn: `192.168.1.100`)
* Secondary DNS: boş (veya aynı)
* WAN DNS: kullanma

### Gateway (DPI için zorunlu)

DPI bypass LAN cihazlarına da uygulanacaksa:

```text
Default Gateway / Router / Gateway: 192.168.1.100
```

---

## 🧪 Doğrulama

DNS test:

```bash
dig google.com @127.0.0.1 -p 5335
dig google.com @127.0.0.1
```

Gateway kontrol (DPI ön koşulu):

* Windows:

```bat
ipconfig /all
```

* Linux/macOS:

```bash
ip route | head
```

Beklenen:

```text
default via 192.168.1.100
```

Zapret kontrol (sunucu üstünde):

```bash
sudo service zapret status
ps aux | grep -E "nfqws|zapret" | grep -v grep
```

---

## 🧠 Felsefe

```text
DNS senin değilse, internet de senin değildir.
```

---
