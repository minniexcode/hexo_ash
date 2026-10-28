---
title: Tailscale 实战：先跑通，再谈组网
date: 2026-10-28
categories: AI与工具
tags:
- Tailscale
- Networking
- WireGuard
- SelfHosted
---

我用 Tailscale 最核心的诉求，不是“科学上网”，而是在**不暴露公网入口**的前提下，稳定访问自己的机器：家里的 NAS、常开的开发机、云上的测试服务器。

这类需求用 frp 也能做，但 frp 需要一台有公网 IP 的中转机，流量绕一圈，权限模型也基本靠自己拼。Tailscale 把“打洞、密钥、身份、ACL”这几件麻烦事一起收了，代价是控制面在它手里。

这篇不打算把 `tailscale --help` 翻译一遍——那种文章你不如直接看官方文档。这里只走一条主线：**从两台设备开始，一步步做到“能远程访问、能只给团队看、能接进整个局域网”**。命令以 1.8x–1.10x 的 CLI 为准，更老的版本参数不一样，遇到对不上的先 `tailscale <cmd> --help`。

<!-- more -->

## 一、先搞清楚它到底解决什么问题

Tailscale 是基于 WireGuard 的覆盖网络（overlay network）。每台加入的设备会拿到一个 `100.64.0.0/10` 网段里的地址，彼此之间像在一个局域网里。

真正值钱的不是“加密”——WireGuard 本来就加密——而是它顺手解决的四件事：

- **NAT 穿透**：能打洞就点对点直连，打不通自动走 DERP 中继
- **密钥管理**：WireGuard 的公私钥、轮换、失效都由客户端和协调服务器处理
- **身份**：设备绑定到 Google/GitHub/微软账号，不再靠一张到处复制的配置文件
- **访问控制**：ACL 精确到“谁、访问哪台设备的哪个端口”

它不适合什么，最好一开始就说清楚：

- **不是翻墙代理**。它只负责把“你的设备”连起来，不改变你访问公网的方式
- **不是 frp/ngrok 的替代品**。如果目标用户根本没有 Tailscale 账号，那还是要 Funnel 或正经反向代理
- **不是零信任的全部**。它解决网络层身份，应用层该有的鉴权一样不能省

一个简单的判断：

| 你的需求 | 选什么 |
| --- | --- |
| 自己或团队成员访问内部机器 | Tailscale（Serve） |
| 匿名公网用户也要访问 | Funnel，或自建反代 |
| 访问一整段装不了客户端的内网 | 子网路由 |
| 出差时让流量从家里出去 | Exit Node |
| 只是临时传个文件 | Taildrop，或者别用它 |

## 二、最小闭环：两台机器进同一个 tailnet

先别管 ACL、子网、出口，先把两台机器连起来，确认 `tailscale ping` 能通。

### 安装

macOS 有两条路，**不要同时装**：

```bash
# CLI + 后台守护进程，适合开发机/常开服务器
brew install tailscale
sudo brew services start tailscale

# 或者官网/App Store 的图形客户端，适合日常桌面
```

Linux 最省事：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
sudo tailscale up
```

Windows 用官方安装包，装完从托盘登录。容器里跑就挂 `/var/lib/tailscale` 持久化状态，并且用 `--network host`，否则容器内的服务没法被 tailnet 直接访问。

### 登录

```bash
tailscale up
```

它会吐一个授权链接，浏览器登录后设备就进 tailnet 了。

### 验证

```bash
tailscale status
tailscale ip -4
tailscale ping my-laptop
```

`tailscale status` 里最关键的一列是连接方式：`direct` 表示打洞成功，`relay "xxx"` 表示走了 DERP 中继。中继不是故障，但延迟和带宽会明显差一截，值得知道。

`tailscale ping` 测的是 tailnet 层的连通性，不是应用层。它通了但服务打不开，问题通常在端口、监听地址或 ACL，不在网络。

### 顺手处理一件事：密钥过期

设备密钥默认有有效期（常见是 180 天）。笔记本过期了重新登录一下无所谓，**常驻服务器建议在后台关掉 key expiry**，否则半年后某个凌晨它会自己掉线。

## 三、别记 IP：MagicDNS

后台开启 MagicDNS 后，每台设备都有稳定域名：

```
<hostname>.<tailnet-name>.ts.net
```

```bash
ping my-laptop.<tailnet>.ts.net
```

三个实际好处：

- 服务地址不用再记 `100.x.y.z`
- 配合 Serve 能拿到受信任的 HTTPS 证书
- SSH 直接 `ssh user@host.<tailnet>.ts.net`

有两个坑：

1. **公司内网 DNS 可能和 Tailscale 抢解析**。如果开了 MagicDNS 后内网域名解析异常，先试 `tailscale set --accept-dns=false`，仍然可以用 `100.x` IP 或 `tailscale ssh`。
2. **主机名会冲突**。同名设备入网后 Tailscale 会自动加后缀，别猜，`tailscale status` 里看实际名字。

## 四、场景一：远程访问家里的机器

这是最常见的用法。优先用 SSH，Web 管理页再考虑 Serve。

### SSH：用身份代替密钥分发

服务端开启：

```bash
sudo tailscale up --ssh
```

客户端：

```bash
tailscale ssh user@nas.<tailnet>.ts.net
```

它和系统自带的 `sshd` 是两套机制：Tailscale SSH 的身份由 tailnet 认证，权限由 ACL 管，不用把公钥撒得到处都是。普通 `ssh` 也仍然能用，前提是 MagicDNS 正常、目标机器上有你的公钥。

ACL 里给 SSH 单独授权：

```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["tag:server"],
      "users": ["autogroup:nonroot", "root"]
    }
  ]
}
```

`users` 控制能登到哪些系统用户。敏感机器可以把 `action` 换成 `check`，每次连接需要浏览器或设备再确认一次。

### Web 管理页：Serve 比直接开端口体面

NAS 面板、路由器后台这类服务，用 `serve` 挂到 HTTPS 上：

```bash
tailscale serve --bg 8080
# 访问 https://nas.<tailnet>.ts.net/
```

`--bg` 让配置常驻，不加就是前台运行，终端一关就没了。

## 五、场景二：把本机服务只给团队看

`tailscale serve` 做的事很明确：让本机 tailscaled 监听 HTTPS，把请求反向代理到本地端口，证书由 Tailscale 自动签发。**只有 tailnet 成员能访问，流量不经过公网。**

基本用法：

```bash
tailscale serve --bg 3000
```

同一台机器要挂多个服务，两种方式：

```bash
# 按路径拆分
tailscale serve --bg --set-path /api 3000
tailscale serve --bg --set-path /docs 8080

# 或者按端口拆
tailscale serve --bg --https=8443 3000
```

非 HTTP 协议走 TCP 转发：

```bash
tailscale serve --bg --tcp=5432 tcp://127.0.0.1:5432
```

管理配置：

```bash
tailscale serve status
tailscale serve reset
tailscale serve get-config > serve.json
tailscale serve set-config serve.json
```

证书通常不用手动管。确实需要导出时：

```bash
tailscale cert nas.<tailnet>.ts.net
```

前提是后台已经开启 **DNS → HTTPS Certificates**。

几个容易踩的限制：

- 反代目标只支持 `http://127.0.0.1` 这类本机地址，服务必须监听在 `127.0.0.1` 或 `0.0.0.0`
- Mac App Store / 独立系统扩展版的 macOS 客户端只能用 Serve 转发端口，**不能用它直接托管文件或目录**
- 同一个端口不能同时被 Serve 和 Funnel 占用，后配置的会覆盖前面的
- 手机端不能作为 Serve 主机

如果对方不是同一个 tailnet 的成员，用后台的 **Share** 把单台设备共享过去，比把人拉进整个 tailnet 干净。

## 六、Serve 还是 Funnel：这是一个决定，不是两个命令

| | Serve | Funnel |
| --- | --- | --- |
| 可见范围 | 仅 tailnet | 整个公网 |
| 证书 | 自动 | 自动 |
| 前置条件 | 节点在线 | MagicDNS + HTTPS + funnel 属性 |
| 端口 | 随意 | 仅 443、8443、10000 |
| 适用 | 团队内部工具 | 对外演示、Webhook 回调 |

Funnel 的额外要求值得单独说清楚：

1. tailnet 开启 MagicDNS 和 HTTPS Certificates
2. 策略文件里有 funnel 属性授权。用 CLI 启用时 Tailscale 默认会允许 `autogroup:member` 使用 Funnel，需要收紧就手动改策略
3. 只监听 443、8443、10000
4. macOS 上必须使用**开源版客户端**，App Store 版和独立系统扩展版不行

```bash
tailscale funnel --bg 3000
tailscale funnel status
```

我的建议很简单：**默认用 Serve**。只有当访问者确实没有 Tailscale 账号时才考虑 Funnel，而且必须补应用层鉴权（Basic Auth、Token、OAuth 都行）。

管理面板、数据库、SSH 这几类东西，不要往 Funnel 上放。

## 七、场景三：让 tailnet 伸进整个局域网

家里有一台打印机、一个监控 NVR、云上一个 VPC，这些设备装不了 Tailscale，但你希望 tailnet 里的设备能直接访问它们的局域网 IP。这就是子网路由。

### 宣告 → 批准 → 接受

网关节点上宣告网段：

```bash
sudo tailscale up --advertise-routes=192.168.1.0/24,10.0.0.0/16
```

后台 **Machines → 该设备 → Edit route settings** 勾选批准。

客户端接受路由：

```bash
# Linux 需要显式接受
sudo tailscale up --accept-routes
```

Windows、macOS、Apple TV 在后台批准后会自动接受，不需要这一步。

### 各系统的转发配置

**Linux** 作为网关必须开转发：

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

如果机器上跑着 firewalld/ufw，还要放行 `tailscale0` 的转发，必要时开 masquerade，否则包进来了出不去。

**macOS** 同样需要开转发，这一点很多教程会漏：

```bash
sudo sysctl -w net.inet.ip.forwarding=1
sudo sysctl -w net.inet6.ip6.forwarding=1
```

要持久化就写进 `/etc/sysctl.conf`。

**Windows** 打开路由并重启：

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" /v IPEnableRouter /t REG_DWORD /d 1 /f
```

### 两个常见问题

- **网段冲突**：两边都是 `192.168.1.0/24` 时，路由会打架。要么改一端的网段，要么在网关上做 NAT
- **子网路由不是出口**：它只影响“访问这个网段”的流量，不会把你的公网流量带走。想要全局出口，看下一节

## 八、场景四：出差时用家里网络出口

Exit Node 让其他设备的**所有公网流量**从某台机器出去，效果接近传统 VPN。

### 提供出口

```bash
sudo tailscale up --advertise-exit-node
```

后台 **Machines → 设备 → Edit route settings → Use as exit node** 批准。

Linux 和 macOS 作为出口都要先开 IP 转发，命令和上一节一样。Linux 上如果防火墙拦截转发，还要放行并开 masquerade。

### 使用出口

```bash
# 指定出口
tailscale up --exit-node=100.x.y.z

# 增量修改，推荐
tailscale set --exit-node=100.x.y.z

# 取消
tailscale set --exit-node=

# 走出口的同时仍能访问本地局域网（打印机、路由器）
tailscale set --exit-node=100.x.y.z --exit-node-allow-lan-access
```

验证：

```bash
tailscale status        # 节点上会标出 exit node
curl ifconfig.me        # 出口 IP 应该变成出口节点的公网 IP
```

两个提醒：

- **手机也能当出口节点**。Android、iOS、tvOS 官方都支持，别被“手机只能当客户端”的旧文章误导
- 流量绕行出口节点，带宽和延迟受它限制；出口节点必须稳定在线，否则整条链路一起断

## 九、ACL：默认的 `*:*` 只是能用

新 tailnet 的默认策略是：

```json
{
  "acls": [
    { "action": "accept", "src": ["autogroup:member"], "dst": ["*:*"] }
  ]
}
```

意思是：所有成员可以访问所有设备的所有端口。设备少、全是自己的机器时够用；一旦有服务器、有别人、有公网入口，就该收紧。

一个够用的起点：

```json
{
  "tagOwners": {
    "tag:server": ["autogroup:admin"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["tag:server:443,22"]
    },
    {
      "action": "accept",
      "src": ["tag:server"],
      "dst": ["autogroup:member:*"]
    }
  ]
}
```

要点：

- `src` / `dst` 可以是 `autogroup:member`、`autogroup:admin`、`group:*`、`tag:*`、邮箱、IP 或网段
- **`dst` 必须带端口**，`tag:server:443` 对，`tag:server` 错
- 规则是白名单，没有匹配就是拒绝
- 后台的 **Access controls → Test** 可以输入源和目标验证，别靠猜
- ACL 改动对已有连接不一定立即生效，相关服务重连一下更稳妥

### 用 tag 管理服务器

给服务器打 `tag:server` 之后，它的归属从“某个人的设备”变成“一个角色”。这对共享机器、自动化接入很重要——人走了，设备不用跟着走。

```bash
sudo tailscale up --advertise-tags=tag:server
```

前提是 `tagOwners` 里已经允许你使用这个 tag。

## 十、无人值守与批量接入

服务器、CI Runner、临时环境不适合走浏览器登录，用 auth key：

1. 后台 **Settings → Keys → Generate auth key**
2. 按需勾选：
   - **Reusable**：可重复使用
   - **Ephemeral**：节点离线后自动移除，适合 CI
   - **Pre-approved**：跳过设备审批
   - **Tags**：入网即打标签
3. 部署时：

```bash
sudo tailscale up \
  --auth-key="$TS_AUTHKEY" \
  --advertise-tags=tag:ci \
  --hostname=ci-runner-001
```

几条实践：

- auth key 放密钥管理系统或 CI Secret，**不要硬编码进镜像和仓库**
- 临时节点用 Ephemeral，省得后台积一堆僵尸设备
- 定期轮换 key，员工离职时顺手清掉
- 如果 key 带了 tag，`--advertise-tags` 必须和它一致，否则会失败

## 十一、出问题先看这三件事

大部分“连不上”都能按这个顺序定位。

### 1. 节点在线吗，走的哪条路

```bash
tailscale status
tailscale ping <host>
tailscale netcheck
```

`tailscale ping` 显示 `via DERP` 说明 UDP 打洞失败。Tailscale 是**主动出站**的，通常不需要在防火墙上开入站端口；如果网络环境限制很严，重点看 UDP 41641 和 DERP 的连通性，而不是盲目开端口。

### 2. 名字解析对不对

```bash
tailscale dns status
tailscale whois 100.x.y.z
```

解析异常先确认 MagicDNS 是否开启、`--accept-dns` 是否被关掉。临时用 `100.x` IP 绕过，能快速区分是 DNS 问题还是网络问题。

### 3. 是不是被 ACL 拦了

后台 **Access controls → Test**，输入源和目标，看规则是否放行。最常见的原因是 `dst` 端口写错，或者忘了给新 tag 加规则。

### Serve 打不开

```bash
tailscale serve status
curl -v https://<host>.<tailnet>.ts.net/
lsof -nP -iTCP:3000 -sTCP:LISTEN
```

按这个顺序看：本地服务有没有监听、Serve 配置在不在、证书有没有签发。401/403 通常是应用层鉴权，不是 Tailscale 的问题。

### 其他

```bash
# Linux
journalctl -u tailscaled -f

# 服务状态
sudo systemctl status tailscaled
brew services list | grep tailscale

# 实在不行，生成诊断包
tailscale bugreport
```

## 十二、我的实践清单

如果只记一份清单，我会留这些：

1. 常驻服务器关掉 key expiry
2. 服务器用 tag 管理，不绑定个人账号
3. ACL 不要长期停留在 `*:*`
4. 内部服务优先 Serve，而不是开公网端口
5. Funnel 一定要加应用层鉴权，管理面板和数据库永远不放
6. auth key 进密钥管理系统，CI 用 Ephemeral
7. 子网路由先确认网段不冲突，Linux/macOS 记得开转发
8. 至少准备一个备用出口节点，别让出差时的网络命悬一线
9. 出问题先 `status` → `ping` → `netcheck`，别一上来就重装

## 附录：常用命令速查

| 目的 | 命令 |
| --- | --- |
| 连接 / 断开 | `tailscale up` / `tailscale down` |
| 登出 | `tailscale logout` |
| 节点列表 | `tailscale status` |
| 本机 IP | `tailscale ip -4` |
| 连通性 | `tailscale ping <host>` |
| 网络诊断 | `tailscale netcheck` |
| 身份查询 | `tailscale whois 100.x.y.z` |
| 发布服务 | `tailscale serve --bg <port>` |
| 查看 / 清空 Serve | `tailscale serve status` / `tailscale serve reset` |
| 公网发布 | `tailscale funnel --bg <port>` |
| 证书 | `tailscale cert <fqdn>` |
| 子网路由 | `sudo tailscale up --advertise-routes=<cidr>` |
| 接受路由 | `sudo tailscale up --accept-routes` |
| 出口节点 | `sudo tailscale up --advertise-exit-node` |
| 使用出口 | `tailscale set --exit-node=<ip>` |
| SSH | `tailscale up --ssh` / `tailscale ssh user@host` |
| 传文件 | `tailscale file cp a.zip host:` / `tailscale file get .` |
| 自动更新 | `tailscale set --auto-update` |
| 诊断包 | `tailscale bugreport` |

## 参考

- 官方文档：<https://tailscale.com/kb>
- 管理后台：<https://login.tailscale.com/admin>
- Serve：<https://tailscale.com/kb/1242/tailscale-serve>
- Funnel：<https://tailscale.com/kb/1223/funnel>
- ACL 语法：<https://tailscale.com/kb/1018/acls>
- 子网路由：<https://tailscale.com/kb/1019/subnets>
- Exit Nodes：<https://tailscale.com/kb/1103/exit-nodes>
- Tailscale SSH：<https://tailscale.com/kb/1193/tailscale-ssh>
- Auth keys：<https://tailscale.com/kb/1085/auth-keys>
