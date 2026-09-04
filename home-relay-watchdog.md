# Relay recovery and the historical server watchdog

## Current behavior

Starting with `flexaccess-iroh` v0.0.4, servers use iroh 1.1.x's native relay
WebSocket liveness detection and reconnect behavior. There is no server
watchdog, delayed `network_change()` nudge, or timed endpoint rebuild in
flexaccess-iroh, flextunnel, ezvpn, or tunnel-rs. A relay outage leaves the
server endpoint and its healthy direct connections alive. Initial custom-relay
validation remains strict.

This removal is a deliberate return to native recovery, **not confirmation
that iroh 1.1.0 fixes the historical permanent stall**. The original incident
has not yet been reproduced and cleared on 1.1.0 without the workaround.
Iroh 1.1.0 includes [#4444](https://github.com/n0-computer/iroh/pull/4444), which
keeps priority messages responsive during relay reconnect backoff; that is a
related fix, not proof that this incident is resolved. ezvpn retains its
existing iroh 1.1.x fork through its workspace patch.

## Original flextunnel incident

The server workaround originated in flextunnel on iroh 1.0.3. After a routine
relay WebSocket reset in a Cloudflare Tunnel deployment (observed roughly
hourly), the server permanently lost its home-relay registration: no further
dial retries, no warnings, and no registration on either configured relay.
Off-LAN clients, including the iOS app, timed out. A LAN Mac client could still
connect through mDNS, masking the failure. Restarting the service restored
relay access. The reset was the observed trigger; the underlying iroh defect
was not established.

History:

- [flextunnel 3a1d25d](https://github.com/flexaccessdev/flextunnel/commit/3a1d25d836f114c813f9ba8cdec0ec5845649be5): server watchdog and endpoint rebuild.
- [flextunnel 4f398c7](https://github.com/flexaccessdev/flextunnel/commit/4f398c7803f1dd96e2394f43a539fd7dc0724f33): upgrade to 1.1.0, explicitly retaining the workaround because the release did not claim to fix the incident.
- [flextunnel 3fc6169](https://github.com/flexaccessdev/flextunnel/commit/3fc6169e0248c00b53c1ada44c1950429b6d3275): backoff to avoid repeatedly dropping healthy LAN clients when the relay itself is unavailable.
- [flexaccess-iroh v0.0.3](https://github.com/flexaccessdev/flexaccess-iroh/tree/v0.0.3): last shared release containing `src/relay_watchdog.rs` and its tests.

## If the same failure returns

1. Capture the exact iroh version or fork revision and relay logs around the
   reset, with `RUST_LOG=info,iroh=debug,iroh_relay=debug`. Record home-relay
   status, whether reconnect attempts continue, whether the relays accept a
   fresh endpoint, and whether LAN access still works. An unavailable relay or
   rejected token alone is not the historical permanent stall.
2. After capturing evidence, restart the affected server service to restore
   access if a fresh endpoint can register. Existing sessions will disconnect.
3. Reproduce with repeated WebSocket disconnects and idle periods, on one and
   two custom relays. Test both relay-only access and healthy direct clients;
   tunnel-rs is the relay-only reference. Verify registration and new inbound
   dials recover after the relay becomes reachable, without replacing the
   server endpoint.
4. If the permanent stall is confirmed, restore the temporary workaround in
   **flexaccess-iroh**, tag a new release, and update all consumer tags and
   server integrations. Use the historical implementation as a reference;
   do not keep disabled code or copy the watchdog into individual apps.
   Investigate and report the underlying iroh failure with the reproduction.

### Workaround restoration requirements

The previous workaround observed `Endpoint::home_relay_status()` for custom
relays only. It treated any connected home relay as healthy and reset its
outage clock on recovery. Non-home relay idle disconnects were ignored.

After 60 seconds without a connected home relay it called `network_change()`;
after 180 seconds total it requested a server endpoint replacement. The caller
closed the old endpoint and bound a new one with the same identity, ALPNs,
allowlists, transport settings, and relay configuration. Rebuilds skipped the
startup per-relay probe and tolerated the online wait failing; binding failures
were retried every 30 seconds.

The outage result recorded whether a home relay had ever connected. Consecutive
endpoints that never registered extended the rebuild deadline to 6, 12, 24,
then 30 minutes; successful registration reset that escalation. This matters:
a genuinely unavailable relay cannot be repaired by rebuilding, and every
rebuild terminates healthy direct sessions too.

Preserve graceful endpoint close, shutdown signals, and quick-mode idle exits
through any restored rebuild loop. Stop bridge tasks tied to the retired
endpoint. In ezvpn, update the status endpoint and TUN self-encapsulation UDP
port filter when the endpoint changes. Restore the outage timing tests and
consumer backoff tests, then run clippy, unit tests, and relay recovery tests.

## Separate client recovery

Flextunnel's client `RebuildableEndpoint` escalation remains in use. It came
from a separate WSL2 incident: a relay ping timeout followed by repeated
30-second reconnect failures, immediately repaired by a client restart
([a2f94c8](https://github.com/flexaccessdev/flextunnel/commit/a2f94c8ba82c3868cfb87d0fb829b72db6b5e0f0)). Removing the server watchdog does not
remove the shared client rebuild API or its tests.
