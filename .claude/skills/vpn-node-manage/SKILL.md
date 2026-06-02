---
name: vpn-node-manage
description: >-
  Manage and maintain an existing Xray/V2Ray VPN node. Add/delete nodes, update
  software, check logs, troubleshoot issues, and keep the server secure. Triggers
  on "管理节点", "维护VPN", "增加节点", "删除节点", "更新xray", "VPN出问题了",
  "节点连不上", "查看日志", "vpn维护", "manage vpn node", "xray add", "xray del".
---

# VPN 节点管理 / VPN Node Management

适用于通过 233boy/Xray 一键脚本搭建的 VLESS+Reality 节点（翻墙节点 + 回国节点通用）。

---

## 零、SSH 登录服务器

所有操作都要先登录。在本地电脑打开终端（PowerShell / CMD / 终端）：

```bash
ssh root@你的服务器IP
```

输入 root 密码后回车。登录成功会看到命令行提示符。

> 忘记 IP？去 RackNerd 面板 → 点你的 VPS → 看 **Primary IP**。
> 忘记密码？面板里点 **Reset Root Password** 重置。

### 免密登录（省得每次输密码）

本机执行：
```bash
ssh-copy-id root@你的服务器IP
```

之后直接 `ssh root@IP` 就进去了，不用输密码。

---

## 一、自动维护（设置一次，永久生效）

如果还没设，SSH 登录后一条一条粘贴：

```bash
apt install unattended-upgrades needrestart -y
```

```bash
echo '0 3 1 * * root xray update && systemctl restart xray' > /etc/cron.d/xray-update
```

```bash
echo '0 4 * * 0 root apt update && apt upgrade -y && apt autoremove -y && apt autoclean' > /etc/cron.d/apt-update
```

> ⚠️ 逐条粘贴，不要多行一起贴。如有弹窗按 Tab 跳转到 Yes 然后 Enter。

### 验证是否生效

```bash
cat /etc/cron.d/xray-update /etc/cron.d/apt-update
```

看到两行 cron 配置内容就说明成功了。

### 这套自动维护做了什么

| 定时 | 功能 |
|------|------|
| 每月1号 3am | 自动更新 Xray + 重启 |
| 每周日 4am | 自动更新系统 + 清理旧包 |
| 实时 | 紧急安全补丁自动安装 |

设完之后永远不用管维护。

---

## 二、增加节点

同一台服务器上创建多套连接参数（免费、不限数量）。

```bash
xray add
```

跟着菜单选协议类型、端口号。完成后会输出新节点的 `vless://` 链接，复制到客户端即可。

### 什么时候需要增加节点？

| 场景 | 说明 |
|------|------|
| 端口被封 | 新建一个不同端口的节点，旧的不用管 |
| 要多条备用线 | 443 挂了切 8443，不用临时搞 |
| 分享给家人 | 各自用各自的端口，互不干扰 |

### 多个节点在客户端里长一样怎么办

IP 相同、协议相同，客户端默认显示服务器地址所以看起来一样。在 Shadowrocket 或 v2rayN 里长按节点 → 编辑 → 改备注：

```
主用 (443)
备用 (8443)
```

改完就能分了。

---

## 三、删除节点

```bash
xray del
```

显示节点列表 → 输入对应数字 → 回车删除。

---

## 四、查看现有节点

```bash
xray change
```

显示所有节点的端口和协议信息，但不输出完整链接。想看完整链接执行 `xray add` 还是 `xray change` 都可以翻。

---

## 五、查看日志

```bash
xray log
```

正常日志没什么有用信息。出问题的时候看最后几行，通常能看到连接失败原因。

### 实时监控（查看谁在连）

```bash
xray log
# 或
journalctl -u xray -f
```

Ctrl+C 退出。

---

## 六、系统状态检查

### 查看 CPU 和内存

```bash
htop
```

如果没装：`apt install htop -y`

### 查看流量使用

```bash
vnstat -m    # 本月流量
vnstat -d    # 每日流量
```

如果没装：`apt install vnstat -y && vnstat -u -i eth0`

### 查看磁盘空间

```bash
df -h
```

---

## 七、故障排查

### 节点突然连不上

```bash
# 1. 检查 Xray 是否在运行
systemctl status xray
# 看到 active (running) 就是正常

# 2. 没在运行就启动
systemctl start xray

# 3. 重启试试
systemctl restart xray

# 4. 检查端口是否被防火墙挡了
ufw status
# 或
iptables -L -n | grep <你的端口号>
```

### 排查流程

| 步骤 | 操作 | 看什么 |
|------|------|--------|
| 1 | `systemctl status xray` | 是不是 running |
| 2 | `ping 你的服务器IP`（本地电脑执行） | 服务器通不通 |
| 3 | `xray log` | 有没有报错 |
| 4 | `xray add` 新建一个节点 | 新节点能不能连 |

### 完全失联（IP 被墙）

1. 去 RackNerd 面板 → 点你的 VPS
2. 找 **"Change IP"** 或 **"Request New IP"**
3. 换完 IP 后 `xray change` 更新配置
4. 客户端更新地址

### 端口被封

```bash
xray add     # 新建一个不同端口的节点
```

然后用新链接，旧节点不管它。

---

## 八、安全加固

### 改 root 密码

```bash
passwd
```

### 改 SSH 端口（防扫描）

```bash
sed -i 's/#Port 22/Port 22222/' /etc/ssh/sshd_config
systemctl restart sshd
```

改完后下次 SSH 用新端口：`ssh root@IP -p 22222`

### 只允许密钥登录（可选，高级）

```bash
# 在本地电脑生成密钥对
ssh-keygen -t ed25519

# 复制公钥到服务器
ssh-copy-id root@你的IP
```

然后关闭密码登录：
```bash
sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd
```

---

## 九、RackNerd 面板操作

| 操作 | 路径 |
|------|------|
| 重启 VPS | Control Panel → Reboot |
| 重装系统 | Control Panel → Reinstall |
| 换 IP | Control Panel → Change IP |
| 看流量 | Control Panel → Bandwidth |
| VNC 救援 | Control Panel → VNC（SSH 挂了用这个） |

## 九点五、阿里云面板操作（回国节点专用）

| 操作 | 路径 |
|------|------|
| 重启 ECS | ECS 控制台 → 实例 → 重启 |
| 重置密码 | ECS 控制台 → 实例 → 重置实例密码 |
| 安全组放行端口 | ECS 控制台 → 安全组 → 添加规则（入方向，放行你的端口） |
| 查看 IP | ECS 控制台 → 实例 → 查看公网 IP |
| VNC 救援 | ECS 控制台 → 实例 → 远程连接 → VNC |

---

## 十、快速参考卡

```
每月:                    出问题时:
apt update && upgrade      systemctl status xray
xray update                ping IP（本地）
                           xray log
增加节点:                  xray add 建新节点
xray add
                           IP 被墙:
删除节点:                  面板 → Change IP
xray del                   xray change 更新
                           客户端更新 IP
查看:
xray change
xray log
```
