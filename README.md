# mtbridge

A two-hop, DPI-resistant Telegram proxy bridge for a small (≤100) circle of users in Russia, with operational tooling to run it unattended.

```
RU client ──MTProto Fake-TLS (SNI=rutube.ru)──▶ RU VPS
                                                  │ VLESS + XHTTP + Reality
                                                  ▼
                                                EU VPS ──▶ Telegram DCs
```

SSH access is via Tailscale only — neither node has public SSH. Both nodes are Ubuntu 24.04, exposing `:443/tcp` only.

## Documentation

- **[docs/setup.md](docs/setup.md)** — full setup guide for both VPSes. Start here.
- **[docs/ops-design.md](docs/ops-design.md)** — design spec for the operational tooling (stats, auto-update, self-heal, alerts). The `ops/` directory implements this spec.
- **[docs/archive/](docs/archive/)** — historical artifacts (v1 setup guide, v1→v2 migration plan).

## Status

- Setup guide: production-ready (v2, revised 2026-05-17).
- Operational tooling: design approved; implementation pending.

## License

[LICENSE](LICENSE).
