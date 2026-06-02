---
name: vpn-node-setup
description: >-
  Guides through setting up a personal VPN/proxy node on a VPS. Covers Shadowsocks,
  Xray VLESS+Reality, Hysteria2, and V2ray VMess+WS+TLS+Nginx. Triggers on "搭建VPN",
  "搭建梯子", "搭建节点", "科学上网", "翻墙", "setup vpn node", "deploy proxy",
  "v2ray", "xray", "hysteria2", "shadowsocks", "reality protocol", "VPS 翻墙",
  "一键脚本 vpn", "机场搭建", "自建节点", "private vpn server",
  "回国节点", "回国线路", "大陆节点", "国内节点", "回国代理", "反向代理".
---

# VPN 节点搭建指南 / VPN Node Setup Guide

基于 sucong426/V2ray-WS-TLS-Web、clown-coding/vpn 项目及 2025 年最新方案整理。

---

## 一、快速决策：选哪个方案？

| 方案 | 需要域名 | 速度 | 抗封锁 | 难度 | 推荐场景 |
|------|---------|------|--------|------|---------|
| **Shadowsocks 极简** | ❌ 不需要 | ⭐⭐⭐ | ⭐⭐ | ⭐ | **5 分钟入门**，最省资源 |
| **Xray VLESS+Reality** | ❌ 不需要 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | **首选推荐**，最省心 |
| **Hysteria2** | ⚠️ 建议有 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | 追求极致速度 |
| **V2ray VMess+WS+TLS+Nginx** | ✅ 必须 | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | 经典方案(sucong426) |

> **2025 年建议：新手先用 Shadowsocks 快速跑通，然后升级到 Xray VLESS+Reality 长期使用。** Reality 协议无需域名、无需 TLS 证书，流量特征与正常 HTTPS 无法区分。

---

### 1.1 特殊场景：回国节点（大陆 IP）

**如果你已经有一台大陆 VPS（阿里云/腾讯云国内版等），想在海外时通过它访问大陆限定内容（爱奇艺、网易云、银行等），这就是"回国节点"。**

| | 翻墙节点 | 回国节点 |
|---|---|---|
| 服务器位置 | 境外 VPS | **大陆 VPS（阿里云/腾讯云国内版）** |
| 流量方向 | 国内→境外 | 境外→国内 |
| GFW 风险 | 高，需要伪装 | **几乎为零**（出站不管） |
| 最优方案 | Xray VLESS+Reality | **同样用 Xray VLESS+Reality** ✅ |

> **关键原则：回国节点和翻墙节点用同一套方案（233boy/Xray VLESS+Reality）。** 不要因为方向不同就切到 Shadowsocks 或其他方案。统一方案的好处：
> - 管理命令一致（`xray add`、`xray log`、`xray update`）
> - 客户端配置格式相同（都是 `vless://` 链接）
> - 你不需要记两套操作方式

**回国节点搭建流程（与翻墙节点完全相同）：**
1. SSH 登录大陆 VPS
2. 跑 233boy/Xray 一键脚本（见方案一 4.1）
3. 阿里云需在**安全组**放行对应端口
4. 装 BBR、设自动维护
5. 客户端导入 `vless://` 链接

> 唯一注意：阿里云安全组比 RackNerd 更严格，记得在**阿里云网页控制台 → ECS → 安全组**里手动添加入方向规则放行你的端口。

---

## 二、前置准备（所有方案通用）

### 2.1 购买 VPS

| 服务商 | 特点 | 推荐 |
|--------|------|------|
| **RackNerd** | 美国公司，$22/年洛杉矶，实测可用，支持支付宝 | ⭐⭐⭐⭐ |
| **Bandwagon (搬瓦工)** | CN2 GIA 线路，国内延迟低 | ⭐⭐⭐⭐ |
| **Vultr** | 日本/新加坡节点，新用户有赠金 | ⭐⭐⭐ |
| **V5.NET** | 香港/日本节点，¥15.6/月 | ⭐⭐⭐⭐ |

> **实际价格说明：** VPS 博客常见 $10/年的价格是黑五/新年促销价，平时买不到。当前实际最低价：RackNerd ~$22/年（¥13.3/月）。每年 11 月底黑五有 $10-15/年的神价，可以先买年付用着，黑五再抢便宜的。

**最低配置要求：**
- CPU: 1 核
- RAM: 512MB+
- 系统: Ubuntu 22.04+ / Debian 12 (推荐) 或 CentOS 8+
- 带宽: 不限流量或 ≥500GB/月

**💡 搬瓦工隐藏优惠码技巧（来自 clown-coding/vpn）：**

购买搬瓦工时，在套餐选择页面按 **F12** 打开开发者工具 → 按 `Ctrl+F` 搜索 **`code`** → 找到 `"Try this promo code: xxxxx"` → 复制优惠码 → 结算时填入 → 点击 **Validate Code**。通常可再省 $1-5。

**购买后的操作：**
1. 记录 VPS 的 **IP 地址**
2. 记录 **root 密码**（或配置 SSH Key）
3. 在 VPS 管理面板的**防火墙/安全组**中放行端口：`22`, `80`, `443`（Reality 方案只需放行自定义端口；Shadowsocks 方案放行你的 SS 端口如 8989）

### 2.2 域名准备（仅方案三需要）

- 推荐：腾讯云/阿里云购买 `.club` / `.xyz` 等小众域名（首年 ¥1-5）
- 在域名控制台添加 **A 记录**，指向 VPS IP
- 等待 5-10 分钟 DNS 生效

### 2.3 SSH 连接 VPS

```bash
ssh root@<你的VPS_IP>
```

Windows 用户可使用 Putty、Termius 或 VS Code Remote SSH。

---

## 三、方案零：Shadowsocks 极简入门（clown-coding 方案 🟢）

**来源：** clown-coding/vpn — 最快让 VPS 跑起来的方案，5 分钟搞定。

**特点：** 极简、省资源、不需要域名。但协议较旧，流量指纹可能被识别，适合**快速测试 VPS 或作为备用**。

### 3.1 架构

```
客户端(Shadowsocks) ──→ VPS(ss-server 裸跑)
                         │
                         ├── 加密: aes-256-cfb
                         └── BBR 加速
```

> ⚠️ Shadowsocks 没有 TLS 伪装，流量特征已被 GFW 深度识别。建议跑通后升级到方案一（Reality）。

### 3.2 一键安装

基于 teddysun 的 Shadowsocks 一键脚本：

```bash
# 安装 wget（如果还没有）
yum install wget -y

# 下载并运行 Shadowsocks 安装脚本
wget --no-check-certificate -O shadowsocks.sh \
  https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks.sh
chmod +x shadowsocks.sh
./shadowsocks.sh 2>&1 | tee shadowsocks.log
```

运行后依次输入：
1. **密码**（自定义，如 `MyPass123`）
2. **端口号**（默认 8989，或自定义）
3. **加密方式**（输入 `7` 选 **aes-256-cfb**）

安装完成后会显示连接信息：
```
IP地址:   你的VPS_IP
密码:     你设置的密码
端口:     你设置的端口
加密方式:  aes-256-cfb
```

> 📌 如果 VPS 是 Ubuntu/Debian 系统，把 `yum` 换成 `apt`：
> ```bash
> apt update && apt install wget -y
> ```

### 3.3 安装 BBR 加速

```bash
wget --no-check-certificate https://github.com/teddysun/across/raw/master/bbr.sh
chmod +x bbr.sh
./bbr.sh
```

安装完成后按提示输入 `y` 重启。重启后验证：
```bash
lsmod | grep bbr
# 看到 tcp_bbr 即成功
```

### 3.4 客户端连接

| 平台 | 客户端 |
|------|--------|
| **Windows** | Shadowsocks-Windows (GitHub: shadowsocks/shadowsocks-windows) |
| **macOS** | ShadowsocksX-NG |
| **iOS** | Shadowrocket（小火箭，$2.99）/ Potatso Lite（免费） |
| **Android** | Shadowsocks Android |

填入 **IP、端口、密码、加密方式**，启动代理即可。

> 💡 各客户端均支持 `ss://` 开头的分享链接，复制粘贴自动导入。

### 3.5 Shadowsocks 常见问题

```bash
# 检查 Shadowsocks 是否在运行
ps aux | grep ssserver

# 查看日志
less /var/log/shadowsocks.log

# 手动启动/停止
/etc/init.d/shadowsocks start
/etc/init.d/shadowsocks stop
```

| 问题 | 解决 |
|------|------|
| 连不上 | 检查 VPS 防火墙是否放行了你的端口（`ufw allow 8989`） |
| 速度慢 | 确认 BBR 已启用（`lsmod \| grep bbr`） |
| 端口被封 | 修改端口：编辑 `/etc/shadowsocks.json` → `restart` |
| IP 被墙 | 换 IP 或升级到方案一（Reality） |

---

## 四、方案一：Xray VLESS + Reality（首选 🥇）

**特点：** 无需域名、无需证书、自动伪装成正常 HTTPS、抗封锁能力最强。

### 4.1 一键安装

**脚本 A — 233boy/Xray（实测可用，推荐）：**

```bash
bash <(curl -s -L https://github.com/233boy/Xray/raw/main/install.sh)
```

> 如果 `curl` 报错，改用 wget 版本：
> ```bash
> bash <(wget -qO- -o- https://github.com/233boy/Xray/raw/main/install.sh)
> ```

安装过程：出现使用协议后按 **1** 开始安装，等 3-5 分钟，看到 `vless://` 链接即成功。

安装后管理命令：
```bash
xray add        # 添加新用户/节点
xray change     # 修改现有配置
xray del        # 删除用户
xray log        # 查看日志
xray update     # 更新 Xray
```

**脚本 B — sindricn/s-xray（2025 年新版，功能更强）：**

```bash
curl -fsSL https://raw.githubusercontent.com/sindricn/s-xray/main/install.sh | sudo bash
```

**脚本 C — 极简纯净版（仅 Reality）：**

```bash
curl -O https://raw.githubusercontent.com/ADALIY1/xray-reality-installer/refs/heads/main/install_xray-reality.sh
chmod +x install_xray-reality.sh
./install_xray-reality.sh my-node 443 www.microsoft.com
```

### 4.2 Reality 关键参数说明

- **SNI（伪装域名）：** 推荐 `www.microsoft.com`、`www.cloudflare.com`、`dl.google.com`
- **端口：** 默认 443，也可用 8443、2096 等
- **Flow：** 默认 `xtls-rprx-vision`，最新且最优
- **Dest：** 回落目标，默认 `127.0.0.1:8080`

### 4.3 安装 BBR 加速

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/charmtv/bbr/main/bbr.sh)
```

选对应选项安装内核并启用 BBR，完成后重启 VPS。

---

## 五、方案二：Hysteria2（极速 🚀）

**特点：** 基于 QUIC 协议，速度快延迟低，支持端口跳跃。

### 5.1 一键安装

**脚本 A — Hi Hysteria（最全面，⭐3.2k+）：**

```bash
bash <(curl -fsSL https://git.io/hysteria.sh)
```

安装后输入 `hihy` 进入管理菜单。

**脚本 B — Sing-Box Hysteria2 + Reality 共存：**

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/charmtv/sing-box/main/hysteria2_reality.sh)
```

### 5.2 关键配置

- **端口跳跃：** 设置端口范围如 `20000-50000`，客户端随机切换端口防封锁
- **伪装：** 建议使用 `proxy` 模式伪装成正常代理流量
- **证书：** 自动 ACME 申请或自签证书，也可无域名 IP 模式

### 5.3 防火墙放行 UDP

```bash
# 如果使用 UFW
ufw allow 443/udp
ufw allow 20000:50000/udp

# 如果使用 firewalld
firewall-cmd --add-port=443/udp --permanent
firewall-cmd --add-port=20000-50000/udp --permanent
firewall-cmd --reload
```

---

## 六、方案三：V2ray VMess + WS + TLS + Nginx（sucong426 方案）

**特点：** 经典方案，通过 Nginx 反向代理伪装成正常网站。

### 6.1 架构

```
客户端 → HTTPS(443) → Nginx(TLS解密)
                        ├── /你的路径 → V2Ray(本地端口)
                        └── 其他请求 → 伪装网站
```

### 6.2 一键安装（sucong426 原始脚本）

```bash
curl -O https://raw.githubusercontent.com/luyiming1016/ladderbackup/master/v2ray_ws_tls.sh && chmod +x v2ray_ws_tls.sh && ./v2ray_ws_tls.sh
```

1. 选择选项 **1** 开始安装
2. 等待 5-6 分钟安装 Nginx
3. 输入你已解析到 VPS 的**域名**
4. 安装完成后**保存配置信息**（UUID、端口、路径、alterId 等）
5. 浏览器访问域名，看到伪装网站即成功

### 6.3 备选脚本（更现代）

**wulabing 经典脚本：**

```bash
wget -N --no-check-certificate -q -O install.sh "https://raw.githubusercontent.com/wulabing/V2Ray_ws-tls_bash_onekey/master/install.sh" && chmod +x install.sh && bash install.sh
```

**xyz690 小白友好脚本（2025 已验证）：**

```bash
bash <(curl -s -L https://raw.githubusercontent.com/xyz690/v2ray/master/onestep.sh)
```

### 6.4 安装 BBR

```bash
cd /usr/src && wget -N --no-check-certificate "https://raw.githubusercontent.com/luyiming1016/ladderbackup/master/tcp.sh" && chmod +x tcp.sh && ./tcp.sh
```

- 选 **1** 安装内核 → 重启 VPS
- 重新登录后再次执行 `./tcp.sh`，选 **5** 启用 BBR

### 6.5 手动启动 V2Ray

```bash
service v2ray start
systemctl enable v2ray   # 设置开机自启
```

---

## 七、客户端配置

### 7.1 Windows

| 客户端 | 适用协议 | 推荐度 |
|--------|---------|--------|
| **v2rayN** | VMess / VLESS / Reality / Hysteria2 | ⭐⭐⭐⭐⭐ |
| **Shadowsocks** | SS / SS-libev | ⭐⭐⭐ |
| **NekoRay** | VLESS / Reality | ⭐⭐⭐⭐ |
| **Clash Verge** | 通用 | ⭐⭐⭐⭐ |

**v2rayN 配置步骤：**
1. 下载 v2rayN（GitHub: 2dust/v2rayN）
2. 服务器 → 添加服务器 → 选择协议类型
3. 填入地址/IP、端口、UUID、Flow 等参数
4. Reality 方案：`流控(flow)` 填 `xtls-rprx-vision`，`伪装域名(sni)` 填脚本输出值，`指纹(fingerprint)` 选 `chrome`
5. 右键托盘图标 → 系统代理 → 全局模式
6. 或复制 `vless://` / `vmess://` 开头的分享链接，v2rayN 会自动识别导入

### 7.2 macOS

| 客户端 | 适用协议 | 推荐度 |
|--------|---------|--------|
| **ShadowsocksX-NG** | SS / SS-libev | ⭐⭐⭐ |
| **V2rayU** | VMess / VLESS | ⭐⭐⭐⭐ |
| **ClashX Pro** | 通用 | ⭐⭐⭐⭐⭐ |
| **NekoRay** | VLESS / Reality | ⭐⭐⭐⭐ |

### 7.3 iOS

| 客户端 | 价格 | 下载方式 |
|--------|------|---------|
| **Shadowrocket (小火箭)** | $2.99 | 美区 App Store |
| **Stash** | $3.99 | 美区 App Store |
| **sing-box (SFM)** | 免费 | App Store |

**iOS 快速配置：** 复制 `vless://` 或 `vmess://` 链接，打开 Shadowrocket → 自动识别粘贴。

> 💡 多个节点（主用/备用）在 Shadowrocket 里显示一样？长按节点 → 编辑 → 改备注名即可区分。

### 7.4 Android

| 客户端 | 推荐度 |
|--------|--------|
| **v2rayNG** | ⭐⭐⭐⭐⭐ |
| **NekoBox** | ⭐⭐⭐⭐ |
| **Hiddify-Next** | ⭐⭐⭐⭐ |

### 7.5 通用配置要素

无论哪个客户端，以下信息必须正确填入：

| 字段 | 说明 |
|------|------|
| **地址 (Address)** | VPS IP 或域名 |
| **端口 (Port)** | 脚本输出的端口号 |
| **UUID / 密码** | 脚本输出的用户ID |
| **传输协议** | TCP / WebSocket / QUIC |
| **TLS** | 开启 (Reality 选 reality) |
| **SNI / 伪装域名** | Reality方案必填 |
| **指纹 (Fingerprint)** | chrome (Reality方案) |
| **路径 (Path)** | WS方案必填 (如 /ray) |

---

## 八、常见问题排查

### 8.1 连不上节点

```bash
# 1. 检查服务是否运行
## Xray / V2Ray
systemctl status xray
systemctl status v2ray

## Shadowsocks
ps aux | grep ssserver
/etc/init.d/shadowsocks status

# 2. 查看日志
## Xray / V2Ray
journalctl -u xray -f
xray log

## Shadowsocks
less /var/log/shadowsocks.log

# 3. 检查端口是否监听
netstat -tlnp | grep <你的端口>
ss -tlnp | grep <你的端口>

# 4. 检查防火墙
ufw status
firewall-cmd --list-ports

# 5. 验证 BBR 是否生效
lsmod | grep bbr
sysctl net.ipv4.tcp_congestion_control
```

### 8.2 常见原因

| 问题 | 可能原因 | 解决 |
|------|---------|------|
| 完全连不上 | 防火墙未放行端口 | 检查 VPS 安全组 + 系统防火墙 |
| 连上了但没网 | 客户端时间不准 | 同步系统时间 (`timedatectl`) |
| 间歇性断开 | VPS IP 被墙 | 换 IP 或换端口 |
| 速度慢 | 没有开 BBR | 安装 BBR 加速 |
| Reality 连不上 | SNI 域名被封 | 换 SNI (如 www.apple.com) |
| Shadowsocks 端口被封 | SS 流量被识别 | 换端口，或升级到 Reality 方案 |
| SS 密码或加密方式不匹配 | 客户端配置错误 | 检查 `/etc/shadowsocks.json` 确认参数 |

### 8.3 换端口/IP

```bash
# Xray (233boy)
xray change

# 手动换端口：修改 /usr/local/etc/xray/config.json
# 然后重启
systemctl restart xray
```

### 8.4 测试延迟和速度

```bash
# 在本地电脑上 ping VPS
ping <你的VPS_IP>

# 测试 TCP 连通性
tcping <VPS_IP> <端口>

# 在 VPS 上测速
curl -s https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py | python3 -
```

---

## 九、最佳实践

1. **定期更新：** 每月执行 `xray update` 或 `apt update && apt upgrade`
2. **不要分享节点：** 个人使用，分享增加被封风险
3. **备选端口：** 装完立刻 `xray add` 建一个备用端口（如 8443），存好链接
4. **流量伪装：** 不用时断开连接，不要 7×24 小时跑大流量
5. **监控 VPS：** 安装 `htop`、`vnstat` 监控资源和流量
6. **备份配置：** 保存好 `vless://` 链接，重建时可直接复用

### 9.1 设置自动维护（一劳永逸）

部署完成后立即执行，以后不用再管：

```bash
apt install unattended-upgrades needrestart -y
echo '0 3 1 * * root xray update && systemctl restart xray' > /etc/cron.d/xray-update
echo '0 4 * * 0 root apt update && apt upgrade -y && apt autoremove -y && apt autoclean' > /etc/cron.d/apt-update
```

> 三条命令一条一条粘贴，不要多行一起粘贴。完成后用 `cat /etc/cron.d/xray-update /etc/cron.d/apt-update` 验证。

之后你的服务器会自己更新、自己清理，再也不用 SSH 进去维护。

---

## 十、快速参考：完整搭建流程

```
1. 买 VPS → 获取 IP + root 密码
   └── 搬瓦工技巧: F12 搜索 "code" 找隐藏优惠码
2. (方案三需要) 买域名 → DNS A 记录指向 VPS IP
3. SSH 登录 VPS
4. 复制粘贴对应方案的一键脚本 → 回车
5. 记录输出的配置参数（截图或复制）
6. 安装 BBR 加速脚本
7. 重启 VPS
8. 下载对应平台的客户端
9. 填入配置参数（或导入分享链接）
   └── SS: 填 IP/端口/密码/加密 → ss://链接
   └── Xray/V2ray: 导入 vmess:// 或 vless:// 链接
10. 连接 → 打开 google.com 测试
```

---

## 十一、免责声明

- 本指南仅供技术学习和研究网络协议使用
- 使用前请了解并遵守所在地区的法律法规
- 脚本来自开源社区，使用前请审查脚本内容
- 不要在 VPN 连接下进行违法活动
