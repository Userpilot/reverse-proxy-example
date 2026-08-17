# Userpilot IP-masking proxy

Reverse proxy so end-user IP addresses never reach Userpilot's servers directly. The
Userpilot SDK normally connects straight to `js.userpilot.io` (script CDN) and
`analytex.userpilot.io` (realtime events websocket, at `/v1/events/websocket`) — this
account's real backend endpoint `analytex.userpilot.io`.

## What this does

`nginx.conf` proxies two paths:

- `/sdk/` &rarr; `https://js.userpilot.io/sdk/`
- `/userpilot/websocket` &rarr; `https://analytex.userpilot.io/v1/events/websocket` (with
  websocket upgrade handling)

Both locations strip `X-Forwarded-For` / `X-Real-IP` / `X-Client-IP` before forwarding, so
the end user's real IP isn't leaked to Userpilot in a proxy header.

Hostnames are stored in `set $var` variables rather than hardcoded in `proxy_pass`, and a
`resolver 127.0.0.11 valid=30s` (Docker's embedded DNS) is configured — community nginx only
resolves a hostname hardcoded in `proxy_pass` once at startup, and Userpilot's backend IPs
rotate.

## Local test

```
docker compose up -d
```

Open `http://localhost:8080/test.html`, fill in a real token in the script tag, and check
DevTools Network: the SDK script should load via `/sdk/latest.js`, the websocket should open
to `/userpilot/websocket`, and neither should hit a `userpilot.io` domain directly.

## Still open before production

1. This is HTTP/`localhost` only. The real SDK likely requires HTTPS/WSS — put a TLS cert in
   front of nginx (mkcert for local testing, a real cert for production).
2. This can't live on `localhost` in production — it needs to run on a real server with a
   public domain (e.g. `proxy.yourcompany.com`) reachable by all end users, not just one
   machine.
3. The `127.0.0.11` resolver only works inside Docker. Swap for a real DNS resolver (e.g.
   `8.8.8.8`) if deploying outside Docker.
