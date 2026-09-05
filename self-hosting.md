# Self-Hosting Iroh Infrastructure

How to self-host iroh relay servers and the address lookup service that goes
with them, for fully independent operation. This applies to [tunnel-rs],
[ezvpn], and [flextunnel] alike — they share the relay design described in
[relays-and-address-lookup.md](relays-and-address-lookup.md).

A custom-relay deployment contacts **no public iroh infrastructure**. It needs
two things of yours: **at least two relays**, and **one address lookup service**
(`iroh-dns-server`) that servers publish their current relay to and every
endpoint resolves from. The lookup service is mandatory with custom relays;
its outage is survivable (dials are carried by relay hints), so a single
instance on its own hostname is the intended shape — see
[relay-failover-findings.md](relay-failover-findings.md).

> **Reference program: [tunnel-rs].** It is the only one of the three with a
> first-class relay-only mode (`--relay-only`) and a fully offline two-relay e2e
> suite, which makes it the fastest way to prove a freshly deployed relay
> actually works before pointing a VPN or proxy at it. The verification commands
> below use it for that reason; the resulting relay serves all three programs.

## Before you start: the startup contract

Every configured custom relay is probed **individually** at startup and **all**
of them must come online, or the process refuses to start. A dead backup relay
is a startup failure, not a silent degradation. Plan your relay list
accordingly, and configure **both sides with the full list** — see the warning
in [relays-and-address-lookup.md](relays-and-address-lookup.md#custom-relays).

## Quick start: local relay

```bash
cargo install iroh-relay --version 1.0.2
iroh-relay --dev  # local testing on http://localhost:3340
```

> **Ports (verified against iroh-relay 1.0.2):** `--dev` runs the relay over
> plain HTTP on port **3340** (`http_bind_addr`) and starts a Prometheus
> **metrics** server on **9090** (`metrics_bind_addr`); it does **not** start a
> QUIC endpoint, because QUIC address discovery requires TLS, which `--dev`
> ignores. If port 9090 is already in use (e.g. by Cockpit), turn the metrics
> server off with `enable_metrics = false` in a config file, or move it with
> `metrics_bind_addr = "127.0.0.1:9099"` — `--dev` still honors non-TLS config
> fields. For production with TLS the relay serves HTTP on **80**, HTTPS on
> **443**, and QUIC address discovery on **7842** (`quic_bind_addr`) when
> `enable_quic_addr_discovery = true`.

> [!NOTE]
> **Endpoint-ID allowlisting is usually the wrong tool here.** iroh-relay 1.0.2
> *does* support `access.allowlist` and `access.denylist` (lists of
> `EndpointId`s), alongside `access.http` for an external authorization endpoint.
> But all three programs give clients **ephemeral** endpoint identities that
> change on every run, so a static allowlist cannot enumerate them and would
> reject legitimate clients — only long-lived identities (a server's
> `secret_file`) are stable enough to list.
>
> Primary access control therefore belongs to each program's own application
> authentication — Ed25519 public-key auth in tunnel-rs and flextunnel, the VPN
> handshake in ezvpn — with the [shared bearer token](#relay-access-token) below
> gating the relay itself. `access.allowlist` / `access.denylist` remain useful as
> defense in depth where the identities *are* stable (pinning a known set of
> servers, or blocking a specific abusive endpoint).

## Production relay with TLS

```bash
cargo install iroh-relay --version 1.0.2
iroh-relay --config-path relay.toml   # -c is the short form; there is no --config
```

Example `relay.toml`:

```toml
# Enable QUIC address discovery
enable_quic_addr_discovery = true

# TLS configuration (required for production)
[tls]
cert_mode = "Manual"
manual_cert_path = "/etc/letsencrypt/live/relay.example.com/fullchain.pem"
manual_key_path = "/etc/letsencrypt/live/relay.example.com/privkey.pem"

# Alternative: use Let's Encrypt automatic certificates
# [tls]
# cert_mode = "LetsEncrypt"
# hostname = "relay.example.com"
```

## Simple production setup: relay behind Cloudflare Tunnel (single TCP port)

If you don't want to manage TLS certificates or open inbound ports, run the
relay over **plain HTTP on a single TCP port** and let Cloudflare Tunnel
terminate TLS at the edge and forward decrypted HTTP to it. Only outbound
connectivity is needed on the relay host — no public IP, no 443, no QUIC/UDP.

**How it works:** omitting the `[tls]` section makes iroh-relay serve *all*
services (the `/relay` WebSocket and the `healthz` routes) over plain HTTP on
`http_bind_addr`. QUIC address discovery defaults to off, so no TLS is required
and the relay starts cleanly without `--dev`. This is the non-`--dev` equivalent
of the local dev config — verified against iroh-relay 1.0.2.

```toml
# Plain-HTTP relay on 3340 (the non-dev default is port 80). Must match the
# cloudflared ingress service and the relay URL the clients use.
http_bind_addr = "[::]:3340"

# Metrics server defaults to port 9090 and often collides with other services;
# turn it off (or move it to a private address you do NOT expose via the tunnel).
enable_metrics = false
# metrics_bind_addr = "127.0.0.1:9099"

# Recommended for a publicly reachable relay: require a bearer token.
# Generate a real one before starting (see "Relay access token" below) — never
# ship the literal placeholder:
# access.shared_token = ["<paste output of: openssl rand -base64 32>"]
```

**1. Run the relay** (no `--dev`):

```bash
iroh-relay -c relay-prod.toml
```

**2. Point cloudflared at it.** The tunnel's ingress must forward the hostname
to `http://localhost:3340`. With a token-based (dashboard-managed) tunnel this
is one line in the dashboard; for a locally-managed tunnel, `config.yml`:

```yaml
tunnel: <tunnel-uuid>
credentials-file: /root/.cloudflared/<tunnel-uuid>.json

ingress:
  - hostname: relay.example.com
    service: http://localhost:3340
  - service: http_status:404
```

```bash
# Locally-managed tunnel:
cloudflared tunnel run <tunnel-name>
# Or dashboard/token-managed tunnel:
cloudflared tunnel run --token <token>
```

**3. Verify** end to end (see [Verifying a relay](#verifying-a-relay) below).

> [!NOTE]
> No paid Cloudflare plan or HTTP/1.1 override is needed — the iroh relay client
> sends no TLS ALPN, so Cloudflare's edge negotiates HTTP/1.1 and the WebSocket
> upgrade works through both quick and named tunnels. A bare
> `curl https://relay.example.com/relay` returning `400` is expected (the relay
> answers 400 to any non-WebSocket request); it is not a tunnel problem. See
> [iroh-relay-connection-trace.md](iroh-relay-connection-trace.md) for the full
> trace.

> **Trade-off:** this routes all relayed traffic through Cloudflare and, because
> there is no QUIC endpoint, disables QUIC address discovery (one of the signals
> iroh uses to help peers hole-punch to a direct connection). For a relay whose
> job is a pure relay-only fallback this is fine; if you want to maximize direct
> P2P success, use the TLS + QUIC production config above instead.

## Address lookup service: iroh-dns-server behind Cloudflare Tunnel

The lookup service is [`iroh-dns-server`](https://github.com/n0-computer/iroh/tree/main/iroh-dns-server),
the same program n0 runs as `dns.iroh.link`, run once on its **own hostname**
(not on a relay host — the relay ingress stays single-purpose) behind the same
kind of Cloudflare Tunnel as the relays. Only its HTTP API is used: servers
`PUT /pkarr/<id>` a signed record, everyone `GET /pkarr/<id>` it. Its DNS
listener is unused and stays on localhost.

`iroh-dns-server` has **no authentication**: anyone can publish under their own
id and read any record. Access is gated by a **capability URL** instead — the
secret is a path segment in front of the API, checked by a small proxy in front
of the server. `cloudflared` cannot rewrite paths, so that proxy (Caddy below)
also strips the secret before forwarding.

```
clients/servers ──https──▶ Cloudflare edge ──tunnel──▶ cloudflared ──▶ Caddy :8080 ──▶ iroh-dns-server :8053
   /lks1-…/pkarr/<id>                                                  strips /lks1-…      /pkarr/<id>
```

**1. Run iroh-dns-server** (version matching the iroh the programs use):

```bash
cargo install iroh-dns-server --version 1.1.0
iroh-dns-server --config dns.toml
```

`dns.toml` (verified against iroh-dns-server 1.1.0; the top-level keys come
first because in TOML a key after a `[table]` header belongs to that table):

```toml
# Every request arrives from Caddy on localhost, so a per-IP limit would put
# all publishers in one bucket. The capability URL gates writes instead.
pkarr_put_rate_limit = "disabled"

data_dir = "/var/lib/iroh-dns"

# Plain HTTP on localhost only; Cloudflare terminates TLS, Caddy fronts this.
[http]
port = 8053
bind_addr = "127.0.0.1"
# No [https] section: nothing here is exposed directly.

# The DNS listener is mandatory in the config but unused by our programs
# (they resolve over the HTTP API). Keep it off the tunnel. The root origin
# "." must be listed: the server keeps its static zone there and refuses to
# start without an SOA for it.
[dns]
port = 5353
bind_addr = "127.0.0.1"
default_ttl = 30
origins = ["lookup.example.com", "."]
default_soa = "ns1.lookup.example.com hostmaster.lookup.example.com 0 10800 3600 604800 3600"

# Never fall back to the public BitTorrent DHT for unknown ids.
[mainline]
enabled = false

[metrics]
disabled = true
```

Records are republished by each server every 5 minutes with a 30 s TTL; the
store keeps a record for 7 days without a republish, so a server that is down
for a while still resolves to its last relay until it comes back.

**2. Front it with Caddy**, which enforces and strips the secret:

```
# /etc/caddy/Caddyfile
:8080 {
	handle_path /lks1-REPLACE_WITH_YOUR_SECRET/* {
		reverse_proxy 127.0.0.1:8053
	}
	respond 404
}
```

`handle_path` strips the matched prefix, so `/lks1-…/pkarr/<id>` reaches the
dns server as `/pkarr/<id>` and `/lks1-…/healthz` as `/healthz`; every other
path is a 404.

**3. Point cloudflared at Caddy** on a dedicated hostname:

```yaml
ingress:
  - hostname: lookup.example.com
    service: http://localhost:8080
  - service: http_status:404
```

**4. Verify:**

```bash
# {"status":"ok","version":"1.1.0",...}
curl -s https://lookup.example.com/lks1-…/healthz

# 404 before the server has published, 200 once it has (body is the signed record)
curl -s -o /dev/null -w '%{http_code}
' https://lookup.example.com/lks1-…/pkarr/<server-endpoint-id>

# 404: without the secret there is no service at all
curl -s -o /dev/null -w '%{http_code}
' https://lookup.example.com/pkarr/<server-endpoint-id>
```

### Generating the lookup secret

The secret is a `lks1-`-prefixed z-base-32 token over 20 random bytes
followed by their CRC-32, so every program can reject a mistyped or truncated
secret at config load. z-base-32 is lowercase letters and digits only — the
alphabet iroh already uses for the endpoint ids that share this URL — so the
secret survives anything that lowercases a URL, which a mixed-case base64
token would not. Generate it with the crate's generator —
tunnel-rs exposes it as a subcommand:

```bash
tunnel-rs generate-lookup-secret
# lks1-<39 z-base-32 characters>
```

Put the same value in the Caddyfile (step 2) and in `lookup_secret` on **every**
server and client, alongside `lookup_url = "https://lookup.example.com"`. Treat
it as a credential separate from the relay token: it appears in cloudflared,
Caddy, and dns-server access logs, so rotate it by changing the Caddyfile and
the configs, never by reusing the relay token.

| Repo | Config keys | CLI | Env |
|---|---|---|---|
| [tunnel-rs] | `[iroh].lookup_url`, `[iroh].lookup_secret` | `--lookup-url`, `--lookup-secret` | `TUNNEL_RS_LOOKUP_URL`, `TUNNEL_RS_LOOKUP_SECRET` |
| [ezvpn] | `[iroh].lookup_url`, `[iroh].lookup_secret` | — | — |
| [flextunnel] | `lookup_url`, `lookup_secret` | `--lookup-url`, `--lookup-secret` | — |

> **Status:** tunnel-rs is the first consumer; the ezvpn and flextunnel rows
> are the planned shape and land when those programs bump to the crate tag
> that carries the lookup.

### When the lookup service is down

Nothing stops. Dials are carried by the relay hints, `online()` waits for a
relay and not for a publish, and a failed publish is retried with a growing
delay until the service is back. What is lost is only the propagation of a
relay change while it is down. The one exception is deliberate: a **server
refuses to start** if its initial publish fails, because a server that starts
unpublished would be found only through hints. Fix the lookup service, then
start the server.

## Relay access token

For a publicly reachable relay, require a shared bearer token so only your
deployments can use it. **Generate a unique secret before first start** — a
guessable or copy-pasted token is the same as having no access control:

```bash
openssl rand -base64 32   # or: head -c 32 /dev/urandom | base64
```

Put it in the config file:

```toml
access.shared_token = ["<the generated secret>"]
```

…or keep it out of the config entirely by setting the environment variable, which
**takes precedence over `access.shared_token`** and sets a single allowed token:

```bash
IROH_RELAY_ACCESS_TOKEN="<the generated secret>" iroh-relay -c relay-prod.toml
```

Either way the relay refuses to start if the token list is empty or contains an
empty string, so a misconfigured token fails loudly rather than silently
disabling access control. Use the config-file form (which accepts a *list*) when
you need to rotate: serve both the old and new token, migrate the clients, then
drop the old one.

Then configure the same token on **both** sides of every program. It is only
ever accepted alongside custom relay URLs — supplying it without them is a hard
configuration error in all three programs.

| Repo | Config key | CLI | Env |
|---|---|---|---|
| [tunnel-rs] | `[iroh].relay_auth_token` | `--relay-auth-token` | `TUNNEL_RS_RELAY_AUTH_TOKEN` |
| [ezvpn] | `[iroh].relay_auth_token` | — | — |
| [flextunnel] | `relay_auth_token` | `--relay-auth-token` | — |

The token rides the relay WebSocket upgrade as `Authorization: Bearer <token>`,
which means the per-relay startup probe validates it: a relay that rejects the
token never comes online and startup fails with that relay named.

## Verifying a relay

**End to end, with tunnel-rs** (the reference relay-only path — this exercises a
real tunnel over the relay with direct P2P disabled, so a pass means the relay
alone is carrying traffic):

```bash
# From a tunnel-rs checkout
./test-scripts/run_e2e.sh --relay-url https://relay.example.com --relay-only
```

> The snippets in this section assume a **tokenless** relay. If the relay has
> `access.shared_token` set, the token must also be supplied — for tunnel-rs, via
> `TUNNEL_RS_RELAY_AUTH_TOKEN` or `--relay-auth-token` (see the table above);
> without it the relay rejects the connection and startup fails.

**Two-relay failover behavior**, fully offline, against local
`iroh-relay --dev` instances:

```bash
./test-scripts/run_relay_failover_e2e.sh
```

**Just the HTTP/WebSocket upgrade**, without running a tunnel. `--no-alpn`
disables the TLS ALPN extension, matching what the iroh relay client sends:

```bash
# Should return 101 Switching Protocols
curl -v --no-alpn \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Protocol: iroh-relay-v2, iroh-relay-v1" \
  -H "Authorization: Bearer $IROH_RELAY_TOKEN" \
  https://relay.example.com/relay
```

> [!IMPORTANT]
> **A 101 does not mean your token was accepted.** This checks the HTTP and
> WebSocket upgrade only — that the relay (and anything proxying it) is reachable
> and speaks the upgrade correctly. In iroh-relay 1.0.2 the server returns `101
> Switching Protocols` *before* the relay handshake runs, and the access-control
> check (`access.shared_token`, `allowlist`, `denylist`, `http`) happens after
> that, inside the relay protocol handshake. A relay with a token configured
> answers `101` to this request even with the `Authorization` header omitted or
> wrong; the connection is dropped a moment later.
>
> Validating the token needs a relay-protocol-aware client configured with it —
> which is exactly what the per-relay startup probe is, so **the end-to-end
> tunnel-rs run above is the real token check.** Include the header here anyway,
> so the command matches what a genuine client sends.

A bare `curl https://relay.example.com/relay` returns `400 Bad Request` — that is
the relay answering any non-WebSocket request, not a proxy problem. Only the full
upgrade request above is a meaningful reachability check. The relay's `/healthz`
route is likewise unauthenticated, so it too confirms the relay is *up* and
nothing more.

## Using your infrastructure

Point both sides at the relays **and** the lookup service. Exact flags differ
per program; the shape is the same:

```bash
# tunnel-rs
tunnel-rs server --relay-url https://relay-a.example.com --relay-url https://relay-b.example.com \
  --lookup-url https://lookup.example.com --lookup-secret lks1-… \
  --secret-file ./server.key --allowed-tcp 127.0.0.0/8 \
  --authorized-keys-file ./authorized_keys

tunnel-rs client --relay-url https://relay-a.example.com --relay-url https://relay-b.example.com \
  --lookup-url https://lookup.example.com --lookup-secret lks1-… \
  --server-node-id <ID> --source tcp://127.0.0.1:22 \
  --target 127.0.0.1:2222 --private-key-file ./client.key
```

For ezvpn and flextunnel, set `relay_urls`, `lookup_url`, and `lookup_secret`
(plus `relay_auth_token` if used) in the server and client config files. See
each repo's own configuration docs.

## Relay behavior

The relay is used for both **signaling/coordination** and as a **data transport
fallback**:

1. The initial connection goes through the relay for signaling.
2. iroh attempts coordinated hole punching (similar to libp2p's DCUtR).
3. If it succeeds, traffic flows directly between peers. How often that happens
   varies with network conditions — NAT type and filtering behavior on both
   sides, address family, and any upstream CGNAT all matter; see
   [nat-traversal-and-transport.md](nat-traversal-and-transport.md#nat-traversal-capability-by-nat-type).
4. If hole punching fails, **traffic continues through the relay**.

> [!NOTE]
> **Bandwidth:** signaling-only coordination *without* relay data fallback is not
> supported by iroh. The relay always acts as fallback when a direct connection
> cannot be established. Size relay bandwidth for the traffic of peers that
> cannot hole-punch.

[tunnel-rs]: https://github.com/flexaccessdev/tunnel-rs
[ezvpn]: https://github.com/flexaccessdev/ezvpn
[flextunnel]: https://github.com/flexaccessdev/flextunnel
