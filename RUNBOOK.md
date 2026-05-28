# kura runbook

Operational guide for the self-hosted services in this repo, deployed via Dokploy
on the shared Hetzner box. Read this before touching a deploy or debugging an outage.

Host: Hetzner VPS, public IP `204.168.128.178`. Shared by ekam, navi, Home Assistant,
and Dokploy itself. A mistake here can take all of them down (it has, once - see
[Incident log](#incident-log)).

---

## How deploys work (Dokploy internals worth knowing)

Dokploy is **not** a simple `docker compose up`. It is a Swarm stack:

- The control plane runs as Swarm **services**: `dokploy`, `dokploy-postgres`,
  `dokploy-redis`, `dokploy-traefik`. They talk over the `dokploy-network` overlay.
- Dokploy stores all app config (including env vars) in **`dokploy-postgres`**.
- It runs the **deploy queue on `dokploy-redis`**. Clicking "Deploy" enqueues a job;
  a worker pulls it, clones the repo into `/etc/dokploy/compose/<app>/code`, writes
  a `.env` from the saved env vars, then runs `docker compose --env-file .env up`.

Two consequences:
1. If Dokploy can't reach `dokploy-redis`, **clicking Deploy silently does nothing** -
   no job runs, no `.env` is written. Looks like an env or compose bug; it is not.
2. The `.env` is written **at deploy time**, not on Save. Saving env only persists it
   to Postgres. No deploy = no `.env` file on disk.

App working dirs: `/etc/dokploy/compose/<app-id>/code/` (repo clone + the generated
`.env` live here).

---

## Standard deploy (Compose app)

1. New Dokploy **Compose** app, Provider = this Git repo, Branch = `main`,
   Compose Path = `<service>/docker-compose.yml` (e.g. `livekit/docker-compose.yml`).
2. **Environment** tab: paste the vars from the service's `.env.example`,
   **with real values filled in** (empty `KEY=` counts as unset and fails `${VAR:?}`).
3. **Save**, then **Deploy**.
4. Verify on the box: `ls -la /etc/dokploy/compose/<app>/code/.env` should exist with
   real values. If it doesn't, the deploy didn't run - see troubleshooting.

Dokploy auto-appends a `networks: dokploy-network: external: true` block to every
compose. If your repo's compose already declares it (required when a service attaches
to `dokploy-network` for Traefik routing), you may end up with a duplicate top-level
key. If a deploy errors on that, remove the duplicate in Dokploy's inline compose
editor.

---

## LiveKit service specifics

Config verified against LiveKit source (`cmd/server/main.go`, `config-sample.yaml`):

- Config env var is **`LIVEKIT_CONFIG`** (the YAML body). `LIVEKIT_CONFIG_BODY` does
  **not** exist - using it means the server boots with empty config and dies with
  `one of key-file or keys must be provided`.
- Architecture mirrors LiveKit's official VM deploy (Caddy + livekit on a bridge
  network): runs on **`dokploy-network`** (not host networking) so Dokploy's Traefik
  can terminate TLS for the wss domain and proxy `wss://` to port 7880 over the
  overlay. Traefik plays the role Caddy plays in LiveKit's reference compose.
- Media (`7881/tcp` + `7882-7892/udp`) is **published via `ports:`** so it reaches
  the host directly. The proxy cannot carry media.
- Media uses a **UDP mux range `udp_port: 7882-7892`**, not the default 50000-60000.
  LiveKit recommends a range >= the machine's vCPU count. **Never widen this** -
  publishing 10k ports spawns 10k `docker-proxy` processes and pins the host (see
  Incident log).
- Redis is **not required** for a single node (only enables distributed mode).

### Ports to open

| Port        | Proto | Purpose                        |
| ----------- | ----- | ------------------------------ |
| 7880        | TCP   | HTTP/WebSocket signaling + API |
| 7881        | TCP   | RTC over TCP (UDP fallback)    |
| 7882-7892   | UDP   | RTC media (UDP mux)            |

**TWO firewall layers - both must allow, or you get a silent timeout:**

1. **Hetzner Cloud Firewall** (web UI): add inbound rules, source Any IPv4 + Any IPv6.
   UDP range port field = `7882-7892`. Confirm the firewall is **attached to the
   server** under its Resources tab, not just created.
2. **Host `ufw`** (it is active on this box): mirror the rules. ufw uses `:` for ranges.
   ```sh
   sudo ufw allow 7880/tcp comment 'LiveKit signaling'
   sudo ufw allow 7881/tcp comment 'LiveKit TCP fallback'
   sudo ufw allow 7882:7892/udp comment 'LiveKit media'
   sudo ufw status
   ```

Diagnosing a timeout (7880 is NOT published on the host - it lives behind Traefik;
7881 and the UDP range ARE published):
```sh
docker ps --filter name=livekit                          # container Up?
curl -s http://localhost:7880/                           # FAILS now (7880 not on host)
docker exec <livekit-container> wget -qO- localhost:7880 # OK from inside the container
curl -s https://lk.vashishtamithra.com/                  # OK = Traefik -> livekit works
sudo ufw status                                          # 7881/tcp + 7882:7892/udp listed?
sudo ss -lnp | grep -E '7881|788[2-9]'                   # docker-proxy bound for media
```
Note: SSH on this box is port **2222**, not 22 (ufw rule reflects that).

### TLS / wss (the wss domain)

Dokploy's Traefik terminates TLS for the LiveKit domain and proxies
`wss://lk.vashishtamithra.com` -> `livekit:7880` over `dokploy-network`. To set up:

1. **DNS:** A record `lk.vashishtamithra.com` -> `204.168.128.178`, **DNS-only
   (grey cloud)** in Cloudflare so Traefik's Let's Encrypt HTTP challenge succeeds.
2. **Dokploy domain:** in the livekit app, Domains tab, add: Host
   `lk.vashishtamithra.com`, Path `/`, Container Port `7880`, HTTPS on (auto-SSL).
3. Verify: `curl -sv https://lk.vashishtamithra.com/` returns `200 OK` with body `OK`
   and a valid Let's Encrypt cert.

### Key/secret parity

`LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` must be **identical** in two places:
LiveKit's env (in Dokploy) and ekam's env (used by `apps/web` + `apps/switchboard`).
ekam also needs `LIVEKIT_URL=wss://lk.vashishtamithra.com`. Generate a pair with:
```sh
KEY="API$(openssl rand -hex 6)"
SECRET="$(openssl rand -base64 32 | tr -d '/+=')"
```

### Production tuning

LiveKit warns `UDP receive buffer too small`. Bump it once:
```sh
echo 'net.core.rmem_max=5000000' | sudo tee /etc/sysctl.d/99-livekit.conf
echo 'net.core.wmem_max=5000000' | sudo tee -a /etc/sysctl.d/99-livekit.conf
sudo sysctl --system
```
Then redeploy/restart the container.

---

## Troubleshooting

### "Error deploying compose" / clicking Deploy does nothing

Almost always Dokploy can't reach its Redis queue (common after a host reboot - the
Swarm overlay DNS goes stale). Check:
```sh
docker service logs --since 30s dokploy
```
If you see `getaddrinfo ENOTFOUND dokploy-redis` (repeating), the queue is dead.
Fix by re-registering the backing services in Swarm DNS, then bouncing Dokploy:
```sh
docker service update --force dokploy-redis
docker service update --force dokploy-postgres
docker service update --force dokploy
docker service logs --since 20s -f dokploy   # ENOTFOUND spam should stop
```
Note: `docker restart dokploy` does NOT work - these are Swarm services, use
`docker service update --force <name>`.

### `required variable LIVEKIT_API_KEY is missing a value`

The env var is empty or unset at deploy time. Either env wasn't saved, the deploy ran
before saving, or the value is blank. Check what Dokploy actually has:
```sh
docker exec $(docker ps -q -f name=dokploy-postgres) \
  psql -U dokploy -d dokploy -c "SELECT name, left(env, 200) FROM compose;"
```
And whether a `.env` exists with real values:
```sh
sudo cat /etc/dokploy/compose/<app>/code/.env
```

### `one of key-file or keys must be provided` (LiveKit)

LiveKit booted with no config. The env var name is wrong (must be `LIVEKIT_CONFIG`)
or the config body interpolated to empty keys.

### The host hangs / SSH dies / everything unreachable

**Cause: publishing a wide port range via Docker `ports:`** (e.g. a UDP media range).
Docker spawns one `docker-proxy` process per published port; 10k ports = 10k processes
= pinned CPU = dead host. This took the whole box down once.

**Prevention:** keep published ranges small (the LiveKit `udp_port: 7882-7892` mux is
11 ports - safe). Never publish a default-sized media range (50000-60000). Alternative
escapes if you ever need a wide range: `network_mode: host` (no docker-proxy at all,
but incompatible with Dokploy's domain/Traefik routing), or set Docker daemon
`"userland-proxy": false` in `/etc/docker/daemon.json` (iptables-only, no per-port
process).

**Recovery:**
1. If SSH is alive: `docker rm -f $(docker ps -aq --filter name=<service>)` - removing
   the container tears down the proxies. Do NOT `systemctl stop docker` (kills Dokploy).
2. If SSH is dead: hard reboot from the Hetzner console (Power -> Reset). Safe. Then
   immediately stop the offending app in Dokploy before it relaunches.
3. After any reboot, expect to fix the Dokploy/Redis DNS (see first item above).

Cost note: Hetzner bills flat monthly. A CPU storm or reboots cost $0 extra.

---

## Manual unblock (bypass Dokploy entirely)

To prove a stack works independent of Dokploy, or to run it while Dokploy is broken.
The compose dir is root-owned, so use `sudo tee` (not `>` redirects, which run as you):
```sh
cd /etc/dokploy/compose/<app>/code
echo "LIVEKIT_API_KEY=$KEY"       | sudo tee .env
echo "LIVEKIT_API_SECRET=$SECRET" | sudo tee -a .env
echo "LIVEKIT_WEBHOOK_URL=https://ekam.v19.tech/api/voice/livekit/webhook" | sudo tee -a .env
docker compose -f <service>/docker-compose.yml --env-file .env up -d
docker logs $(docker ps -q --filter name=<service>) --tail 30
```
Bring it down before letting Dokploy take over (avoids host-port conflict):
```sh
docker compose -f <service>/docker-compose.yml --env-file .env down
```
Caveat: a Dokploy git redeploy overwrites a hand-written `.env`. Manual `.env` is a
test/stopgap, not a fix.

Avoid heredocs over SSH - pasted indentation breaks the `EOF` terminator. Use the
per-line `tee` form above.

---

## Incident log

### 2026-05-28 - host outage from published UDP range
First livekit compose published `50000-60000/udp` via `ports:`. Docker spawned ~10k
`docker-proxy` processes, pinned CPU, killed SSH and took down ekam/navi/HA. Recovered
by hard reboot + `docker rm -f` of the livekit container. Initial fix: switched to
`network_mode: host` + `udp_port: 7882-7892` mux. After the reboots, Dokploy's deploy
queue couldn't resolve `dokploy-redis` (stale Swarm overlay DNS), making every deploy
silently no-op and masquerade as an env bug; fixed by force-updating the Swarm services.
LiveKit `v1.12.0` confirmed running once env had real values.

### 2026-05-28 - architecture pivot to bridge + Traefik (final)
Host networking made Dokploy's Traefik unable to route the `lk.vashishtamithra.com`
domain (Traefik connects to the container over `dokploy-network`; host-net containers
aren't on it). Refactored to match LiveKit's official VM deployment (Caddy + livekit
on a bridge network in their reference; Traefik in our case): `network_mode: host`
dropped, service attached to `dokploy-network`, `expose: 7880` for Traefik, 7881/tcp
and 7882-7892/udp published. Verified end-to-end with `curl -sv
https://lk.vashishtamithra.com/` returning `200 OK` over a Let's Encrypt cert.
