# Sync Task Resilience

## Architecture

The sync task (`crates/notebook-sync/src/sync_task.rs`) is a background tokio
task that owns the socket connection to the daemon. It runs a biased `select!`
loop with this priority order:

1. **Frame** (incoming daemon frames) — highest priority, keeps the socket drained
2. **Changed** (local mutations via `DocHandle::with_doc`) — generates outbound sync
3. **Command** (requests, confirm_sync) — daemon RPC and sync confirmation
4. **Maintenance** (50ms tick) — in-flight watchdog, confirm_sync retries

Document mutations do NOT go through the sync task. Callers mutate directly via
`DocHandle::with_doc` (acquires `Arc<Mutex<SharedDocState>>`). The sync task
only handles network synchronization.

## Automerge Panic Recovery

### The Problem

automerge 0.7 has a known panic in `BatchApply::apply` when the internal patch
log actor table gets out of order during concurrent sync
(automerge/automerge#1187). Without protection, the panic poisons the
`std::sync::Mutex` wrapping `SharedDocState`, rendering the session permanently
unusable.

### The Pattern

Both `AutomergeSync` (frame `0x00`) and `RuntimeStateSync` (frame `0x05`)
handlers use the same catch_unwind pattern:

```rust
let mut state = self.io.doc.lock().unwrap_or_else(|e| e.into_inner());

// Step 1: catch_unwind around receive_sync_message
let recv_result = std::panic::catch_unwind(AssertUnwindSafe(|| {
    state.receive_sync_message(msg)   // or receive_state_sync_message
}));
match recv_result {
    Ok(Ok(())) => {}                  // applied successfully
    Ok(Err(e)) => { warn!(...); return; }  // automerge error, skip
    Err(panic_payload) => {
        warn!(...);
        rebuild_shared_doc_state(&mut state);  // or state.rebuild_state_doc()
        return;
    }
}

// Step 2: catch_unwind around generate_sync_message (can also panic)
match std::panic::catch_unwind(AssertUnwindSafe(|| {
    state.generate_sync_message().map(|msg| msg.encode())
})) {
    Ok(bytes) => bytes,
    Err(_) => {
        warn!(...);
        rebuild_shared_doc_state(&mut state);  // or state.rebuild_state_doc()
        None
    }
}
```

### Rebuild Functions

**`rebuild_shared_doc_state(state)`** — for the notebook document:
1. Save the doc to bytes via `state.doc.save()`
2. Load a fresh `AutoCommit` from those bytes
3. Cell-count guard: if the rebuilt doc has fewer cells, skip the rebuild
   (only reset sync state) to prevent silent cell loss
4. Preserve the actor ID
5. Reset `state.peer_state = sync::State::new()` to force a fresh sync handshake

**`state.rebuild_state_doc()`** — for the RuntimeStateDoc:
1. Round-trip save/load via `state.state_doc.rebuild_from_save()`
2. Reset `state.state_peer_state = sync::State::new()`

Both follow the automerge-protocol skill's principle: "reset transport state,
preserve document truth." The sync state reset forces a fresh handshake, which
will reconcile any divergence.

### Why Both Handlers Need Protection

The notebook doc handler had catch_unwind from the start. The RuntimeStateDoc
handler was added later (PR #2454) after the breaker gremlin demonstrated that
concurrent `interrupt_kernel` + `create_cell` could trigger a panic in
RuntimeStateSync processing, killing the sync task and losing pending cell
mutations.

The key insight: any code path that calls automerge's `receive_sync_message` or
`generate_sync_message` on a document that receives concurrent sync frames is
vulnerable to the automerge#1187 panic. Both the notebook doc and the state doc
are synced concurrently.

## Mutex Handling

The sync task uses `std::sync::Mutex` (not `tokio::sync::Mutex`) for
`SharedDocState`. This is correct because:

1. The mutex is never held across `.await` points
2. `unwrap_or_else(|e| e.into_inner())` recovers from poisoned mutexes
   (which happen when catch_unwind catches a panic inside a lock scope)
3. The lock scope is always a block `{ ... }` — never leaked into async code

## Confirm Sync

`confirm_sync` is waiter-based, not blocking:

1. Caller captures current heads via `DocHandle::confirm_sync()`
2. A `ConfirmSync` command is sent to the sync task with target heads
3. The sync task registers the target heads as a waiter
4. Normal inbound `AutomergeSync` handling checks waiters after each receive
5. When `shared_heads` include all target heads, the waiter resolves
6. Timeout: 10s total, with 200ms retry ticks

This design avoids blocking the frame loop — the frame reader keeps draining
while the waiter resolves in the background.

## Common Mistakes

1. **Adding a recv loop inside a command handler.** The sync task must keep
   draining frames. Any per-command blocking starves broadcasts, state sync,
   and sync replies.

2. **Holding the mutex across I/O.** The lock scope must be a block that
   drops before any `.await`. The ack bytes are computed inside the lock,
   then sent outside it.

3. **Forgetting catch_unwind on new sync handlers.** If you add a new
   Automerge sync stream (e.g., PoolStateSync frame `0x06`), it needs the
   same catch_unwind + rebuild pattern.

4. **Assuming generate_sync_message always returns a message.** It returns
   None when there's an in-flight unacknowledged message (automerge's
   `in_flight` flag). Don't try to work around this.

5. **Skipping the cell-count guard in rebuild.** `save()` on a
   panic-corrupted doc may drop ops, producing fewer cells. The guard
   prevents silent cell loss by falling back to sync-state-only reset.
