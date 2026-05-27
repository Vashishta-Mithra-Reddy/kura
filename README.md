# kura

Self-hosted infrastructure, one folder per service. Each folder is a self-contained
Docker Compose stack deployed independently on Dokploy.

> kura (蔵) - a traditional Japanese storehouse. This repo is the storehouse for the
> services that sit beneath the apps.

**Deploying or debugging an outage? Read [RUNBOOK.md](RUNBOOK.md) first** - Dokploy
internals, the deploy flow, firewall/TLS, and the recovery procedures for the failures
we have actually hit.

## Layout

```
kura/
  <service>/
    docker-compose.yml   # the stack
    .env.example         # required env, copy to .env (gitignored)
    README.md            # what it is, deploy notes, gotchas
```

Each `<service>` directory maps to exactly one Dokploy "Compose" application.

## Services

| Service    | Purpose                          | Notes                          |
| ---------- | -------------------------------- | ------------------------------ |
| `livekit`  | WebRTC media server (voice/RTC)  | Needs UDP port range, see README |

## Adding a service

1. Create `kura/<service>/` with the three files above.
2. Document required env in `.env.example` and the gotchas in its `README.md`.
3. Create a new Compose app in Dokploy pointing at the folder.

Resist building a "platform." Keep each stack self-contained until a real shared
need (common network, shared secret store) actually shows up.
