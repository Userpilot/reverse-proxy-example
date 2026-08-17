# Userpilot IP-masking proxy

Reverse proxy so end-user IP addresses never reach Userpilot's servers directly. The
Userpilot SDK normally connects straight to `js.userpilot.io` (script CDN) and
`analytex.userpilot.io` (realtime events websocket, at `/v1/events/websocket`) — this
account's real backend endpoint, confirmed via Userpilot support, is `analytex.userpilot.io`.

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

## Debugging

`nginx.conf` logs a custom `proxied` format (`docker compose logs -f proxy`) that shows,
for every request, the exact upstream URL nginx forwarded to plus the response status —
e.g. `"GET /sdk/latest.js HTTP/1.1" -> "https://js.userpilot.io/sdk/latest.js" status=200
upstream_status=200`. This is the fastest way to confirm a path is being built correctly,
since it's the literal string nginx used, not the location's `proxy_pass` line.

## Troubleshooting: duplicated path segments

If a proxy mounted under a prefix that doesn't match the real upstream path (e.g.
`/services/some-integration/proxy/`, unlike this repo's `/sdk/` which happens to mirror
`js.userpilot.io`'s real `/sdk/` path) shows a resource's path repeated, e.g.:

```
/services/userpilot-integration/proxy/services/userpilot-integration/proxy/uploads/<hash>.png
```

the cause is `proxy_pass` with a variable host (required here for DNS re-resolution, see
above) not stripping the matched `location` prefix — nginx only does that automatic
stripping when the upstream host is hardcoded. With a variable host, `$request_uri` carries
the *entire* original path, prefix included, straight to the upstream. If the upstream (or
the SDK) then builds a link from that already-prefixed path and something requests it
through the same proxy again, the prefix gets added a second time.

The fix is to strip the location's own prefix explicitly with a regex capture instead of
forwarding `$request_uri` verbatim:

```nginx
location ~ ^/services/some-integration/proxy/(.*)$ {
    set $upstream real-upstream-host.com;
    proxy_pass https://$upstream/$1$is_args$args;
}
```

Confirmed locally: proxying a request through the "$request_uri, unstripped" pattern to an
upstream whose real path doesn't include the local prefix 404s; switching to the regex-strip
pattern above resolves correctly.

## Still open before production

1. This is HTTP/`localhost` only. The real SDK likely requires HTTPS/WSS — put a TLS cert in
   front of nginx (mkcert for local testing, a real cert for production).
2. This can't live on `localhost` in production — it needs to run on a real server with a
   public domain (e.g. `proxy.yourcompany.com`) reachable by all end users, not just one
   machine.
3. The `127.0.0.11` resolver only works inside Docker. Swap for a real DNS resolver (e.g.
   `8.8.8.8`) if deploying outside Docker.
