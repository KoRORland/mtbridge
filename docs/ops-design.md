# Bridge Operational Tooling — Design Spec

**Created:** 2026-05-17
**Targets:** the v2 Telegram-bridge setup defined in `mtproxy-v2.md`
**Status:** approved through brainstorming; ready for implementation plan

## Goals

Operate the two-node Telegram bridge unattended for weeks at a time. Specifically:

1. **Stats** — at any time, the operator can SSH to either node and immediately see who's connected, per-IP breakdown, and recent operational signal (probe results, reject rate, time since last update).
2. **Self-update** — both nodes keep their stack (Xray, mtg, Tailscale, apt) current automatically, with safety gates that rollback binary updates on failed health probes and skip cascading updates when the peer rolled back.
3. **Self-heal** — service-level failures (xray or mtproto crashing) recover automatically with a flap-cap that prevents infinite restart loops and escalates to email when something needs the operator's attention.
4. **Alerting** — operator gets email for the operationally meaningful events (rollback, flap-cap reached, sustained no-clients, reject-rate spike); a dead-man's-switch (healthchecks.io) covers the case where the node is so broken it can't even self-report.

## Non-goals

- **Auto-rotating the MTProto secret on detected blocks.** Clients can't receive a new secret automatically, so any auto-rotation breaks user access until out-of-band redistribution. Operator handles secret rotation (procedure documented in `mtproxy-v2.md` §8.2).
- **Auto-promoting a backup RU IP.** Same reason: the `tg://` link points at one IP. Multi-IP support in this design is *passive* (probing + reporting), not active failover.
- **Per-user metering or quotas.** Out of scope for this version; if needed later, add a separate component reading mtg's stats endpoint (which itself requires migrating mtg from `simple-run` to `run` with a TOML config — a v3 concern).
- **A web dashboard.** `bridge-stats` is a CLI command run over Tailscale SSH. Building a UI is a separate project.

## Architecture overview

One Python 3.12 codebase (stdlib only — `tomllib`, `sqlite3`, `smtplib`, `http.server`, `subprocess`, `urllib`, `unittest`), deployed identically to both nodes. Per-node behaviour is gated by a single `node.role = "ru" | "eu"` value in the shared config.

```
┌────────────────────────────────┐                ┌────────────────────────────────┐
│  RU VPS                        │                │  EU VPS                        │
│                                │  Tailscale     │                                │
│  mtproto.service (mtg)         │ ────────────▶  │  xray.service (Reality inbound)│
│  xray.service (Reality client) │                │  bridge-receiver.service (EU)  │
│                                │  ◀───────────  │  bridge-update.service (Sun 04)│
│  bridge-stats.service (1min)   │  HTTP via      │  bridge-stats.service (1min)   │
│  bridge-heal.service (2min)    │  tailnet       │  bridge-heal.service (2min)    │
│  bridge-update.service (Sun 06)│                │  bridge-prune.service (Sun 03) │
│  bridge-prune.service (Sun 03) │                │                                │
└────────────────────────────────┘                └────────────────────────────────┘
        │                                                      │
        │ heartbeat                                            │ heartbeat
        ▼                                                      ▼
                       healthchecks.io  ─ (dead-man's-switch email if no ping) →  operator
                                                                                       ▲
                                                                                       │
                                                        SMTP (Gmail relay)             │
                                  EU bridge-receiver → smtplib ───────────────────────┘
```

**Key invariants**:
- Only `:443/tcp` is publicly exposed on either node (per v2). The receiver listens on the tailnet interface only.
- mtproto.service (the actual mtg daemon) is never touched by the ops tooling except via `systemctl restart` from `bridge-heal`.
- Code lives in `/usr/local/lib/python3/dist-packages/bridge/` and is updated only when the operator runs `git pull && make install`. The ops layer is *not* in scope for self-update.

## Filesystem layout

```
/etc/bridge/config.toml                          # operator-edited; chmod 600
/etc/systemd/system/bridge-*.{service,timer}     # installed by make install
/usr/local/bin/bridge-{stats,update,heal,prune,receiver}   # entry-point scripts
/usr/local/lib/python3/dist-packages/bridge/     # Python package
/var/lib/bridge/state.db                         # SQLite (WAL mode)
/var/backups/bridge/                             # binary + config snapshots for rollback
/opt/bridge-ops/                                 # git checkout of the repo
```

## Config file (`/etc/bridge/config.toml`)

```toml
[node]
role          = "ru"           # "ru" or "eu"
self_tailnet  = "ru-bridge"
peer_tailnet  = "eu-bridge"

[network.ru]
public_ips    = ["198.51.100.10", "198.51.100.11"]   # all RU IPs this box owns
primary_ip    = "198.51.100.10"                      # the one currently in the user-facing tg:// link

[network.eu]
public_ip     = "203.0.113.45"

[email]                                              # EU populates; RU ignores entire table
smtp_host     = "smtp.gmail.com"
smtp_port     = 587
smtp_user     = "you@example.com"
smtp_password = "app-password"
from_addr     = "bridge-alerts@example.com"
to_addrs      = ["you@example.com"]

[healthchecks]
heartbeat_url = "https://hc-ping.com/<uuid-per-node>"   # each node has its own

[receiver]                                              # EU only — RU ignores
tailnet_listen_port = 8742                              # bound to tailscale0, never public

[peer]                                                  # RU points here; EU ignores
url = "http://eu-bridge:8742"

[active_hours_utc]
start = 6
end   = 22

[update]
day             = "sunday"
hour_utc        = 4
ru_offset_hours = 2
```

**Validation** at load time:
- `node.role` ∈ {`ru`, `eu`}.
- `network.<role>.primary_ip` is a member of `network.<role>.public_ips`.
- All IP strings parse via `ipaddress.ip_address`.
- If `role == "eu"`: `email.*` and `receiver.*` present.
- If `role == "ru"`: `peer.url` present and resolves via tailnet DNS.

Violations raise `ConfigError`; the failing CLI exits 78 (EX_CONFIG) and logs to journalctl.

## SQLite schema (`/var/lib/bridge/state.db`, WAL mode)

```sql
CREATE TABLE schema_version (
  version INTEGER PRIMARY KEY,
  applied INTEGER NOT NULL
);

CREATE TABLE connection_snapshot (
  ts          INTEGER NOT NULL,
  src_ip      TEXT NOT NULL,
  local_ip    TEXT NOT NULL,
  conn_count  INTEGER NOT NULL,
  PRIMARY KEY (ts, src_ip, local_ip)
);
CREATE INDEX idx_snapshot_ts ON connection_snapshot(ts);

CREATE TABLE probe_result (
  ts          INTEGER NOT NULL,
  target_ip   TEXT NOT NULL,
  kind        TEXT NOT NULL CHECK (kind IN ('tcp','tls','tunnel')),
  success     INTEGER NOT NULL CHECK (success IN (0,1)),
  latency_ms  INTEGER,
  error       TEXT
);
CREATE INDEX idx_probe_ts ON probe_result(ts, target_ip);

CREATE TABLE xray_event (
  ts     INTEGER NOT NULL,
  kind   TEXT NOT NULL,
  count  INTEGER NOT NULL,
  PRIMARY KEY (ts, kind)
);

CREATE TABLE update_event (
  ts            INTEGER NOT NULL,
  component     TEXT NOT NULL,
  old_version   TEXT,
  new_version   TEXT,
  probe_passed  INTEGER NOT NULL CHECK (probe_passed IN (0,1)),
  rolled_back   INTEGER NOT NULL CHECK (rolled_back IN (0,1)),
  notes         TEXT
);
CREATE INDEX idx_update_ts ON update_event(ts DESC);

CREATE TABLE heal_state (
  service_name          TEXT PRIMARY KEY,
  last_restart_ts       INTEGER,
  restart_count_window  INTEGER DEFAULT 0,
  cooldown_until_ts     INTEGER
);

CREATE TABLE alert (
  ts         INTEGER NOT NULL,
  level      TEXT NOT NULL CHECK (level IN ('info','warn','crit')),
  kind       TEXT NOT NULL,
  message    TEXT NOT NULL,
  delivered  INTEGER NOT NULL CHECK (delivered IN (0,1))
);
CREATE INDEX idx_alert_ts ON alert(ts DESC);

CREATE TABLE ipinfo_cache (
  ip          TEXT PRIMARY KEY,
  asn         TEXT,
  country     TEXT,
  region      TEXT,
  org         TEXT,
  fetched_ts  INTEGER NOT NULL
);

CREATE TABLE state_cursor (
  name      TEXT PRIMARY KEY,    -- e.g. 'journalctl_xray', 'journalctl_mtproto', 'probe_tick_counter'
  cursor    TEXT NOT NULL,       -- opaque value (e.g. journalctl --cursor= output, or stringified int)
  updated   INTEGER NOT NULL
);
```

**Retention** (run by `bridge-prune` weekly):

| Table                  | Keep      |
|------------------------|-----------|
| connection_snapshot    | 30 days   |
| probe_result           | 30 days   |
| xray_event             | 30 days   |
| alert                  | 90 days   |
| update_event           | forever   |
| heal_state             | forever (one row per service) |
| ipinfo_cache           | refresh entries older than 30 days on next access; never auto-delete |
| state_cursor           | forever (one row per cursor)  |

**Concurrency**: WAL mode; each invocation opens its own `sqlite3.Connection`, uses `BEGIN IMMEDIATE` for writes, `.close()` on exit. The receiver service holds one long-lived connection.

**Initialization**: at first run of any CLI/service, `bridge.db.ensure_schema()` checks `schema_version`; if empty, runs the CREATE statements and inserts `(1, now)`. Future schema migrations are gated by version comparison and ALTER TABLE/INSERT statements; v1 has only the initial schema.

## Scripts

### `bridge-stats`

**Two invocation modes**, same script:

- `bridge-stats` (interactive, no flags): read-only, prints current state to stdout.
- `bridge-stats --tick` (timer): scrape, write to DB, emit alerts, ping heartbeat.

**Tick flow**:

1. Parse `ss -Htn state established '( sport = :443 )'` output. Group by `(src_ip, local_ip)`, insert one row per group into `connection_snapshot` with the current timestamp.
2. For each known journal cursor (`journalctl_xray`, `journalctl_mtproto`), read `journalctl --after-cursor=<saved> --unit=<svc> --output=export` since last tick. The `__CURSOR=...` line in each record is the opaque cursor value; save the last one to `state_cursor.cursor`. Count regex matches per kind (`reject|fail|reset` → kind), insert `xray_event` deltas. First-run (cursor empty): seed with `--since "5 minutes ago"`.
3. Probe sub-step (every 15th tick): for each IP in `network.<role>.public_ips`, attempt TCP connect, TLS handshake, and a tunnel probe (RU: curl-via-socks5; EU: curl-direct). Record one `probe_result` row per (IP, kind).
4. Evaluate alert conditions:
   - **`no_clients`** (warn): zero rows in `connection_snapshot` for the last 30min AND current UTC hour ∈ `[active_hours_utc.start, active_hours_utc.end)`.
   - **`reject_spike`** (warn): current-tick `reject` count > `mean(reject counts in last 24h excluding last hour) + 3σ`, with a minimum absolute threshold of 10 to avoid false positives on near-zero baselines.
   - **`probe_failure`** (crit): a public IP probe failed `tcp` or `tls` on the most recent probe tick.
5. POST any new alerts via `bridge.alerts.send(...)` (see Alert dispatch below).
6. Sweep undelivered `crit` alerts older than 5 min; retry up to 3 total attempts per alert row.
7. GET `healthchecks.heartbeat_url` (HTTP 200 expected; failure logs but doesn't error).

**Interactive output** — three sections separated by blank lines:

```
Connections (now)                            [from 2 unique src IPs]
  COUNT  SOURCE IP        COUNTRY  ORG                   LOCAL IP
     29  95.24.149.161    RU       Rostelecom            198.51.100.10
      1  62.163.138.205   RU       MTS                   198.51.100.10

Public IP probes (last 60min)
  198.51.100.10  tcp ✓  tls ✓  tunnel ✓   (3/3 healthy)
  198.51.100.11  tcp ✓  tls ✓  tunnel ✓   (3/3 healthy)

Recent operational signal
  Rejects (last hour): 2  (baseline 1.4 ± 0.9 — normal)
  Last update:         2026-05-15 04:00 UTC  xray 1.8.27 → 1.8.30  ok
  Last alert:          2026-05-12 14:22 UTC  no_clients (resolved)
```

IP→country/ASN comes from `ipinfo_cache`; cache miss triggers one-shot `https://ipinfo.io/<ip>` with 3s timeout. Unavailable → column shows `?`.

### `bridge-update`

**Invoked once weekly** by `bridge-update.timer` (Sun 04:00 EU, Sun 06:00 RU).

**Flow**:

1. **Lock**: `flock` on `/var/lib/bridge/update.lock`. If already held → exit 0 (can't double-run).
2. **Peer gate** (RU only): GET `peer.url/update-status`. Parse JSON `{ "components": {"xray": {"ts":..., "rolled_back": 0|1}, ...} }`. If any component has `rolled_back=1` within the past 7 days → log "EU rolled back recently, skipping" → ping `&fail` healthcheck → exit 0.
3. **Component loop** for `["apt", "xray", "mtg", "tailscale"]`:
   - **Version check**:
     - apt: `apt-get -s upgrade | grep -c '^Inst '` → number of upgradable packages. If 0, skip.
     - xray: `/usr/local/bin/xray version | head -1` → compare to remote `curl -s https://api.github.com/repos/XTLS/Xray-core/releases/latest | jq -r .tag_name` (no jq — parse with `json.loads` from `urllib`).
     - mtg: `cd /opt/mtg && git fetch && git rev-list HEAD..origin/master --count`. If 0, skip.
     - tailscale: `dpkg-query -W -f='${Version}' tailscale` vs `apt-cache policy tailscale` candidate.
   - **Snapshot** (if proceeding):
     - apt: dpkg state list to `/var/backups/bridge/dpkg-pre-<ts>.txt`.
     - xray: `cp /usr/local/bin/xray /var/backups/bridge/xray-<ts>.bin`; `cp /usr/local/etc/xray/config.json /var/backups/bridge/xray-config-<ts>.json`.
     - mtg: `cp /usr/local/bin/mtg /var/backups/bridge/mtg-<ts>.bin`.
     - tailscale: snapshot package version string only.
   - **Update**:
     - apt: `apt-get update && DEBIAN_FRONTEND=noninteractive apt-get -y upgrade`.
     - xray: `bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install`.
     - mtg: `cd /opt/mtg && git pull && go build -o /usr/local/bin/mtg.new . && mv /usr/local/bin/mtg.new /usr/local/bin/mtg && systemctl restart mtproto`.
     - tailscale: `apt-get install -y --only-upgrade tailscale`.
   - **Health probe** (Section 4 from Q7-B):
     - `systemctl is-active xray mtproto` → both `active`.
     - `ss -Hltn 'sport = :443'` returns ≥1 row.
     - `journalctl -u xray -u mtproto --since "60s ago" --grep "panic|fatal|critical|FATAL" --no-pager` returns empty.
     - **Tunnel test**:
       - RU: `curl -m 8 -x socks5h://127.0.0.1:1080 https://ifconfig.me/ip` → matches `network.eu.public_ip`.
       - EU: `curl -m 5 -k -o /dev/null -w '%{http_code}\n' https://127.0.0.1/` → any 3-digit code (Reality serves fallback from `dest`).
   - **On probe failure** for xray/mtg: restore from snapshot, `systemctl restart`, re-probe. If still failing → mark `rolled_back=1` regardless of re-probe outcome; alert `crit`.
   - **On probe failure** for apt/tailscale: do not auto-downgrade; alert `crit`, mark `rolled_back=0` but include note.
   - Insert `update_event(ts, component, old_version, new_version, probe_passed, rolled_back, notes)`.
4. **Heartbeat**: ping `healthchecks.heartbeat_url` (with `&fail` query if any rollback).
5. **Summary alert**: if any rollback OR any new versions installed → email summary (single email per run, not per component).

**Why mtg gets full rollback but apt doesn't**: apt updates touch dozens of packages with inter-dependencies; "rolling back" is not a single operation. xray/mtg are single binaries — `cp` is the entire rollback.

### `bridge-heal`

**Invoked every 2 minutes** by `bridge-heal.timer`.

**Flow**:

1. For each `service_name` in `["xray", "mtproto"]`:
   - `systemctl is-active <svc>` → if `active`:
     - If `last_restart_ts` exists and (`now - last_restart_ts > 3600`) → set `restart_count_window = 0` in `heal_state` (decay).
     - Continue to next service.
   - Service is not active:
     - Read `heal_state` row. If `cooldown_until_ts > now` → skip.
     - If `restart_count_window >= 3` → set `cooldown_until_ts = now + 3600`, emit `service_flapping` crit alert, skip restart.
     - Else: `systemctl restart <svc>`, sleep 5s, re-check `is-active`.
       - Recovered: `last_restart_ts = now`, `restart_count_window += 1`, `cooldown_until_ts = now + 300`, emit `service_restarted` info alert.
       - Still failing: emit `restart_failed` crit alert, `cooldown_until_ts = now + 600`.
2. Ping `healthchecks.heartbeat_url` (always — heal noise is normal; only the email alerts escalate).

**Heal does NOT do**:
- Tunnel probing (lives in `bridge-stats`).
- Restart anything beyond xray/mtproto.
- Touch the systemd unit files.

## Alert dispatch (`bridge.alerts`)

`def send(level: str, kind: str, message: str) -> None`:

1. INSERT `alert(ts, level, kind, message, delivered=0)`.
2. If `node.role == "eu"`:
   - Format email: subject `[bridge:{level}] {kind}`, body includes `message`, source node, ts, and the last 5 `update_event` rows for context.
   - `smtplib.SMTP(host, port).starttls().login(user, pw).sendmail(...)`.
   - On success: UPDATE alert SET delivered=1.
   - On SMTP failure: log to journalctl, leave delivered=0 (retried by stats sweep).
3. If `node.role == "ru"`:
   - POST `peer.url/alert` with `{level, kind, message, source: self_tailnet, ts}` as JSON body.
   - On 2xx: UPDATE alert SET delivered=1.
   - Otherwise: log, leave delivered=0.

Crit alerts undelivered for >5min are retried by the next `bridge-stats --tick`, max 3 attempts.

## EU receiver (`bridge.receiver`)

`bridge-receiver.service` runs continuously on EU only. Python `http.server.ThreadingHTTPServer` bound to the tailnet IP (resolved via `tailscale ip -4` at startup), port from `receiver.tailnet_listen_port`.

**Endpoints**:

- `GET /update-status` → 200 with JSON:
  ```json
  {
    "components": {
      "xray":      {"ts": 1715900000, "version": "1.8.30", "rolled_back": 0},
      "mtg":       {"ts": 1715900000, "version": "deadbeef", "rolled_back": 0},
      "apt":       {"ts": 1715900000, "version": null,     "rolled_back": 0},
      "tailscale": {"ts": null,       "version": "1.78.0",  "rolled_back": 0}
    }
  }
  ```
  Latest row per component from `update_event`. `ts: null` if never updated.
- `POST /alert` (max body 16KB): parse JSON body, validate fields, dispatch email via `bridge.alerts.send`, return 204.

**Error handling**:
- Bad JSON → 400.
- Body > 16KB → 413.
- Handler exception → 500, log to journalctl.
- Email failure inside POST: persist alert row with `delivered=0`; HTTP response is still 204 (RU shouldn't retry — the next stats sweep handles it).

**Auth**: tailnet binding is the auth. The Tailscale admin console's default-deny ACL gates which devices can hit the port; tighten with `tag:bridge-ops` ACL rules if you want device-level restrictions.

**Restart policy**: `Restart=always, RestartSec=5` in the unit.

## systemd units

All units share `User=root`, `StandardOutput=journal`, `StandardError=journal`. Listed by file:

```ini
# /etc/systemd/system/bridge-stats.service
[Unit]
Description=Bridge stats scraper
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/bridge-stats --tick
```

```ini
# /etc/systemd/system/bridge-stats.timer
[Unit]
Description=Run bridge-stats every minute

[Timer]
OnCalendar=*:0/1
AccuracySec=10s
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/bridge-heal.service
[Unit]
Description=Bridge service self-heal
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/bridge-heal
```

```ini
# /etc/systemd/system/bridge-heal.timer
[Unit]
Description=Run bridge-heal every 2 minutes

[Timer]
OnCalendar=*:0/2
AccuracySec=10s

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/bridge-update.service
[Unit]
Description=Bridge auto-update with safety gates
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/bridge-update
TimeoutStartSec=15min
```

```ini
# /etc/systemd/system/bridge-update.timer (RU: 06:00; EU: 04:00 — rendered by make install)
[Unit]
Description=Weekly bridge auto-update

[Timer]
OnCalendar=Sun *-*-* @HOUR@:00:00 UTC   # @HOUR@ replaced by make install
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/bridge-prune.service + .timer  (OnCalendar=Sun 03:00 UTC)
```

```ini
# /etc/systemd/system/bridge-receiver.service (EU only)
[Unit]
Description=Bridge EU HTTP receiver
After=tailscaled.service
Requires=tailscaled.service

[Service]
Type=simple
ExecStart=/usr/local/bin/bridge-receiver
Restart=always
RestartSec=5
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
```

## Deployment

**Repo layout** (under `/home/asharov/Scripts/mtproxy/ops/`):

```
ops/
├── Makefile
├── config.toml.example
├── bridge/
│   ├── __init__.py
│   ├── config.py
│   ├── db.py
│   ├── alerts.py
│   ├── ipinfo.py
│   ├── stats.py
│   ├── update.py
│   ├── heal.py
│   ├── receiver.py
│   └── prune.py
├── bin/
│   ├── bridge-stats
│   ├── bridge-update
│   ├── bridge-heal
│   ├── bridge-prune
│   └── bridge-receiver
├── systemd/
│   ├── bridge-stats.service
│   ├── bridge-stats.timer
│   ├── bridge-heal.service
│   ├── bridge-heal.timer
│   ├── bridge-update.service
│   ├── bridge-update.timer.in
│   ├── bridge-prune.service
│   ├── bridge-prune.timer
│   └── bridge-receiver.service
└── tests/
    ├── test_config.py
    ├── test_db.py
    ├── test_ss_parser.py
    ├── test_journalctl_parser.py
    ├── test_alerts.py
    └── test_update_flow.py
```

**Makefile**:

```make
PREFIX ?= /usr/local
PYLIB  ?= /usr/local/lib/python3/dist-packages
SYSD   ?= /etc/systemd/system
CFG    ?= /etc/bridge/config.toml

install:
	@test -f $(CFG) || (echo "config missing at $(CFG); copy config.toml.example first"; exit 1)
	install -m 0755 -d $(PREFIX)/bin $(PYLIB)/bridge $(SYSD) /var/lib/bridge /var/backups/bridge
	install -m 0755 bin/* $(PREFIX)/bin/
	install -m 0644 bridge/*.py $(PYLIB)/bridge/
	install -m 0644 systemd/bridge-stats.{service,timer} $(SYSD)/
	install -m 0644 systemd/bridge-heal.{service,timer} $(SYSD)/
	install -m 0644 systemd/bridge-update.service $(SYSD)/
	install -m 0644 systemd/bridge-prune.{service,timer} $(SYSD)/
	@ROLE=$$(python3 -c 'import tomllib; print(tomllib.load(open("$(CFG)","rb"))["node"]["role"])'); \
	if [ "$$ROLE" = "ru" ]; then HOUR=06; else HOUR=04; fi; \
	sed "s/@HOUR@/$$HOUR/" systemd/bridge-update.timer.in > $(SYSD)/bridge-update.timer; \
	if [ "$$ROLE" = "eu" ]; then \
	  install -m 0644 systemd/bridge-receiver.service $(SYSD)/; \
	fi
	systemctl daemon-reload

uninstall:
	systemctl disable --now bridge-*.timer bridge-receiver.service 2>/dev/null || true
	rm -f $(PREFIX)/bin/bridge-*
	rm -rf $(PYLIB)/bridge
	rm -f $(SYSD)/bridge-*.{service,timer}
	systemctl daemon-reload

test:
	python3 -m unittest discover tests -v

lint:
	python3 -m py_compile bridge/*.py bin/*

smoke:
	bridge-stats --dry-run
	bridge-update --dry-run
	bridge-heal --dry-run
	bridge-prune --dry-run
```

**First-time deployment** per node (over Tailscale SSH):

```bash
sudo git clone https://github.com/<your>/mtproxy /opt/bridge-ops
sudo install -m 700 -d /etc/bridge /var/lib/bridge /var/backups/bridge
sudo cp /opt/bridge-ops/ops/config.toml.example /etc/bridge/config.toml
sudo chmod 600 /etc/bridge/config.toml
sudo $EDITOR /etc/bridge/config.toml          # fill in role, IPs, SMTP, hc urls, peer url
cd /opt/bridge-ops/ops && sudo make install
sudo systemctl enable --now bridge-stats.timer bridge-heal.timer bridge-update.timer bridge-prune.timer
# EU only:
sudo systemctl enable --now bridge-receiver.service
```

**Updating the ops layer itself** (operator-driven, NOT auto-updated):

```bash
cd /opt/bridge-ops && git pull
cd ops && sudo make install
sudo systemctl restart bridge-receiver.service    # EU only, if receiver.py changed
```

Timers pick up the new code on their next firing — no per-timer restart needed since each invocation is a fresh process.

## Testing strategy

**Unit tests** (stdlib `unittest`, run via `make test`):

- `test_config.py` — valid configs load; invalid role/IPs raise `ConfigError` with specific messages.
- `test_db.py` — `ensure_schema` is idempotent; retention pruning deletes correct rows.
- `test_ss_parser.py` — given canned `ss` output, produces expected (src_ip, local_ip, count) tuples.
- `test_journalctl_parser.py` — given canned journalctl lines, classifies into reject/fail/reset/info correctly; cursor advances.
- `test_alerts.py` — RU sends POST, EU sends SMTP, both insert with correct `delivered` flag; retry path works.
- `test_update_flow.py` — full update flow with `subprocess.run` patched: version check → snapshot → update → probe success/failure → correct DB row.

**Smoke tests** (run on a live box, idempotent):

- Every CLI accepts `--dry-run`: goes through the motions, prints what *would* happen, no DB writes or external calls.
- `make smoke` runs all four in sequence.

**No CI initially**. Operator runs `make test` before `git push`.

## Operational concerns

### Multi-IP semantics (clarified)

`network.ru.public_ips` lists every public IP attached to the RU interface (per the netplan procedure in `mtproxy-v2.md` §8.3). `bridge-stats`:
- Groups `connection_snapshot` rows by `local_ip` so the operator sees which of their IPs each client hit.
- The probe sub-step inside `bridge-stats --tick` runs every 15th tick (≈ every 15 minutes), iterating over each entry in `public_ips` and recording `probe_result` rows (kinds: `tcp`, `tls`, `tunnel`). Probe targets the box from inside (loopback for tunnel/tls, connect to the public IP for tcp via the local interface). External-vantage probing (check-host.net) is out of scope for v1 — the on-box probe catches the local-service-broken case, not the IP-blocklisted-by-RKN case. Add external probing in a future iteration if needed.

The `primary_ip` field is informational: it's what's in the user-facing `tg://` link. If the operator changes the link out-of-band, they update this field and `bridge-stats` knows which IP is the "advertised" one for display.

### Email deliverability from EU

SMTP from a small EU VPS is moderate-difficulty:
- Use Gmail relay (or another reputable provider) with an app password; do not run a local MX.
- `from_addr` should be on a domain whose SPF/DKIM you control, not a Gmail address (Gmail rejects "via" relayed mail with a stricter heuristic).
- Test deliverability before relying on it; send a manual `python3 -c "import bridge.alerts; bridge.alerts.send('info', 'test', 'hello')"` after first install.

### Healthchecks.io setup

Two checks in the operator's healthchecks.io account: `ru-bridge-heartbeat` and `eu-bridge-heartbeat`. Set "grace period" to 5 min, "schedule" to 1/minute. Notification channel: same email as `email.to_addrs`. The free tier (20 checks, monthly cap) is enough.

### Rollback playbook (operator-side)

If `bridge-update` rolled back a binary and the alert email arrives:
1. SSH to the affected node via Tailscale.
2. `journalctl -u bridge-update --since "1 hour ago"` to read the rollback context.
3. `bridge-stats` to see current state.
4. `ls /var/backups/bridge/` to see what snapshots are available.
5. Decide whether to: (a) re-run the update manually with `bridge-update --component xray` (future CLI flag — not in v1), (b) wait for upstream fix, or (c) pin a known-good version in the config (also a future feature).

For v1, the rollback alert means "automation did its job; the bridge is on the old version; investigate at your leisure."

### Multi-IP rotation flow (operator-side)

This is unchanged from `mtproxy-v2.md` §8.3 — the ops tooling doesn't automate it. But the visibility is improved:
- Before rotation: `bridge-stats` shows both old and new IP probe results, helping pick which to promote.
- After rotation: update `primary_ip` in config, `bridge-stats` displays the new active IP context.

## Open questions / explicitly deferred

1. **mtg stats endpoint migration**: switching from `simple-run` to `run` with `stats-bind` would give real per-session metrics. Deferred to v3 because it requires changing the `mtproto.service` ExecStart line.
2. **Active-probing detection from passive traffic shape** (short connections, no MTProto handshake completed) is *not* implemented in v1. The reject/fail count from xray is a strong-enough proxy.
3. **Auto-IP-promotion** and **auto-secret-rotation**: explicitly non-goals; revisit only if operator load proves too high.
4. **Web dashboard**: not in v1. Could be added later as a fourth CLI (`bridge-dashboard`) running a small `http.server` on the tailnet with a single static HTML page.

## Out of scope

- CI/CD for the ops repo (operator runs `make test` locally).
- Backup of `state.db` (it's stats history; loss is recoverable from journalctl + rescraping).
- Disaster recovery beyond "redeploy from `mtproxy-v2.md` + clone ops repo + `make install`".
- Per-user identity or authentication for clients (they're identified by source IP only — Telegram MTProto secret is shared).
