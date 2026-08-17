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
DevTools Network: the SDK script should load via `/sdk/latest.js`. The websocket, however,
will fail here — see "Local test with TLS" below.

## Local test with TLS

The SDK opens the events websocket as `wss://` against whatever host:port `endpoint` points
to, regardless of the page's own protocol — plain `http://localhost:8080` can't serve that
handshake itself (the TLS `ClientHello` hits a plain-HTTP listener and gets rejected), so
`endpoint` needs to point at a TLS-terminated port. To test locally, generate a trusted local
certificate with [mkcert](https://github.com/FiloSottile/mkcert):

```
brew install mkcert
mkcert -install                 # adds a local CA to your system/browser trust store
mkdir -p certs
mkcert -cert-file certs/localhost.pem -key-file certs/localhost-key.pem localhost 127.0.0.1 ::1
```

`docker-compose.yml` mounts `./certs` into the container and exposes `8443:443`;
`nginx.conf`'s `server` block listens on both `80` and `443 ssl` using those certs.
`test.html`'s `endpoint` is set to `localhost:8443/userpilot` to match, so it works whether
the page is opened via `http://localhost:8080/test.html` or `https://localhost:8443/test.html`
— confirmed both ways: DevTools Network shows the SDK script loading via `/sdk/latest.js` and
the websocket opening (`wss://`, `101 Switching Protocols`) to `/userpilot/websocket`, neither
hitting a `userpilot.io` domain directly. (Browsers don't block an insecure page from opening
a *secure* `wss://` connection elsewhere — only the reverse — so only `endpoint`'s target needs
TLS, not the page itself.)

`mkcert -install` can be undone later with `mkcert -uninstall`. `certs/` holds a private key —
keep it out of version control (see `.gitignore`).

## Debugging

`nginx.conf` logs a custom `proxied` format (`docker compose logs -f proxy`) that shows,
for every request, the exact upstream URL nginx forwarded to plus the response status —
e.g. `"GET /sdk/latest.js HTTP/1.1" -> "https://js.userpilot.io/sdk/latest.js" status=200
upstream_status=200`. This is the fastest way to confirm a path is being built correctly,
since it's the literal string nginx used, not the location's `proxy_pass` line.

The same log line also shows IP-stripping happening in real time: `client_sent[XFF="..."
X-Real-IP="..." X-Client-IP="..."]` prints whatever IP-identifying headers the client actually
sent, and `ip_stripping=...` confirms whether/how this location zeroes them out before
forwarding (`n/a (not proxied)` for non-proxied locations like `/`). Trigger it with e.g.
`curl -H "X-Forwarded-For: 1.2.3.4" http://localhost:8080/sdk/latest.js` and watch
`docker compose logs -f proxy` — the spoofed IP shows up in `client_sent[...]` while
`ip_stripping=stripped (...)` confirms it never reached the upstream request.

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

1. Locally this uses an mkcert-issued cert, trusted only on this machine (see "Local test with
   TLS"). Production needs a real cert (e.g. Let's Encrypt) for the public domain below.
2. This can't live on `localhost` in production — it needs to run on a real server with a
   public domain (e.g. `proxy.yourcompany.com`) reachable by all end users, not just one
   machine.
3. The `127.0.0.11` resolver only works inside Docker. Swap for a real DNS resolver (e.g.
   `8.8.8.8`) if deploying outside Docker.
