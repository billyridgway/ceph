# RGW gateway: RDMA-semantics transport and read-through cache for multisite

Status: design for a proof-of-concept branch, Nov 22, 2026.
Branch: `wridgway/rgw-rdma-gateway` (billyridgway/ceph).
Prototype and measurements: github.com/billyridgway/rgw-rdma-gateway
(`bench/RESULTS.md` there is the evidence base for every number below).

## 1. Summary

Add a paired-gateway data plane to RGW multisite, in the shape of
Vcinity-style gateway products but built from RGW's own machinery:

- A **read-through cache**: a GET (or a sync fetch) for an object version
  whose metadata is local but whose data lives in a peer zone is served by
  streaming chunks from the peer zone's gateway, populating a local cache
  so later reads never cross the WAN.
- A **windowed, loss-tolerant transport** between paired gateways for the
  WAN leg: RDMA *semantics* (large sliding window, selective
  retransmission, no congestion backoff) over ordinary UDP on a
  configurable port, with HTTP as the always-available fallback and real
  RDMA (verbs/rsockets) as a later transport where the fabric allows it.

RGW remains the system of record at both ends. The gateway moves bytes
for versions that RGW's metadata sync already knows about; it originates
no metadata, makes no access decisions, and degrades to today's HTTP
sync when it is unavailable.

## 2. What the prototype measured (and what it changed)

Test link: two hosts, 24.6 ms RTT, ~1.1 Gbps ceiling. All payloads
SHA-256-verified end to end.

- **Transport, clean link:** custom reliable UDP ties tuned TCP at
  64 MiB (0.90 vs 0.88 Gbps) and trails ~13% at 1 GiB (0.99 vs
  1.12 Gbps). Whole-object 256 MiB: HTTP 2.45 s, HTTPS 2.78 s,
  gateway-UDP 3.44 s. *The custom protocol is not faster than HTTPS on a
  clean link*, and it is the most CPU-hungry option (~2 s server CPU per
  256 MiB vs ~1 s for nginx TLS — per-packet syscalls plus the
  prototype's extra user-space hop).
- **Transport, 1% packet loss:** HTTPS collapses to 293.9 s for
  256 MiB; the windowed transport holds 5.4 s (~54x). TCP reads loss as
  congestion and backs off; between two gateways on an operator-owned
  link, loss is weather, not a congestion signal. *This regime — lossy
  long-haul WAN — is where the transport earns its keep.*
- **RoCE v2 is not assumable:** UDP 4791 is filtered on the prototype's
  provider path while general UDP passes. Any design that requires
  stock RoCE across a WAN is dead on arrival; the transport must run on
  an arbitrary port, with real RDMA reserved for fabrics that offer it.
- **The cache is the largest win:** warm hits served 64 MiB in 0.16 s
  (~9x vs direct) and 256 MiB in 0.35 s (~17x); cold misses stream
  through at ~direct speed. ETag revalidation (STAT before serve) never
  served a stale byte across a version change.
- **Pipelining is not optional:** connection-per-chunk fetching measured
  21.3 s for a cold 256 MiB read where the pipelined design (persistent
  peer connection, chunk requests pipelined 8-deep, stream-through
  delivery) measures 3.0 s at the same chunk size. With pipelining,
  chunk size nearly stops mattering for sequential reads (2.9–3.8 s
  across 1–64 MiB chunks).
- **Chunk size is a sparse-access knob:** wire amplification for sparse
  reads tracks chunk size (1.1x / 4.5x / 18x / 40x at 1 / 4 / 16 /
  64 MiB). 4 MiB is the balanced default.
- **Synchronous prefetch does not work:** it measured *slower* than no
  prefetch (1.92 s vs 1.73 s) because the fetch happens inside the
  current request. Prefetch requires an asynchronous fetcher or none.
- **RGW is a good citizen as a source:** 1 MiB range GETs answer in
  6–9 ms, offset-independent, byte-exact. Gateway vs an RGW source:
  cold 256 MiB 3.48 s at 4 MiB chunks, warm 0.41 s, sparse 1.22 s —
  the transport-floor class, ~25x faster than against a source whose
  ranges are expensive.

## 3. Design decisions

- **D1 — Cache first, transport second.** Read-through caching is the
  headline feature and ships value on any transport, including HTTP.
  The custom transport is the WAN-leg upgrade for lossy links, not the
  product.
- **D2 — Pluggable transport, one fetch interface.** HTTP (existing,
  permanent fallback), windowed reliable UDP on a configurable port
  (the measured transport), RDMA verbs/rsockets later where available.
  Nothing in the design may assume UDP 4791 is routable.
- **D3 — Loss policy is explicit.** No congestion backoff between paired
  gateways; SACK-based repair; large window. Pacing is configurable,
  and the fairness trade (a non-backing-off flow is unfair on shared
  links) is documented at the config surface, not buried.
- **D4 — Pipelining is part of the protocol contract.** Persistent peer
  connections, pipelined chunk requests, stream-through delivery.
  Connection-per-chunk is prohibited by design (measured ~7x worse).
- **D5 — Chunk size 4 MiB default**, configurable per zone pair; chosen
  for sparse amplification and cache granularity, not throughput.
- **D6 — No synchronous prefetch.** An async fetcher is a separate,
  later component; v1 omits prefetch entirely.
- **D7 — Daemon + thin in-RGW client.** The transport/cache daemon is
  paired 1:1 with a zone's RGWs (isolates transport crashes from S3
  serving, keeps driver/UDP code out of the radosgw process). RGW links
  a small client used by the fetch paths. The daemon serves bytes only
  against scoped tokens (see Security); in the POC it reads the source
  zone through the local RGW's S3 with sync credentials, as the
  prototype did.

## 4. Where it lands in the RGW tree

Integration points, in landing order:

- **4a — Config scaffolding (first code on this branch).** Per-zone-pair
  gateway config in the multisite period config: peer gateway endpoints,
  transport preference, chunk size, enable flags, QoS caps. Surfaced via
  `radosgw-admin zonegroup modify` / `zone modify` alongside the existing
  endpoint lists. No behavior change.
- **4b — Data-sync fetch path.** Multisite data sync already fetches
  object data from the source zone remotely
  (`src/rgw/driver/rados/rgw_data_sync.cc`,
  `RGWRados::fetch_remote_obj` in `src/rgw/driver/rados/rgw_rados.cc`,
  over the REST client in `src/rgw/rgw_rest_client.cc`). Behind a config
  flag, the fetch goes through the gateway client instead of HTTP; any
  gateway failure falls back to the HTTP fetch. No client-visible
  semantics change — this is the safest first code, because sync already
  has the "fetch this version from that zone" shape.
- **4c — Read-through on the GET path.** In `RGWGetObj::execute`
  (`src/rgw/rgw_op.cc`): metadata present, data absent locally → fetch
  via the gateway client, stream to the client, populate the cache.
  Cache placement follows the original sketch: a dedicated pool fronted
  by a storage class so placement is visible in the object manifest;
  the POC may start with the daemon's disk cache as prototyped. Whether
  the D4N cache driver (`src/rgw/driver/d4n`) is the right in-tree cache
  substrate is an open question below.
- **4d — Tier target.** A peer zone registered as a lifecycle tier
  target (`src/rgw/driver/rados/rgw_lc_tier.cc`), except transitions
  move data over the gateway transport and reads of tiered objects are
  served read-through instead of restore-only, reusing the cloud-tier
  manifest/head conventions.

## 5. Wire protocol (transport v1)

Evolved from the prototype (`bench/udp_transport.h` in the prototype
repo), with its known debts fixed rather than carried:

- Datagram stream on a configurable UDP port; 1,400-byte prototype
  packets become MTU-sized datagrams; 8,192-packet sliding window;
  cumulative ACK + 64-bit SACK bitmap; retransmit holes only.
- **RTT-estimated RTO** (the prototype's fixed 25 ms RTO is a measured
  tail-latency debt at small sizes — do not ship it).
- **Batched syscalls** (`sendmmsg`/`recvmmsg`) — per-packet syscalls
  were the prototype's CPU ceiling.
- **Explicit FIN handshake** — the prototype's drain-on-close is a
  stand-in, not a protocol.
- Transfers are (bucket, key, version-id, range) addressed, per-chunk
  checksummed, resumable from a chunk bitmap, and idempotent per
  version.

## 6. Cache design

- Revalidation before serve: the serving side stats the source version
  (ETag) before serving a cached chunk set; a version change is a miss.
  A cached copy is *never* authoritative.
- Eviction is LRU over whole objects in the POC (range-level eviction
  is a later refinement); invalidation also arrives via mdlog on
  overwrite/delete.
- Read-through traffic has strict priority over sync/tiering traffic in
  the bulk scheduler (per-tenant/per-bucket token buckets); read-through
  is latency-bound, replication is not.

## 7. Consistency rules

- The gateway moves bytes for exactly one version at a time; version
  identity (version-id + ETag) travels with every chunk and is verified
  on completion.
- Deletes and overwrites propagate only through mdlog/bilog as today;
  the gateway never originates a metadata change.
- Read-through may serve a version only if local metadata already
  reflects it. If metadata sync has not arrived, the GET is a normal
  local miss — the gateway is not a side channel around eventual
  consistency.
- Tiered-remote objects keep their index entry and manifest locally,
  with data location pointing at the peer zone + gateway, mirroring
  cloud-tiered objects' local head.

## 8. Security

- Control plane: mTLS with per-zone certs, the multisite trust model.
- Data plane: per-transfer AEAD (AES-GCM) chunk encryption, keys from
  the control channel, rotated per session; packet sequence numbers as
  nonces. Do not assume link encryption — RoCE provides none either.
  (The prototype ran plaintext UDP between two test hosts; that was a
  benchmarking shortcut, not the design.)
- Fetch authorization: every fetch carries a short-lived token scoped
  to bucket + key + version + range, issued by the requesting zone's
  RGW after it authorizes the original request. The peer gateway serves
  bytes only against that token and makes no access decision of its
  own; cross-cluster IAM is not federated by the gateway.

## 9. Failure behavior

- Gateway or transport down → sync falls back to the HTTP fetch path;
  read-through disables itself; tiered reads fall back to
  restore-over-HTTP. Degradation is visible in zone status, never
  silent.
- Partial transfer → chunk bitmap persists with the sync marker; resume
  on reconnect. A version whose data never completes stays in the
  existing "sync pending" state.
- Split-brain is out of scope by construction: multisite's rules decide
  which version wins; the gateway moves the winner's bytes faster.

## 10. Open questions

- Cache substrate in-tree: dedicated pool + storage class (sketch) vs
  the D4N cache driver.
- Daemon packaging: a mode of radosgw vs a separate daemon under
  cephadm/orch; the POC keeps the prototype's standalone daemon.
- Pacing defaults on shared links (D3's fairness trade made concrete).
- Whether an async fetcher (D6) earns its complexity after 4c
  measurements.
- Per-bucket / sync-policy opt-in granularity: data sharing is an
  explicit per-bucket or per-zone-pair choice, never a zonegroup-wide
  default.
