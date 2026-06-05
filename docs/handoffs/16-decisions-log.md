# Decision log — transport-agnostic `runtime_agent` (#16)

A running trail of non-obvious calls so any session can take the work back and
understand *why*. One entry per decision: what, the alternative, why. Append as you go;
commit alongside the code that realizes each decision. Newest at the bottom.

## Seeded from the design session (2026-06-05)

1. **Make `runtime_agent` transport-agnostic; do not reimplement the kernel drive.**
   Alternative: ship the spike's `kernel_host.rs` (a working standalone runtime_peer
   that launches its own kernel). Why: that's a second kernel driver to untangle later,
   and it bypasses the daemon's env pools / launcher cache / supervision. Reusing the
   daemon and swapping only the sync transport is the smaller, more correct surface.

2. **The daemon stays the kernel manager; the runtime is a peer of the cloud *room*,
   not an unmanaged process.** Why: "peer" is about the transport (how it reaches the
   cloud), not the absence of a supervisor. Lifecycle events (death, hang, error) flow
   through the daemon's existing `handle_lifecycle_signal` path; a bare peer with no
   manager misses them.

3. **`FrameTransport` trait + UDS impl live in `notebook-protocol`; the cloud-WS impl
   lives in a separate lib crate.** Alternative: put the WS impl in `notebook-protocol`.
   Why: keeps `notebook-protocol` tungstenite-free and wasm-safe, and avoids the daemon
   depending on a binary. The trait belongs next to the framing it abstracts.

4. **Phase 1 is behavior-preserving (UDS impl only), gated by `cargo test -p runtimed`.**
   Why: de-risk the extraction with zero functional change before any cloud code touches
   the load-bearing daemon agent. The daemon tests are the contract.

5. **The kernel-host spike is a closed reference PR (#3408), not merged.** Why: it
   proved the cloud wire + the full cross-machine lifecycle end to end, but it
   duplicates the daemon's kernel drive. Closed-but-linkable preserves the record
   without entrenching the duplicate on `main`.

6. **Consumer-side RuntimeStateDoc receive uses `receive_sync_message_with_changes`, not
   `receive_sync_message`.** Why: the plain receive is daemon-authoritative and *strips
   incoming changes*; a cloud peer is a consumer of the room's authoritative state, so
   stripping silently discards the room's queued executions and stalls convergence. This
   cost hours to diagnose in the spike — carry it forward.

7. **A lifecycle safety net is required before relying on cloud hosting (Phase 3).**
   Why: `kernel.lifecycle` is `runtime_peer`-only-writable (`policy.rs:403-405`), so when
   the runtime itself vanishes no surviving room participant can correct the doc and the
   room has no watchdog — a dropped workstation strands the room with a phantom-live
   kernel. Needs a cloud-room watchdog + a narrow policy relaxation (or a `Disconnected`
   lifecycle the room can stamp). See `16-lifecycle-analysis.md`.

8. **Output path: plain nbformat manifest + a minted `output_id` is sufficient.**
   Verified live: it persists across peer disconnect and renders in the cloud viewer
   without the daemon's richer `OutputManifest`/blob-store shape. Don't over-build the
   output side.

9. **A cloud `runtime_peer` needs an explicit `runtime_peer` ACL row** (owner alone is
   403; `aclRowsCoverScope` special-cases the scope). Grant via
   `POST /api/n/:id/acl {subject_kind:"principal", subject, scope:"runtime_peer"}`.

10. **Stack one branch/PR per phase; this log + PR STATUS are the trail.** Why: headless
    with no reviewer between phases, stacking keeps each phase independently reviewable
    and lets the takeback session merge/rebase in order.

## Appended by subsequent sessions

<!-- Add entries here as you make decisions. Format: N. **Decision.** Alternative. Why. -->

### Phase 1 session (2026-06-05, lab2, branch `quod/16-frame-transport`)

11. **Split the transport into `FrameSource` (recv) + `FrameSink` (send) halves, plus a
    `FrameTransport` connector that yields the pair — not the single
    `recv_frame`/`send_frame` object the handoff sketched.** Alternative: one
    `trait FrameTransport { recv_frame(&mut self); send_frame(&mut self); }` held in one
    variable. Why: the agent's `tokio::select!` awaits the recv future in one arm
    (borrowing the read half for the whole `select!`) while other arms call send in their
    bodies. A single `&mut self` object makes those two borrows conflict; the existing code
    only compiles because `framed_reader` and `writer` are *separate* variables. Keeping
    two halves preserves that structure exactly and is the minimal, behavior-preserving
    shape. The connector (`connect() -> (Source, Sink)`) owns the transport-specific
    dial+handshake, which is what `reconnect_with_backoff` needs.

12. **Traits use `async fn` in trait consumed through generics, not `#[async_trait]` or
    `Box<dyn>`.** Alternative: `Box<dyn FrameTransport>` for runtime polymorphism. Why:
    matches the neighbouring `KernelConnection` pattern in `runtimed`, keeps
    `notebook-protocol` free of an `async-trait` dependency (stays wasm-safe and
    dependency-light per decision #3), and the agent has exactly one transport per process
    so monomorphisation at the single call site is free. The cloud transport (Phase 2) is a
    second impl selected at construction, not at runtime per-call.

13. **`UdsFrameTransport::connect` normalises the `send_json_frame` anyhow error to
    `io::Error::other`.** Alternative: make the trait's `connect` return `anyhow::Error`.
    Why: keeps the whole transport surface io-typed (`recv_frame`/`send_frame` are already
    `io::Result`), and the only error source is the effectively-impossible serialization
    failure of a `Handshake`. The message text is preserved; the sole caller
    (`main.rs`) only Displays it. Verified non-lossy by adversarial review.

14. **`FramedReader` capacity (16) hoisted to a named const `FRAME_READER_CAPACITY` in the
    transport module.** Why: the value was duplicated as a literal at the initial-connect
    and reconnect sites in `runtime_agent`; centralising it in the one place that now spawns
    the reader removes the duplication without changing the value.

Phase 1 verification (the contract): `cargo test -p runtimed` → 944 passed, 0 failed
(incl. `tokio_mutex_lint` and `tokio_select_cancel_safe` CI lints); `cargo clippy -p
runtimed --all-targets` and `-p notebook-protocol --all-targets` clean; `cargo build
--workspace` clean; `cargo fmt --check` clean. Net diff: +48/-97 in runtime_agent +
new transport.rs. Adversarial subagent review found zero behavioral differences.

Note for Phase 2: `/tmp/stage-oidc.txt` is **NOT present on lab2** (an earlier note in this
log claimed it was — that was a false positive from a shell check where `head` exits 0 on
empty stdin; corrected). So the live cross-machine re-proof is **not** runnable from this
host without creds being provided. preview.runt.run is reachable (HTTP 200), so only the
staging bearer is missing. `pi`/`opencode` CLIs are **not** installed here either — used a
spawned subagent for adversarial review instead.

15. **Push to fork `quillaid/desktop` and open the PR against the `nteract/nteract`
    handoff branch from there.** Alternative: push the branch directly to `origin`
    (nteract/nteract), as the prior `quod/*` branches were. Why: the `quillaid` git
    identity this run commits under has **no push access** to `nteract/nteract`
    (`{push:false, pull:true, triage:true}`); both HTTPS and SSH pushes 403. The existing
    `quod/*` branches were pushed by Kyle (repo owner), not reproducible here. GitHub's
    fork of `nteract/nteract` under this account already exists as `quillaid/desktop`
    (a rename of the fork; `push:true`), so the standard fork-PR flow is the only headless
    path. **Phase 1 PR: nteract/nteract#3409** (base `quod/runtime-agent-transport-handoff`,
    head `quillaid:quod/16-frame-transport`). A `fork` git remote
    (`git@github.com:quillaid/desktop.git`) is configured in the worktree for subsequent
    phase pushes. The takeback session (if it has direct push) may re-push these branches
    to `origin` and retarget the PRs; optimize for the reviewable trail, not remote
    identity. Stack Phase 2 on `quod/16-frame-transport` and PR it against this same branch
    or #3409's head.

### Phase 2 session (2026-06-05, lab2, branch `quod/16-cloud-transport`)

16. **`runt-cloud-peer` already exists on `main` (merged #3397) as the WS-sync *binary*,
    without kernel hosting.** The spike branch `quod/runtime-peer-kernel-host` (#3408) added
    `--host-kernel` + `kernel_host.rs` on top of it; that half stays retired per decision #1.
    Phase 2 lifts the merged binary's WS wire (dial + header auth + `cloud_room_ready` +
    one-frame-per-binary-message) into a **library**, `notebook-cloud-transport`, that
    implements the Phase 1 `FrameTransport` trait. The binary keeps working unchanged; the
    library is what the daemon's `runtime_agent` will write to.

17. **Phase 2 ships the cloud transport *library only*; the daemon spawn-path wiring moves
    to Phase 3.** Alternative (handoff's literal Phase 2): also add "a daemon path to spawn
    `runtime_agent` with the cloud transport." Why deferred: wiring the spawn path requires
    two agent-loop changes that are unsafe before Phase 3's fixes — (a) authoring the
    NotebookDoc/RuntimeStateDoc under the `cloud_room_ready` principal (not the daemon's
    `runtime_agent_id`), and (b) the consumer-side `receive_sync_message_with_changes`
    (decision #6) instead of the daemon's `receive_sync_and_foreign_comms_recovering`. Most
    critically, the handoff itself flags that a cloud-WS EOF currently falls into
    `kernel.shutdown()` (lifecycle-analysis req #1), so spawning the agent on the cloud
    transport *before* the EOF-policy-by-transport split would let a transient WS blip kill a
    healthy kernel. Shipping the transport alone keeps each PR independently safe and
    reviewable; Phase 3 adds the spawn path together with the EOF fix that makes it correct.
    A compile-time assertion (`cloud_transport_is_a_frame_transport`) proves the library is
    already drop-in for the agent's generic bound, so Phase 3 is purely additive.

18. **The cloud `connect()` reads up to `cloud_room_ready` and surfaces the room principal
    via an `OnceLock` getter, rather than widening the `FrameTransport` trait.** Alternative:
    add a `principal()`/`on_ready()` method to the trait. Why: the UDS transport has no such
    concept, and the Phase 1 trait PR (#3409) is open for review — widening it now would
    churn that PR. The principal is cloud-specific, so it lives on the concrete
    `CloudWsFrameTransport`. Data frames that arrive before `cloud_room_ready` are buffered in
    the source's `pending` queue and drained first, so the ready-wait loses no frames.

19. **Frame decode skips empty/unknown frame types (returns `None`, keeps reading) to mirror
    the UDS `FramedReader`/`recv_typed_frame` forward-compat behavior.** Why: a cloud room on
    a newer protocol may send frame types this peer doesn't know; dropping them silently (with
    a warn) matches the local path rather than erroring the stream.

20. **The connect-time ready-wait surfaces `cloud_frame_rejected` as a
    `PermissionDenied` connect error, and warns on a principal mismatch across reconnect.**
    These came from an adversarial review of the Phase 2 crate. The original loop silently
    ignored every non-`cloud_room_ready` control frame, so a room rejection delivered as a
    `cloud_frame_rejected` control frame (auth/ACL failure surfaced *after* a successful WS
    upgrade) would hang the connect until the socket closed, then return an opaque EOF —
    discarding the room's stated `reason`. The reference binary at least logs it. Now
    `classify_ready_control` returns `Ready(principal)` / `Rejected(reason)` / `Other`, and
    `connect_cloud` returns `Err(PermissionDenied: "room rejected attach before ready:
    <reason>")` on a rejection. Separately, the `OnceLock` principal cache now warns if a
    reconnect observes a *different* principal than the one the agent is authoring under
    (silent staleness would otherwise make the room drop all the agent's changes — the exact
    failure mode the module docs warn about). Pre-ready data frames are still buffered (not
    dropped like the reference) — judged strictly safer.

Phase 2 verification: `cargo test -p notebook-cloud-transport` → 11 passed, 0 failed;
`cargo clippy -p notebook-cloud-transport --all-targets` clean; `cargo test -p runtimed`
still 944 passed (no regression from the new workspace member). Adversarial subagent review
ran against the crate vs the `runt-cloud-peer` reference; its one BLOCKER (silent rejection
handling) is fixed per decision #20. Live cross-machine re-proof is deferred with the Phase 3
spawn path — there is no daemon path to *invoke* the cloud transport yet (decision #17), and
the staging bearer (`/tmp/stage-oidc.txt`) is **not** present on this host regardless.

Build-host note: `gh repo fork --clone` left a stray 139 MB `desktop/` clone inside the
worktree (the fork is named `quillaid/desktop`); it tripped `cargo xtask lint` (JS/TS
formatting over the nested checkout). Removed it. Future fork operations should use
`--clone=false` or clone outside the worktree.

### Phase 3a session (2026-06-05, lab2, branch `quod/16-lifecycle`)

21. **Phase 3 is split: 3a (transport-aware clean-EOF policy) lands now; 3b+ (spawn path,
    WS reconnect/re-auth specifics, cloud-room watchdog, policy relaxation, inbound request
    channel) are planned but not yet built.** Why: Phase 3 spans three codebases (Rust agent,
    TS Cloudflare worker, `runtime-doc` policy) and its safety-critical pieces (the DO
    watchdog, the policy relaxation) need integration/live verification that isn't fully
    headless. But lifecycle-analysis **req #1** — "a cloud-WS clean EOF must NOT fall into
    `kernel.shutdown()`" — is a clean, behavior-preserving, unit-testable Rust change and is
    the keystone that makes spawning the agent on the cloud transport *safe*. Landing it
    first de-risks 3b and keeps each PR independently reviewable.

22. **The clean-EOF teardown policy is a defaulted trait method
    `FrameTransport::clean_eof_is_recoverable()` (default `false`), overridden to `true` by
    the cloud transport.** Alternative: a runtime flag threaded through `run_runtime_agent`,
    or a per-call parameter. Why a defaulted trait method: the policy is an intrinsic property
    of the transport (the daemon socket's clean close means "daemon gone → tear down"; a
    cloud WS clean close means "blip/eviction → reconnect"), so it belongs on the transport.
    Default `false` keeps the UDS/desktop path byte-for-byte unchanged (verified: 944
    runtimed tests still green). The agent's `None` (clean-EOF) arm now consults it and, when
    recoverable, runs the *same* reconnect+resync dance as the existing framing-error (`Err`)
    arm — drop source, `reconnect_with_backoff`, reset `coordinator_sync_state`, kick
    `state_kick_tx`. This mirrors the deliberate "kernel stays running" policy the framing-
    error branch already applies (lifecycle-analysis: that branch is the correct model).

23. **The recoverable clean-EOF arm enforces a 1s reconnect floor
    (`CLEAN_EOF_RECONNECT_FLOOR`).** From an adversarial review of 3a: `reconnect_with_backoff`
    only sleeps between *failed* connects, so a cloud sink that accepts the connection and then
    immediately closes cleanly every cycle (a flapping/evicting room) would spin a reconnect
    storm at network-RTT rate (not a CPU busy-loop, but a self-inflicted DoS on the room). The
    `Some(Err)` arm has the same structural property, but clean-EOF-on-every-reconnect is a
    *more plausible* trigger for a cloud room than instant framing errors. The fix: track the
    last clean-EOF reconnect time and, if a clean EOF recurs within the floor, sleep the
    remainder before redialing. Only the recoverable (cloud) path uses it; the UDS path never
    reconnects on clean EOF, so its `last_clean_reconnect` stays `None`. The room-side watchdog
    (3d) and a fuller circuit-breaker remain future work, but this client-side floor is the
    prerequisite the review flagged for the 3c spawn path.

Phase 3a verification: `cargo test -p runtimed` → 944 passed, 0 failed (UDS default
unchanged); `cargo test -p notebook-protocol` → 89 passed; `cargo test -p
notebook-cloud-transport` → 12 passed; clippy clean across all three; `cargo fmt --check`
clean. The `tokio_mutex_lint` + `tokio_select_cancel_safe` CI lints still pass. Adversarial
subagent review of the EOF-policy change found no BLOCKERs; its one concern (reconnect-storm
on a flapping sink) is addressed by decision #23.

## Phase 3 remaining plan (3b+) — for the takeback session

Ordered by the lifecycle analysis. 3a (req #1) is **done**. Remaining, in dependency order:

- **3b — WS reconnect/re-auth + full-resync on cloud reconnect (req #2).** `reconnect_with_backoff`
  is already generic over `FrameTransport`, and the cloud `connect()` re-dials + re-auths +
  re-reads `cloud_room_ready`, so the *mechanism* exists. What's missing is verifying the
  resync kick (`state_kick_tx`) drives a full RuntimeStateDoc re-send after a cloud reconnect
  (it fires today on both reconnect arms — confirm it converges against a real room), and
  that re-auth uses a *fresh* token if the original expired (the `CloudAuth` is currently a
  static token; a token-provider closure may be needed for long-lived sessions). Unit-test
  the reconnect arm with a mock `FrameTransport` whose `connect` fails N times then succeeds.

- **3c — daemon spawn path + doc-actor identity + consumer-side receive (the integration).**
  Add a `run_runtime_agent`-equivalent (or a parameter) that builds a `CloudWsFrameTransport`
  instead of `UdsFrameTransport`. Two agent-loop changes gated on this path:
  (a) author the RuntimeStateDoc (and NotebookDoc if synced) under the
  `cloud_room_ready` principal (`transport.principal()`), as `<principal>/<operator>`, not the
  daemon's `runtime_agent_id` — else the room's `validate_room_notebook_change_actors` drops
  every change (decision #6, ADR "load-bearing findings"). (b) Apply incoming RuntimeStateSync
  with `receive_sync_message_with_changes` (consumer semantics) rather than the daemon's
  `receive_sync_and_foreign_comms_recovering` (which strips incoming changes). Gate (a)/(b) on
  a transport-kind discriminant so the UDS path is untouched. **This is where the live
  cross-machine re-proof runs** (requires a staging bearer — see creds gate below): spawn the
  daemon's real `runtime_agent` against a preview room as `runtime_peer` (needs the explicit
  `runtime_peer` ACL row, decision #9) and confirm a cloud-submitted cell runs on the
  daemon-managed kernel and renders in the viewer — through the *real* agent, not the spike.
  **Creds gate:** this needs a staging bearer at `/tmp/stage-oidc.txt`, which is **not**
  present on lab2 as of the Phase 3a session — provide it (or run on a host that has it) to
  do the live re-proof. preview.runt.run itself is reachable from lab2 (HTTP 200).

- **3d — cloud-room DurableObject watchdog (reqs #3, #7; the safety net the daemon can't
  provide).** TypeScript in `apps/notebook-cloud/`. Thread peer **scope** through
  `removePeer` → `room-materializer.removePeer` → `RoomHostHandle.remove_peer`
  (`notebook-room.ts:635`, `runtimed-wasm/src/lib.rs:523`), give `RoomHostHandle` a
  reconciliation mutator (it has none), and use a DO `alarm()` with a grace period to
  terminalize running/queued executions + flip lifecycle when a `runtime_peer` departs.
  Verify with `cd apps/notebook-cloud && node --import tsx --test test/*.test.ts`.

- **3e — policy relaxation (req #4; gates 3d being legal).** `crates/runtime-doc/src/policy.rs:403-405`
  blocks editor/owner from writing `state.kernel` ("daemon-owned"). The watchdog (room host)
  needs a *narrow* authority to terminalize lifecycle on `runtime_peer` departure. Recommended:
  both (a) a scoped relaxation for the lifecycle→terminal transition, and (b) a model-level
  `Disconnected` `RuntimeLifecycle` / `last_seen` on `KernelState`
  (`crates/runtime-doc/src/types.rs:272`) so viewers distinguish gone-but-recoverable from
  dead. Unit-test the policy in `runtime-doc`.

- **3f — inbound request channel (req #5) + terminal-delta buffering (req #6).** Route
  interrupt/restart `RuntimeAgentRequest`s to the cloud agent (hosted REQUEST dispatch), and
  don't `break` the agent loop on a single writer error (`runtime_agent.rs` outbound arm) —
  buffer + replay across a blip. 3f's req #6 is partly addressed by 3a (the loop no longer
  tears down on a clean close) but the writer-error `break` in the `state_changed_rx` arm
  remains.

Branch/PR chain so far (all stacked, all in fork `quillaid/desktop` except #3409):
`main` → `quod/runtime-agent-transport-handoff` (docs) → `quod/16-frame-transport` (Phase 1,
**nteract/nteract#3409**) → `quod/16-cloud-transport` (Phase 2, **quillaid/desktop#1**) →
`quod/16-lifecycle` (Phase 3a, PR pending).
