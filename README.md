# torrent-setup

Headless qBittorrent, tailnet-only Web UI, no incoming peer connections. Reproduces the
setup running on Liran's home machine so a friend can clone this on their own box and have
Claude Code install and configure it.

- **Client**: `qbittorrent-nox` (headless), run as a per-user systemd service.
- **Storage**: config/state live under `profile/` in this repo's clone location (kept out of
  git); downloads land in `~/Downloads/torrents`, outside the repo.
- **Access**: Web UI binds to `127.0.0.1:8080` only, published to the tailnet via
  `tailscale serve`. It is never exposed to the public internet.
- **Peers**: incoming connections are blocked (ufw denies the BitTorrent port), UPnP is off.
  Downloads still work — only inbound seeding/connectability is disabled.

To set this up on a new machine, open this folder in Claude Code and say "set this up" —
`CLAUDE.md` has the full runbook, including generating a fresh Web UI password (nothing
secret is committed to this repo).
