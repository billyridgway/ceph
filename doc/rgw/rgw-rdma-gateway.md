# RGW long-haul gateway: WAN transport acceleration for multisite

Status: refactored design, Nov 25, 2026, following an implementation
review of the original proof-of-concept branch.
Branch: `wridgway/rgw-rdma-gateway` (billyridgway/ceph).
Standalone gateway and transport: github.com/billyridgway/rgw-rdma-gateway
(measurements in `bench/RESULTS.md` there).

## 1. Objective and non-goals

Improve object storage data movement over long-haul, high-latency,
congested networks for:

- RGW multisite asynchronous replication.
- RGW's existing read-through cache functionality (the D4N cache
  driver), whose data arrives through the multisite fetch machinery.
- Potential future consumers (ZGW, GPFS AFM) of the standalone
  transport engine.

Non-goals, explicitly: no new storage tiering semantics, no change to
multisite consistency guarantees, no redesign of lifecycle
management, no on-demand replication on the GET path. The guiding
principle is to improve WAN data movement without changing existing
object storage semantics, with the smallest possible Ceph
integration, concentrating effort on the transport.

## 2. Architecture

The gateway is a site-local, standalone service deployed in pairs:

```
Site A RGW ---- local HTTP ---- Site A Gateway
                                     |
                              optimized WAN transport
                                     |
Site B RGW ---- local HTTP ---- Site B Gateway
```

Only the gateway-to-gateway connection traverses the WAN. In its
preferred form the gateway pair is a transport-level tunnel: it
carries RGW's HTTP byte streams without terminating them, so SigV4
signatures, Host headers, multisite `rgwx-*` parameters, object
metadata, and (where configured) TLS all remain end to end between
the RGWs. RGW authorization decisions are untouched; the gateway
never sees a credential it could use and cannot mint one; a client
request cannot trigger a privileged transfer, because gateways only
carry connections RGW itself opened.

Two consumers need no Ceph change at all:

- **Cloud-S3 tiering / restore.** The tier endpoint is already an
  operator-configured URL; pointing it at the local gateway inserts
  the gateway pair into the restore path with existing semantics
  intact.
- **Any HTTP client of a remote RGW/S3 endpoint** (including future
  ZGW/AFM consumers) can be routed the same way.

Multisite replication is the one consumer whose destination is not a
free-form URL: RGW dials the source zone's endpoint from the period,
and zone endpoints are the zone's identity, not a routing knob. For
multisite there are two options:

- **Deployment routing** (zero Ceph diff): the peer endpoint is
  reachable via the gateway pair at the network level.
- **The minimal native hook** (implemented on this branch, §3): RGW
  connects to its *local* gateway, which routes by destination zone
  identity.

## 3. What this branch implements

Exactly three things:

1. **Minimal gateway configuration on RGWZone**
   (`RGWZoneGatewayConfig` in the period): `enabled` plus the zone's
   **local gateway endpoints**. Nothing transport-specific is in RGW
   configuration: wire protocol, ports, chunking, and QoS belong to
   the gateway service, and changing them does not commit a new
   period. Set via `radosgw-admin --gateway-enabled` /
   `--gateway-endpoints` on zone create/modify and zonegroup add.
   The binary/JSON period encoding stays able to read the earlier
   POC layout (extra fields are read and dropped / ignored).

2. **Gateway routing inside `RGWRados::fetch_remote_obj`** — the
   existing multisite fetch primitive used by data sync
   (`RGWFetchRemoteObjCR`) and cross-zone CopyObj. When the local
   zone has a gateway configured, the zone service builds one gateway
   connection per source zone alongside the direct zone connections
   (`RGWSI_Zone::gateway_conn_map`), addressed to the local gateway
   endpoints and carrying the source zone's identity; fetches use it
   instead of the direct connection. The wire protocol is the
   ordinary sync protocol with the usual system key. Replication
   traffic uses the gateway as transport only: nothing is cached or
   stored twice, and bytes land in RADOS exactly as a direct fetch
   would put them. Metadata sync, data-log processing, version
   selection, conflict resolution, sync policies, checkpoints, and
   delete handling are all unchanged — they never reach this code.

   Failure handling is classified (helpers above `fetch_remote_obj`):
   - *Transport-class* failures (connect/reset/timeout/EIO/...):
     fall back once to the direct zone connection and back the
     gateway off for that source zone
     (`rgw_gateway_failure_backoff_secs`, default 30) so a dead
     gateway is not retried on every object. Note, from rig testing:
     the REST layer surfaces some path failures (e.g. a refused
     connection to a dead gateway) as an internal error at
     `complete_request` rather than as a transport errno; those take
     the server-error path below — fallback without backoff — so a
     dead gateway costs one fast-failing attempt per fetch until a
     success or a true transport errno changes the state.
   - *Server errors* (503 / internal error): may fall back, never
     mark the gateway unhealthy — either end may have produced them.
   - *Application-level answers* (404, 403, 401, 400, 304, ...):
     propagate unchanged. They are the source zone's answer, not a
     path failure.
   - The fallback can be disabled (`rgw_gateway_fallback_direct`,
     default true) to make the gateway mandatory.
   - A `complete_request` failure after a full transfer is safe to
     retry: the fetch writes through a temporary object and completes
     atomically, so a re-fetch restarts cleanly; the cost is a
     repeated WAN transfer, which the classification above bounds to
     a single fallback.

3. **An independent coroutine throttle fix** in `fetch_remote_obj`:
   use `rgw::make_throttle(window, rctx.y)` instead of an
   unconditional `BlockingAioThrottle`. With a coroutine caller, a
   blocking throttle deadlocks once a multi-stripe object fills the
   window, because librados completions post to the very context the
   blocking wait stalls (root-caused with gdb; reproduced with
   objects >= 8 MiB, which hung indefinitely, and verified fixed at
   8/32/256 MiB). This stands alone from the gateway work — cross-zone
   CopyObj passes a real yield and hits the same latent deadlock —
   and is a plausible upstream patch in its own right.

## 4. What was removed, and why

The original POC branch carried two features the review rejected;
both are removed in full, and the branch's remaining diff is only §3:

- **Peer-zone lifecycle tiering** (a new `peer-zone` tier type with
  manifest, placement, lifecycle transition/restore, and copy
  changes). A new tier type is not necessary to accelerate existing
  remote operations; the existing cloud-S3 tier path (§2) covers the
  tiering consumer with zero semantic change.
- **GET-path multisite read-through** (on a local miss, fetch the
  whole object from the master zone, persist it in RADOS, then serve
  the GET; triggered from the REST handler via a SAL hint). That is
  on-demand replication, not acceleration of RGW's existing
  read-through cache: it fires before authorization completes and it
  assumes the master zone is authoritative. RGW's existing
  read-through cache is the **D4N** SAL filter driver
  (`rgw_filter=d4n`): an L1 block cache with version-pinned blocks
  over the Rados driver, whose misses drain from the wrapped store —
  i.e. through the same multisite fetch machinery §3 accelerates.
  Known upstream limitation, documented rather than absorbed: D4N's
  block directory carries a `// TODO: Retrieve remotely` for blocks
  held by another cache host.

## 5. The standalone transport engine (primary contribution)

The windowed reliable-UDP transport and the tunnel that carries TCP
streams over it live in the standalone repository, independent of
Ceph, consumable by RGW (via the gateway), ZGW, and AFM alike. The
prototype demonstrated the loss-resilience signature on one link
profile (25 ms RTT, 1 Gbps, random loss): transfers stayed ~flat
from 0% to 1% loss while TCP relays collapsed (~37x at 1% loss for
256 MiB).

It is not yet demonstrated to be good transport, and this design
does not claim otherwise. The prototype has a fixed RTO with no RTT
estimation, no congestion control (an external pacing gate had to be
added during testing to stop its window from overfilling an in-RTT
queue — the prototype can be antisocial and self-defeating without
one), a single connection per pair, and no fairness story. Before
further custom-UDP investment, the alternatives get benchmarked on
the same rig: kernel TCP with BBR, tuned CUBIC, parallel TCP
streams, and a QUIC tunnel. The custom transport earns its place
only if it beats those under the controlled matrix (bandwidth
1/10/25 Gbps where infrastructure permits; RTT 25/50/100/200 ms;
loss 0/0.01/0.1/1%, random and burst; competing traffic; concurrent
transfers; full and sparse range reads; goodput, completion time,
TTFB, p95/p99, retransmitted bytes, CPU, memory, fairness,
replication backlog recovery, integrity). Caching is disabled for
transport comparisons; caching benefits are benchmarked separately.

If the transport earns its place, the hardening list is: adaptive
RTT estimation and RTO, pacing, congestion-aware flow control,
BDP-aware windows, selective retransmission, efficient
acknowledgments, burst-loss recovery, fairness to competing traffic,
batching, concurrent connections, bounded memory, backpressure, and
connection reuse with cancellation/recovery.

## 6. Caching

There is no gateway cache in the RGW path: replication is
transport-only by default, and RGW's own caches (D4N, cloud restore)
keep their semantics with the transport underneath. The standalone
repository's caching gateway (built for the original prototype) is a
separate component for consumers that want it, with a known defect
to fix before any reliance: chunk fetches are not pinned to the
version the initial STAT returned, so an object changing between
STAT and FETCH could assemble chunks from two versions. The
acceptance bar for that component: immutable version pinning with
conditional chunk fetches and abort-on-mismatch, collision-resistant
cache identities, tenant/bucket isolation, request coalescing,
atomic publication, bounded eviction, per-chunk integrity.

## 7. Security

Current state, plainly: the prototype gateway and tunnel run
plaintext with no authentication — acceptable between two benchmark
hosts, disqualifying for production traffic. Requirements before any
production use:

- Mutual authentication between gateway pairs, with encryption and
  integrity on the WAN leg and replay protection (the design intent
  is an mTLS control plane with per-transfer AEAD keys).
- Under the tunnel model, RGW's own SigV4 (and TLS, if configured)
  stays end to end; the gateway holds no RGW credentials and the
  multisite system key never leaves the RGWs. If a future gateway
  mode terminates application-layer security instead of carrying it,
  the added trust must be documented explicitly.
- Resource-consumption limits on the gateway itself (unauthenticated
  traffic must not be able to exhaust it).

## 8. Validation status

- The §3.2 routing was runtime-proven in its earlier form on a
  two-VM rig (two vstart clusters, real cross-cluster multisite,
  shaped 25 ms / 1 Gbps links): sync and fetches carried over the UDP
  tunnel, byte-exact at 256 MiB across the loss matrix. The
  refactored form was re-verified on the same rig on Nov 25, 2026:
  the new build (397/397 steps) read a period written by the old
  build without error; a fresh object's immediate GET on the
  secondary returned 404 (no on-demand fetch — §4's removal behaving
  as stock); 1 MiB and 32 MiB objects then synced byte-exact through
  the secondary's local gateway endpoint (log: "via this zone's
  local gateway"); with the gateway killed, the next objects still
  landed byte-exact via the classified fallback (NOTICE logged), and
  gateway sync resumed when the gateway returned.
- The §3.3 throttle fix is rig-proven as described.
- The §5 congestion objective is **not** established; §5's matrix is
  the gate for any such claim.
