# Field Notes: Self-Hosted Proxy Behind the Great Firewall (Nov 2026)

> A real troubleshooting log: from "I thought the server was broken" to "Hysteria2 saturating the link" — the dead ends, the diagnostic methods, and the final answer.

[中文版 / Chinese version](./README.md)

## TL;DR

| Approach | Latency | 10 MB download | Verdict |
|---|---|---|---|
| **Hysteria2 (direct, UDP 8443)** | 0.6–1.6 s | **2.3 MB/s** | ✅ final answer |
| Commercial airport (US West node) | 1.5–2.1 s | 2.0 MB/s | good fallback |
| Cloudflare Tunnel (VLESS+WS) | 1.3–3 s | 0.2 MB/s | works, but crawling |
| Direct TLS (Trojan/VLESS over TCP) | intermittent | unusable | IP-level interference |

**Three findings that mattered:**

1. **ICMP passes while TCP drops means a single IP is being targeted**, not the whole subnet.
2. **A free Cloudflare Tunnel is throttled to 0.1–0.5 MB/s** as a proxy — QUIC and HTTP/2 both fail to fix it.
3. **UDP 443 is blocked, UDP 8443 is not.** Moving Hysteria2 to 8443 turned "cannot connect" into "line saturated".

---

## 1. Environment

- VPS: US West / Dallas, Debian 12, 1 vCPU / 1 GB
- Server: Xray (VLESS/Trojan + TLS), Hysteria2 added later
- Client: Clash Verge Rev (mihomo core)
- Home line: ~12 Mbps (~1.5 MB/s on domestic speed tests)

## 2. Symptoms and two wrong conclusions

### Symptoms
- `ping`: **0% loss**, RTT ≈ 207 ms
- TCP connect: only 6–10 successes out of 20 attempts
- Port 443: SYN never answered
- Occasionally connects, TLS handshake completes, then data stalls

### Wrong conclusion #1: "the server is broken"
All "direct connection" tests were run with **Clash TUN mode enabled** — which transparently routed every packet through the commercial airport node, and **that node blocks port 8443**, so everything timed out.

> **How to check**: `GET /configs` → `tun.enable`; `route print` → is the default route pointing at `198.18.0.x`?; a domain resolving to `198.18.x.x` means fake-ip is active.

### Wrong conclusion #2: "the whole subnet is blocked"
After switching to another IP inside the **same /26 subnet**, the connection success rate went from 30–50% to 80% — so the interference targets **individual IPs** (random SYN drops + port-specific blocking), not the subnet. A subnet-level blackhole would also kill ICMP.

## 3. Diagnostics (reusable)

```bash
# 1. ICMP baseline
ping -n 10 <IP>

# 2. TCP success rate (20 attempts)
for i in $(seq 1 20); do timeout 3 bash -c "echo > /dev/tcp/<IP>/<PORT>" 2>/dev/null && echo ok; done | wc -l

# 3. Can TLS complete? (yes => TCP+TLS layer is fine)
echo | openssl s_client -connect <IP>:<PORT> -servername <SNI> -brief

# 4. Port probe (RST vs timeout)
curl -sv --connect-timeout 6 https://<IP>:<PORT>/ 2>&1 | head -5
```

**Interpretation table**:

| Observation | Meaning |
|---|---|
| Ping OK, TCP SYN success 30–80% | Per-IP probabilistic drop |
| One port 100% blocked, another flaky | Fine-grained per-IP+port policy |
| TLS handshake succeeds, then stalls | Interference after the handshake |
| Same subnet, different IP behaves better | IP reputation problem, not routing |

## 4. Why Cloudflare Tunnel is not enough

```
Direct:  client ──(one encrypted connection)── VPS ── target
Tunnel:  client ──(VLESS+TLS)── CF edge ──(CF backbone)── VPS ── target
```

- **Double encryption** and an extra hop: header and handshake overhead pile up
- Free tunnels carry no bandwidth guarantee; a single app is often capped at 0.1–0.5 MB/s
- With loss + RTT, congestion control collapses (Mathis: `throughput ≈ MSS / (RTT × √loss)`; 207 ms + 0.5% loss ≈ 100 KB/s — matches what we measured)

Tried `protocol: http2`, `--edge-ip-version 4`, and reconnecting for a closer PoP (check the `cf-ray` suffix) — none produced a meaningful improvement.

## 5. The fix: Hysteria2 on UDP 8443

### The decisive experiment: change the port
Hysteria2 listens on UDP 443 by default. The client failed with `connect error: timeout: no recent network activity` — UDP 443 is blocked.
**Changing `listen` to `:8443` made it work immediately**: 0.6–1.6 s latency, 2.3 MB/s download.

> Tip: if UDP 443 fails, try 8443 / 3478 / 20000 / 2087 in order.

### Server config (Debian 12)

```yaml
# /etc/hysteria/config.yaml
listen: :8443
tls:
  cert: /usr/local/etc/xray/c.pem   # reuse the existing self-signed cert
  key: /usr/local/etc/xray/k.pem
auth:
  type: password
  password: <strong-random-password>
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

### Client (mihomo / Clash Meta)

```yaml
- name: my-hy2
  type: hysteria2
  server: <IP>
  port: 8443
  password: <same as server>
  sni: <cert CN, e.g. www.cloudflare.com>
  skip-cert-verify: true     # required for self-signed certs
  up: "10 Mbps"
  down: "50 Mbps"            # set to your real bandwidth; Brutal uses it
```

### Why Hysteria2 wins
- **Single hop**: client connects straight to the VPS, no CDN in between
- **Loss tolerance**: Brutal congestion control keeps throughput high under loss (TCP halves its window)
- **QUIC underneath**: stream-level multiplexing without TCP head-of-line blocking

## 6. Other gotchas

1. **A VNC console can drop keystrokes**; uppercase letters need an explicit `Shift+letter`, and passwords with capitals should be typed slowly.
2. When **the SSH port is intermittently blocked**, write deployment scripts as "connect → fail → wait 20 s → retry N times" rather than giving up on the first failure.
3. **Cloudflare's add-site onboarding must be completed**: Add site → select a plan (click *Select plan* on the Free card) → on the DNS page click **Continue to activation**. Skip a step and the zone stays `Pending setup`, invisible to `cloudflared tunnel login`.
4. **`cloudflared tunnel login` needs a system proxy** to fetch the certificate, otherwise it reports `Failed to fetch resource`.
5. Browsers refuse remote debugging on the **default profile** (Chrome 136+); automating an already-logged-in browser requires a separate temporary profile.

## 7. Numbers (measured at the same time, on the same line)

| Node | Latency (avg of 3) | 10 MB download |
|---|---|---|
| Hysteria2 UDP 8443 | 1.1 s | 2.3 MB/s |
| Commercial airport (US West) | 1.8 s | 2.0 MB/s |
| Cloudflare Tunnel | 2.2 s | 0.2 MB/s |

## 8. Closing

None of this would run without the open-source community: **Hysteria** (transport), **mihomo/Clash.Meta** (client), **cloudflared** (tunnel), **Xray** (node). We are only users — writing down the dead ends is our way of paying it forward.

> These notes document network diagnostics and self-hosted service experience. Use them in compliance with local laws and the terms of your network providers.

---

**Compiled by Yuge (宇歌)** · field testing by **Zhang Yangfei** · 2026-11-24

> May every blocked road have someone who writes down the detour.

## License

MIT
