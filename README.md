# quadlet-opengym

Quadlet setup for [openGym](https://opengym.duarte-santos.ch) — a self-hosted gym
and body-weight tracker with passkey login
(`registry.gitlab.com/duartesantos8/opengym/{api,web}:1.3`, AGPL-3.0).

This project was created with the help of Claude Code and
https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `opengym.network` | Shared network for the two containers |
| `opengym-api.container` | Quadlet unit — Node API, passkeys and all user data |
| `opengym-web.container` | Quadlet unit — nginx serving the PWA, proxying `/api` |
| `opengym-media.service` | Systemd oneshot: one-time ~140 MB exercise-media download |
| `opengym.env` | Default environment variables |
| `opengym.override.env.template` | Template for local overrides (domain, admin id) |
| `opengym-backup.service` | Systemd service: tars the data directory |
| `opengym-backup.timer` | Systemd timer: triggers the backup daily |

## Shape of the deployment

Two containers on a shared `opengym.network`, plus a oneshot that downloads the
exercise images once:

- **`opengym-web`** is the only one published, on `127.0.0.1:8085`. It serves the
  frontend and proxies `/api` to `opengym-api` over the shared network. Single
  origin — passkeys require it.
- **`opengym-api`** is not published at all; nothing outside the network reaches it.
- **`opengym-media`** runs once, writes `~opengym/media/{img,gif}`, then skips
  itself on every later boot via `ConditionPathExists=`.

### Read this before choosing a hostname

openGym signs in with passkeys (WebAuthn). Browsers bind a passkey to an **exact
hostname** and only allow it over **HTTPS** (`http://localhost` is the one
exception). Two consequences:

- Reaching this instance over the LAN by IP will never show a passkey prompt.
  It has to be the real HTTPS hostname, via the reverse proxy.
- **Changing `RP_ID` later invalidates every existing passkey** and everyone has
  to register again. Settle the domain before anyone joins.

### Neither image declares a USER — and that is deliberate here

Both images run as root *inside* the container (verified against the registry:
the image config's `User` is null for both). Under rootless podman that maps to
the host `opengym` user, so bind-mounted files come out owned by `opengym`.

This is why the `.container` files carry **no `UserNS=`/`User=`/`Group=`**, unlike
the template in `quadlet-my-guidelines`. Adding `UserNS=keep-id` would map the
host uid to the same uid inside the container, and container-root would then no
longer be `opengym` — the data directory would end up owned by a subuid.

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/opengym -s /usr/sbin/nologin opengym

REPO_URL=https://github.com/mkoester/quadlet-opengym.git
REPO=~opengym/quadlet-opengym
```

```sh
# 2. Enable linger
sudo loginctl enable-linger opengym

# 3. Clone this repo into the service user's home
sudo -u opengym git clone $REPO_URL $REPO

# 4. Create quadlet, systemd-user, data and media directories
sudo -u opengym mkdir -p ~opengym/.config/containers/systemd
sudo -u opengym mkdir -p ~opengym/.config/systemd/user
sudo -u opengym mkdir -p ~opengym/data ~opengym/media/img ~opengym/media/gif

# 5. Create .override.env from template and fill in RP_ID / ORIGIN
#    (leave INVITE_ONLY=0 uncommented for now — see "First run" below)
sudo -u opengym cp $REPO/opengym.override.env.template $REPO/opengym.override.env
sudo -u opengym nano $REPO/opengym.override.env

# 6. Symlink the quadlet files
sudo -u opengym ln -s $REPO/opengym.network ~opengym/.config/containers/systemd/opengym.network
sudo -u opengym ln -s $REPO/opengym-api.container ~opengym/.config/containers/systemd/opengym-api.container
sudo -u opengym ln -s $REPO/opengym-web.container ~opengym/.config/containers/systemd/opengym-web.container
sudo -u opengym ln -s $REPO/opengym.env ~opengym/.config/containers/systemd/opengym.env
sudo -u opengym ln -s $REPO/opengym.override.env ~opengym/.config/containers/systemd/opengym.override.env

# 7. Symlink the media oneshot (a plain systemd unit, NOT a quadlet)
sudo -u opengym ln -s $REPO/opengym-media.service ~opengym/.config/systemd/user/opengym-media.service

# 8. Reload and start (web pulls in api and media via Requires=)
sudo -u opengym XDG_RUNTIME_DIR=/run/user/$(id -u opengym) systemctl --user daemon-reload
sudo -u opengym XDG_RUNTIME_DIR=/run/user/$(id -u opengym) systemctl --user start opengym-web

# 9. Verify
sudo -u opengym XDG_RUNTIME_DIR=/run/user/$(id -u opengym) systemctl --user status opengym-media opengym-api opengym-web
```

The first start downloads ~140 MB of exercise media and blocks `opengym-web`
until it finishes — `TimeoutStartSec=1800` covers that.

### Reverse proxy

Add to `/etc/caddy/Caddyfile`, alongside the other services:

```caddyfile
@opengym host gym.example.com   # openGym — gym & body-weight tracker
handle @opengym {
        reverse_proxy localhost:8085
}
```

Then create the DNS record for the hostname and `sudo systemctl reload caddy`.

`ORIGIN` in `opengym.override.env` must match what the browser shows, exactly —
scheme included, trailing slash excluded.

### First run: the invite-only chicken-and-egg

`opengym.env` ships the hardened end state (`INVITE_ONLY=1`, `ALLOW_GUEST=0`).
Invite codes are generated from the admin dashboard, the dashboard needs an
admin, and an admin is a registered user — so the first profile cannot be created
under `INVITE_ONLY=1`. Bootstrap in two steps:

```sh
# a) With INVITE_ONLY=0 in the override, visit https://gym.example.com and
#    create your profile with a passkey.

# b) Find your user id
sudo -u opengym grep -o '"id":"[^"]*"' ~opengym/data/db.json | head

# c) Put it in ADMIN_UIDS and REMOVE the INVITE_ONLY=0 line
sudo -u opengym nano $REPO/opengym.override.env

# d) Apply — a plain restart re-reads the EnvironmentFiles
sudo -u opengym XDG_RUNTIME_DIR=/run/user/$(id -u opengym) systemctl --user restart opengym-api opengym-web
```

Then confirm the **Admin dashboard** link appears in Settings, and generate invite
codes from there for anyone else.

## Configuration

`opengym.env` holds the non-sensitive defaults; `opengym.override.env` is loaded
after it and wins. Full reference: upstream's `.env.example`.

| Variable | Default here | Description |
|---|---|---|
| `RP_ID` | *(override)* | Bare hostname passkeys are bound to — no scheme, no port |
| `ORIGIN` | *(override)* | Full URL the app is served from, no trailing slash |
| `RP_NAME` | `openGym` | Name shown in the passkey prompt |
| `BACKEND` | `opengym-api` | Container name `/api` is proxied to; matches `ContainerName=` |
| `PORT` | `3000` | API port; the web container proxies to the same value |
| `NGINX_PORT` | `8080` | Port nginx listens on inside the web container |
| `INVITE_ONLY` | `1` | New profiles need an invite code |
| `ALLOW_GUEST` | `0` | Removes "Continue without account" |
| `ADMIN_UIDS` | *(override)* | Comma-separated user ids that get the admin dashboard |
| `SESSION_DAYS` | `90` | How long a sign-in lasts |
| `AUDIT_LOG` | `1` | Record sign-ins and admin actions to `data/audit.log` |
| `AUDIT_IP` | `off` | `off` / `net` / `full` — addresses are opt-in |
| `CF_CONNECTING_IP` | *(empty)* | Left empty: Caddy is in front, not Cloudflare |
| `COACH_DISABLED` | `1` | Kill switch for the AI Coach |

Two things that look like settings and are not: `DATA_DIR` is pinned to `/data` by
the image (change the host side of the volume instead), and `WEB_PORT` is a
docker-compose-only knob — here the host port is the `PublishPort=` line in
`opengym-web.container`.

### Ports

| | Host | Container |
|---|---|---|
| web | `127.0.0.1:8085` | `8080` (`NGINX_PORT`) |
| api | not published | `3000` (`PORT`) |

`NGINX_PORT=8080` rather than the image default of `80` so nothing inside needs a
privileged port. The web image renders its nginx config from these at start-up,
so this works on the prebuilt image with no rebuild.

## Updating

`AutoUpdate=registry` on **series tags** (`:1.3`), never `:latest` — a major bump
should be a deliberate act, not something that lands unattended. Both images must
move together.

```sh
sudo -u opengym XDG_RUNTIME_DIR=/run/user/$(id -u opengym) podman auto-update
```

To move to the next series, edit the `Image=` line in **both** `.container` files,
then `daemon-reload` and restart. Releases are announced on
[GitLab](https://gitlab.com/DuarteSantos8/opengym/-/releases).

`data/` and the downloaded media are untouched by an update.

## Backup

Follows the shared convention: `/var/backups/opengym/` owned
`opengym:backup-readers`, mode `2750`, pulled offsite by `backupuser`.

`data/` is the whole of the state — users, passkeys, workouts, `audit.log` and
`vapid.json` — as plain JSON. `media/` is deliberately excluded: it is
third-party content that `opengym-media.service` re-downloads.

```sh
# Create backup staging directory (setgid so the group is inherited)
sudo mkdir -p /var/backups/opengym
sudo chown opengym:backup-readers /var/backups/opengym
sudo chmod 2750 /var/backups/opengym

# Symlink backup units from the repo
sudo -u opengym ln -s $REPO/opengym-backup.service ~opengym/.config/systemd/user/opengym-backup.service
sudo -u opengym ln -s $REPO/opengym-backup.timer ~opengym/.config/systemd/user/opengym-backup.timer

# Enable and start the timer
sudo -u opengym XDG_RUNTIME_DIR=/run/user/$(id -u opengym) systemctl --user daemon-reload
sudo -u opengym XDG_RUNTIME_DIR=/run/user/$(id -u opengym) systemctl --user enable --now opengym-backup.timer
```

## Image pruning

```sh
sudo -u opengym XDG_RUNTIME_DIR=/run/user/$(id -u opengym) systemctl --user enable --now podman-image-prune@30.timer
```

## Acceptance checks

Each of these fails visibly if the thing it checks is broken:

```sh
# API answers, and reports itself healthy
curl -fsS http://127.0.0.1:8085/api/health          # => {"ok":true,...}

# The media actually landed. Measured 2026-09-07: 1324 in each, and the two
# counts MATCHING matters — the dataset is one gif per jpg, so a mismatch means
# one of the two cp's was partial. Check this now: ExecStartPost has already
# created .download-complete, so ConditionPathExists=! will skip the unit on
# every future boot and a partial copy would latch silently.
sudo -u opengym ls ~opengym/media/img | wc -l
sudo -u opengym ls ~opengym/media/gif | wc -l

# Both containers are healthy, not merely running
sudo -u opengym XDG_RUNTIME_DIR=/run/user/$(id -u opengym) podman ps --format '{{.Names}} {{.Status}}'

# Data is owned by the service user, not a subuid — the point of the no-UserNS choice
sudo ls -ln ~opengym/data

# Through Caddy, from outside
curl -fsS https://gym.example.com/api/health
```

## Notes

- **⚠ The web image's nginx proxies `/api` through DOCKER's DNS, so `BACKEND` here
  is an IP, not a container name** (found on the first deploy, 2026-09-07).
  `web/nginx.conf.template` hardcodes `resolver 127.0.0.11 valid=10s ipv6=off;`
  and reaches the API through a *variable* `proxy_pass`, which forces per-request
  DNS. Podman's aardvark-dns listens on the network gateway, not on Docker's
  `127.0.0.11`, so every `/api/` request 502s. `BACKEND`/`PORT` are envsubst
  placeholders in that template; the resolver address is not, so no env setting
  fixes it. **Everything about the symptom points elsewhere** — both containers
  report `(healthy)` because the web healthcheck probes `/` (static, no
  resolver); `podman exec opengym-web wget -qO- http://opengym-api:3000/api/health`
  succeeds, because `/etc/resolv.conf` is correct and nginx's `resolver`
  directive ignores it (and `/etc/hosts`); and Caddy returns a 502 that reads as
  a Caddy or upstream-down problem. **Only `podman logs opengym-web` names it:**
  `resolver: 127.0.0.11:53 … Connection refused`. The fix is a pinned
  `Subnet=`/`Gateway=` in `opengym.network`, `IP=` on `opengym-api.container`,
  and `BACKEND` set to that address — an IP literal makes nginx skip DNS
  entirely. Deliberately *not* a bind-mounted copy of their patched template:
  `AutoUpdate=registry` would let a newer image drift away from the copy
  silently. Reported upstream 2026-09-09 on
  [issue #69](https://gitlab.com/DuarteSantos8/opengym/-/issues/69), which another
  Podman user had already opened; a fix is also already in flight as
  [!110](https://gitlab.com/DuarteSantos8/opengym/-/merge_requests/110)
  (`ENV RESOLVER=`, from the Kubernetes side). A second option exists that needs no
  configuration at all: the `nginx:alpine` image ships
  `/docker-entrypoint.d/15-local-resolvers.envsh`, which exports
  `NGINX_LOCAL_RESOLVERS` from the container's own `/etc/resolv.conf` when
  `NGINX_ENTRYPOINT_LOCAL_RESOLVERS` is set — verified present in the running
  `opengym-web`. **Keep the pinned IP regardless**: it makes the recreate-drift
  case behind upstream's `#16` structurally impossible rather than merely fixed.
- **`opengym-media` needs `--entrypoint sh` — an image's ENTRYPOINT silently eats
  your command** (found on the first deploy, 2026-09-07). `docker.io/alpine/git`
  declares `ENTRYPOINT ["git"]`, so `podman run … alpine/git sh -c '…'` runs
  `git sh -c '…'`: git takes `sh` as a subcommand, prints *"The most similar
  commands are"* with a tab-indented list, and exits 1. In the journal that
  surfaces as a few bare words (`show`, `push`) with no error line near them, so
  it reads as a broken download script rather than a wrong entrypoint. The unit's
  `ConditionPathExists=!` guard means it retries on every boot, so the failure
  repeats rather than latching. **Check `Entrypoint` before writing any
  `podman run <image> sh -c` line** — the registry answers it without a pull:
  fetch an anonymous token, `GET /v2/<repo>/manifests/<tag>` with the OCI index
  Accept headers, then `curl -sL` the config blob (the `-L` matters; the blob is
  a redirect).
- **Notifications** (rest-timer and workout-day push) need no server-side setup:
  VAPID keys are generated on first run into `data/vapid.json`. They require a
  signed-in profile and HTTPS.
- **The AI Coach** is off (`COACH_DISABLED=1`). API-key providers would work in
  this default image; the Claude Agent SDK / Codex providers need upstream's
  `coach` build target, which means building from source rather than pulling.
- **Exercise media licensing**: the images and GIFs are © Gym visual, used under
  the [exercises-dataset](https://github.com/hasaneyldrm/exercises-dataset) terms —
  not openGym's AGPL, and not redistributed by this repo. They are fetched from
  upstream at deploy time. Reusing them yourself needs your own licence.
