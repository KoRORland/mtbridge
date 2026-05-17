# Telegram 2-Hop DPI-Proof Bridge — v2

**Revised 2026-05-17. Supersedes v1 (`mtproxy.md`).**
If you already deployed v1, jump to **Appendix A: Migration from v1**.

## What this is

Two-hop bridge so a small (≤100) circle of Russian Telegram users keeps working after the federal block:

```
RU client ──MTProto Fake-TLS (SNI=rutube.ru):443──▶ RU VPS
                                                     │
                                                     │ VLESS + XHTTP + Reality
                                                     │ (SNI=gateway.icloud.com)
                                                     ▼
                                                   EU VPS ──Telegram DCs
```

The RU VPS speaks MTProto to clients and tunnels its egress through a Reality channel to a clean EU box that talks to Telegram directly. SSH is **not** publicly exposed on either box — both are reached only over a private Tailscale mesh.

## Changes vs v1 (highlights)

- Reality `dest` is `gateway.icloud.com` (v1 used the over-fingerprinted `www.microsoft.com`)
- ALPN pinned to `h2`, XHTTP mode pinned to `packet-up`
- mtg launched with `--secure-only --anti-replay-cache-size 128MB --multiplex-per-connection 1`
- EU outbound restricted to Telegram DC ranges (was unrestricted `freedom`)
- SSH access via Tailscale; sslh removed; `:443` is pure Xray (was multiplexed, leaked SSH banner on idle probes)
- IP rotation runbook with netplan procedure (was missing)
- Maintenance: secret rotation, ASN pre-flight, burn signals

Both VPSes: Ubuntu 24.04. Only `:443/tcp` is exposed to the public internet.

---

## Pre-flight: ASN check the candidate VPS IPs

Before you provision either VPS, check the ASN of the IP the provider will hand you (most show this in the panel before purchase). Run from anywhere:

```bash
whois CANDIDATE_IP | grep -iE 'origin|netname'
```

Avoid heavily flagged hosters for the **EU** side — Hetzner (AS24940), DigitalOcean (AS14061), Vultr (AS20473), OVH (AS16276) — their /16s are on shared greylists and the bridge will burn in weeks. Prefer Aruba, BuyVM, Servarica, or a smaller regional EU provider. RU side is less sensitive since you want it to *look* Russian; any reputable RU hoster (Timeweb, Selectel, VK Cloud) is fine.

---

## 1. Prerequisites — run on BOTH VPSes

Bring the system current and install only what we need:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ufw curl golang-go proxychains-ng -y
```

Initial firewall (we'll tighten in §5 once Tailscale is up):

```bash
sudo ufw allow 22/tcp     # bootstrap only — closed in §5
sudo ufw allow 443/tcp
sudo ufw --force enable
```

SSH key auth (run on your laptop, once per VPS):

```bash
ssh-keygen -t ed25519 -C "bridge-admin"   # if you don't have a key already
ssh-copy-id root@RU_BOOTSTRAP_IP
ssh-copy-id root@EU_BOOTSTRAP_IP
```

Disable password login on both VPSes:

```bash
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

---

## 2. SSH via Tailscale (both VPSes)

Tailscale gives you a private WireGuard mesh with MagicDNS hostnames. SSH lives only there — no public banner, no port-sharing tricks, IP-rotation doesn't break your access.

Install on the **RU VPS**:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname=ru-bridge
```

Install on the **EU VPS**:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname=eu-bridge
```

Each `tailscale up` prints a login URL — open it once per node and approve the device at https://login.tailscale.com/admin/machines. Set "Disable key expiry" on both nodes so they don't drop off the tailnet during a reauth.

Install Tailscale on your **laptop** (same tailnet). From now on:

```bash
ssh root@ru-bridge
ssh root@eu-bridge
```

Verify both nodes are reachable over the tailnet before proceeding.

**Emergency fallback** if Tailscale itself fails: every reputable VPS provider offers a web console (Hetzner Cloud Console, Timeweb console, etc.). Use it to log in and run `sudo tailscale up` again. Do **not** temporarily re-open public `:22` to recover.

---

## 3. European VPS — Xray server

### 3.1 Generate keys

```bash
xray x25519           # → PrivateKey + PublicKey
openssl rand -hex 8   # → ShortID
uuidgen               # → client UUID
```

Save all four values; they appear in both server and client configs.

### 3.2 Verify the masquerade target reachable

The Reality `dest` must support TLS 1.3 + X25519 + HTTP/2 and respond cleanly. From the EU VPS:

```bash
xray reality test gateway.icloud.com:443
```

All three checks must say OK. If not, fall back to one of: `swdist.apple.com`, `dl.google.com`, `mensura.cdn-apple.com`. Pick one and use it consistently in §3.3 below and §4.4 (client side).

### 3.3 Server config

```bash
sudo nano /usr/local/etc/xray/config.json
```

```json
{
  "log": {"loglevel": "warning"},
  "inbounds": [{
    "listen": "0.0.0.0",
    "port": 443,
    "protocol": "vless",
    "settings": {
      "clients": [{"id": "YOUR_UUID_HERE"}],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "xhttp",
      "xhttpSettings": {"path": "/xh", "mode": "packet-up"},
      "security": "reality",
      "realitySettings": {
        "show": false,
        "dest": "gateway.icloud.com:443",
        "xver": 0,
        "serverNames": ["gateway.icloud.com"],
        "privateKey": "YOUR_PrivateKey_HERE",
        "shortIds": ["YOUR_ShortID_HERE"],
        "alpn": ["h2"]
      }
    },
    "sniffing": {"enabled": true, "destOverride": ["http", "tls", "quic"]}
  }],
  "outbounds": [
    {"protocol": "freedom", "tag": "direct"},
    {"protocol": "blackhole", "tag": "block"}
  ],
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "ip": [
          "149.154.160.0/20",
          "91.108.4.0/22",
          "91.108.8.0/22",
          "91.108.16.0/22",
          "91.108.56.0/22",
          "95.161.64.0/20",
          "2001:b28:f23d::/48",
          "2001:b28:f23f::/48",
          "2001:67c:4e8::/48"
        ],
        "outboundTag": "direct"
      },
      {"type": "field", "network": "tcp,udp", "outboundTag": "block"}
    ]
  }
}
```

The routing block restricts outbound to Telegram DC ranges only. If the EU IP leaks (gets reused for browsing, scraping, anything), there's nothing for an observer to fingerprint beyond Telegram traffic.

### 3.4 Install Xray and start

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
sudo systemctl enable --now xray
sudo systemctl status xray
```

### 3.5 Fix the "nobody" warning

```bash
sudo nano /etc/systemd/system/xray.service
# Replace User=nobody with DynamicUser=yes
sudo systemctl daemon-reload && sudo systemctl restart xray
```

---

## 4. Russian VPS — MTProto + Xray client

### 4.1 Install latest Go and build mtg

```bash
sudo apt remove --purge golang-go -y
cd /tmp
wget https://go.dev/dl/go1.26.1.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.26.1.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/go.sh
source /etc/profile.d/go.sh

git clone https://github.com/9seconds/mtg.git /opt/mtg
cd /opt/mtg
go build -o /usr/local/bin/mtg .
sudo chmod +x /usr/local/bin/mtg
```

### 4.2 Generate the MTProto secret

```bash
mtg generate-secret --hex rutube.ru
# Copy the ee... string → YOUR_SECRET_HERE
```

The hex argument is the Fake-TLS SNI, not the proxy hostname. `rutube.ru` is a real Russian video service; clients will appear to be visiting it.

### 4.3 Xray client config

```bash
sudo nano /usr/local/etc/xray/config.json
```

```json
{
  "log": {"loglevel": "warning"},
  "inbounds": [{
    "listen": "127.0.0.1",
    "port": 1080,
    "protocol": "socks",
    "settings": {"udp": true}
  }],
  "outbounds": [{
    "protocol": "vless",
    "settings": {
      "vnext": [{
        "address": "EU_IP_HERE",
        "port": 443,
        "users": [{"id": "YOUR_UUID_HERE", "encryption": "none"}]
      }]
    },
    "streamSettings": {
      "network": "xhttp",
      "xhttpSettings": {"path": "/xh", "mode": "packet-up"},
      "security": "reality",
      "realitySettings": {
        "serverName": "gateway.icloud.com",
        "publicKey": "YOUR_PublicKey_HERE",
        "shortId": "YOUR_ShortID_HERE",
        "fp": "chrome",
        "alpn": ["h2"]
      }
    }
  }]
}
```

### 4.4 Install Xray and start

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
sudo systemctl enable --now xray
```

Fix the `nobody` warning the same way as §3.5.

### 4.5 proxychains config

```bash
sudo nano /etc/proxychains.conf
```

```ini
strict_chain
proxy_dns
quiet_mode

[ProxyList]
socks5 127.0.0.1 1080
```

### 4.6 MTProto systemd service

```bash
sudo nano /etc/systemd/system/mtproto.service
```

```ini
[Unit]
Description=MTProto Proxy with tunnel
After=network.target xray.service

[Service]
Type=simple
ExecStart=/usr/bin/proxychains -q /usr/local/bin/mtg simple-run \
  --secure-only \
  --anti-replay-cache-size 128MB \
  --multiplex-per-connection 1 \
  -n 1.1.1.1 -i prefer-ipv4 -t 30s -a 512kib \
  0.0.0.0:443 YOUR_SECRET_HERE
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload and enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mtproto
```

---

## 5. Firewall lockdown

With Tailscale running, public SSH is no longer needed. Close it on both VPSes:

```bash
sudo ufw delete allow 22/tcp
sudo ufw reload
sudo ufw status
```

Expected: only `443/tcp ALLOW Anywhere`. Tailscale uses outbound UDP (with DERP relay fallback) — no ingress rule required.

---

## 6. Verification

### 6.1 On the RU box

```bash
# Egress goes via EU
curl -x socks5h://127.0.0.1:1080 https://ifconfig.me
# → must print the EU VPS IP

# mtg up, no errors
sudo systemctl status xray mtproto
sudo journalctl -u mtproto -n 20 --no-pager

# Connected clients (likely 0 right after deploy)
sudo ss -Htn state established sport = :443 | wc -l
```

### 6.2 On the EU box

```bash
sudo systemctl status xray
sudo journalctl -u xray -n 20 --no-pager
# Should be quiet; spikes of "reject" or "fail" mean Reality is being probed
```

### 6.3 External smoke test

Run from outside both VPSes — **do not** use your laptop's residential IP. Use check-host.net, a throwaway probe, or a dedicated RU monitoring VPS.

TCP reachability:

```bash
nc -vz RU_IP 443
```

TLS plausibility — Reality should mirror the masquerade target's cert chain:

```bash
echo | openssl s_client -connect RU_IP:443 -servername gateway.icloud.com 2>/dev/null \
  | grep -E 'subject=|issuer='
```

Expected: subject/issuer that match the real `gateway.icloud.com` certificate. Anything else (self-signed, default Xray cert, SSH banner) means Reality is misconfigured or the wrong service is bound to :443.

No public SSH endpoint:

```bash
nc -vz RU_IP 22   # expected: refused/timeout
nc -vz EU_IP 22   # expected: refused/timeout
```

---

## 7. Client side — Telegram on phones

Single-tap link to send to your users:

```
tg://proxy?server=RU_IP&port=443&secret=YOUR_SECRET_HERE
```

Or manual: Settings → Data and Storage → Proxy → MTProto. Server `RU_IP`, Port `443`, Secret `YOUR_SECRET_HERE`.

Distribute out-of-band — Signal, in person, encrypted email. Never post the link inside Telegram (it can't reach the user if their Telegram is blocked, and it teaches your adversary the same secret).

---

## 8. Operating the bridge

### 8.1 Who's connected right now

```bash
sudo ss -Htn state established sport = :443 \
  | awk '{print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn
```

Each row: connection count + source IP. Telegram clients open several parallel TCP streams; 4–10 per device is normal, 20+ likely means multiple devices behind a NAT.

To resolve who an IP belongs to:

```bash
curl -s https://ipinfo.io/SOURCE_IP | grep -E '"country"|"region"|"org"|"city"'
```

For real session counts (not raw TCP), migrate `simple-run` → `mtg run` with a TOML config and `stats-bind = "127.0.0.1:3128"`; then `curl http://127.0.0.1:3128/stats`. This is the prerequisite for any proper monitoring/dashboarding setup.

### 8.2 Rotate the MTProto secret (every 30–60 days)

Leaked secrets get harvested off client devices and shared blocklists. Schedule rotation:

```bash
NEW_SECRET=$(mtg generate-secret --hex rutube.ru)
sudo sed -i "s/YOUR_OLD_SECRET/$NEW_SECRET/" /etc/systemd/system/mtproto.service
sudo systemctl daemon-reload && sudo systemctl restart mtproto
echo "tg://proxy?server=RU_IP&port=443&secret=$NEW_SECRET"
```

Distribute the new link out-of-band. Keep the old service config around for 24h in case of stragglers, then update permanently.

### 8.3 Rotate the RU public IP

Use when: the IP appears on a blocklist, RU clients can't reach the box but the box itself is healthy, or you're migrating providers.

**8.3.1 Pre-flight**

ASN-check the candidate (see top of this doc). Decide whether you're doing IP-only or coordinated rotation (IP + MTProto secret + optionally Reality keys).

**8.3.2 System-side IP setup**

Russian providers attach additional IPs at the panel level but **do not configure them on the OS interface** — you must add to netplan. Authoritative reference for Timeweb: https://timeweb.cloud/docs/unix-guides/adding-ip-addresses.

Inspect current state on the RU VPS (reached via Tailscale):

```bash
ip addr show eth0
ip route show default
ls /etc/netplan/
```

Note the existing IP, default gateway, and interface name. On Timeweb you'll typically see `50-cloud-init.yaml`.

Create `/etc/netplan/99-ipv4.yaml` with **both** IPs side by side (so the overlap window works — old clients keep hitting OLD_IP, new clients hit NEW_IP, both on the same kernel):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - "OLD_IP/24"
        - "NEW_IP/24"
      routes:
        - to: "0.0.0.0/0"
          via: "GATEWAY_IP"
      nameservers:
        addresses:
          - "1.1.1.1"
          - "1.0.0.1"
```

Verify CIDR (`/24` is usual for Timeweb) and interface name against `ip addr` output.

Lock permissions and neutralise cloud-init so it doesn't overwrite this on reboot:

```bash
sudo chmod 600 /etc/netplan/99-ipv4.yaml
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml-backup-$(date +%Y%m%d) 2>/dev/null
echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

Apply:

```bash
sudo netplan --debug apply
ip addr show eth0       # must list BOTH OLD_IP and NEW_IP
ip route show default
```

Services already bind to `0.0.0.0` so they accept on both. Restart so mtg refreshes its public-IP self-check:

```bash
sudo systemctl restart xray mtproto
sudo journalctl -u mtproto -n 5 --no-pager
```

> **Risk:** if you misconfigure netplan and `netplan apply` strands the box, Tailscale survives interface reconfigs and you keep access — but only because §2 was applied. Without Tailscale you'd need VPS-console recovery.

**8.3.3 External reachability**

Probe NEW_IP from an RU vantage (check-host.net or your monitoring VPS). Both TCP and TLS must succeed.

**8.3.4 Mint and distribute the new link**

```bash
echo "tg://proxy?server=NEW_IP&port=443&secret=YOUR_SECRET_HERE"
```

Out-of-band only.

**8.3.5 Overlap window and cleanup**

After 24–48h, remove OLD_IP from the `addresses:` list and re-apply netplan. On your laptop:

```bash
ssh-keygen -R OLD_IP   # only if you ever SSHed via raw IP, not Tailscale hostname
```

Update any external monitoring scripts that reference OLD_IP literally.

**8.3.6 Coordinated rotation (if you suspect a block)**

In the same swap: rotate the MTProto secret (§8.2), regenerate Reality keys on EU (`xray x25519`) and update both ends, and pick a fresh `dest` from §3.2's alternates. A single coordinated rotation is far harder for a passive observer to track than three separate ones spread over days.

### 8.4 Burn signals to watch

- `journalctl -u xray -n 50 | grep -ciE 'reject|fail|reset'` on EU spiking → Reality being actively probed
- Sudden latency increase on the RU socks5 egress (§6.1) → carrier-level shaping
- Multiple users report Telegram stuck on "connecting…" → MTProto secret likely on a blocklist; rotate

### 8.5 Software updates

```bash
# Xray on both boxes
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
sudo systemctl restart xray

# mtg
cd /opt/mtg && git pull && go build -o /usr/local/bin/mtg .
sudo systemctl restart mtproto

# Tailscale
sudo apt update && sudo apt install --only-upgrade tailscale -y
```

---

## Appendix A: Migration from v1 to v2

For users who deployed the original `mtproxy.md` and want to land on v2 without a full redeploy. Do these in order; each step is reversible until the last.

### A.1 Snapshot configs (both VPSes)

```bash
sudo cp -a /usr/local/etc/xray/config.json /usr/local/etc/xray/config.json.v1-backup
sudo cp -a /etc/systemd/system/mtproto.service /etc/systemd/system/mtproto.service.v1-backup 2>/dev/null
sudo cp -a /etc/default/sslh /etc/default/sslh.v1-backup 2>/dev/null
sudo cp -a /etc/ufw /etc/ufw.v1-backup
```

### A.2 Install Tailscale on both VPSes (§2)

Do this *before* touching anything else, while public SSH still works as your safety net. Verify `ssh root@ru-bridge` and `ssh root@eu-bridge` succeed before continuing.

### A.3 Remove sslh

Only after Tailscale SSH is confirmed working:

```bash
sudo systemctl disable --now sslh 2>/dev/null
sudo apt purge sslh -y
```

### A.4 Update EU Xray config (§3.3)

Edit `/usr/local/etc/xray/config.json`:

- Inbound: `"listen": "0.0.0.0"`, `"port": 443` (revert if you previously moved to :8443 for sslh)
- `xhttpSettings`: add `"mode": "packet-up"`
- `realitySettings`: change `dest` and `serverNames` to `gateway.icloud.com` (after running `xray reality test` to confirm reachability — see §3.2)
- `realitySettings`: add `"alpn": ["h2"]`
- Replace the `outbounds` and add the `routing` block (Telegram-DC restriction) per §3.3

```bash
sudo systemctl restart xray
sudo journalctl -u xray -n 20 --no-pager
```

### A.5 Update RU Xray client config (§4.3)

Edit `/usr/local/etc/xray/config.json`:

- `realitySettings.serverName`: `gateway.icloud.com`
- `realitySettings.publicKey`: keep your existing Reality public key (no need to regenerate unless coordinated rotation)
- Rename `YOUR_Password_HERE` placeholder if you literally pasted it — the field has always been `publicKey` semantically
- Add `"alpn": ["h2"]`
- `xhttpSettings`: add `"mode": "packet-up"`

```bash
sudo systemctl restart xray
```

### A.6 Update mtg systemd service (§4.6)

Edit `/etc/systemd/system/mtproto.service`. Change the `ExecStart` line to include the new flags and revert the listen port back to `:443` if you moved to `:8443` for sslh:

```ini
ExecStart=/usr/bin/proxychains -q /usr/local/bin/mtg simple-run \
  --secure-only \
  --anti-replay-cache-size 128MB \
  --multiplex-per-connection 1 \
  -n 1.1.1.1 -i prefer-ipv4 -t 30s -a 512kib \
  0.0.0.0:443 YOUR_SECRET_HERE
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart mtproto
```

### A.7 Firewall lockdown (§5)

On both VPSes:

```bash
sudo ufw delete allow 22/tcp
sudo ufw reload
sudo ufw status
```

### A.8 (Optional but recommended) Coordinated secret rotation

If your bridge has been up for more than a month under v1, take this opportunity to rotate. The v1 secret may have been harvested. See §8.2 — rotate, distribute the new link to users out-of-band.

### A.9 Verification (§6)

Run all three verification blocks. Pay particular attention to the external `openssl s_client` check — the returned cert subject/issuer must mirror `gateway.icloud.com`, not Microsoft. If it still shows the old `dest`, Xray didn't pick up the config change.

### A.10 Decommission v1 artefacts

After 48h of stable v2 operation:

```bash
sudo rm /usr/local/etc/xray/config.json.v1-backup
sudo rm /etc/systemd/system/mtproto.service.v1-backup
# /etc/default/sslh.v1-backup already gone with apt purge sslh
```

---

## Appendix B: Original brief (preserved from v1)

> So, telegram was completely blocked in russia. I want to keep talking with family and firends, so I want to create a bridge proxy solution to allow telegram to still work for the chosen few - like 50-100 clients.
>
> The idea is to do 2-hop system - on Europe side, e.g. on AWS, I create a out relay, in Russia I do a local cloud VPS that is serving MTProto proxy with fake tls. There are instructions to do Proxy-only here: https://habr.com/ru/articles/1010942/ or here https://habr.com/ru/articles/994934/
>
> Both VPSes wil be running ubuntu 24.04.
>
> Russian side should use SNI to something russian and looking like videostreaming - e.g. rutube.com, and expose only port 443. We should still be able to ssh with the key there, but it should be hidden. All traffic from the MTPRoxy should go to the european VPS.
>
> Tunnel should be DPI-proof, too - so no Wireguards or any other easily detectible protocols, maximum obfuscation - e.g. VLESS + XHTTP + Reality
>
> I need reasonably digestible instruction on setting up both hosts, and then setting up client to ssh into both - and how to setup russian client telegrams to talk to this proxy.
