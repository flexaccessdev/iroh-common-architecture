# Relay failover findings: why custom relays need a lookup service

Analysis (2026-09-04, against iroh **1.1.0** sources, iroh PR [#4435], and
tailscale/headscale sources) behind making a self-hosted address lookup
service **mandatory** with custom relays, and behind the plan to retire the
[home-relay watchdog](home-relay-watchdog.md). Companion to
[relay-discovery-findings.md](relay-discovery-findings.md), which established
that relay hints alone are enough to *connect*; this document is about what
happens after that, when a relay goes away.

## Question

The watchdog exists because a custom-relay server was observed to lose its
home relay for good and become undialable until restarted. iroh 1.1.0 did not
fix that. Is there a standard iroh way to survive relay loss that makes the
watchdog unnecessary, and what does it need from the deployment?

## Findings

### 1. iroh 1.1.0 changes nothing about relay reconnect

Checked against the released sources: `iroh-relay`'s client (the WebSocket
layer the observed reset happened in) is byte-identical between 1.0.3 and
1.1.0, and the reconnect machinery in `iroh/src/socket/transports/relay/actor.rs`
is unchanged. On disconnect the actor marks the relay `Disconnected` and
retries the *same* URL; nothing triggers a fresh net report. The one relay
change that shipped ([#4444]) keeps priority messages answered during reconnect
backoff and is a fragment split out of [#4435].

### 2. Upstream's answer is failover, and failover needs a publish path

[#4435] (open `[wip]` draft, untouched since 2026-07-27, pinned to an
unreleased `noq` branch, n0-computer/noq#770) is upstream's design for relay
loss. It is a **failover**: when the home relay is lost the endpoint forces a
full net report, homes on a **different** relay, republishes its address, and
peers **re-resolve** it through address lookup and open a path over the new
relay. Existing connections keep their LAN paths and migrate their relay path;
nothing is torn down.

Two consequences for these programs:

- It only works with **at least two relays** and **an address lookup service
  both sides use**. With a single relay the forced report re-picks the same
  URL, which the relay actor treats as "no change". With custom relays as
  deployed until now, address lookup was off entirely, so the re-resolve half
  had nothing to resolve against: iroh stops retrying a lookup when no service
  is configured.
- It does **not** touch the reconnect loop. If the actor for the current home
  relay stops retrying while the relay itself stays healthy (the shape of the
  original incident, where the server ended up registered on *neither* relay),
  a net report keeps choosing that same relay and the failover never fires.

So the missing piece in our deployments was never the watchdog's job; it was
the publish path that every standard iroh deployment has and ours removed.

### 3. This is how Tailscale and headscale do it

Headscale does almost nothing here: it hands out the DERP map, stores each
node's preferred DERP region, and pushes a patch to every peer when that
region changes. That patch **is** the lookup service. All failover logic is in
the client:

- The DERP client reconnects on every error, backing off to at most 5 s,
  forever. The server sends a keepalive every 60 s and the client keeps a 120 s
  read deadline on the socket, so a half-open connection is torn down and
  re-dialed within two minutes.
- A region has several meshed servers; the client tries them in order.
- The home region is re-evaluated by a netcheck every 20–26 s while active,
  with stickiness (keep the old home unless it is unreachable, or the new one
  is at least 10 ms and 33 % better). A new home goes to control and out to
  every peer. WireGuard sessions do not care which address packets arrive
  from, so nothing is torn down.
- **The client refuses to change home while it is not connected to control**,
  because peers could not learn the change and moving would break all
  connectivity. That is the mirror image of finding 2.

The two lessons pull apart: failover needs a publish path (we now add one);
the reconnect loop must be unkillable on its own (deadline-based; iroh's relies
on its 15 s ping timer inside the actor, and that is the loop that went quiet).

### 4. The service keeps working when the lookup service is down

Checked in iroh 1.1.0, provided clients keep dialing with relay hints:

- **Connecting:** with a relay hint present, iroh resolves the remote
  immediately and starts on that path; the lookup runs alongside as an extra
  source, and a failed lookup is only recorded.
- **Server startup:** `online()` waits for a relay connection, not a publish.
- **Publishing:** a failed publish is logged and retried with a growing delay
  (one second per consecutive failure), forever. The record reappears within a
  republish cycle once the service is back.

The lookup service is therefore load-bearing only **at the moment a server
changes relays**. While it is down, existing connections are unaffected and
new dials still work through the hinted relays; only propagation of a relay
change is lost, which is the same as today. That is why one dedicated instance
is enough and it does not need to share a host with a relay.

### 5. Authentication: iroh-dns-server has none, but a proxy can add it

`iroh-dns-server` 1.1.0 verifies that a published packet is signed by the id in
the URL and rate-limits `PUT` per IP. That is all: anyone can publish under
their own id, and reads (`GET /pkarr/{id}`, DNS-over-HTTPS, plain DNS) are
open. The iroh publisher and resolver build a plain HTTP client with no hook
for headers, so a bearer token needs a code change on both sides (about thirty
lines; the relay already has the same `auth_token` / `Access` design, and no
upstream issue asks for it on the dns server). Two proxy-level options need no
code change:

1. **Allowlist node ids on `PUT`** — the id is in the path, and only servers
   (fixed ids) ever publish; clients with ephemeral ids only `GET`.
2. **Capability URL** — both `PkarrPublisher::builder(url)` and
   `PkarrResolver::builder(url)` append `/{id}` to whatever base URL they are
   given, so a base of `https://lookup.example.com/<secret>/pkarr` works
   unchanged, and it gates reads as well as writes.

Option 2 was chosen: it covers reads, and it keeps the per-server allowlist out
of the proxy config. `cloudflared` cannot rewrite paths, so a small proxy
(Caddy) strips the secret before `iroh-dns-server`; see
[self-hosting.md](self-hosting.md#address-lookup-service-iroh-dns-server-behind-cloudflare-tunnel).

## Decision

- **Custom relays require a self-hosted lookup service**, configured as two
  options on both sides like the relay token: `lookup_url` (scheme and host)
  and `lookup_secret` (a prefixed, checksummed token, so a mistyped secret is
  rejected at config load instead of silently 404ing). The crate composes
  `<lookup_url>/<lookup_secret>/pkarr`. Servers with a persistent identity
  publish (relay URLs only); everyone resolves over HTTP. Relay hints stay
  attached to every dial, so the lookup is additive and its outage is
  survivable. Specified in
  [relays-and-address-lookup.md](relays-and-address-lookup.md#custom-relays),
  implemented in `flexaccess-iroh`, adopted by tunnel-rs first.
- **One dedicated `iroh-dns-server` behind Cloudflare Tunnel**, on its own
  hostname, with a capability URL. Not co-hosted with a relay: finding 4 means
  it is not a single point of failure for service, and keeping it off the relay
  hosts keeps the relay ingress single-purpose.
- **Run at least two relays.** With one relay there is nothing to fail over to,
  lookup or not.
- **The watchdog stays until removal is validated** against this setup; see
  [home-relay-watchdog.md](home-relay-watchdog.md#removal-plan). Finding 2
  means the publish path enables the standard failover but does not by itself
  address a reconnect loop that stops retrying.

## Not chosen

- **Forking iroh to carry [#4435]** on ezvpn's fork: ported and built for
  evaluation (it compiles on 1.1.0 with small conflicts), but it changes no
  reconnect decision, depends on an unreleased `noq` branch, and its useful
  half needs the lookup service first. Revisit when it lands upstream.
- **Making relay hostnames dual-purpose** (lookup on every relay host, publish
  to all, resolve from all): works, but unnecessary given finding 4.

[#4435]: https://github.com/n0-computer/iroh/pull/4435
[#4444]: https://github.com/n0-computer/iroh/pull/4444
