# Bitcoin Client — Cross-Language Design

This repo is the umbrella design doc for the `bitcoin-client-*` family
(`bitcoin-client-java`, plus `-cpp`/`-python`/`-go`/`-rust`/`-ts`/`-cs`
ports). It states what every port must be and why, and points to each
repo's own `DESIGN.md`/`TODO.md` for language-specific implementation
detail rather than duplicating it here - those documents are the ones to
keep in sync with actual source; this one should stay stable.

## Status, as of 2026-09-26

| Repo | State |
|---|---|
| `bitcoin-client-java` | Real, working implementation - both halves below, plus the anonymity fixes described here already applied and verified. The reference every other port is specified against. |
| `bitcoin-client-rust` | An old, minimal stub (RPC-only, ~1 method, against an outdated `Envelope` shape) predating this spec - not yet a real port. |
| `bitcoin-client-cpp`/`-python`/`-go`/`-ts`/`-cs` | Empty repos. Requirements documented (`README.md`/`DESIGN.md`/`TODO.md` in each), nothing implemented yet. |

## The shape every port must have

`bitcoin-client-java` (package `ra.btc`) is two genuinely different clients
behind one `BitcoinClient` interface, selected by whether a local `bitcoind`
is already running:

1. **A local-RPC client** (`ra.btc.rpc.LocalBitcoinClient` in Java) - a full
   Bitcoin Core JSON-RPC wrapper (~80 methods across `blockchain`/
   `control`/`generate`/`mining`/`network`/`tx`/`util`/`wallet`) talking to
   a local, already-trusted `bitcoind` over loopback HTTP+JSON. No
   anonymity concern here at all - it's a loopback call to a process the
   operator already runs and trusts. Every port should build this on top of
   its own sibling `http-client-*` library (all of them already do
   HTTP+JSON+Basic-auth) rather than adding a new dependency.
2. **An embedded SPV client** (`ra.btc.bitcoinj.BitcoinJClient` in Java,
   built on `bitcoinj`) - for when no local node exists: its own P2P peer
   connections, DNS-seed discovery, offline signing, and a network-mode
   state machine (`TOR`/`RELAY_ONLY`/`OFFLINE`, implemented one layer up in
   `1m5-core-java`'s `BitcoinService`) so it still works with no direct
   network path of its own. **This is the half with real anonymity
   requirements** - see below - and the one every non-Java port currently
   lacks a mature library for (see each repo's own `DESIGN.md` "Phase 2
   library" section for what exists in that language).

Don't build a single class/type that tries to be both - the two halves
share almost nothing in implementation, and `bitcoin-client-java` itself
doesn't either.

## Anonymity requirements (non-negotiable, not an afterthought)

These came from a real, confirmed bug, not a theoretical concern: on
2026-09-25, `bitcoin-client-java`'s embedded SPV client was found to be
leaking a plain clearnet DNS query for bitcoinj's built-in Bitcoin DNS
seed hostnames, even on a node otherwise correctly routing every actual
peer connection through Tor. `InetAddress.getAllByName` (Java's default
DNS-seed lookup) ignores any `java.net.Proxy` entirely - a `Proxy` object
only affects `Socket`/`URLConnection` connects, never hostname resolution.
The same class of bug is possible in any language: a SOCKS/HTTP proxy
setting on an HTTP or P2P client does not, by itself, guarantee hostname
resolution goes through it too.

The fix, and the shape every port must reproduce:

- **A resolver seam** (`BitcoinClient.ProxiedHostResolver` in Java:
  `resolve(hostname, timeout) -> IP`) that a Tor client can satisfy via a
  real proxied-DNS mechanism - Tor's own SOCKS5 `RESOLVE` extension
  (`TorSocksRelay#resolve` in `tor-client-java`, added 2026-09-26 for
  exactly this purpose) is the reference implementation.
- **A DNS-seed discovery implementation built on that seam**
  (`BitcoinJClient.ProxiedDnsSeedDiscovery` in Java) instead of whatever
  the underlying Bitcoin library's own default seed-lookup does - used
  whenever a resolver is supplied and no explicit peer list is configured.
  Check what the chosen library actually does by default (read its source,
  don't assume) before trusting it unmodified - the same bug can exist in
  `NBitcoin`, `btcsuite`, `python-bitcoinlib`, or any other library's own
  seed-discovery code.
- **Genuine SOCKS5 for every P2P connection**, not merely "a proxy
  setting." Several sibling `http-client-*` ports' proxy support is
  HTTP-CONNECT-shaped, not real SOCKS5, and can't reach a SOCKS5-only relay
  like `tor-client-java`'s `TorSocksRelay` at all - see each `http-client-*`
  repo's own `DESIGN.md` "Identity metadata leaks" section, and don't
  assume a Bitcoin library's own "proxy" option is SOCKS5 either without
  checking.
- **The `TOR`/`RELAY_ONLY`/`OFFLINE` network-mode model**: `TOR` when a
  real SOCKS proxy and seed resolver are both wired and usable; `RELAY_ONLY`
  when only I2P is usable - the wallet still starts from cached state
  (balance/receive-address/offline signing all work), makes zero direct P2P
  connections of its own, and hands a signed transaction to a peer to
  broadcast on this node's behalf instead; `OFFLINE` when neither is
  usable - a signed send is queued until availability changes. I2P alone
  is never a real fallback for reaching the Bitcoin network directly (no
  general clearnet-TCP outproxy) - don't build a mode that tries to route
  bitcoinj-equivalent P2P traffic over I2P.

## Other requirements worth stating once, here

- **Never hardcode RPC credentials.** `LocalBitcoinClient.AUTHN` in Java is
  a fixed `Basic` header (a fixed username/password baked into source) -
  tolerable only because it's a loopback call to a locally-configured node
  the operator controls, but still worth not repeating. Take RPC
  credentials from config in every port.
- **Independent security review before real funds touch any of this.**
  Every anonymity-relevant claim above should be verified against the
  actual chosen library's source, the same way the Java fix and
  `tor-client-java`'s embedded-Tor work this session were verified by
  reading real bytecode/source rather than trusting documentation - and
  even that verification is not a substitute for real, independent review
  once a port is complete enough to handle live funds.

## See also

Each language repo's own `README.md`/`DESIGN.md`/`TODO.md` for
implementation-level detail, library choices, and a phased checklist:
`bitcoin-client-java`, `bitcoin-client-cpp`, `bitcoin-client-python`,
`bitcoin-client-go`, `bitcoin-client-rust`, `bitcoin-client-ts`,
`bitcoin-client-cs`.
