# livekit

Self-hosted [LiveKit](https://livekit.io) WebRTC media server for ekam voice.

## Deploy (Dokploy)

1. New Compose app pointing at this folder.
2. Set env from `.env.example` (Dokploy env UI, not a committed file).
3. Deploy.

There is no `livekit.yaml`. The full server config lives inline in
`docker-compose.yml` as `LIVEKIT_CONFIG_BODY`, with `${...}` placeholders filled
from the env above. One file to edit, no mounted-config substitution gotchas.

## The gotchas (read before deploying)

**WebRTC is not HTTP.** Traefik / the Dokploy reverse proxy only handles HTTP(S).
LiveKit media flows over UDP and a TCP fallback that the proxy does not touch. This
stack uses `network_mode: host` so the server binds the VM's real ports directly.

Open these on the host firewall / cloud security group:

| Port          | Proto | Purpose                          |
| ------------- | ----- | -------------------------------- |
| 7880          | TCP   | HTTP/WebSocket signaling + API   |
| 7881          | TCP   | RTC over TCP (UDP fallback)      |
| 50000-50100   | UDP   | RTC media                        |

`use_external_ip: true` is set so LiveKit advertises the VM's public IP in ICE
candidates. On a NATed host this is required or clients never connect.

**Signaling TLS.** Clients connect to `wss://`. Terminate TLS for port 7880 either
via a thin Traefik HTTP route (signaling only) or LiveKit's built-in TLS. Media on
50000-50100/UDP is encrypted by WebRTC (DTLS-SRTP) regardless.

**Secret parity.** `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` here MUST equal the
values ekam uses. Mismatch = every token rejected. Single source of truth, document
where it lives.

**Webhook reachability.** LiveKit pushes events to `LIVEKIT_WEBHOOK_URL`, so this
server must be able to reach ekam's public URL outbound.

## Generate keys

```sh
docker run --rm livekit/livekit-server generate-keys
```
