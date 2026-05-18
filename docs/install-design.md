# Install Scripts — Design Spec

**Created:** 2026-05-18
**Targets:** automate `docs/setup.md` §1–§8 for both bridge nodes
**Status:** approved through brainstorming; ready for implementation plan
**Related:** `docs/ops-design.md` (downstream — runs after install completes)

## Goals

Reduce a fresh-VPS-to-working-bridge deploy to two commands per node (`cp install.toml.example /etc/bridge/install.toml && nano /etc/bridge/install.toml && sudo install/eu-setup.py`) with idempotent recovery and TDD-shaped code.

1. **Automate the manual setup procedure.** Cover apt prereqs, SSH key auth, Tailscale install + join, Xray install + Reality config, mtg install + MTProto secret (RU), systemd units, UFW lockdown. The bridge must be operationally working after the script exits 0.
2. **Idempotent.** Re-run is always safe. Each step checks system state and skips if already done. A re-run after partial failure resumes from where it left off.
3. **v1 detection and migration.** Refuse to run on systems showing v1 artefacts (sslh present, old Xray dest, old mtproto.service ExecStart) without `--migrate-from-v1`, which removes/replaces the v1 pieces inline.
4. **TDD throughout.** Every step function has a unit test (happy path), an idempotent-skip test, and a precondition-failure test. The implementation plan enforces test-first.
5. **Operator UX.** Single `install.toml` file (chmod 600) is the source of truth. Missing fields prompt interactively; pre-filled fields run unattended. Validation errors name the specific field.

## Non-goals

- **Bootstrap of the ops tooling.** Installing the bridge and installing the ops layer are separate concerns. After both `*-setup.py` scripts succeed, the operator runs `cd ops && sudo make install` (per `ops-design.md`).
- **SSH key generation or distribution.** Operator already has an SSH key; the script tightens `sshd_config` (password-auth off) but does not provision keys.
- **VPS provisioning.** The script assumes both VPSes already exist and operator has SSH access.
- **Multi-IP setup automation.** Adding a second IP to the RU box is documented in `docs/setup.md` §8.3 (netplan procedure). The install script reads multi-IP from `install.toml` if pre-configured but does not run netplan itself — operator handles netplan separately.
- **Rollback of install changes.** If `eu-setup.py` fails halfway, re-run after fixing the underlying issue. There is no `--rollback` flag.
- **Generic configuration management.** This is a one-off bootstrap tool, not Ansible.

## Architecture

Two Python 3.12 scripts, stdlib only:

- `install/eu-setup.py` — EU node installer
- `install/ru-setup.py` — RU node installer

Shared logic in `install/_lib/`. Each entry script is a thin orchestrator: it loads config, validates, then calls a sequence of step functions imported from `_lib/steps_eu.py` or `_lib/steps_ru.py`. Step functions compose primitive helpers (`_lib/apt.py`, `_lib/systemd.py`, etc.) which are the boundary between testable code and real system calls.

```
eu-setup.py / ru-setup.py
  │
  ▼
_lib/steps_eu.py (step_apt_prereqs, step_install_tailscale, ...)
  │
  ▼
_lib/apt.py, _lib/systemd.py, _lib/tailscale.py, _lib/ufw.py,
_lib/fs.py, _lib/net.py, _lib/checks.py, _lib/prompts.py,
_lib/templates.py, _lib/toml_io.py
  │
  ▼
subprocess.run(...) / Path.write_text(...) / urllib.request.urlopen(...)
```

Tests mock the primitive helpers; step-function tests assert helpers were called in the right order with the right arguments. Helpers themselves get targeted unit tests against canned stdin/stdout fixtures.

## Operator workflow

```
EU node:                                    RU node (after EU completes):
1. clone repo to /opt/bridge-ops            5. clone repo
2. sudo install -m 700 -d /etc/bridge       6. sudo install -m 700 -d /etc/bridge
3. cp install/install.eu.toml.example \     7. cp install/install.ru.toml.example \
      /etc/bridge/install.toml                    /etc/bridge/install.toml
   sudo chmod 600 /etc/bridge/install.toml     sudo chmod 600 /etc/bridge/install.toml
   nano /etc/bridge/install.toml               nano /etc/bridge/install.toml
                                                  (paste [from_eu] block from step 4)
4. sudo install/eu-setup.py                 8. sudo install/ru-setup.py
   → prints [from_eu] block at end             → ends with tg:// link
   
later: cd ops && sudo make install          later: cd ops && sudo make install
```

## `install.toml` schemas

`install.toml` is the single source of truth — install-time and ops-layer fields combined. After install succeeds, the script writes a projection to `/etc/bridge/config.toml` (ops-layer subset only) for the ops layer to consume.

### EU template (`install/install.eu.toml.example`)

```toml
[node]
role          = "eu"
self_tailnet  = "eu-bridge"
peer_tailnet  = "ru-bridge"

[tailscale]
auth_key = ""                          # empty → interactive URL approval

[network.eu]
public_ip = ""                          # empty → auto-detect from default route

[network.ru]                            # echoed into config.toml for ops layer
public_ips = ["198.51.100.10"]
primary_ip = "198.51.100.10"

[reality]
dest_host = "gateway.icloud.com"

[email]
smtp_host     = "smtp.gmail.com"
smtp_port     = 587
smtp_user     = "you@example.com"
smtp_password = ""                      # prompted if empty (getpass)
from_addr     = "bridge-alerts@example.com"
to_addrs      = ["you@example.com"]

[healthchecks]
heartbeat_url = ""                      # prompted if empty

[receiver]
tailnet_listen_port = 8742

[active_hours_utc]
start = 6
end   = 22

[update]
day             = "sunday"
hour_utc        = 4
ru_offset_hours = 2

# === Generated by eu-setup.py — do not hand-edit ===
[generated]
reality_private_key = ""
reality_public_key  = ""
reality_short_id    = ""
xray_client_uuid    = ""
```

### RU template (`install/install.ru.toml.example`)

```toml
[node]
role          = "ru"
self_tailnet  = "ru-bridge"
peer_tailnet  = "eu-bridge"

[tailscale]
auth_key = ""

[network.ru]
public_ips = []                         # empty → auto-detect; list for multi-IP
primary_ip = ""

[network.eu]
public_ip = ""                          # filled from [from_eu] if left empty

[fake_tls]
sni = "rutube.ru"

[mtg]
upstream_dns    = "1.1.1.1"
bandwidth_limit = "512kib"

[healthchecks]
heartbeat_url = ""

[peer]
url = "http://eu-bridge:8742"

[active_hours_utc]
start = 6
end   = 22

[update]
day             = "sunday"
hour_utc        = 4
ru_offset_hours = 2

# === Paste from EU's eu-setup.py final output ===
[from_eu]
public_ip          = ""
xray_client_uuid   = ""
reality_public_key = ""
reality_short_id   = ""
reality_dest_host  = ""

# === Generated by ru-setup.py — do not hand-edit ===
[generated]
mtproto_secret = ""
```

### Validation (in `_lib/checks.py`)

- `[node].role` must match the script invoked. `eu-setup.py` refuses if `role != "eu"`; vice versa.
- Auto-detect: if `network.<self>.public_ip(s)` is empty, populate from `ip -j route get 1.1.1.1` (source IP). Multi-IP detection on RU uses `ip -j addr show` filtered to scope-global non-loopback. Auto-detected values are written back to `install.toml`.
- `primary_ip ∈ public_ips` (RU only).
- RU only: every `[from_eu]` field must be non-empty after prompts. Refuse otherwise with exact message: `"paste the [from_eu] block from the EU node's eu-setup.py output before re-running"`.
- All IPs parse via `ipaddress.ip_address`.
- Reality `dest_host`: verified via `xray reality test <host>:443` during step 8. On failure, list the three alternates from `docs/setup.md` §3.2 and exit non-zero.
- `[email].to_addrs` must be a non-empty list (EU only).
- `[active_hours_utc]`: `0 ≤ start < end ≤ 24`.
- `[update].hour_utc ∈ [0, 23]`; `[update].day == "sunday"` (only supported value in v1).

Validation runs **before any side-effecting step**. Errors include the exact TOML key path.

## EU install flow

17 step functions, executed in order. Each idempotent.

| # | Step | Idempotency probe |
|---|------|-------------------|
| 1 | Pre-flight: root, OS, network, install.toml present + mode 600, v1 detection | — |
| 2 | Load + validate `install.toml`, auto-detect missing fields, prompt for required-but-empty | — |
| 3 | `apt-get update` | `/var/cache/apt/pkgcache.bin` mtime < 1h |
| 4 | `apt-get install -y ufw curl uuid-runtime` | `dpkg-query -W` for each |
| 5 | UFW bootstrap: allow 22/tcp, allow 443/tcp, `--force enable` | `ufw status` lists both + active |
| 6 | SSH hardening: `PasswordAuthentication no` | grep config |
| 7 | Tailscale install via curl install.sh | `which tailscale` |
| 8 | Tailscale up + authenticate (auth key or interactive URL, poll until ready) | `tailscale status --json` shows authenticated AND matching hostname |
| 9 | Install Xray via XTLS install-release.sh | `/usr/local/bin/xray` present and < 30d old |
| 10 | Generate Reality keys, ShortID, UUID; write to `[generated]` | `[generated]` fields all non-empty |
| 11 | `xray reality test {dest_host}:443` | (always run; cheap) |
| 12 | Render + write `/usr/local/etc/xray/config.json` | SHA-256 of rendered = SHA-256 on disk |
| 13 | Patch xray.service: `User=nobody` → `DynamicUser=yes`; `daemon-reload` | grep service file |
| 14 | `systemctl enable --now xray`, wait, assert active + :443 listening + no panic in journal | `systemctl is-active xray == "active"` AND `ss -Hltn '( sport = :443 )'` non-empty |
| 15 | Close UFW :22 | `ufw status` shows only 443/tcp public |
| 16 | Render + write `/etc/bridge/config.toml` (ops-layer projection) | SHA-256 match |
| 17 | Print `[from_eu]` handoff block + next steps | — |

## RU install flow

21 step functions. Common steps (1–8 of EU pattern) reused; RU-specific:

| # | Step | Idempotency probe |
|---|------|-------------------|
| 1 | Pre-flight (same as EU; v1 markers include sslh, old mtproto.service) | — |
| 2 | Load + validate (`role=ru`; `[from_eu]` fully populated required) | — |
| 3 | apt update + install (`ufw curl proxychains-ng`) | dpkg-query |
| 4 | UFW bootstrap | `ufw status` |
| 5 | SSH hardening | grep |
| 6 | Tailscale install + up (hostname=ru-bridge) | tailscale status |
| 7 | Install Go 1.26.1 to `/usr/local/go` if absent or wrong version | `go version` |
| 8 | Clone or `git pull` `/opt/mtg`; `go build -o /usr/local/bin/mtg .` | binary present + age < 30d |
| 9 | Generate MTProto secret: `mtg generate-secret --hex {fake_tls.sni}` | `[generated].mtproto_secret` non-empty |
| 10 | Install Xray (same as EU) | binary present + age check |
| 11 | TCP probe to `[from_eu].public_ip:443` (warn-only) | — |
| 12 | Render + write `/usr/local/etc/xray/config.json` (CLIENT variant: VLESS outbound + SOCKS inbound :1080) | SHA-256 match |
| 13 | Patch xray.service `DynamicUser=yes` | grep |
| 14 | `systemctl enable --now xray`; assert active + 127.0.0.1:1080 listening | systemctl + ss |
| 15 | Write `/etc/proxychains.conf` (idempotent) | SHA-256 match |
| 16 | Write `/etc/systemd/system/mtproto.service` from template with pinned flags | SHA-256 match |
| 17 | `systemctl daemon-reload`; `systemctl enable --now mtproto`; assert active | systemctl |
| 18 | **End-to-end tunnel test** (gating): `curl -m 8 -x socks5h://127.0.0.1:1080 https://ifconfig.me/ip` returns `[from_eu].public_ip` | — (always run; fail = exit 1) |
| 19 | Close UFW :22 | ufw status |
| 20 | Render + write `/etc/bridge/config.toml` | SHA-256 match |
| 21 | Print success + `tg://proxy?server={primary_ip}&port=443&secret={mtproto_secret}` link | — |

## v1 detection + migration

`_lib/checks.detect_v1_artefacts() → list[str]`:

| Marker | Detection | Migration action |
|---|---|---|
| sslh installed | `dpkg-query -W sslh` non-empty | `systemctl disable --now sslh`, `apt purge -y sslh`, restore UFW to `allow 22 + 443` |
| Old Xray dest (microsoft) | `grep -q microsoft.com /usr/local/etc/xray/config.json` | Backup config to `/var/backups/bridge/xray-v1-{ts}.json` then rewrite in step 12 |
| Old mtproto.service ExecStart | ExecStart lacks `--secure-only` | Backup unit to `/var/backups/bridge/mtproto-v1-{ts}.service`, stop service before step 16 rewrites it |
| Xray/mtg on `:8443` (interim sslh+backend layout) | `ss -Hltn 'sport = :8443'` non-empty | Service reconfig in steps 12/16 moves them back to `:443` (UFW already allows 443) |

Refuse to start without `--migrate-from-v1` if any marker is present, listing exactly which. With the flag: print the list, perform each migration action, log to `/var/log/bridge-install.log`, then proceed normally.

## CLI

Both scripts share the same flag set:

```
sudo install/eu-setup.py [OPTIONS]

  --config PATH              path to install.toml (default /etc/bridge/install.toml)
  --migrate-from-v1          accept v1 artefacts and migrate inline
  --skip-tailscale-auth      assume Tailscale is already up (testing/e2e)
  --skip-service-start       write files but don't enable/start services (e2e)
  --unattended               fail rather than prompt for missing values
  --verbose                  log every command invoked
  -h, --help
```

Exit codes:

- `0` — success
- `1` — runtime failure (one of the steps did not complete its assertion)
- `2` — pre-condition failure (config missing, wrong role, v1 detected without flag)
- `78` (EX_CONFIG) — `install.toml` malformed / failed validation

## Testing strategy

TDD-shaped across all of `install/` (and matching the existing ops layer plan). Three layers; same `python3 -m unittest discover` runner; stdlib only.

### Layer 1 — pure-function unit tests

| Module | Coverage |
|---|---|
| `_lib/checks.py` | `validate_install_toml(cfg)` raises with named field on each violation; `detect_v1_artefacts` returns expected list given fake inputs |
| `_lib/templates.py` | `render_xray_server_config(values)` produces dict with correct nesting + values; `render_xray_client_config(values)`; `render_mtproto_service(values)` produces string with all required flags; `render_proxychains_conf(values)`; `render_ops_config_toml(install_toml)` projects only ops fields |
| `_lib/toml_io.py` | Load + update `[generated]` + save preserves comments and unrelated keys; refuses to load with mode > 600 |
| `_lib/prompts.py` | Multi-line paste of `[from_eu]` block is parsed correctly; `getpass` is used for password fields |

### Layer 2 — mocked-subprocess unit tests

| Module / area | Coverage |
|---|---|
| `_lib/apt.py` | `apt_install([pkg])` invokes correct apt-get command with DEBIAN_FRONTEND set; `dpkg_query` parses output |
| `_lib/systemd.py` | `is_active`, `enable_now`, `restart`, `daemon_reload` each call expected systemctl args; parse output correctly |
| `_lib/tailscale.py` | `is_authenticated()` returns True/False from canned `--json` output; `up(hostname)` uses auth key when provided |
| `_lib/ufw.py` | `status()` parses the `ufw status` text format; `allow`, `delete_allow` call expected args |
| `_lib/fs.py` | `write_file_idempotent` returns False (no change) when content matches; True when it differs; sets correct mode |
| `_lib/net.py` | `detect_default_route_src` parses `ip -j route get 1.1.1.1` output |
| `_lib/steps_eu.py` | Each `step_*` function: happy path, idempotent skip, refuse-on-precondition |
| `_lib/steps_ru.py` | Same |

**Every step has three tests minimum** (happy / skip / refuse). Step tests patch the primitive helpers; primitive-helper tests patch `subprocess.run`.

### Layer 3 — Docker e2e

`tests/e2e/Dockerfile.test`:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y python3 python3-pip systemd
COPY . /repo
WORKDIR /repo
```

Three e2e scenarios (`tests/e2e/run-eu.sh`, `run-ru.sh`, `assert-idempotent.sh`):

1. **Fresh EU install.** Write minimal install.toml → `python3 install/eu-setup.py --skip-tailscale-auth --skip-service-start` → assert `/usr/local/etc/xray/config.json` parses, contains `gateway.icloud.com`, ALPN h2, packet-up; `[generated]` populated in install.toml; `/etc/bridge/config.toml` projected.
2. **Idempotent re-run.** Run install twice; compare SHA-256 of every written file before/after second run; must be identical.
3. **v1 detection.** Pre-seed `dpkg -i` a stub sslh package + a sample old xray config → run without flag → assert exit 2 and stderr lists exactly `["sslh", "xray dest=microsoft.com"]`. Re-run with `--migrate-from-v1` → assert sslh purged, install completes, exit 0.

E2E for ops layer (after EU install runs): `cd ops && make install`, `bridge-stats --tick` (with the database stubbed via `BRIDGE_DB_PATH=/tmp/test.db`), `python3 -m unittest discover -s ops/tests`.

### Layer 4 — manual smoke

The `docs/setup.md` §6 verification block, run by the operator after deploying to actual VPSes. Not automated; the final human gate.

### Test layout

```
install/tests/           # per-module unit + step tests
ops/tests/               # per-module unit tests (already in ops plan)
tests/fixtures/          # repo-root: install.eu.minimal.toml, ss_sample.txt,
                         # journalctl_sample.bin, dpkg_v1_sample.txt
tests/e2e/               # repo-root: Dockerfile.test, run-eu.sh, run-ru.sh, assert-idempotent.sh
```

### Top-level Makefile

```make
.PHONY: test test-install test-ops e2e lint

test: test-install test-ops

test-install:
	cd install && python3 -m unittest discover -s tests -v

test-ops:
	cd ops && python3 -m unittest discover -s tests -v

lint:
	cd install && python3 -m py_compile _lib/*.py *-setup.py
	cd ops && python3 -m py_compile bridge/*.py

e2e:
	docker build -t mtbridge-e2e -f tests/e2e/Dockerfile.test .
	docker run --rm mtbridge-e2e bash /repo/tests/e2e/run-eu.sh
	docker run --rm mtbridge-e2e bash /repo/tests/e2e/run-ru.sh
	docker run --rm mtbridge-e2e bash /repo/tests/e2e/assert-idempotent.sh
```

`make test` is the inner-loop command and must complete in under 10 seconds. `make e2e` runs Docker; expected ~2 minutes; run before commits that change install logic.

## Open questions / explicitly deferred

1. **GitHub Actions CI.** Hook `make test` and `make e2e` into a workflow. Trivial when wanted, out of v1 scope.
2. **Network setup automation.** The netplan procedure in `docs/setup.md` §8.3 (multi-IP) is operator-manual for v1. If multi-IP becomes common, automate via a `--add-public-ip` flag that updates netplan and re-applies.
3. **Tailscale ACL provisioning.** Currently operator approves devices manually at login.tailscale.com. If using a Tailscale account with the API enabled, the install script could auto-approve via API — deferred until/unless this becomes friction.
4. **Cross-node coordination during install.** Currently EU→RU handoff is manual paste. If EU and RU are installed simultaneously (e.g. via Terraform), a Tailscale-based fetch (option B from brainstorming) could be revisited.

## Out of scope

- Provisioning the VPSes themselves (Terraform, cloud-init, etc.).
- Distributing the `tg://` link to end users.
- Backup or disaster recovery of `install.toml` / `config.toml` / SQLite state.
- Per-user customization of `mtproto.service` flags beyond `mtg.upstream_dns` and `mtg.bandwidth_limit` in `install.toml`.
