# Relays and Address Lookup (Default vs Custom)

How a dialer finds a peer depends on the relay mode. This is the single most
important shared decision across [tunnel-rs], [ezvpn], and [flextunnel], and
one implementation serves all three: the `relay` and `endpoint` modules of
[`flexaccess-iroh`](https://github.com/flexaccessdev/flexaccess-iroh).

The default-vs-custom distinction is resolved **once**, at config time, into a
`RelayConfig` enum — `Default` vs `Custom` — and it selects **both** which relay
map iroh uses **and** which address lookup stack is installed. Lookup is *not*
independently configurable; it strictly follows the relay mode: n0's public
stack with the default relays, a **self-hosted, mandatory** lookup service with
custom relays.

| | Relay map | pkarr publish | Lookup | How the dialer finds the peer |
|---|---|---|---|---|
| **Default** | n0 public relays | to n0, persistent identity only | n0 DNS | resolve the published record by endpoint ID |
| **Custom** | configured relays | to the self-hosted service, persistent identity only | the self-hosted service, over HTTP | relay hints attached to the peer's `EndpointAddr`, **plus** the published record |

> **Status (2026-09-04):** the custom-relay row is the design specified here
> and being implemented in `flexaccess-iroh` (next tag after v0.0.3), with
> [tunnel-rs] as the first consumer; [ezvpn] and [flextunnel] follow later.
> A program still on v0.0.3 runs the previous design — custom relays with no
> lookup at all — until it bumps the tag. The reasoning is in
> [relay-failover-findings.md](relay-failover-findings.md).

Whether an endpoint publishes at all is the program's call
(`EndpointOptions::publish_address`): a server with a persistent identity does,
a client that only dials out never advertises its ephemeral id.

mDNS is deliberately left out of that table: unlike the n0 lookup stack it does
**not** follow the relay mode, and it is the one piece of address lookup the
three programs do not enable alike. See [mDNS](#mdns) below.

## Background: iroh address lookup

Dialing an iroh endpoint by `EndpointId` alone works because of [address
lookup](https://docs.iroh.computer/concepts/address-lookup): each endpoint signs
a pkarr record containing its **home relay** URL (and optionally direct
addresses) and publishes it to n0's `iroh-dns-server`; a dialer resolves
`_iroh.<z32-endpoint-id>.dns.iroh.link TXT` to learn `relay=<url>` / `addr=<addr>`
and knows where to reach the peer. Two facts matter:

1. An endpoint has **one home relay** at a time — it holds a persistent
   connection to exactly one relay, so that relay is its only *relay* route for
   inbound connections. It is not necessarily the only route: when direct
   connectivity is available, a dialer can also reach the endpoint through the
   direct addresses in its published record.
2. Relay servers are **stateless and independent** — they do not sync who is
   connected where and do not forward to each other. Traffic sent to a relay the
   peer is not connected to goes nowhere.

## Default relays

The full lookup stack is enabled (`DnsAddressLookup`, plus `PkarrPublisher` when
the endpoint has a persistent identity): the server publishes its current home
relay, the client resolves it by endpoint ID, and iroh's relay failover works —
if the server's home relay dies, it re-homes to another relay from the default
map and republishes; dialers find the new record. The client has no relay hints
to add, so findability relies entirely on n0's public lookup (`dns.iroh.link`).

Pkarr publishing is gated on having a secret key so that an ephemeral endpoint
(a client with no persistent identity) resolves peers but never advertises
itself.

## Custom relays

n0's public infrastructure is never contacted — nothing is published to or
resolved from `dns.iroh.link`. Two things replace it, and **both are required**:

1. **Relay hints.** The dialer attaches every configured relay URL to the
   peer's `EndpointAddr` as transport-address hints. iroh sends QUIC Initials
   to every configured relay, so the handshake succeeds via whichever relay the
   peer is currently homed on, and hole punching is still attempted for a
   direct P2P path. This is what *connects*; see
   [relay-discovery-findings.md](relay-discovery-findings.md).
2. **A self-hosted lookup service** (an `iroh-dns-server` you run — see
   [self-hosting.md](self-hosting.md#address-lookup-service-iroh-dns-server-behind-cloudflare-tunnel)).
   A server with a persistent identity publishes its current home relay to it;
   every endpoint resolves peers from it over HTTP. This is what lets a relay
   change *propagate*: it is the publish path every standard iroh deployment
   has, the one Tailscale's control plane provides, and the one
   [#4435]-style failover in iroh depends on. Without it a server that moves to
   another relay can only be found by clients that happen to hint that relay.

The lookup is **additive**: hints carry every dial, the record is an extra
source, and iroh neither waits for nor fails on a lookup when a hint is
present. So an outage of the lookup service costs only the propagation of a
relay change; existing connections and new dials are unaffected. That is why
one instance is enough and it need not share a host with a relay.

### Configuration

Two options, on **both** sides, next to the relay URLs and the relay token:

| Option | Value |
|---|---|
| `lookup_url` | Scheme and host of the lookup service, e.g. `https://lookup.example.com`. No path, query, or fragment — the crate owns the layout below. |
| `lookup_secret` | The service's capability secret: a `lks1-`-prefixed z-base-32 token (lowercase only, the alphabet iroh uses for endpoint ids in the same URL) carrying its own CRC-32 (see [self-hosting.md](self-hosting.md#generating-the-lookup-secret)). The crate checks the checksum at config load, so a mistyped secret is a startup error, not a silent 404. |

The crate composes `<lookup_url>/<lookup_secret>/pkarr` and hands that base to
iroh's `PkarrPublisher` (servers with a persistent identity, relay URLs only —
never direct addresses) and `PkarrResolver` (everyone). Both options are
**required with custom relays and rejected without them**, exactly like the
relay token: the default relays never see them.

**Startup is strict here too.** A server publishes its record once, in the
foreground, before it starts serving; if the lookup service rejects or does not
answer, startup fails naming it. A client does not probe the lookup service —
its dial is carried by the hints — but it validates the secret's checksum like
the server does.

> [!WARNING]
> Still configure **both sides with the full relay list**, and run **at least
> two relays**. The lookup service lets a client learn a relay it did not hint,
> but only after the server has re-homed and republished; the hints are what
> keep dials working in the meantime, and with one relay there is nothing to
> re-home onto. After its home relay goes offline, an endpoint re-homes onto
> another configured relay within ~30 seconds (net_report re-probes every
> 20–26 s) and republishes.

[#4435]: https://github.com/n0-computer/iroh/pull/4435

## mDNS

mDNS local-network discovery is independent of the relay mode: where it is
enabled at all, it stays on in **both** default and custom mode. It is the
crate's `mdns` feature (which the crate itself compiles out on iOS), and unlike
the rest of the lookup stack it is not uniform across the three programs:

| Repo | mDNS |
|---|---|
| [tunnel-rs] | on in both relay modes; **disabled under `--relay-only`**, which drops mDNS and the direct transports (the self-hosted lookup stays: its records carry relay URLs only) |
| [ezvpn] | **never enabled** — the `mdns` feature is off |
| [flextunnel] | on in both relay modes; **compiled out on iOS**, where raw multicast needs the `com.apple.developer.networking.multicast` entitlement |

So the only place mDNS is switched off *by mode* is tunnel-rs's relay-only mode
(see below); everywhere else it is a fixed per-program property.

## Optional shared relay token

A private relay deployment can require a shared bearer token (iroh-relay's
`IROH_RELAY_ACCESS_TOKEN` / `access.shared_token`). When `relay_auth_token` is
configured it is carried on the `RelayConfig::Custom` variant and applied to
every entry in the custom relay map (`RelayMap::with_auth_token`), which iroh
sends as an `Authorization: Bearer <token>` header on each relay WebSocket
upgrade.

It is **strictly gated to custom relays**: `RelayConfig::from_urls_with_token`
rejects a token supplied without relay URLs, so the default n0 relays never
receive one and the feature is inert in default mode. Blank/whitespace-only
tokens normalize to "no token" rather than erroring. Server and clients sharing a
private relay must configure the same token.

## Custom relay validation: the per-relay startup probe

Before binding the real endpoint (`flexaccess_iroh::endpoint::create_endpoint`),
each configured relay is probed **individually** (`relay::probe_custom_relays`;
the lookup service gets its own strict check, described under
[Custom relays](#configuration))
by binding a throwaway, relay-only endpoint
(`clear_ip_transports`, ephemeral identity) for just that one URL and waiting on
`endpoint.online()`, bounded by a 10 s `RELAY_CONNECT_TIMEOUT`. All probes run in
parallel. **Startup fails if any relay does not come online.**

This is stricter than — and replaces — a single endpoint-wide `online()` wait,
which only proved that *one* relay (the eventual home relay) connected and so
gave a misleading all-clear when a backup relay was down. Because the auth token
rides the relay WebSocket upgrade, this probe also validates the token: a relay
that rejects it never comes online and startup fails.

The strictness is deliberate. A configured backup relay that is silently dead is
worse than a startup failure: it gives false confidence in a failover path that
does not exist. **Startup is strict; runtime is not** — once a process is
running, losing a relay is survivable and the endpoint re-homes onto a surviving
one. For the same reason a mid-run **rebuild** of an endpoint (see
[home-relay-watchdog.md](home-relay-watchdog.md)) skips the probe: during an
outage that strictness would block recovery through the one relay that still
answers.

`clear_ip_transports()` on the probe endpoint is what makes `online()` a *pure
relay* reachability signal: a holepunched direct path can never mask a dead or
auth-rejecting relay.

## Relay-only mode

Relay-only (`EndpointOptions::relay_only`) drops the direct IP transports and
mDNS on the *real* endpoint, so it is reachable only over the configured
relays. The self-hosted lookup service stays installed: its records carry
relay URLs only, so it can never produce a direct path, and a relay-only
deployment needs relay changes to propagate like any other. It requires a custom relay set — the
rate-limited default relays cannot serve it. Because a relay-only dialer tries
the relays one at a time, `RelayConfig` keeps the configured order (deduping
only exact repeats): the first URL is the preferred relay.

**[tunnel-rs] is the reference program for relay-only setup.** It is the only one
of the three that exposes relay-only as a first-class user-facing mode
(`--relay-only` on both `server` and `client`, CLI-only so it cannot be switched
on accidentally from a config file), and it carries the matching sequential
per-relay failover dial path and a fully offline two-relay e2e suite
(`test-scripts/run_relay_failover_e2e.sh`). Use it to validate a self-hosted
relay end to end before pointing other programs at it — see
[self-hosting.md](self-hosting.md). ezvpn and flextunnel keep
`clear_ip_transports()` only inside the startup probe described above.

## Connect path

With custom relays, all three attach every configured relay URL to the peer's
`EndpointAddr` and dial **once**, letting iroh fan the QUIC Initials out across
the hints. Relay-only is the exception: with no direct path to fall back on,
tunnel-rs dials the relays **one at a time** so a dead relay fails fast and the
next is tried.

Every dial is wrapped in a timeout — an untimed `endpoint.connect` can hang
indefinitely when no path is reachable.

## Where this lives in each repo

The shared part is one crate; each program keeps a thin layer over its
builder.

| Repo | Implementation | Notes |
|---|---|---|
| [flexaccess-iroh] | `src/relay.rs`, `src/endpoint.rs`, `src/lookup.rs` | `RelayConfig` (relay URLs, token, `lookup_url` + `lookup_secret`), the per-relay probe, the lookup secret format and generator, the base builder (`endpoint_builder` + `EndpointOptions`), `create_endpoint` vs `rebuild_endpoint` |
| [tunnel-rs] | `src/iroh_mode/endpoint.rs` | `mf/4` ALPN, transport tuning, user-facing `--relay-only` + sequential relay failover dial, `generate-lookup-secret`; `mdns` on. **First consumer of the mandatory lookup** |
| [ezvpn] | `src/transport/endpoint.rs`, `src/transport/paths.rs` | VPN ALPN, transport tuning, bounded connect; `mdns` off; iroh fork via `[patch.crates-io]`; on-demand `/healthz` per-relay health check for status UIs |
| [flextunnel] | `crates/flextunnel-core/src/transport/endpoint.rs`, `.../transport/paths.rs` | three ALPNs + native allowlist hook; `mdns` on (crate compiles it out on iOS); outbound bridges attach the same relay hints; on-demand `/healthz` health check |

The `/healthz` status check in ezvpn/flextunnel is a *different* thing from the
startup probe: it runs only when a status snapshot is requested, hits the relay's
**unauthenticated** HTTP health endpoint, and so confirms the relay is *up*, not
that the token is accepted. Token validation is the startup probe's job.

[tunnel-rs]: https://github.com/flexaccessdev/tunnel-rs
[ezvpn]: https://github.com/flexaccessdev/ezvpn
[flextunnel]: https://github.com/flexaccessdev/flextunnel
