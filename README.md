# Userpilot IP-masking proxy

Reverse proxy so end-user IP addresses never reach Userpilot's servers directly.

Without this proxy, the SDK connects straight to:
- `js.userpilot.io` — script CDN
- `analytex.userpilot.io` — realtime events websocket (`/v1/events/websocket`)

## What it does

`nginx.conf` proxies two paths:

- `/sdk/` → `https://js.userpilot.io/sdk/`
- `/userpilot/websocket` → `https://analytex.userpilot.io/v1/events/websocket` (with websocket upgrade handling)

Both locations:
- Strip `X-Forwarded-For` / `X-Real-IP` / `X-Client-IP` before forwarding, so the end user's real IP is never leaked
- Use `set $var` for the upstream hostname (not hardcoded in `proxy_pass`), with `resolver 127.0.0.11 valid=30s` (Docker's DNS) — community nginx only resolves a hardcoded `proxy_pass` hostname once at startup, and Userpilot's backend IPs rotate

## Quick start

```bash
docker compose up -d
```

Open `http://localhost:8080/test.html`, then check DevTools → Network:
- SDK script should load via `/sdk/latest.js`
- Neither request should hit a `userpilot.io` domain directly

The websocket needs TLS to work — see below.

## Local test with TLS

The SDK always opens the events websocket as `wss://`, regardless of the page's own protocol. To test it locally:

```bash
brew install mkcert
mkcert -install                 # adds a local CA to your system/browser trust store
mkdir -p certs
mkcert -cert-file certs/localhost.pem -key-file certs/localhost-key.pem localhost 127.0.0.1 ::1
```

- `docker-compose.yml` mounts `./certs` and exposes port `8443`
- `nginx.conf` listens on `443 ssl` using those certs
- `test.html`'s `endpoint` points at `localhost:8443/userpilot`

Works whether the page itself is loaded via `http://localhost:8080/test.html` or `https://localhost:8443/test.html` — a page can always open a *secure* `wss://` connection elsewhere, only the reverse is blocked. Confirm in DevTools → Network: SDK script loads, and the websocket opens with `101 Switching Protocols`.

To undo: `mkcert -uninstall`. Keep `certs/` out of version control (already gitignored) — it holds a private key.

## Debugging

Watch live traffic:

```bash
docker compose logs -f proxy
```

Every request logs the exact upstream URL nginx forwarded to, the response status, and IP-stripping status, e.g.:

```
"GET /sdk/latest.js HTTP/1.1" -> "https://js.userpilot.io/sdk/latest.js" status=200 upstream_status=200
client_sent[XFF="-" X-Real-IP="-" X-Client-IP="-"] ip_stripping=stripped (...)
```

To see the stripping happen with a spoofed IP:

```bash
curl -H "X-Forwarded-For: 1.2.3.4" http://localhost:8080/sdk/latest.js
```

Then check the log — the spoofed IP shows up in `client_sent[...]`, but `ip_stripping=stripped` confirms it never reached the upstream request.

> Note: `curl`'s own output is just the response body (the SDK JS file) — the log line only appears in `docker compose logs`, not in `curl`'s terminal.

## Troubleshooting: duplicated path segments

**Symptom:** a proxied resource's path repeats itself, e.g.:

```
/services/some-integration/proxy/services/some-integration/proxy/uploads/<hash>.png
```

**Cause:** this happens when the local proxy prefix doesn't match the real upstream path (unlike this repo's `/sdk/`, which happens to mirror `js.userpilot.io`'s own `/sdk/` path). With a variable upstream host, `proxy_pass` does **not** auto-strip the matched `location` prefix — so `$request_uri` (prefix included) gets forwarded as-is. If the upstream then returns a link built from that prefixed path, and something requests it through the same proxy again, the prefix doubles up.

**Fix:** strip the prefix explicitly with a regex capture instead of forwarding `$request_uri` verbatim:

```nginx
location ~ ^/services/some-integration/proxy/(.*)$ {
    set $upstream real-upstream-host.com;
    proxy_pass https://$upstream/$1$is_args$args;
}
```

Confirmed locally: the unstripped pattern 404s against an upstream whose real path doesn't include the local prefix; the regex-strip pattern above resolves correctly.

## Still open before production

- Needs a real cert (e.g. Let's Encrypt) for the public domain — locally this uses an mkcert cert trusted only on this machine
- Can't run on `localhost` in production — needs a real server with a public domain (e.g. `proxy.yourcompany.com`) reachable by all end users
- The `127.0.0.11` resolver only works inside Docker — swap for a real DNS resolver (e.g. `8.8.8.8`) if deploying outside Docker
