# livekit

Self-hosted [LiveKit](https://livekit.io) WebRTC media server for ekam voice.

## Deploy (Dokploy)

1. New Compose app pointing at this folder.
2. Set env from `.env.example` (Dokploy env UI, not a committed file).
3. Deploy.

There is no `livekit.yaml`. The full server config lives inline in
`docker-compose.yml` under the `LIVEKIT_CONFIG` env var (the name the image reads -
NOT `LIVEKIT_CONFIG_BODY`), with `${...}` placeholders filled from the env above.
Mirrors the proven ekam compose. One file to edit.

## The gotchas (read before deploying)

**Host networking, not port publishing.** This stack uses `network_mode: host`.
Do NOT switch this service to `ports:`. Publishing a UDP range (e.g. 50000-60000)
makes Docker spawn a `docker-proxy` process per port (10k+), which pins CPU and
hangs the entire host - this caused a real outage. Host net binds ports directly
with zero proxy. The single muxed UDP port (7882) keeps the firewall trivial too.

The server binds the host's real ports. Open just these three on the firewall /
security group:

| Port        | Proto | Purpose                                |
| ----------- | ----- | -------------------------------------- |
| 7880        | TCP   | HTTP/WebSocket signaling + API         |
| 7881        | TCP   | RTC over TCP (UDP fallback)            |
| 7882-7892   | UDP   | RTC media (UDP mux, all sessions)      |

Media uses a UDP mux range (`rtc.udp_port: 7882-7892`), not the default
50000-60000. LiveKit recommends a range >= the machine's vCPU count
(config-sample.yaml); 11 ports suits this box. Tiny firewall, nothing wide to
ever accidentally publish.

`use_external_ip: true` is set so LiveKit advertises the VM's public IP in ICE
candidates. On a NATed host this is required or clients never connect.

**Signaling TLS.** With host networking, Dokploy's Traefik does NOT auto-route
7880 - the server binds the host directly. Clients connect to `wss://`, so put TLS
in front of 7880 yourself: a Cloudflare proxied DNS record, or a standalone
reverse proxy / LiveKit's built-in TLS. Media on 50000-60000/UDP is encrypted by
WebRTC (DTLS-SRTP) regardless.

**Secret parity.** `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` here MUST equal the
values ekam uses. Mismatch = every token rejected. Single source of truth, document
where it lives.

**Webhook reachability.** LiveKit pushes events to `LIVEKIT_WEBHOOK_URL`, so this
server must be able to reach ekam's public URL outbound.

## Generate keys

```sh
docker run --rm livekit/livekit-server generate-keys
```
