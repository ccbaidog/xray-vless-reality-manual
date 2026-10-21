# 海外 VPS 自建节点手册（Ubuntu 22.04 · VLESS + Reality）

> 2026-10-11 整理，2026-10-21 优化。从上往下做，每步有"验证"。密钥和密码存记事本或密码管理器，别发聊天、别截图。
>
> 文中 `<SSH端口>`、`<服务器IP>` 是占位符，换成你自己的值。

## 目录

0. [名词速查](#0-名词速查)
1. [买之前](#1-买之前)
2. [登录并改密码](#2-登录并改密码)
3. [更新系统](#3-更新系统)
4. [1G 虚拟内存](#4-1g-虚拟内存swap)
5. [防火墙](#5-防火墙顺序是命门)
6. [关 IPv6（可选）](#6-关-ipv6可选)
7. [SSH 加固：fail2ban + 密钥登录](#7-ssh-加固fail2ban--密钥登录)
8. [自动安全更新](#8-自动安全更新)
9. [开 BBR](#9-开-bbr)
10. [搭 VLESS + Reality 节点](#10-搭-vless--reality-节点)
11. [换端口，以及网页面板怎么放](#11-换端口以及网页面板怎么放)
12. [排错](#12-排错节点连不上按这个顺序查)
13. [升级、备份、卸载](#13-升级备份卸载)
14. [完成后的检查清单](#14-完成后的检查清单)
15. [安全守则](#15-安全守则)
16. [底层原理](#16-底层原理这个协议到底在干什么)

## 0. 名词速查

- **面板密码**：登录服务商后台用，能重装或删除机器。
- **root 密码**：SSH 登录系统用。
- **SSH 端口**：以服务商后台显示为准，不是默认的 22。
- **UFW**：防火墙，只放行你要的端口。
- **fail2ban**：有人反复猜密码就封他的 IP。
- **BBR**：拥塞控制算法，跨洋线路有丢包时不容易降速。
- **Reality**：服务器伪装成一个真网站，陌生人看到的是真证书，只有带密钥的客户端才会被当成代理。
- **伪装域名**：被借"脸"的真网站，必须支持 TLS1.3 和 h2。

## 1. 买之前

- 一律**月付**，用满一个月再决定要不要换季付。
- 用页面给的测试 IP 在本机 ping，晚高峰再测一次（Windows：`ping -n 50 测试IP`；macOS/Linux：`ping -c 50 测试IP`）。
- 选带回国优化的线路（9929 / CMIN2）。纯国际线路可能直接 ping 不通，买了就是废的。
- 机房 IP 开不了奈飞完整内容，大概率只能看自制剧，别抱太大期望。

## 2. 登录并改密码

```bash
ssh -p <SSH端口> root@<服务器IP>
passwd
```

密码 16 位以上，大小写加数字加符号，和面板密码不要一样。

## 3. 更新系统

```bash
apt update && apt upgrade -y
```

- 提示 `Waiting for cache lock`：系统在后台自动更新，等 5 到 10 分钟。不要 `kill -9`，不要删锁文件。
- 提示 `dpkg was interrupted`：先执行 `dpkg --configure -a`，再重新 `apt update`。
- 升级过程中弹出紫色/粉色的 needrestart 对话框：直接回车选默认即可。
- 提示需要重启就 `reboot`，一分钟后重新登录。

## 4. 1G 虚拟内存（swap）

```bash
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
sysctl vm.swappiness=10
echo 'vm.swappiness=10' >> /etc/sysctl.conf
```

验证：`free -h`，Swap 一行显示 1.0G。

`swapon` 报错说明是容器型 VPS（OpenVZ/LXC），不支持，跳过。若 `mkswap` 或 `swapon` 提示 "holes" / "unsupported"（常见于 btrfs/xfs），改用 `dd if=/dev/zero of=/swapfile bs=1M count=1024` 重建。

## 5. 防火墙（顺序是命门）

**先放行 SSH 端口，再 enable。** 反了会把自己锁在门外。

```bash
apt install ufw -y
ufw allow <SSH端口>/tcp
ufw allow 443/tcp
ufw enable
ufw status numbered
```

`enable` 后先别关当前窗口，另开一个窗口 ssh 测试能登录，再关旧的。

## 6. 关 IPv6（可选）

先看有没有：`ip -6 addr | grep inet6`。没有输出就不用关。

```bash
cat > /etc/sysctl.d/99-disable-ipv6.conf <<EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
sysctl --system
sed -i 's/^IPV6=yes/IPV6=no/' /etc/default/ufw
ufw reload
```

## 7. SSH 加固：fail2ban + 密钥登录

### 7.1 fail2ban（防爆破）

```bash
apt install fail2ban -y
cat > /etc/fail2ban/jail.local <<EOF
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
banaction = ufw

[sshd]
enabled = true
port = <SSH端口>
EOF
systemctl enable --now fail2ban
systemctl restart fail2ban
fail2ban-client status sshd
```

> 注意：`<SSH端口>` 要换成真实数字再执行，这里是 `<<EOF`，不会替你替换。

验证：能看到 `Currently banned` 和 `Total failed`。**jail 里的 port 必须是你的真实 SSH 端口。**

自己输错被封了：用服务商后台的 VNC 或控制台进去，执行 `fail2ban-client set sshd unbanip 你的IP`。

### 7.2 密钥登录并关闭密码登录（强烈推荐）

fail2ban 只是减速，密码登录本身才是风险。上了密钥就不再怕爆破。

**① 在你自己的电脑上生成密钥并上传公钥：**

```bash
# macOS / Linux / Windows PowerShell 通用
ssh-keygen -t ed25519
# macOS / Linux
ssh-copy-id -p <SSH端口> root@<服务器IP>
```

Windows PowerShell 没有 `ssh-copy-id`，用：

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p <SSH端口> root@<服务器IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

**② 先在新窗口验证密钥能登录（不输密码），确认后再继续。**

**③ 服务器上关闭密码登录：**

```bash
cat > /etc/ssh/sshd_config.d/00-hardening.conf <<EOF
PasswordAuthentication no
PermitRootLogin prohibit-password
EOF
sshd -t && systemctl reload ssh
```

- 文件名必须以 `00-` 开头：sshd 对同一个选项"先读到的生效"，云厂商常见的 `50-cloud-init.conf` 里写着 `PasswordAuthentication yes`，排在后面会被它抢先。
- `sshd -t` 没有输出才是通过；有报错就别 reload。
- reload 后**不要关当前窗口**，另开窗口测试，失败就把文件删掉再 reload。
- 私钥文件丢了又关了密码登录，只能走服务商 VNC 救援，备份好私钥。

## 8. 自动安全更新

```bash
apt install unattended-upgrades -y
dpkg-reconfigure -plow unattended-upgrades
```

界面选 Yes。只自动装安全补丁。

## 9. 开 BBR

```bash
echo "net.core.default_qdisc=fq" > /etc/sysctl.d/99-bbr.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.d/99-bbr.conf
sysctl --system
sysctl net.ipv4.tcp_congestion_control
```

验证：输出 `= bbr`。

## 10. 搭 VLESS + Reality 节点

### 10.1 选伪装域名

要求同时支持 TLS1.3 和 h2。在服务器上测：

```bash
for h in www.microsoft.com addons.mozilla.org www.lovelive-anime.jp; do
  echo "== $h"
  openssl s_client -connect $h:443 -servername $h -tls1_3 -alpn h2 </dev/null 2>/dev/null \
    | grep -E "TLSv1.3|ALPN protocol"
done
```

每个域名下面要同时出现 `TLSv1.3` 和 `ALPN protocol: h2` 才算过。出现 `No ALPN negotiated` 的不能用（`bilibili-hk.biliimg.com` 就是这样被淘汰的）。本次使用 `addons.mozilla.org`。

选域名的两条经验：

- 尽量选和服务器同地区、不被墙、站点稳定的大站，别选自己的小站。
- 别用 `www.microsoft.com` 这类教程里出镜率最高的，用的人太多，本身就成了特征。

### 10.2 装 Xray（手动，不用一键脚本）

```bash
apt install -y unzip curl
cd /tmp
curl -LO https://github.com/XTLS/Xray-core/releases/latest/download/Xray-linux-64.zip
curl -LO https://github.com/XTLS/Xray-core/releases/latest/download/Xray-linux-64.zip.dgst
sha256sum Xray-linux-64.zip
grep -i 256 Xray-linux-64.zip.dgst
```

肉眼比对两行 sha256 一致，再继续（不一致就重下，别装）：

```bash
unzip -o Xray-linux-64.zip -d xray
install -m 755 xray/xray /usr/local/bin/xray
mkdir -p /usr/local/share/xray /usr/local/etc/xray
install -m 644 xray/geoip.dat xray/geosite.dat /usr/local/share/xray/
xray version
```

### 10.3 生成身份并写配置（下面整段在同一个窗口里做）

UUID 是用户名；PrivateKey 是锁，留在服务器；Password（公钥）是钥匙，给客户端；SID 是额外暗号。

```bash
UUID=$(xray uuid)
SID=$(openssl rand -hex 8)
xray x25519
echo "UUID=$UUID"; echo "SID=$SID"
```

把四样存到记事本：UUID、SID、PrivateKey、Password。然后把 `PRIV` 换成你的 PrivateKey：

> 不同 Xray 版本输出的标签不同：新版是 `PrivateKey` / `Password`，旧版是 `Private key` / `Public key`。含义一样，Password = Public key。

```bash
PRIV='粘贴PrivateKey'
HOST='addons.mozilla.org'
PORT=443
cat > /usr/local/etc/xray/config.json <<EOF
{
  "log": {"loglevel": "warning"},
  "inbounds": [{
    "port": $PORT,
    "protocol": "vless",
    "settings": {
      "clients": [{"id": "$UUID", "flow": "xtls-rprx-vision"}],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp",
      "security": "reality",
      "realitySettings": {
        "dest": "$HOST:443",
        "serverNames": ["$HOST"],
        "privateKey": "$PRIV",
        "shortIds": ["$SID"]
      }
    }
  }],
  "outbounds": [{"protocol": "freedom"}]
}
EOF
chown root:nogroup /usr/local/etc/xray/config.json
chmod 640 /usr/local/etc/xray/config.json
xray run -test -config /usr/local/etc/xray/config.json
```

最后一行必须输出 `Configuration OK`。`dest` 和 `serverNames` 必须是同一个域名。

如果中途换了窗口，`UUID`、`SID` 会丢，heredoc 会写出空值。用 `grep -E '"id"|shortIds' /usr/local/etc/xray/config.json` 检查一下，值是空的就重做这一节。

### 10.4 做成服务并启动

```bash
cat > /etc/systemd/system/xray.service <<EOF
[Unit]
Description=Xray
After=network.target

[Service]
User=nobody
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
LimitNOFILE=1048576
ExecStart=/usr/local/bin/xray run -config /usr/local/etc/xray/config.json
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now xray
systemctl status xray --no-pager
ss -tlnp | grep 443
```

443 已经在第 5 步放行过，这里不用重复 `ufw allow`。

验证：`active (running)`，并且 443 在监听。

### 10.5 客户端（v2rayN）

选"从剪贴板导入分享链接"，把下面的占位符换成你自己的值：

```
vless://UUID@<服务器IP>:443?encryption=none&flow=xtls-rprx-vision&security=reality&sni=addons.mozilla.org&fp=chrome&pbk=Password公钥&sid=SID&type=tcp#us-node
```

- `pbk` 填的是 **Password（公钥）**，不是 PrivateKey。填反了是最常见的连不上原因。
- 端口不是 443 的话，把 `:443` 一并改掉。

测试：`netflix.com/title/81280792`（能播是全解锁，只显示自制剧是半解锁）、ChatGPT、Claude。

## 11. 换端口，以及网页面板怎么放

**节点换端口：** 不要重跑 10.3（新窗口里变量都丢了，会写出空配置），直接改配置文件里的端口：

```bash
sed -i 's/"port": 443,/"port": 8443,/' /usr/local/etc/xray/config.json
xray run -test -config /usr/local/etc/xray/config.json
ufw allow 8443/tcp
ufw delete allow 443/tcp
systemctl restart xray
```

注意 `dest` 里的 `:443` 是伪装站的端口，**不要改**。客户端的端口要同步改。

**更推荐的做法：节点留在 443（伪装最像真 HTTPS），面板换到别的端口。** 面板不对公网开放，只监听 `127.0.0.1`，用 SSH 隧道访问：

```bash
ssh -p <SSH端口> -L 8080:127.0.0.1:<面板端口> root@<服务器IP>
```

隧道保持不关，浏览器打开 `http://127.0.0.1:8080`。这样面板根本不暴露在公网，不需要放行任何防火墙端口。

如果一定要公网访问面板：必须改掉默认端口、强密码、尽量上 HTTPS，并只对自己的 IP 放行：

```bash
ufw allow from 你的IP to any port <面板端口> proto tcp
```

## 12. 排错：节点连不上，按这个顺序查

1. `systemctl status xray`：是不是 active。
2. `ss -tlnp | grep 端口`：端口有没有在监听。
3. `ufw status numbered`：有没有放行该端口。
4. 服务商后台的防火墙或安全组：有没有放行该端口。
5. 客户端逐项核对：UUID、Password 公钥（不是私钥）、SID、SNI、流控、指纹。
6. `xray run -test -config /usr/local/etc/xray/config.json`：配置是否合法。
7. `journalctl -u xray -n 50 --no-pager`：看最近的报错日志。
8. 本机 ping 不通服务器：线路或 IP 被封，换 IP 或提工单，配置没用。
9. ping 通但握手超时：用 `date` 看服务器时间，偏差大会导致 Reality 校验失败；`timedatectl set-ntp true`。

## 13. 升级、备份、卸载

**备份**（改动配置前都做一次）：

```bash
cp /usr/local/etc/xray/config.json /root/xray-config.json.bak
```

**升级 Xray**：重做 10.2（下载、校验、`install`），然后：

```bash
xray version
systemctl restart xray
```

配置文件不受影响。

**卸载**：

```bash
systemctl disable --now xray
rm /etc/systemd/system/xray.service /usr/local/bin/xray
rm -r /usr/local/etc/xray /usr/local/share/xray
systemctl daemon-reload
```

## 14. 完成后的检查清单

- [ ] root 密码已改，和面板密码不一样
- [ ] `apt upgrade` 完成
- [ ] swap 1G，`free -h` 可见
- [ ] ufw active，只放行 SSH 端口和节点端口
- [ ] 新窗口测试过能 ssh 登录
- [ ] fail2ban 的 port 和 SSH 端口一致
- [ ] 密钥登录已验证，密码登录已关闭，私钥已备份
- [ ] BBR 输出 bbr
- [ ] Xray active (running)
- [ ] 客户端能连，三项测试都跑过
- [ ] 月付，记下到期日

## 15. 安全守则

- PrivateKey、root 密码、面板密码不要发聊天、不要截图。
- 已经出现在聊天记录里的那一套密钥，如果要分享对话，先重新生成：重新跑 10.3，再改客户端。
- 先放行再启用防火墙，先测试新窗口再关旧窗口。
- 遇到不认识的报错，先看报错的第一行，不要乱删文件、不要乱重启。
- 这个仓库是文档，不要把真实 IP、UUID、密钥、SSH 端口提交进来。

## 16. 底层原理：这个协议到底在干什么

### 16.1 一次正常的 HTTPS 访问

你访问一个网站时，先和服务器做 TCP 三次握手，再做 TLS 1.3 握手：

1. **ClientHello**：你告诉服务器"我要访问 xxx.com"（这就是 SNI，明文），并附上这次临时生成的 x25519 公钥。
2. **ServerHello**：服务器回它的临时公钥，双方各自算出同一个会话密钥。
3. 之后的证书和数据全部加密。

从外面只能看到 IP、端口、SNI、包的大小和时间，看不到内容。

### 16.2 为什么普通代理会被认出来

- Shadowsocks 这类老式代理的流量是一团随机字节，不像任何正常协议，统计特征很显眼。
- 防火墙还会主动探测：直接连你的端口，发乱码或发一个 HTTPS 请求，看你怎么回。正常网站会回网页或标准错误，代理服务器常常露馅（沉默、断开、回不对的东西）。
- 自己买域名配证书的 TLS 代理，要办域名、续证书，域名本身也可能被盯上。

### 16.3 VLESS：只负责"你是谁、要去哪"

VLESS 的头部很简单：版本 + 你的 UUID + 目标地址和端口。服务器核对 UUID，对就转发，不对就拒绝。它**自己不加密**，必须套在加密外壳（TLS 或 Reality）里面。所以 VLESS 解决的是认人和转发，Reality 解决的是外壳。

### 16.4 Reality：不用自己的证书，借别人的"脸"

核心思想：服务器对外表现成 addons.mozilla.org，但只有持钥匙的人才能走后门。

**密钥原理（x25519，椭圆曲线 Diffie–Hellman）：**

- 服务器有一对长期密钥：PrivateKey 留在服务器，公钥（Password）给客户端。
- 每次连接，客户端生成一对一次性的临时密钥。
- 客户端用「自己的临时私钥 + 服务器公钥」，服务器用「自己的私钥 + 客户端临时公钥」，算出**同一个共享秘密**。只有持有 PrivateKey 的一方才算得出来。
- 客户端把一段用这个共享秘密加密的认证信息（含 ShortId）藏在 ClientHello 的一个正常字段里。从外面看，这就是一次普通的 TLS 1.3 握手。

**服务器收到后分两种情况：**

1. **认证通过**（你自己的客户端）：服务器切换成代理模式，用共享秘密派生出的临时证书签名向客户端证明身份，客户端验证后开始传 VLESS 数据。
2. **认证不通过**（路人、探测者）：服务器把整个连接原封不动转给 `dest` 指向的真网站，对方拿到的是真证书和真页面，看不出任何异常。

所以配置里 `dest`（服务器真正去连的网站）和 `serverNames`（允许客户端声称的名字）必须对应同一个网站。

**为什么伪装域名必须支持 TLS 1.3 和 h2：** Reality 的握手依赖 TLS 1.3 的结构；h2 是现代浏览器连大站时默认协商的结果，不支持 h2 的站反而是少见的特征。`bilibili-hk.biliimg.com` 返回 `No ALPN negotiated`，就是因为这个被淘汰的。

### 16.5 XTLS Vision（配置里的 flow）

问题：你的浏览器访问 https 网站时本身就有一层 TLS，再套一层 Reality，就是"TLS 里面套 TLS"，内层握手包的长度和时序有固定规律，是个可识别的特征。

Vision 做两件事：

- 把内层 TLS 握手阶段的包填充成不规则长度，抹掉特征。
- 内层握手完成后，不再对内层 TLS 数据重复加密，直接透传，既省 CPU，也少一个特征。

所以服务端配置和客户端的流控必须一致，都是 `xtls-rprx-vision`。

### 16.6 一条数据完整走一遍

浏览器 → v2rayN 本地代理端口 → Reality 外壳（TCP 443，伪装成访问 addons.mozilla.org）→ 你的美国服务器 → Xray 的 freedom 出站 → 真正的目标（Netflix、ChatGPT 等）。

目标网站看到的是**服务器的 IP**，不是你家的。这也是机房 IP 开不了奈飞完整内容的原因：奈飞看到的是一个机房 IP。

### 16.7 BBR 和 Hysteria2 的区别

- **传统拥塞控制（CUBIC）**：看到丢包就当作网络堵了，立刻降速。跨洋链路有少量随机丢包，速度就一直上不去。
- **BBR**：测量瓶颈带宽和最小延迟，按"带宽 × 延迟"决定发送量，不被少量丢包吓住。
- **Hysteria2**：基于 QUIC（UDP），自带拥塞策略，按你设定的带宽硬发，丢包线路上很猛。但国内移动网络对 UDP 经常限速或阻断，所以不适合你家现在的网络。

### 16.8 这套方案的边界

- Reality 让主动探测和特征检测拿不到明显把柄，但**不是隐身**。服务器 IP 本身仍然可能被封，这和协议无关。
- 声称访问 mozilla 的连接，却连到一个不属于 Mozilla 的美国 IP，理论上存在 IP 和域名对不上的特征。选和服务器同地区的伪装域名更稳。
- 流量模式也会暴露：长时间高流量的单一连接，比偶尔访问更显眼。
- 以后遇到新协议，先问三个问题：它怎么认人？它的外壳像什么？被探测时它怎么回应？答得出来，就算理解了。
