# 中国大陆网络环境下自建代理的踩坑实录（2026-11）

[English version](./README.en.md)

> 一次真实的排查过程记录：从「以为是服务器坏了」到「Hysteria2 跑满带宽」，中间踩过的坑、用过的判定方法、最后的结论。

## 结论先行

| 方案 | 延迟 | 10MB 下载 | 结论 |
|---|---|---|---|
| **Hysteria2（UDP 8443 直连）** | 0.6~1.6 s | **2.3 MB/s** | ✅ 最终方案 |
| 商业机场（美西节点） | 1.5~2.1 s | 2.0 MB/s | 可作备用 |
| Cloudflare Tunnel（VLESS+WS） | 1.3~3 s | 0.2 MB/s | 能通但慢，只适合浏览 |
| 直连 TLS（Trojan/VLESS over TCP） | 时通时断 | 不可用 | 被针对性干扰 |

**三个关键发现**：

1. **ICMP 通、TCP 丢包 = 单个 IP 被针对性处理**，不是整段被封。
2. **Cloudflare 免费隧道做代理，带宽被压在 0.1~0.5 MB/s**，QUIC / HTTP2 都救不了。
3. **UDP 443 被拦、UDP 8443 通** —— Hysteria2 换端口后从「连不上」直接变成「跑满」。

---

## 一、环境

- VPS：美国西部/达拉斯，Debian 12，1C1G
- 服务端：Xray（VLESS/Trojan + TLS），后来加装 Hysteria2
- 客户端：Clash Verge Rev（mihomo 内核）
- 家用宽带：约 12 Mbps（国内测速 ~1.5 MB/s）

## 二、症状与误判

### 症状
- `ping` **0 丢包**、RTT 约 207 ms
- TCP 连接 20 次只成功 6~10 次
- 443 端口 SYN 完全不响应
- 偶尔连上，TLS 握手也能完成，但随后数据断流

### 误判一：以为服务器坏了
一开始在 Clash 开着 **TUN 模式**的情况下做「直连测试」——实际上本机所有流量都被 TUN 接管，走了机场节点，而**机场节点封了 8443 端口**，于是「怎么测都超时」。

> **判定方法**：`GET /configs` 看 `tun.enable`；`route print` 看默认路由是否指向 `198.18.0.x`；域名解析出 `198.18.x.x` 说明 fake-ip 生效中。

### 误判二：以为整段 IP 被墙
同一 `/26` 网段换一个 IP 后，「能连的比例」从 30~50% 升到 80% —— 说明是**单个 IP 被单独处理**（概率丢 SYN + 封特定端口），不是段级黑洞。整段被封的话连 ping 都不通。

## 三、诊断方法（可复用）

```bash
# 1. ICMP 基线
ping -n 10 <IP>

# 2. TCP 成功率（跑 20 次看比例）
for i in $(seq 1 20); do timeout 3 bash -c "echo > /dev/tcp/<IP>/<PORT>" 2>/dev/null && echo ok; done | wc -l

# 3. TLS 能否握手（能握手说明 TCP+TLS 层通）
echo | openssl s_client -connect <IP>:<PORT> -servername <SNI> -brief

# 4. 端口探测（看是否有 RST 还是直接超时）
curl -sv --connect-timeout 6 https://<IP>:<PORT>/ 2>&1 | head -5
```

**判读表**：

| 现象 | 含义 |
|---|---|
| ping 通、TCP SYN 成功率 30~80% | 单 IP 被针对性丢包 |
| 某端口 100% 不通、另一端口时通时断 | 按 IP+端口 的精细策略 |
| TLS 握手成功但随后断流 | 干扰在握手之后 |
| 同网段换 IP 后明显好转 | 是 IP 声誉问题，不是线路段问题 |

## 四、为什么 Cloudflare Tunnel 不行

路径对比：

```
直连：  客户端 ──(一条加密连接)── VPS ── 目标
隧道：  客户端 ──(VLESS+TLS)── CF 边缘 ──(CF 骨干)── VPS ── 目标
```

- **两层加密**、多一跳，头部与握手开销叠加
- 免费隧道没有带宽保障，单个应用的吞吐经常被压在 0.1~0.5 MB/s
- 跨境丢包 + RTT 下，拥塞控制会把窗口打到地板（Mathis 公式：`吞吐 ≈ MSS / (RTT × √丢包率)`，207 ms + 0.5% 丢包 ≈ 100 KB/s，与实际吻合）

试过 `protocol: http2`、`--edge-ip-version 4`、换 PoP（看 `cf-ray` 后缀），都没有质变。

## 五、正解：Hysteria2 + UDP 8443

### 关键实验：换端口
Hysteria2 默认监听 UDP 443，客户端报 `connect error: timeout: no recent network activity` —— 说明 UDP 443 被拦。
**把服务端 `listen` 改成 `:8443` 后立刻连通**，延迟 0.6~1.6 s、下载 2.3 MB/s。

> 建议：UDP 443 不通时，依次试 8443 / 3478 / 20000 / 2087 等非常见端口。

### 服务端配置（Debian 12）

```yaml
# /etc/hysteria/config.yaml
listen: :8443
tls:
  cert: /usr/local/etc/xray/c.pem   # 可复用现有的自签证书
  key: /usr/local/etc/xray/k.pem
auth:
  type: password
  password: <随机强密码>
masquerade:
  type: proxy
  proxy:
    url: https://www.bing.com
    rewriteHost: true
```

```ini
# /etc/systemd/system/hysteria.service
[Unit]
Description=Hysteria2 Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/hysteria server -c /etc/hysteria/config.yaml
Restart=on-failure
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

### 客户端（mihomo / Clash Meta）

```yaml
- name: my-hy2
  type: hysteria2
  server: <IP>
  port: 8443
  password: <同上>
  sni: <证书的 CN，例如 www.cloudflare.com>
  skip-cert-verify: true     # 自签证书需要
  up: "10 Mbps"
  down: "50 Mbps"            # 按实际带宽填写，Brutal 会据此控制发送
```

### 为什么 Hysteria2 能赢
- **单跳**：客户端直连 VPS，没有 CDN 中间层
- **抗丢包**：Brutal 拥塞控制在丢包时仍能维持较高吞吐（TCP 会在丢包时把窗口砍半）
- **QUIC 底座**：UDP 流级多路复用，没有 TCP 队头阻塞


### 连上了但打不开网页？先查 IPv6

**症状**：客户端显示已连接，百度能开，谷歌打不开；浏览器报 `ERR_CONNECTION_CLOSED` 或 `ERR_QUIC_PROTOCOL_ERROR`。

**原因**：手机/电脑优先把域名解析成 **IPv6**，而服务器只有 IPv4。服务端日志会明说：

```
WARN TCP error ... reqAddr: "[2001:4860:...]:443"
  error: "dial tcp6 [2001:4860:...]:443: connect: cannot assign requested address"
```

**排查与修复**：
- 服务器：`ip -6 addr`（没有 `inet6` 就是没 IPv6）、`sysctl net.ipv6.conf.all.disable_ipv6`（为 1 表示系统关闭）
- 客户端：**关闭 IPv6**，或把 DNS 策略改成「仅 IPv4 / Prefer IPv4」
- 注意：**百度能开不代表代理通**——国内域名走的是分流直连，谷歌才是真正的测试目标

> 备选：RackNerd 等商家可免费申请 IPv6 /64，但需要手工配置网络，新手不建议折腾。

## 六、其他踩过的坑

1. **VPS 控制台（VNC）键盘输入会丢字符**，大写字母要手动 `Shift+字母`；密码含大写时用慢速逐键输入。
2. **SSH 端口时通时断**时，部署脚本要写成「连接失败 → 等 20 秒 → 重试 N 次」的循环，而不是一次失败就放弃。
3. **Cloudflare 后台加域名**要完整走完引导：添加站点 → 选套餐（点 Free 卡片的 Select plan）→ DNS 记录页点 **Continue to activation**。少一步状态会停在 `Pending setup`，隧道授权页看不到这个域名。
4. **`cloudflared tunnel login` 证书下载要走系统代理**，否则报 `Failed to fetch resource`。
5. 浏览器默认配置**不允许第三方程序开远程调试**（Chrome 136+），想自动化已登录的浏览器只能用独立临时配置文件。

## 七、数据汇总（同一时间、同一线路实测）

| 节点 | 延迟（3 次均值） | 10MB 下载 |
|---|---|---|
| Hysteria2 UDP 8443 | 1.1 s | 2.3 MB/s |
| 商业机场（美西） | 1.8 s | 2.0 MB/s |
| Cloudflare Tunnel | 2.2 s | 0.2 MB/s |

## 八、写在最后

这套东西能跑起来，靠的是开源社区：**Hysteria**（传输层）、**mihomo/Clash.Meta**（客户端）、**cloudflared**（隧道）、**Xray**（节点）。我们只是使用者，把踩过的坑记下来回馈给下一个人。

> 本项目仅记录网络诊断与自建服务的技术经验，请在符合当地法律法规与网络服务条款的前提下使用。

---

**整理：宇歌**（AI 助手）· 实测：**张阳飞** · 2026-11-24

> 愿每一条被堵的路，都有人写下绕行的地图。

## License

MIT
