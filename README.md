---

# 🛡️ Ultimate Privacy Server

## Pi-hole · Unbound · PiVPN (WireGuard) · DPI Bypass (Zapret)

A **self-hosted privacy infrastructure** that provides **network-wide ad blocking**, **fully recursive DNS**, **ISP DPI / censorship bypass**, and **secure VPN access** — designed to be **reboot-proof** and survive **power outages**.

---

## 🌍 Languages

* [English](#english)
* [中文](#chinese)
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

<a name="chinese"></a>

# 中文指南 (Chinese Guide)

## ✨ 功能

* 全网广告与追踪器拦截（Pi-hole）
* 使用 Unbound 实现完全递归 DNS（不依赖 Google/运营商 DNS）
* 通过 Zapret（nfqws）实现 DPI/审查绕过
* 使用 WireGuard（PiVPN）提供高速安全 VPN
* NAT 与 IP 转发持久化（重启不丢）
* 局域网 + VPN 客户端统一 DNS 架构

---

## 🧱 架构

```text
设备
→ 路由器（DNS → Pi-hole）
→ Pi-hole（53 端口）
→ Unbound（127.0.0.1:5335）
→ 根 DNS 服务器

VPN 客户端
→ WireGuard（PiVPN）
→ 同样的 DNS 过滤

所有出口流量
→ Zapret（DPI 反制 / desync）

重要（DPI 绕过前提）：
→ 局域网客户端的流量必须经过本服务器（运行 Zapret 的主机）。
   这意味着客户端的默认网关需要指向服务器
   （或由路由器使用策略路由把指定流量转发到服务器）。
```

---

## ⚙️ 环境要求

* Debian/Ubuntu 系统（推荐 Raspberry Pi）
* 服务器使用静态 IP（例如：`192.168.1.100`）
* 路由器支持配置 LAN DNS / DHCP
* **（DPI 绕过）需要网络路由让客户端以服务器作为网关**
* sudo / root 权限

---

## 🚀 安装

### 1) 系统更新

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 2) 安装 Pi-hole（广告拦截 DNS）

安装：

```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

安装过程中：

* 接受 **静态 IP**
* **不要**选择上游 DNS 提供商（后续使用 Unbound）

管理面板：

```text
http://STATIC_IP/admin
```

状态检查（可选）：

```bash
pihole status
```

---

### 3) 安装 Unbound（递归 DNS）

安装并获取 root hints：

```bash
sudo apt install unbound -y
wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints
```

创建配置文件：

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

粘贴内容：

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

保存并退出：

```text
Ctrl + O  → Enter  → Ctrl + X
```

检查配置（推荐）：

```bash
sudo unbound-checkconf
```

重启并检查：

```bash
sudo service unbound restart
sudo service unbound status
```

直接测试 Unbound：

```bash
dig google.com @127.0.0.1 -p 5335
```

---

### 4) Pi-hole → Unbound 绑定

打开 Pi-hole 面板：

```text
http://STATIC_IP/admin
```

路径：

```text
Settings → DNS
```

操作：

* 关闭所有默认上游 DNS
* 在 **Custom 1 (IPv4)** 中填写：

```text
127.0.0.1#5335
```

保存。

可选验证：

```bash
dig google.com @127.0.0.1
```

---

## 🌐 DPI 绕过的网关要求（关键）

Zapret `nfqws` 只对**经过服务器的流量**生效。
如果局域网客户端直接通过路由器/运营商出网，DPI 绕过不会生效。

你必须确保：**客户端默认网关指向服务器**（或路由器做策略路由把流量转发到服务器）。

### 方案 A（推荐）：DHCP 下发服务器作为默认网关

在路由器 DHCP/LAN 设置中配置：

```text
Default Gateway / Router / Gateway: 192.168.1.100   (服务器 IP)
Primary DNS:                     192.168.1.100   (服务器 IP)
Secondary DNS:                   留空（或同上）
```

然后让客户端更新 DHCP（或重启）。

客户端验证默认路由：

* Windows：

```bat
ipconfig /all
```

* Linux/macOS：

```bash
ip route | head
```

期望看到：

```text
default via 192.168.1.100
```

### 方案 B（高级）：路由器策略路由（PBR）

若不希望全网改网关，可在路由器上将特定设备或端口（例如 `80/443`）通过 `192.168.1.100` 转发。

（具体配置因路由器固件而异。）

---

### 5) 安装 Zapret（DPI Bypass）

安装依赖：

```bash
sudo apt update
sudo apt install curl git ipset nftables -y
```

下载并安装：

```bash
cd ~
git clone https://github.com/bol-van/zapret.git
cd zapret
sudo ./install_easy.sh
```

编辑配置：

```bash
sudo nano /opt/zapret/config
```

在文件中找到（`Ctrl + W` 搜索）并设置以下变量；如果不存在，就添加：

```conf
MODE="nfqws"
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"
```

保存并退出：

```text
Ctrl + O  → Enter  → Ctrl + X
```

重启并检查：

```bash
sudo service zapret restart
sudo service zapret status
```

确认进程（可选）：

```bash
ps aux | grep -E "nfqws|zapret" | grep -v grep
```

查看日志（可选）：

```bash
sudo journalctl -u zapret -n 200 --no-pager
```

---

### 6) 安装 PiVPN（WireGuard）

安装：

```bash
curl -L https://install.pivpn.io | bash
```

安装时：

* 协议：**WireGuard**
* 端口：`51820`（默认）

添加客户端：

```bash
pivpn add
pivpn -qr
```

可选：列出客户端：

```bash
pivpn list
```

---

### 7) 🔁 重启不丢（关键）

开启 IPv4 转发（永久）：

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

验证：

```bash
sysctl net.ipv4.ip_forward
```

自动识别出口网卡并添加 NAT：

```bash
INTERFACE=$(ip route get 1.1.1.1 | awk '{print $5}')
echo "$INTERFACE"
sudo iptables -t nat -A POSTROUTING -o "$INTERFACE" -j MASQUERADE
```

验证规则（推荐）：

```bash
sudo iptables -t nat -S POSTROUTING
```

持久化：

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

---

## 🌐 路由器配置（必须）

路由器 → LAN / DHCP 设置

### DNS（必须）

```text
Primary DNS:   192.168.1.100
Secondary DNS: 留空（或同上）
WAN DNS:       不要配置
```

### 网关（DPI 绕过必须）

```text
Default Gateway / Router / Gateway: 192.168.1.100
```

重启路由器并让客户端更新 DHCP。

---

## 🧪 验证

DNS：

```bash
dig google.com @127.0.0.1 -p 5335
dig google.com @127.0.0.1
```

网关（DPI 前提）：

* Windows：

```bat
ipconfig /all
```

* Linux/macOS：

```bash
ip route | head
```

期望：

```text
default via 192.168.1.100
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
