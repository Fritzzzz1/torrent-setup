# torrent-setup

This repo reproduces a headless-qBittorrent setup: Web UI reachable only over the owner's
Tailscale tailnet, no incoming peer connections accepted. Nothing secret is committed here —
follow this runbook to install and wire it up fresh on this machine.

Target OS: Arch / Omarchy. Adjust package manager calls if that's wrong.

## 0. Preconditions

- Confirm this machine is on the intended tailnet: `tailscale status`. If not logged in,
  stop and tell the user to run `sudo tailscale up` themselves (needs interactive auth) —
  don't try to do this for them.
- Note the absolute path this repo is cloned to; call it `$REPO`. All paths below assume
  `$REPO` (e.g. `~/dev/torrent-setup`).

## 1. Install the client

```
sudo pacman -S --needed qbittorrent-nox ufw
```

(Use the sudo-terminal pattern in the root CLAUDE.md if one exists for this machine —
otherwise just ask the user to approve the `sudo pacman` prompt directly.)

## 2. Lay down the profile

```
mkdir -p "$REPO/profile/qBittorrent/config"
sed "s|__HOME__|$HOME|g" "$REPO/config/qBittorrent.conf.template" \
  > "$REPO/profile/qBittorrent/config/qBittorrent.conf"
mkdir -p "$HOME/Downloads/torrents"
```

This ships with `Session\UseUPnP=false` and no incoming-port forwarding — matches step 4.
No WebUI username/password is set yet; first boot uses qBittorrent's built-in default
(`admin`), with a one-time temporary password it prints to its log.

## 3. Install and start the systemd unit

```
sed "s|__REPO_DIR__|$REPO|g" "$REPO/systemd/qbittorrent.service.template" \
  > ~/.config/systemd/user/qbittorrent.service
systemctl --user daemon-reload
systemctl --user enable --now qbittorrent
loginctl enable-linger "$USER"   # so the service survives logout, not just active sessions
```

Grab the temporary admin password qBittorrent generates on first launch:

```
journalctl --user -u qbittorrent -n 50 --no-pager | grep -i "temporary password"
```

## 4. Set a real Web UI username/password

Pick (or ask the user for) a username and a strong password. Then, using the temporary
password from step 3:

```
COOKIE=$(curl -s -i http://127.0.0.1:8080/api/v2/auth/login \
  --data "username=admin&password=<temp password from journal>" \
  | grep -i '^set-cookie' | sed 's/Set-Cookie: //I;s/;.*//')

curl -s http://127.0.0.1:8080/api/v2/app/setPreferences \
  -H "Cookie: $COOKIE" \
  --data-urlencode 'json={"web_ui_username":"<chosen user>","web_ui_password":"<chosen password>"}'
```

Then write those same values to `$REPO/.env` (gitignored, never commit it):

```
cat > "$REPO/.env" <<EOF
WEBUI_USER=<chosen user>
WEBUI_PASSWORD=<chosen password>
EOF
```

Restart the service once so the new credentials take: `systemctl --user restart qbittorrent`.

## 5. Lock down incoming connections

```
sudo ufw deny 6881
sudo ufw status
```

UPnP is already off via the profile config. This means downloads work fine but the client
never accepts inbound peer connections — matches the reference setup.

## 6. Publish the Web UI to the tailnet only

```
sudo tailscale serve --bg 8080
tailscale serve status
```

This exposes `https://<this machine's tailnet name>:<port>` to tailnet devices only —
never to the public internet. Report the resulting URL back to the user along with the
username set in step 4 (never repeat the password back in plaintext in chat — tell them to
read it from `.env` on the machine, or however they'd like to receive it).

## Verify

- `systemctl --user status qbittorrent` — active.
- From the URL in step 6, on a device that's on the tailnet, the Web UI login page loads
  and the new username/password from step 4 works.
- `curl -s http://127.0.0.1:8080` from a non-tailnet vantage point should not resolve/connect
  (it's tailnet-only by construction — Tailscale Serve doesn't listen on the public interface).

## Notes for future changes

- If the Web UI password is ever changed in Settings, update `.env` to match — it's the
  local record, not the source of truth (qBittorrent's own config holds the real hash).
- `profile/` and the real `systemd/qbittorrent.service` are gitignored on purpose: they
  contain machine-specific paths and, in `profile/`, the password hash. Don't remove them
  from `.gitignore`.
