# Ultimate Privacy Server (Beginner-Friendly)

Pi-hole + Unbound + PiVPN (WireGuard) + Zapret (nfqws) on one Linux server.

This guide is written so you can complete setup step-by-step without needing extra documentation.

## 🌍 Languages

- [English](#english)
- [中文](#chinese)
- [Türkçe](#turkce)

---

<a name="english"></a>

---

## 0) What You Are Building

- Network-wide ad/tracker blocking: `Pi-hole`
- Recursive DNS (no ISP/Google upstream): `Unbound`
- VPN access from outside home: `PiVPN (WireGuard)`
- DPI bypass for traffic that passes through this server: `Zapret (nfqws)`

Traffic flow:

```text
LAN client -> Pi-hole:53 -> Unbound:127.0.0.1:5335 -> Root DNS
VPN client -> WireGuard -> same Pi-hole/Unbound DNS
Optional DPI bypass -> only works if client traffic passes through this server
```

---

## 1) Before You Start (Do Not Skip)

You need:

- Debian/Ubuntu server (Raspberry Pi is fine)
- Static server IP (example: `192.168.1.100`)
- Router admin access
- `sudo` access

Important safety note:

- Do **DNS setup first**.
- Only enable "server as default gateway" after DNS is confirmed working.
- If clients lose internet, rollback by setting router DHCP gateway back to the router IP.

Set variables once (change values to your network):

```bash
export SERVER_IP="192.168.1.100"
export ROUTER_IP="192.168.1.1"
```

Find your active WAN interface (needed later for NAT):

```bash
export WAN_IFACE="$(ip route get 1.1.1.1 | awk '{print $5; exit}')"
echo "$WAN_IFACE"
```

---

## 2) Update System

```bash
sudo apt update && sudo apt upgrade -y
```

Install base tools:

```bash
sudo apt install -y curl git wget dnsutils iptables-persistent unbound ipset nftables
```

---

## 3) Install Pi-hole

```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

During installer:

- Keep static IP enabled
- Upstream DNS choice is temporary (we switch to Unbound next)

Check Pi-hole service:

```bash
pihole status
```

Open admin panel:

```text
http://SERVER_IP/admin
```

---

## 4) Install and Configure Unbound

Get root hints:

```bash
wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints >/dev/null
```

Create config:

```bash
sudo tee /etc/unbound/unbound.conf.d/pi-hole.conf >/dev/null <<'EOF'
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
  qname-minimisation: yes

  private-address: 10.0.0.0/8
  private-address: 172.16.0.0/12
  private-address: 192.168.0.0/16
EOF
```

Validate and restart:

```bash
sudo unbound-checkconf
sudo systemctl restart unbound
sudo systemctl enable unbound
sudo systemctl --no-pager --full status unbound
```

Direct test:

```bash
dig +short google.com @127.0.0.1 -p 5335
```

If you get IP addresses, Unbound works.

---

## 5) Connect Pi-hole to Unbound

In Pi-hole UI:

`Settings -> DNS`

Do this:

- Uncheck all public upstream providers
- Set `Custom 1 (IPv4)` to `127.0.0.1#5335`
- Save

Test Pi-hole local resolver:

```bash
dig +short google.com @127.0.0.1
```

If this returns IPs, DNS chain is good.

---

## 6) Router DNS (First Rollout)

On router DHCP/LAN settings:

- Primary DNS: `SERVER_IP`
- Secondary DNS: empty (or same as primary)

Do **not** change gateway yet. First verify clients can browse and ads are blocked.

Client check:

```bash
nslookup pi.hole
```

or check client DNS server shows `SERVER_IP`.

---

## 7) Install Zapret (DPI Bypass)

```bash
cd ~
git clone https://github.com/bol-van/zapret.git
cd zapret
sudo ./install_easy.sh
```

Edit config:

```bash
sudo nano /opt/zapret/config
```

Set (or add):

```conf
MODE="nfqws"
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"
```

Restart and verify:

```bash
sudo systemctl restart zapret
sudo systemctl enable zapret
sudo systemctl --no-pager --full status zapret
ps aux | grep -E "nfqws|zapret" | grep -v grep
```

Critical reminder:

- Zapret affects only traffic that goes through this server.

---

## 8) Install PiVPN (WireGuard)

```bash
curl -L https://install.pivpn.io | sudo bash
```

During installer:

- Select `WireGuard`
- Default port `51820` is fine
- When asked DNS provider, choose Pi-hole/local option if shown

Create client profile:

```bash
pivpn add
pivpn -qr
```

Optional:

```bash
pivpn list
```

## 8.1) Router Port Forwarding for WireGuard (Required)

Without port forwarding, VPN usually works only inside your home LAN.

Add this rule in router `NAT / Port Forwarding`:

| LAN IP      | Start Port | End Port | Source IP | Source Start | Source End | Protocol | Description | Enabled |
| ----------- | ---------- | -------- | --------- | ------------ | ---------- | -------- | ----------- | ------- |
| `SERVER_IP` | `51820`    | `51820`  | `0.0.0.0` | `51820`      | `51820`    | `UDP`    | `PiVPN`     | `Yes`   |

---

## 9) Make Routing Reboot-Proof

Enable IP forwarding permanently:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-privacy-server.conf >/dev/null
sudo sysctl --system
sysctl net.ipv4.ip_forward
```

Add NAT (idempotent, avoids duplicates):

```bash
sudo iptables -t nat -C POSTROUTING -o "$WAN_IFACE" -j MASQUERADE 2>/dev/null || \
sudo iptables -t nat -A POSTROUTING -o "$WAN_IFACE" -j MASQUERADE
```

Persist firewall rules:

```bash
sudo netfilter-persistent save
sudo netfilter-persistent reload
sudo iptables -t nat -S POSTROUTING
```

---

## 10) Enable DPI Bypass for LAN Clients (Second Rollout)

Only do this after all previous tests pass.

### Option A: Full LAN through server (simple, high impact)

In router DHCP settings:

- Default Gateway/Router: `SERVER_IP`
- Primary DNS: `SERVER_IP`

Renew DHCP on clients or reboot clients.

Verify on client:

- Linux/macOS:

```bash
ip route | head -n 2
```

- Windows:

```bat
ipconfig /all
```

Expected route:

```text
default via SERVER_IP
```

### Option B: Policy-Based Routing (safer rollout)

Route only selected clients/ports (`80/443`) via `SERVER_IP` from router PBR rules.

---

## 11) Verification Checklist

Run these on server:

```bash
dig +short cloudflare.com @127.0.0.1 -p 5335
dig +short cloudflare.com @127.0.0.1
sudo systemctl is-active unbound pihole-FTL zapret
sudo iptables -t nat -S POSTROUTING
```

Run these on a LAN client:

- DNS server = `SERVER_IP`
- If DPI mode enabled: default gateway = `SERVER_IP`
- Confirm normal websites + previously blocked/censored target behavior

Run these for VPN:

```bash
pivpn -c
```

Connect with WireGuard client and confirm:

- Internet works
- DNS queries resolve through home server

---

## 12) Fast Rollback (If Something Breaks)

If clients lose internet after gateway change:

1. In router DHCP, set gateway back to router IP (`ROUTER_IP`)
2. Renew DHCP lease on clients (or reboot clients)
3. Keep DNS as `SERVER_IP` if Pi-hole still works

If Unbound fails:

```bash
sudo unbound-checkconf
sudo journalctl -u unbound -n 100 --no-pager
```

If Zapret fails:

```bash
sudo journalctl -u zapret -n 200 --no-pager
```

---

## Notes

- IPv6 is intentionally disabled in this setup for simpler DPI behavior.
- Do not hardcode public DNS on router/clients/VPN if you want single-DNS architecture.
- For best stability, apply changes in two phases:
  1. DNS phase (Pi-hole + Unbound)
  2. Traffic phase (gateway/PBR + Zapret)

---

<a name="chinese"></a>

# 中文指南（新手友好版）

在一台 Linux 服务器上部署：Pi-hole + Unbound + PiVPN (WireGuard) + Zapret (nfqws)。

本指南按“可直接照做”的方式编写，新手也可以一步一步完成整套部署。

---

## 0) 你将搭建什么

- 全网广告/追踪拦截：`Pi-hole`
- 递归 DNS（不依赖 ISP/Google）：`Unbound`
- 外网安全接入：`PiVPN (WireGuard)`
- DPI 绕过（仅对经过本机的流量生效）：`Zapret (nfqws)`

流量路径：

```text
局域网客户端 -> Pi-hole:53 -> Unbound:127.0.0.1:5335 -> Root DNS
VPN 客户端 -> WireGuard -> 同样走 Pi-hole/Unbound
可选 DPI 绕过 -> 仅对经过本服务器的流量有效
```

---

## 1) 开始前（不要跳过）

你需要：

- Debian/Ubuntu 服务器（树莓派可用）
- 服务器静态 IP（示例：`192.168.1.100`）
- 路由器管理权限
- `sudo` 权限

安全提示：

- **先完成 DNS，再动网关**。
- 只有 DNS 验证成功后，才把服务器设为默认网关。
- 如果客户端断网，先把路由器 DHCP 网关改回路由器本机 IP。

先设置变量（按你的网络修改）：

```bash
export SERVER_IP="192.168.1.100"
export ROUTER_IP="192.168.1.1"
```

获取当前出口网卡（后面 NAT 需要）：

```bash
export WAN_IFACE="$(ip route get 1.1.1.1 | awk '{print $5; exit}')"
echo "$WAN_IFACE"
```

---

## 2) 更新系统

```bash
sudo apt update && sudo apt upgrade -y
```

安装基础工具：

```bash
sudo apt install -y curl git wget dnsutils iptables-persistent unbound ipset nftables
```

---

## 3) 安装 Pi-hole

```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

安装过程中：

- 保持静态 IP
- 上游 DNS 先临时选，后面会切到 Unbound

检查 Pi-hole 状态：

```bash
pihole status
```

管理面板：

```text
http://SERVER_IP/admin
```

---

## 4) 安装并配置 Unbound

下载 root hints：

```bash
wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints >/dev/null
```

创建配置：

```bash
sudo tee /etc/unbound/unbound.conf.d/pi-hole.conf >/dev/null <<'EOF'
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
  qname-minimisation: yes

  private-address: 10.0.0.0/8
  private-address: 172.16.0.0/12
  private-address: 192.168.0.0/16
EOF
```

检查并重启：

```bash
sudo unbound-checkconf
sudo systemctl restart unbound
sudo systemctl enable unbound
sudo systemctl --no-pager --full status unbound
```

直接测试：

```bash
dig +short google.com @127.0.0.1 -p 5335
```

有 IP 返回即正常。

---

## 5) 让 Pi-hole 使用 Unbound

在 Pi-hole 界面：

`Settings -> DNS`

操作：

- 取消勾选所有公共上游 DNS
- `Custom 1 (IPv4)` 填 `127.0.0.1#5335`
- 保存

测试 Pi-hole 本地解析：

```bash
dig +short google.com @127.0.0.1
```

有 IP 返回表示 DNS 链路正常。

---

## 6) 路由器 DNS（第一阶段上线）

在路由器 DHCP/LAN 设置：

- Primary DNS: `SERVER_IP`
- Secondary DNS: 留空（或同 Primary）

**先不要改默认网关**。先确认客户端上网和广告拦截都正常。

客户端检查：

```bash
nslookup pi.hole
```

或确认客户端 DNS 服务器为 `SERVER_IP`。

---

## 7) 安装 Zapret（DPI 绕过）

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

设置（或新增）：

```conf
MODE="nfqws"
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"
```

重启并验证：

```bash
sudo systemctl restart zapret
sudo systemctl enable zapret
sudo systemctl --no-pager --full status zapret
ps aux | grep -E "nfqws|zapret" | grep -v grep
```

关键点：

- Zapret 只会处理经过本服务器的流量。

---

## 8) 安装 PiVPN（WireGuard）

```bash
curl -L https://install.pivpn.io | sudo bash
```

安装时：

- 选择 `WireGuard`
- 端口 `51820` 默认即可
- 如果有 DNS 选项，优先选 Pi-hole/本地 DNS

添加客户端：

```bash
pivpn add
pivpn -qr
```

可选：

```bash
pivpn list
```

## 8.1) 路由器端口转发（WireGuard 必需）

不做端口转发时，VPN 通常只能在家里局域网内可用。

在路由器 `NAT / Port Forwarding` 中添加规则：

| 内网 IP     | 起始端口 | 结束端口 | 来源 IP   | 来源起始端口 | 来源结束端口 | 协议  | 说明    | 启用 |
| ----------- | -------- | -------- | --------- | ------------ | ------------ | ----- | ------- | ---- |
| `SERVER_IP` | `51820`  | `51820`  | `0.0.0.0` | `51820`      | `51820`      | `UDP` | `PiVPN` | `是` |

---

## 9) 做到重启不丢

永久开启 IP 转发：

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-privacy-server.conf >/dev/null
sudo sysctl --system
sysctl net.ipv4.ip_forward
```

添加 NAT（幂等，避免重复）：

```bash
sudo iptables -t nat -C POSTROUTING -o "$WAN_IFACE" -j MASQUERADE 2>/dev/null || \
sudo iptables -t nat -A POSTROUTING -o "$WAN_IFACE" -j MASQUERADE
```

持久化规则：

```bash
sudo netfilter-persistent save
sudo netfilter-persistent reload
sudo iptables -t nat -S POSTROUTING
```

---

## 10) 给局域网启用 DPI 绕过（第二阶段上线）

必须在前面全部验证通过后再做。

### 方案 A：全局走服务器（简单但影响面大）

路由器 DHCP 设置：

- Default Gateway/Router: `SERVER_IP`
- Primary DNS: `SERVER_IP`

让客户端续租 DHCP 或重启。

客户端验证：

- Linux/macOS：

```bash
ip route | head -n 2
```

- Windows：

```bat
ipconfig /all
```

期望：

```text
default via SERVER_IP
```

### 方案 B：策略路由 PBR（更稳妥）

仅把指定设备/端口（如 `80/443`）通过 `SERVER_IP` 转发。

---

## 11) 验证清单

服务器执行：

```bash
dig +short cloudflare.com @127.0.0.1 -p 5335
dig +short cloudflare.com @127.0.0.1
sudo systemctl is-active unbound pihole-FTL zapret
sudo iptables -t nat -S POSTROUTING
```

客户端验证：

- DNS 服务器 = `SERVER_IP`
- 若开启 DPI 模式：默认网关 = `SERVER_IP`
- 正常网站可访问，目标站点行为符合预期

VPN 验证：

```bash
pivpn -c
```

使用 WireGuard 连接后确认：

- 能上网
- DNS 解析走家庭服务器

---

## 12) 快速回滚（出问题时）

如果改网关后客户端断网：

1. 路由器 DHCP 网关改回 `ROUTER_IP`
2. 客户端续租 DHCP（或重启）
3. 若 Pi-hole 正常，可保留 DNS 为 `SERVER_IP`

Unbound 故障排查：

```bash
sudo unbound-checkconf
sudo journalctl -u unbound -n 100 --no-pager
```

Zapret 故障排查：

```bash
sudo journalctl -u zapret -n 200 --no-pager
```

---

## 说明

- 该方案默认关闭 IPv6，以简化 DPI 行为。
- 如果要保持单一 DNS 架构，不要在路由器/客户端/VPN 手动填公共 DNS。
- 推荐按两阶段上线：
  1. DNS 阶段（Pi-hole + Unbound）
  2. 流量阶段（Gateway/PBR + Zapret）

---

<a name="turkce"></a>

# Türkçe Rehber (Yeni Başlayan Dostu)

Tek bir Linux sunucuda: Pi-hole + Unbound + PiVPN (WireGuard) + Zapret (nfqws).

Bu rehber, teknik geçmişi az olan kullanıcıların bile adım adım sorunsuz kurulum yapabilmesi için yazıldı.

---

## 0) Ne Kuruyorsun

- Ağ genelinde reklam/izleyici engelleme: `Pi-hole`
- Recursive DNS (ISS/Google yok): `Unbound`
- Ev ağına dışarıdan güvenli erişim: `PiVPN (WireGuard)`
- DPI bypass (yalnızca sunucudan geçen trafikte): `Zapret (nfqws)`

Trafik akışı:

```text
LAN istemcisi -> Pi-hole:53 -> Unbound:127.0.0.1:5335 -> Root DNS
VPN istemcisi -> WireGuard -> aynı Pi-hole/Unbound DNS
Opsiyonel DPI bypass -> sadece sunucudan geçen trafikte çalışır
```

---

## 1) Başlamadan Önce (Atlama)

Gerekenler:

- Debian/Ubuntu sunucu (Raspberry Pi olur)
- Sunucu için statik IP (örnek: `192.168.1.100`)
- Router yönetici erişimi
- `sudo` erişimi

Güvenli kurulum notu:

- **Önce DNS kur, sonra gateway değiştir**.
- DNS doğrulanmadan sunucuyu varsayılan ağ geçidi yapma.
- İstemciler internete çıkamazsa router DHCP gateway ayarını tekrar router IP'sine al.

Önce değişkenleri ayarla (kendi ağına göre değiştir):

```bash
export SERVER_IP="192.168.1.100"
export ROUTER_IP="192.168.1.1"
```

Aktif WAN arayüzünü bul (NAT için gerekli):

```bash
export WAN_IFACE="$(ip route get 1.1.1.1 | awk '{print $5; exit}')"
echo "$WAN_IFACE"
```

---

## 2) Sistemi Güncelle

```bash
sudo apt update && sudo apt upgrade -y
```

Temel paketleri kur:

```bash
sudo apt install -y curl git wget dnsutils iptables-persistent unbound ipset nftables
```

---

## 3) Pi-hole Kurulumu

```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

Kurulum sırasında:

- Statik IP açık kalsın
- Upstream DNS seçimi geçici (bir sonraki adımda Unbound'a geçeceğiz)

Servis kontrolü:

```bash
pihole status
```

Yönetim paneli:

```text
http://SERVER_IP/admin
```

---

## 4) Unbound Kurulumu ve Yapılandırma

Root hints dosyasını al:

```bash
wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints >/dev/null
```

Config oluştur:

```bash
sudo tee /etc/unbound/unbound.conf.d/pi-hole.conf >/dev/null <<'EOF'
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
  qname-minimisation: yes

  private-address: 10.0.0.0/8
  private-address: 172.16.0.0/12
  private-address: 192.168.0.0/16
EOF
```

Doğrula ve yeniden başlat:

```bash
sudo unbound-checkconf
sudo systemctl restart unbound
sudo systemctl enable unbound
sudo systemctl --no-pager --full status unbound
```

Doğrudan test:

```bash
dig +short google.com @127.0.0.1 -p 5335
```

IP dönüyorsa Unbound çalışıyor.

---

## 5) Pi-hole'u Unbound'a Bağla

Pi-hole arayüzünde:

`Settings -> DNS`

Şunları yap:

- Tüm varsayılan public upstream DNS kutularını kapat
- `Custom 1 (IPv4)` alanına `127.0.0.1#5335` yaz
- Kaydet

Pi-hole test:

```bash
dig +short google.com @127.0.0.1
```

IP dönüyorsa DNS zinciri doğru.

---

## 6) Router DNS Ayarı (İlk Geçiş)

Router DHCP/LAN ayarlarında:

- Primary DNS: `SERVER_IP`
- Secondary DNS: boş (veya aynı IP)

**Henüz gateway değiştirme**. Önce istemcilerde internet ve reklam engelleme doğrulansın.

İstemci kontrolü:

```bash
nslookup pi.hole
```

veya istemcinin DNS sunucusunun `SERVER_IP` olduğunu doğrula.

---

## 7) Zapret Kurulumu (DPI Bypass)

```bash
cd ~
git clone https://github.com/bol-van/zapret.git
cd zapret
sudo ./install_easy.sh
```

Config düzenle:

```bash
sudo nano /opt/zapret/config
```

Ayarla (yoksa ekle):

```conf
MODE="nfqws"
NFQWS_OPT="--filter-tcp=80,443 --dpi-desync=fake --dpi-desync-ttl=3"
```

Yeniden başlat ve doğrula:

```bash
sudo systemctl restart zapret
sudo systemctl enable zapret
sudo systemctl --no-pager --full status zapret
ps aux | grep -E "nfqws|zapret" | grep -v grep
```

Kritik not:

- Zapret sadece sunucudan geçen trafik üzerinde etki eder.

---

## 8) PiVPN (WireGuard) Kurulumu

```bash
curl -L https://install.pivpn.io | sudo bash
```

Kurulum sırasında:

- `WireGuard` seç
- `51820` varsayılan port uygundur
- DNS sorulursa Pi-hole/yerel DNS seçeneğini seç

İstemci ekle:

```bash
pivpn add
pivpn -qr
```

Opsiyonel:

```bash
pivpn list
```

## 8.1) Router Port Yonlendirme (WireGuard icin Zorunlu)

Port yonlendirme yoksa VPN genelde sadece ev icindeki LAN'da calisir.

Router `NAT / Port Forwarding` ekranina su kurali ekle:

| LAN IP      | Baslangic Portu | Bitis Portu | Kaynak IP | Baslangic Portu | Bitis Portu | Protokol | Tanim   | Etkin  |
| ----------- | --------------- | ----------- | --------- | --------------- | ----------- | -------- | ------- | ------ |
| `SERVER_IP` | `51820`         | `51820`     | `0.0.0.0` | `51820`         | `51820`     | `UDP`    | `PiVPN` | `Evet` |

---

## 9) Yeniden Başlatmaya Dayanıklı Yap

IPv4 forwarding'i kalıcı aç:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-privacy-server.conf >/dev/null
sudo sysctl --system
sysctl net.ipv4.ip_forward
```

NAT kuralını ekle (tekrar eklemeye dayanıklı):

```bash
sudo iptables -t nat -C POSTROUTING -o "$WAN_IFACE" -j MASQUERADE 2>/dev/null || \
sudo iptables -t nat -A POSTROUTING -o "$WAN_IFACE" -j MASQUERADE
```

Kuralları kalıcılaştır:

```bash
sudo netfilter-persistent save
sudo netfilter-persistent reload
sudo iptables -t nat -S POSTROUTING
```

---

## 10) LAN için DPI Bypass'ı Aç (İkinci Geçiş)

Bunu yalnızca önceki adımlar tamamen doğrulandıktan sonra yap.

### Seçenek A: Tüm LAN trafiğini sunucudan geçir (kolay, etkisi büyük)

Router DHCP ayarlarında:

- Default Gateway/Router: `SERVER_IP`
- Primary DNS: `SERVER_IP`

İstemcilerde DHCP lease yenile veya yeniden başlat.

İstemci doğrulaması:

- Linux/macOS:

```bash
ip route | head -n 2
```

- Windows:

```bat
ipconfig /all
```

Beklenen:

```text
default via SERVER_IP
```

### Seçenek B: Policy-Based Routing (daha güvenli)

Sadece belirli cihaz/portları (`80/443`) `SERVER_IP` üzerinden yönlendir.

---

## 11) Doğrulama Kontrol Listesi

Sunucuda çalıştır:

```bash
dig +short cloudflare.com @127.0.0.1 -p 5335
dig +short cloudflare.com @127.0.0.1
sudo systemctl is-active unbound pihole-FTL zapret
sudo iptables -t nat -S POSTROUTING
```

LAN istemcisinde doğrula:

- DNS sunucusu = `SERVER_IP`
- DPI modu açıksa varsayılan gateway = `SERVER_IP`
- Normal siteler açılıyor, hedeflenen erişim davranışı doğru

VPN doğrulaması:

```bash
pivpn -c
```

WireGuard ile bağlanıp doğrula:

- İnternet var
- DNS sorguları ev sunucusundan çözülüyor

---

## 12) Hızlı Geri Dönüş (Sorun Olursa)

Gateway değişiminden sonra internet kesilirse:

1. Router DHCP gateway'i tekrar `ROUTER_IP` yap
2. İstemcilerde DHCP yenile (veya yeniden başlat)
3. Pi-hole çalışıyorsa DNS'i `SERVER_IP` olarak bırakabilirsin

Unbound arıza kontrolü:

```bash
sudo unbound-checkconf
sudo journalctl -u unbound -n 100 --no-pager
```

Zapret arıza kontrolü:

```bash
sudo journalctl -u zapret -n 200 --no-pager
```

---

## Notlar

- Bu kurulumda daha sade DPI davranışı için IPv6 devre dışı bırakılır.
- Tek DNS mimarisi istiyorsan router/istemci/VPN tarafında public DNS girme.
- En stabil yaklaşım iki aşamalı geçiştir:
  1. DNS aşaması (Pi-hole + Unbound)
  2. Trafik aşaması (Gateway/PBR + Zapret)
