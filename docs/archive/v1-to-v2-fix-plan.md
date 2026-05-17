# mtproxy.md DPI-Hardening Fixes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Patch `mtproxy.md` to close known RKN DPI-evasion gaps and fix structural/cosmetic errors without re-architecting the 2-hop design.

**Architecture:** Edit-in-place on a single document. Tasks are grouped by concern (cosmetic → correctness → DPI-hardening → structural). No code execution; verification is visual diff + optional shell checks for the Reality `dest` swap.

**Tech Stack:** Markdown doc. Edits applied via the Edit tool against `/home/asharov/Scripts/mtproxy/mtproxy.md`. Verification commands use `grep`, `xray`, `curl`.

**Convention:** Each task shows OLD and NEW blocks for the Edit tool. "Verify" steps use `grep -n` to confirm the change landed. Commit after each task.

**Execution paths:** Tasks 1–11 form the baseline. Task 12 is a hot patch that closes the sslh banner-leak — apply immediately regardless of path. Task 13 is the architectural follow-up: replace sslh with Tailscale-based SSH access. **Task 13 supersedes Task 8 and reverses the port changes in Tasks 7 and 8.** Do not execute both Task 8 and Task 13 in the same deployment. Recommended order: 1–7, 9–12 (skip 8) → verify stable → 13. Or, if you want sslh first as a stepping stone: 1–11 → 12 → 13.

---

## Task 1: Fix malformed code-fence language tags

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

Fence tags are inconsistent (`Bashxray`, `Bash`, `JSON`, `bash`, glued to commands). Normalize to lowercase `bash` / `json` / `ini` / `text`.

- [ ] **Step 1: Replace the glued fence at section 2.1**

OLD:
```
### 2.1 Generate keys
Bashxray x25519          # → PrivateKey + Password
openssl rand -hex 8  # → ShortID (8 hex chars)
```

NEW:
```
### 2.1 Generate keys
​```bash
xray x25519          # → PrivateKey + PublicKey
openssl rand -hex 8  # → ShortID (8 hex chars)
​```
```

(Strip the zero-width chars when applying — they are only present here to prevent markdown nesting in this plan.)

- [ ] **Step 2: Replace the glued fence at section 3.1**

OLD:
```
### 3.1 Install latest Go & build mtg (2026 structure)
​```Bashsudo apt remove --purge golang-go -y
```

NEW:
```
### 3.1 Install latest Go & build mtg (2026 structure)
​```bash
sudo apt remove --purge golang-go -y
```

- [ ] **Step 3: Lowercase every remaining `Bash` / `JSON` fence**

Run:
```bash
sed -i 's/^```Bash$/```bash/; s/^```JSON$/```json/' /home/asharov/Scripts/mtproxy/mtproxy.md
```

- [ ] **Step 4: Verify**

Run: `grep -nE '^```[A-Z]' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): normalize code-fence language tags"
```

---

## Task 2: Make SNI consistent (`rutube.ru` everywhere)

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

Header says `rutube.com`; the actual secret-gen command uses `rutube.ru`. The real domain is `rutube.ru` — keep that, fix the header.

- [ ] **Step 1: Edit header line**

OLD: `**Russian VPS**: MTProto (Fake-TLS SNI = rutube.com) + Xray client`
NEW: `**Russian VPS**: MTProto (Fake-TLS SNI = rutube.ru) + Xray client`

- [ ] **Step 2: Verify**

Run: `grep -n 'rutube' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: every match shows `rutube.ru`, none show `rutube.com`.

- [ ] **Step 3: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): unify Fake-TLS SNI to rutube.ru"
```

---

## Task 3: Fix `publicKey` field labelled `Password`

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

`xray x25519` outputs `PrivateKey` + `PublicKey`. The doc's section 2.1 comment used to say "Password" — fixed in Task 1. The client config still references `YOUR_Password_HERE`.

- [ ] **Step 1: Edit client config**

OLD: `"publicKey": "YOUR_Password_HERE",`
NEW: `"publicKey": "YOUR_PublicKey_HERE",`

- [ ] **Step 2: Verify**

Run: `grep -n 'Password' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: no output.

- [ ] **Step 3: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): rename Password placeholder to PublicKey"
```

---

## Task 4: Replace Reality `dest` with a less fingerprinted target

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

`www.microsoft.com` is the most-copy-pasted Reality dest in public configs and appears in RKN/CN classifier corpora. Use `gateway.icloud.com` — Apple iCloud sync endpoint, TLS 1.3 + X25519 + h2, plausibly accessed by RU residential users, low-latency from most EU regions.

- [ ] **Step 1: Add a verification note above section 2.2**

Insert before `### 2.2 Full server config`:

```
> **Pick your `dest` carefully.** Verify your chosen target supports TLS 1.3, X25519, and HTTP/2 before pasting it into the config. From your EU VPS, run:
> ```bash
> xray reality test gateway.icloud.com:443
> # All three checks must say OK. If not, try: swdist.apple.com, dl.google.com, mensura.cdn-apple.com.
> ```
```

- [ ] **Step 2: Replace `dest` in EU server config**

OLD:
```
        "dest": "www.microsoft.com:443",
        "xver": 0,
        "serverNames": ["www.microsoft.com"],
```

NEW:
```
        "dest": "gateway.icloud.com:443",
        "xver": 0,
        "serverNames": ["gateway.icloud.com"],
```

- [ ] **Step 3: Replace `serverName` in RU client config**

OLD: `"serverName": "www.microsoft.com",`
NEW: `"serverName": "gateway.icloud.com",`

- [ ] **Step 4: Verify**

Run: `grep -n 'microsoft' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): swap Reality dest to gateway.icloud.com"
```

---

## Task 5: Pin ALPN to `h2` on both ends

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

Reality's masquerade fails plausibility checks if the negotiated ALPN doesn't match what `dest` actually serves. iCloud Gateway serves `h2`. Pin it explicitly so future Xray default changes don't break it.

- [ ] **Step 1: Add `alpn` to EU server `realitySettings`**

OLD:
```
      "realitySettings": {
        "show": false,
        "dest": "gateway.icloud.com:443",
        "xver": 0,
        "serverNames": ["gateway.icloud.com"],
        "privateKey": "YOUR_PrivateKey_HERE",
        "shortIds": ["YOUR_ShortID_HERE"]
      }
```

NEW:
```
      "realitySettings": {
        "show": false,
        "dest": "gateway.icloud.com:443",
        "xver": 0,
        "serverNames": ["gateway.icloud.com"],
        "privateKey": "YOUR_PrivateKey_HERE",
        "shortIds": ["YOUR_ShortID_HERE"],
        "alpn": ["h2"]
      }
```

- [ ] **Step 2: Add `alpn` to RU client `realitySettings`**

OLD:
```
      "realitySettings": {
        "serverName": "gateway.icloud.com",
        "publicKey": "YOUR_PublicKey_HERE",
        "shortId": "YOUR_ShortID_HERE",
        "fp": "chrome"
      }
```

NEW:
```
      "realitySettings": {
        "serverName": "gateway.icloud.com",
        "publicKey": "YOUR_PublicKey_HERE",
        "shortId": "YOUR_ShortID_HERE",
        "fp": "chrome",
        "alpn": ["h2"]
      }
```

- [ ] **Step 3: Verify**

Run: `grep -nc '"alpn": \["h2"\]' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: `2`.

- [ ] **Step 4: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): pin Reality ALPN to h2 on both ends"
```

---

## Task 6: Pin XHTTP mode to `packet-up`

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

XHTTP default (`auto`) can negotiate a long-poll pattern that RU carriers throttle or actively reset. `packet-up` is the most DPI-invisible variant. Slower under loss but reliable.

- [ ] **Step 1: Edit EU server `xhttpSettings`**

OLD: `"xhttpSettings": {"path": "/xh"},`
NEW: `"xhttpSettings": {"path": "/xh", "mode": "packet-up"},`

(Apply this same edit twice — once in section 2.2, once in section 3.3.)

- [ ] **Step 2: Verify**

Run: `grep -nc '"mode": "packet-up"' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: `2`.

- [ ] **Step 3: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): pin XHTTP mode to packet-up on both ends"
```

---

## Task 7: Harden mtg invocation

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

`simple-run` defaults change between mtg versions. Pin the security-relevant flags so this doc keeps working.

- [ ] **Step 1: Edit the systemd `ExecStart` line in section 3.5**

OLD:
```
ExecStart=/usr/bin/proxychains -q /usr/local/bin/mtg simple-run -n 1.1.1.1 -i prefer-ipv4 -t 30s -a 512kib 0.0.0.0:443 YOUR_SECRET_HERE
```

NEW:
```
ExecStart=/usr/bin/proxychains -q /usr/local/bin/mtg simple-run \
  --secure-only \
  --anti-replay-cache-size 128MB \
  --multiplex-per-connection 1 \
  -n 1.1.1.1 -i prefer-ipv4 -t 30s -a 512kib \
  0.0.0.0:8443 YOUR_SECRET_HERE
```

Note: port changed from `0.0.0.0:443` to `0.0.0.0:8443`. This is required by Task 8 (sslh takes :443).

- [ ] **Step 2: Verify**

Run: `grep -n 'secure-only\|anti-replay-cache-size\|multiplex-per-connection' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: 3 matches on the ExecStart line.

- [ ] **Step 3: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): pin mtg security flags and move to :8443"
```

---

## Task 8: Resolve sslh/Xray port-443 conflict

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

Current doc has BOTH sslh and Xray binding `:443`. sslh must own `:443`; Xray and mtg must move to `:8443`. sslh's `--tls` backend already points at `127.0.0.1:8443`, so the sslh config in section 4.2 is correct — only the Xray/mtg listeners need to change, and the section ordering needs a note.

- [ ] **Step 1: Change EU server Xray inbound `port` to 8443**

OLD:
```
  "inbounds": [{
    "listen": "0.0.0.0",
    "port": 443,
    "protocol": "vless",
```

NEW:
```
  "inbounds": [{
    "listen": "127.0.0.1",
    "port": 8443,
    "protocol": "vless",
```

Note: `listen` also tightens to localhost — only sslh should reach Xray.

- [ ] **Step 2: Update RU client outbound to dial sslh on :443 (no change needed)**

Verify the client `vnext` block still reads:
```
"address": "EU_IP_HERE",
"port": 443,
```
This is correct — sslh on the EU side terminates :443 and demuxes TLS to Xray on :8443.

- [ ] **Step 3: Add a section-order callout at the top of section 4**

Insert under `## 4. Hide port 22`:

```
> **Important — order of operations:** Install and configure sslh **before** changing the Xray/mtg ports to 8443, or you will lock yourself out of the Telegram bridge mid-deploy. If you have already deployed sections 2 and 3, stop Xray (`sudo systemctl stop xray`), apply the port changes, install sslh, then start everything: `sudo systemctl start sslh xray mtproto`.
```

- [ ] **Step 4: Verify**

Run: `grep -n '"port": 443\|"port": 8443\|0.0.0.0:443\|0.0.0.0:8443' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected:
- EU server inbound → `"port": 8443`
- RU client outbound → `"port": 443` (this is sslh)
- mtg ExecStart → `0.0.0.0:8443`

- [ ] **Step 5: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): move Xray/mtg to :8443, sslh owns :443"
```

---

## Task 9: UFW cleanup (drop `22/tcp` once sslh is live)

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

Section 1 opens `22/tcp` for bootstrap, but after sslh is running, SSH is reached via `:443` only. Leaving `22/tcp` open defeats the "only 443 public" promise and leaves a probeable surface.

- [ ] **Step 1: Add a cleanup snippet at the end of section 4.3**

Insert after `sudo ss -ltnp | grep 443   # should show only sslh`:

```
# After sslh is confirmed working, close port 22 externally
sudo ufw delete allow 22/tcp
sudo ufw reload
sudo ufw status   # 443/tcp should be the only public rule
```

- [ ] **Step 2: Verify**

Run: `grep -n 'ufw delete allow 22' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: 1 match.

- [ ] **Step 3: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): close UFW port 22 after sslh handoff"
```

---

## Task 10: Restrict EU outbound to Telegram DC ranges

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

EU `freedom direct` outbound currently passes anything. If the EU IP gets reused for general browsing/scraping it'll get flagged and the bridge dies. Restrict outbound to Telegram DCs.

- [ ] **Step 1: Replace EU server outbounds block**

OLD:
```
  "outbounds": [{"protocol": "freedom", "tag": "direct"}]
```

NEW:
```
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
```

- [ ] **Step 2: Verify**

Run: `grep -n 'blackhole\|149.154.160.0' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: 2 matches.

- [ ] **Step 3: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): restrict EU outbound to Telegram DC ranges"
```

---

## Task 11: Add maintenance items (secret rotation, ASN check)

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

The Maintenance section only covers updates. Add operational hygiene: MTProto secret rotation, ASN sanity check before standing up EU VPS.

- [ ] **Step 1: Insert a pre-flight note above section 2**

Insert above `## 2. European VPS – Xray Server (clean out-relay)`:

```
> **Pre-flight: check the EU VPS ASN.** Run `whois EU_IP | grep -iE 'origin|netname'` from any machine. If the ASN belongs to Hetzner (AS24940), DigitalOcean (AS14061), Vultr (AS20473), or OVH (AS16276), the /16 is likely already on RKN greylists. Prefer Aruba, Servarica, BuyVM, or a small regional EU hoster. The bridge will still work on flagged ASNs but expect higher latency and earlier burn.
```

- [ ] **Step 2: Append rotation guidance to the Maintenance section**

OLD:
```
## Maintenance
​```bash
# Update Xray on both
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
sudo systemctl restart xray
# Update mtg
cd /opt/mtg && git pull && go build -o /usr/local/bin/mtg . && sudo systemctl restart mtproto
​```
```

NEW:
```
## Maintenance
​```bash
# Update Xray on both
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
sudo systemctl restart xray
# Update mtg
cd /opt/mtg && git pull && go build -o /usr/local/bin/mtg . && sudo systemctl restart mtproto
​```

### Rotate MTProto secret (every 30–60 days)

Leaked secrets get harvested from client lists and appear on shared blocklists. Rotate on a schedule:

​```bash
NEW_SECRET=$(mtg generate-secret --hex rutube.ru)
sudo sed -i "s/YOUR_SECRET_HERE/$NEW_SECRET/" /etc/systemd/system/mtproto.service
sudo systemctl daemon-reload && sudo systemctl restart mtproto
echo "tg://proxy?server=RU_IP&port=443&secret=$NEW_SECRET"
​```

Distribute the new link to your users via an out-of-band channel (Signal, in-person). Keep the old service config around for 24h in case some clients haven't refreshed yet — then delete.

### Watch for burn signals

- `journalctl -u xray -n 50 | grep -i 'reject\|fail'` on EU side spiking → Reality handshakes failing, probable active probing
- Sudden latency increase from RU side → carrier-level shaping kicked in; try rotating the Reality `dest` (Task 4 verification command lists alternates)
- Telegram clients reporting "connecting…" stuck → MTProto secret likely on a blocklist; rotate
```

- [ ] **Step 3: Verify**

Run: `grep -n 'Rotate MTProto secret\|check the EU VPS ASN' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: 2 matches.

- [ ] **Step 4: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): add ASN pre-flight and secret-rotation maintenance"
```

---

## Task 12: Close sslh timeout-fallback banner leak

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

Empirically confirmed: `telnet RU_IP 443` followed by 2s of silence currently dumps the client onto sshd, which broadcasts `SSH-2.0-OpenSSH_9.6p1 Ubuntu-...`. sslh's default timeout fallback is the first protocol listed (SSH). Any passive scanner — including RKN's — sees this and learns (a) :443 is sslh, not real TLS, and (b) the exact OpenSSH/distro version. Route the fallback to TLS instead so idle probes hit Xray, which either accepts a Reality handshake or transparently forwards to the masquerade `dest`.

- [ ] **Step 1: Edit the sslh DAEMON_OPTS line in section 4.2**

OLD:
```
DAEMON_OPTS="--user sslh --listen 0.0.0.0:443 --ssh 127.0.0.1:22 --tls 127.0.0.1:8443"
```

NEW:
```
DAEMON_OPTS="--user sslh --listen 0.0.0.0:443 --ssh 127.0.0.1:22 --tls 127.0.0.1:8443 --on-timeout tls --timeout 5"
```

- [ ] **Step 2: Add a banner-leak smoke test to section 4.3**

Insert after `sudo ss -ltnp | grep 443   # should show only sslh`:

```
# Verify: no SSH banner leak on idle connections
( timeout 8 telnet RU_IP 443 < /dev/null ) 2>&1 | grep -i 'ssh-'
# Expected: no output. If you see "SSH-2.0-OpenSSH...", sslh is still falling back to SSH —
# re-check DAEMON_OPTS and `sudo systemctl restart sslh`.
```

Run this from a throwaway probe (check-host.net TCP probe or a temporary VPS), not from your own EU residential IP.

- [ ] **Step 3: Verify the doc edit**

Run: `grep -n 'on-timeout tls' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: 1 match in section 4.2.

- [ ] **Step 4: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): route sslh timeout fallback to TLS, close SSH banner leak"
```

---

## Task 13: Replace sslh with Tailscale-based SSH (supersedes Task 8)

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

sslh exists only to hide SSH on :443. With Tailscale, SSH disappears from the public internet entirely — no banner-leak surface, no timeout-fallback question, no port-share with Xray. :443 returns to being pure Xray + Reality, which is what every plausibility check assumes. SSH access happens over the tailnet (WireGuard, MagicDNS hostnames). Emergency fallback if Tailscale itself breaks: VPS provider's web console.

This task **replaces section 4 of `mtproxy.md` wholesale** and reverses the port relocations from Tasks 7 and 8.

- [ ] **Step 1: Replace section 4 (sslh) with the Tailscale-based section**

OLD (the entire `## 4. Hide port 22` section through the end of section 4.4):
```
## 4. Hide port 22 
4.1 Install sslh
​```bash
sudo apt install sslh -y
​```
### 4.2 Configure sslh (same on both VPSes)
[... entire section through 4.4 ...]
ssh -p 443 root@EU_IP
​```
```

NEW:
```
## 4. SSH access via Tailscale (no public SSH)

We remove sslh entirely. SSH lives only inside a private WireGuard mesh (Tailscale); :443 is pure Xray. There is no public SSH banner to leak and no port collision to manage.

### 4.1 Install Tailscale on both VPSes
​```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname=ru-bridge   # on RU
sudo tailscale up --ssh --hostname=eu-bridge   # on EU
​```
Approve each device in the admin console at https://login.tailscale.com/admin/machines.

### 4.2 Remove sslh if previously installed
​```bash
sudo systemctl disable --now sslh 2>/dev/null
sudo apt purge sslh -y 2>/dev/null
​```

### 4.3 Restore Xray and mtg to port 443
On EU VPS, edit `/usr/local/etc/xray/config.json` — inbound:
​```json
"listen": "0.0.0.0",
"port": 443,
​```
On RU VPS, edit `/etc/systemd/system/mtproto.service` — ExecStart port back to `0.0.0.0:443`:
​```
ExecStart=/usr/bin/proxychains -q /usr/local/bin/mtg simple-run \
  --secure-only --anti-replay-cache-size 128MB --multiplex-per-connection 1 \
  -n 1.1.1.1 -i prefer-ipv4 -t 30s -a 512kib \
  0.0.0.0:443 YOUR_SECRET_HERE
​```
Reload and restart:
​```bash
sudo systemctl daemon-reload
sudo systemctl restart xray mtproto
​```

### 4.4 Tighten UFW
​```bash
sudo ufw delete allow 22/tcp 2>/dev/null
sudo ufw allow 443/tcp
sudo ufw reload
sudo ufw status   # only 443/tcp public; Tailscale uses outbound UDP, no ingress rule needed
​```

### 4.5 New SSH workflow from your laptop
Install Tailscale on your laptop, join the same tailnet, then:
​```bash
ssh root@ru-bridge
ssh root@eu-bridge
​```
MagicDNS resolves the hostnames over WireGuard. No public SSH endpoint exists.

### 4.6 Emergency access if Tailscale fails
Use the VPS provider's web console (Hetzner Cloud Console, AWS EC2 Instance Connect, etc.) to log in and run `sudo tailscale up` again. Do **not** temporarily re-open `:22` in UFW — keep the public surface minimal.
```

(When applying: strip the zero-width chars from the triple-backtick markers in the snippet above; they exist only to prevent markdown nesting in this plan.)

- [ ] **Step 2: Verify section 4 was rewritten**

Run: `grep -n 'Tailscale\|sslh' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: multiple `Tailscale` matches, sslh only appears in the purge command line.

- [ ] **Step 3: Verify port reversions landed**

Run: `grep -nE '"port": 443|"port": 8443|0.0.0.0:443|0.0.0.0:8443' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected:
- EU server inbound → `"port": 443`
- RU client outbound → `"port": 443`
- mtg ExecStart → `0.0.0.0:443`
- No `8443` anywhere.

- [ ] **Step 4: Update section 1's UFW snippet to remove the temporary :22 rule**

OLD (in section 1, Firewall block):
```
sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

NEW:
```
# Bootstrap rule for port 22 — close it in section 4.4 once Tailscale is up
sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

- [ ] **Step 5: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): replace sslh with Tailscale, restore :443 to pure Xray"
```

- [ ] **Step 6: Post-deployment smoke test (run from outside both VPSes)**

From any external host (use check-host.net or a throwaway probe — not your laptop's residential IP):
```bash
# TCP reachability
nc -vz RU_IP 443
# TLS handshake must look like the Reality `dest`
echo | openssl s_client -connect RU_IP:443 -servername gateway.icloud.com 2>/dev/null | grep -E 'subject=|issuer='
```
Expected: the cert subject/issuer should mirror the real `gateway.icloud.com` cert. If you instead see anything resembling SSH, Xray, or a self-signed cert, Reality is misconfigured.

Also verify there is no longer a public SSH endpoint:
```bash
nc -vz RU_IP 22
# Expected: connection refused or timeout (UFW blocks)
```

---

## Task 15: Add RU IP rotation runbook

**Files:**
- Modify: `/home/asharov/Scripts/mtproxy/mtproxy.md`

The doc currently has no procedure for "the RU IP got burned, what do I do." Add a numbered section 7 between section 6 (client side) and the Maintenance block. Captures the dependency-graph walk: what is bound to the RU IP literal, what isn't, and the order to rotate without losing users.

Assumes Task 13 (Tailscale) has been applied — SSH access via `ru-bridge` hostname is IP-independent, which is the only reason this runbook can be a clean "swap IP, services come back" procedure rather than "lose access, recover via console." If you haven't applied Task 13, add a parenthetical note in step 7.3 that says "use VPS provider's web console to reach the box."

- [ ] **Step 1: Insert new section 7 before the Maintenance block**

OLD:
```
## Maintenance
```

NEW:
```
## 7. Rotating the RU public IP

Use when: the IP appears on a blocklist, RU clients can't connect but the box is healthy, or you're migrating providers/regions.

### 7.1 Pre-flight

ASN-check the candidate IP:
​```bash
whois NEW_IP | grep -iE 'origin|netname'
​```
Avoid the same /16 as the burned IP and the same ASN if the burn was DPI-driven. See the ASN guidance above section 2.

Decide the rotation scope:
- **IP-only** — provider blocked one address, MTProto secret and Reality keys are clean.
- **Coordinated rotation** — suspected DPI/classifier block. Rotate IP + MTProto secret + (optionally) Reality `dest` in a single swap. See 7.7.

### 7.2 Apply the IP change

**Same VPS, new IP.** RU providers (Timeweb, Selectel, VK Cloud) attach additional IPs at the panel level but **do not configure them on the interface** — you must add them to netplan yourself, otherwise the kernel never knows the IP belongs here and nothing listens on it. The provider's docs are authoritative; for Timeweb the reference is https://timeweb.cloud/docs/unix-guides/adding-ip-addresses.

Procedure for Ubuntu 24.04 (run on the RU VPS, reached via Tailscale):

Inspect current state:
​```bash
ip addr show eth0
ip route show default
ls /etc/netplan/
​```
Note the existing IP, the default gateway, and what files already live in `/etc/netplan/`. On a fresh Timeweb image you'll typically see `50-cloud-init.yaml`.

Create `/etc/netplan/99-ipv4.yaml` with **both** the old and new IPs (we keep both during the overlap window):
​```yaml
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
​```
Replace `OLD_IP`, `NEW_IP`, `GATEWAY_IP`, and `eth0` (verify the actual interface name from `ip addr`) with real values. CIDR mask must match what the provider documents — usually `/24` for Timeweb VPS IPv4.

Lock permissions and neutralise the cloud-init defaults that would otherwise overwrite this on reboot:
​```bash
sudo chmod 600 /etc/netplan/99-ipv4.yaml
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml-backup-$(date +%Y%m%d) 2>/dev/null
echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
​```

Apply and verify:
​```bash
sudo netplan --debug apply
ip addr show eth0       # must list BOTH OLD_IP and NEW_IP
ip route show default   # default route still via GATEWAY_IP
​```

Services bind to `0.0.0.0` so they accept on both addresses immediately. Restart so mtg re-runs its SNI/public-IP check against the new address set:
​```bash
sudo systemctl restart xray mtproto
sudo journalctl -u mtproto -n 5 --no-pager   # public_ip line now lists both
​```

> **Risk:** if you misconfigure netplan (wrong gateway, wrong CIDR, wrong interface name) and `netplan apply` strands the box, Tailscale will still let you in over the tailnet (it survives interface reconfigs) — but only if you applied Task 13. Without Tailscale, you need VPS-console recovery. Run the IPv4 change first on a non-production host if you can, or schedule it with a console-tab open.

After the overlap window (7.6), remove `OLD_IP` from the `addresses:` list and re-run `sudo netplan apply`.

**New VPS** — redeploy sections 1–4 on the new host. Copy across `/usr/local/etc/xray/config.json`, `/etc/systemd/system/mtproto.service`, the MTProto secret, and the Reality keys. Do **not** copy `/etc/ssh/ssh_host_*` — let the new box generate fresh host keys.

### 7.3 Reconnect and verify

​```bash
ssh root@ru-bridge          # Tailscale MagicDNS, IP-independent
systemctl status xray mtproto
curl -x socks5h://127.0.0.1:1080 https://ifconfig.me   # must show EU_IP
sudo ss -Htn state established sport = :443 | wc -l    # baseline (likely 0 until clients update)
sudo journalctl -u mtproto -n 20 --no-pager            # confirm public_ip line shows NEW_IP
​```

### 7.4 External reachability check

Probe from an RU vantage, not your EU residential IP:
- check-host.net: `https://check-host.net/check-tcp?host=NEW_IP:443&node=ru1.node.check-host.net`
- Or your dedicated RU monitoring VPS

Both should report TCP success and TLS handshake success.

### 7.5 Mint and distribute new client link

​```bash
echo "tg://proxy?server=NEW_IP&port=443&secret=YOUR_SECRET_HERE"
​```
Distribute out-of-band (Signal, in person, encrypted email). Never post the new link inside Telegram itself.

### 7.6 Overlap window

If you can keep the old box running, do so for 24–48h to catch stragglers who haven't updated. Then decommission:
​```bash
sudo systemctl disable --now xray mtproto sslh 2>/dev/null
​```
Destroy the VPS via the provider panel.

### 7.7 Coordinated rotation (if suspected block)

In addition to the IP, rotate the secret in the same operation:
​```bash
NEW_SECRET=$(mtg generate-secret --hex rutube.ru)
sudo sed -i "s/YOUR_SECRET_HERE/$NEW_SECRET/" /etc/systemd/system/mtproto.service
sudo systemctl daemon-reload && sudo systemctl restart mtproto
echo "tg://proxy?server=NEW_IP&port=443&secret=$NEW_SECRET"
​```
Optionally regenerate Reality keys on EU (`xray x25519`) and update both server and client configs; pick a new `dest` from the alternates listed above section 2.2. A single coordinated rotation is much harder for a passive observer to track than three separate ones spread over days.

### 7.8 Cleanup on your laptop

​```bash
ssh-keygen -R OLD_IP        # drop stale host key (only if you ever SSHed via IP, not via Tailscale)
​```
Update any external monitoring scripts that reference `OLD_IP` literally (healthchecks.io URLs, check-host.net cron jobs, RIPE Atlas measurements).

## Maintenance
```

(When applying: strip the zero-width chars from the triple-backtick markers.)

- [ ] **Step 2: Verify the new section landed**

Run: `grep -n '^## 7\. Rotating' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: 1 match, positioned before the `## Maintenance` line.

Run: `grep -n 'Coordinated rotation\|Overlap window' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: 2 matches, both inside section 7.

Run: `grep -n '99-ipv4.yaml\|netplan --debug apply' /home/asharov/Scripts/mtproxy/mtproxy.md`
Expected: at least 2 matches inside section 7.2.

- [ ] **Step 3: Commit**

```bash
git add mtproxy.md
git commit -m "docs(mtproxy): add RU IP rotation runbook (section 7)"
```

---

## Out of scope (not in this plan)

These would require deeper restructuring and were discussed but deferred:

- **Replace proxychains with `dokodemo-door` + chained outbound.** Cleaner architecture, removes proxychains DNS edge cases, but a substantial config rewrite. Defer until current setup is in production and proven stable.
- **Multi-EU-outbound load balancing.** Useful at 100+ users to break the single-IP traffic-shape signature, but requires either a second EU VPS or Cloudflare-fronted WS fallback. Track separately.
- **CDN-fronted secondary path** (VLESS + WS + TLS via Cloudflare). Highest stealth, but adds a whole second deployment. Track separately.

---

## Self-review

- Spec coverage: every gap from the audit conversation maps to a task (cosmetic → 1; SNI → 2; naming → 3; dest → 4; ALPN → 5; XHTTP mode → 6; mtg flags → 7; sslh conflict → 8; UFW → 9; outbound restriction → 10; rotation + ASN → 11; banner leak → 12; Tailscale migration → 13). Deferred items called out explicitly.
- Path conflict explicit: Tasks 8 and 13 are mutually exclusive; header note flags this. Task 12 is path-independent.
- Placeholder scan: no `TODO`/`TBD`. Every step has exact OLD/NEW or exact command.
- Type consistency: port numbers (443 = sslh public, 8443 = Xray/mtg behind sslh) consistent across Tasks 7, 8. Placeholder names (`YOUR_PrivateKey_HERE`, `YOUR_PublicKey_HERE`, `YOUR_ShortID_HERE`, `YOUR_UUID_HERE`, `YOUR_SECRET_HERE`, `EU_IP_HERE`, `RU_IP`) unchanged from original doc.
