# Telegram 2-Hop DPI-Proof Bridge (Russia → Europe)  
**Tested & Working – March 22 2026**  
**Russian VPS**: MTProto (Fake-TLS SNI = rutube.com) + Xray client  
**European VPS**: Xray server (VLESS + XHTTP + Reality)  
**Clients**: Just add normal MTProto proxy in Telegram (50-100 users, zero extra apps)

Both servers: **Ubuntu 24.04**, only port **443** public + SSH (key-only).

---

## 1. Prerequisites (do on BOTH VPSes)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ufw curl git golang-go proxychains-ng -y
```
# Firewall
```
sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```
# SSH key-only (run from your local machine first)
```
ssh-keygen -t ed25519 -C "bridge-admin"
ssh-copy-id root@RU_IP
ssh-copy-id root@EU_IP
```
# Disable password login on both VPSes
sudo nano /etc/ssh/sshd_config
# → PasswordAuthentication no
sudo systemctl restart ssh

## 2. European VPS – Xray Server (clean out-relay)
### 2.1 Generate keys
Bashxray x25519          # → PrivateKey + Password
openssl rand -hex 8  # → ShortID (8 hex chars)
### 2.2 Full server config
```bash
sudo nano /usr/local/etc/xray/config.json
```
```JSON
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
      "xhttpSettings": {"path": "/xh"},
      "security": "reality",
      "realitySettings": {
        "show": false,
        "dest": "www.microsoft.com:443",
        "xver": 0,
        "serverNames": ["www.microsoft.com"],
        "privateKey": "YOUR_PrivateKey_HERE",
        "shortIds": ["YOUR_ShortID_HERE"]
      }
    },
    "sniffing": {"enabled": true, "destOverride": ["http","tls","quic"]}
  }],
  "outbounds": [{"protocol": "freedom", "tag": "direct"}]
}
```
### 2.3 Install & start
```Bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
sudo systemctl enable --now xray
sudo systemctl status xray
```
### 2.4 Fix “nobody” warning
```Bash
sudo nano /etc/systemd/system/xray.service
# Replace User=nobody with:
DynamicUser=yes
sudo systemctl daemon-reload && sudo systemctl restart xray
```

## 3. Russian VPS – MTProto + Xray Client
### 3.1 Install latest Go & build mtg (2026 structure)
```Bashsudo apt remove --purge golang-go -y
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
### 3.2 Generate MTProto secret (rutube.ru SNI)
```Bash
mtg generate-secret --hex rutube.ru
# Copy the ee... string → YOUR_SECRET_HERE
```
### 3.3 Xray client config
```Bash
sudo nano /usr/local/etc/xray/config.json
```
```JSON
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
      "xhttpSettings": {"path": "/xh"},
      "security": "reality",
      "realitySettings": {
        "serverName": "www.microsoft.com",
        "publicKey": "YOUR_Password_HERE",
        "shortId": "YOUR_ShortID_HERE",
        "fp": "chrome"
      }
    }
  }]
}
```
### 3.4 Install Xray client & start
```Bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
sudo systemctl enable --now xray
```
### 3.5 Create MTProto service (proxied through Xray)
```Bash
sudo nano /etc/systemd/system/mtproto.service
```
```ini
[Unit]
Description=MTProto Proxy with tunnel
After=network.target xray.service

[Service]
Type=simple
ExecStart=/usr/bin/proxychains -q /usr/local/bin/mtg simple-run -n 1.1.1.1 -i prefer-ipv4 -t 30s -a 512kib 0.0.0.0:443 YOUR_SECRET_HERE
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
### Fix proxychains (mandatory [ProxyList])
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
```Bash
sudo systemctl daemon-reload
sudo systemctl enable --now mtproto
```
### 3.6 Fix “nobody” warning (same as Europe)
```Bash
sudo nano /etc/systemd/system/xray.service
# → DynamicUser=yes
sudo systemctl daemon-reload && sudo systemctl restart xray
```
## 4. Hide port 22 
4.1 Install sslh
```Bash
sudo apt install sslh -y
```
### 4.2 Configure sslh (same on both VPSes)
```Bash
sudo nano /etc/default/sslh
```
```ini
RUN=yes
DAEMON_OPTS="--user sslh --listen 0.0.0.0:443 --ssh 127.0.0.1:22 --tls 127.0.0.1:8443"
```
### 4.3 Start sslh & clean firewall
```Bash
sudo systemctl enable --now sslh
sudo ufw reload
sudo ss -ltnp | grep 443   # should show only sslh
```
### 4.4. How you SSH now (from your home PC)
```Bash
ssh -p 443 root@RU_IP
ssh -p 443 root@EU_IP
```
## 5. Final Tests (run on Russian VPS)
```Bash
curl -x socks5h://127.0.0.1:1080 ifconfig.me   # ← MUST show EU IP
journalctl -u mtproto -n 20 --no-pager
systemctl status xray mtproto
```
## 6. Client Side (Telegram on phones)
One-click link (send to family):
```text
tg://proxy?server=RU_IP&port=443&secret=YOUR_SECRET_HERE
```
Or manual:
Settings → Data and Storage → Proxy → MTProto
Server = RU_IP, Port = 443, Secret = YOUR_SECRET_HERE

## Maintenance
```Bash
# Update Xray on both
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
sudo systemctl restart xray
# Update mtg
cd /opt/mtg && git pull && go build -o /usr/local/bin/mtg . && sudo systemctl restart mtproto
```


## initial promt
```text
So, telegram was completely blocked in russia. I want to keep talking with family and firends, so I want to create a bridge proxy solution to allow telegram to still work for the chosen few - like 50-100 clients.
 
The idea is to do 2-hop system - on Europe side, e.g. on AWS, I create a out relay, in Russia I do a local cloud VPS that is serving MTProto proxy with fake tls. There are instructions to do Proxy-only here: https://habr.com/ru/articles/1010942/ or here https://habr.com/ru/articles/994934/

Both VPSes wil be running ubuntu 24.04.

Russian side should use SNI to something russian and looking like videostreaming - e.g. rutube.com, and expose only port 443. We should still be able to ssh with the key there, but it should be hidden. All traffic from the MTPRoxy should go to the european VPS.

Tunnel should be DPI-proof, too - so no Wireguards or any other easily detectible protocols, maximum obfuscation - e.g. VLESS + XHTTP + Reality

I need reasonably digestible instruction on setting up both hosts, and then setting up client to ssh into both - and how to setup russian client telegrams to talk to this proxy.
```
