# cli/channel: long-poll `/recv` advances `latest_seq` past messages it never delivered → silent message loss under concurrent senders

## What's wrong

The `/recv` long-poll handler (`cli/src/channel-core.mjs:98`) resolves a waiter with whatever `wakeWaiters` handed it, then reports `latest_seq` as the *global* high-water mark:

```ts
const messages = await waitHandle.promise
return { messages, latest_seq: store.nextSeq - 1 }
```

But `wakeWaiters` (`cli/src/log-store.mjs:42-62`) resolves each waiter with a **single** message and then removes the waiter:

```ts
const drained = list.splice(0)
for (const r of drained) r.resolve([payload])   // one message, waiter removed
```

So when two messages arrive close together while a waiter is registered:

1. msg `N` arrives → `wakeWaiters` resolves the waiter with `[msg N]` and splices it out of the list.
2. msg `N+1` arrives → there is no waiter left to wake; nothing is delivered.
3. The awaited continuation runs and returns `{ messages: [msg N], latest_seq: nextSeq-1 }`, where `nextSeq-1 == N+1`.

The client receives `messages: [N]` but `latest_seq: N+1`. On its next poll it sends `since: N+1`, and `drain` skips seq `N+1` forever (`if (m.seq <= since) continue`).

Concurrent senders are normal: the SDK runs each `/send` (and `/recv`-with-message) handler as an independent async task per WS frame, not serialized.

## Why it matters

**Permanent, silent message loss** in a multi-participant chat under ordinary concurrency — exactly the failure the `since`/cursor design exists to prevent. A participant simply never sees message `N+1` and has no signal that anything was dropped. The agent (or human) acts on an incomplete conversation.

## What to do

The single-message `wakeWaiters` delivery is incompatible with returning `latest_seq = nextSeq-1`. Fix either side:

- After the waiter resolves, **re-`drain` from the caller's `since`** before returning, so `messages` covers everything up to `latest_seq`; or
- Set `latest_seq` to the seq of the **last delivered** message (so the next `since` doesn't skip the gap); or
- Have `wakeWaiters` deliver *all* messages accumulated since the waiter registered, not just the one that woke it.

## Acceptance

- Two `/send`s landing in the same tick while a peer is long-polling result in the peer eventually receiving **both** (across at most one extra poll), with no seq skipped.
- A regression harness scenario reproduces concurrent senders and asserts no gap between delivered messages and the advertised `latest_seq`.

## Provenance

Found during a full line-by-line audit and confirmed against the source: `channel-core.mjs:98` (`latest_seq: store.nextSeq - 1` on the long-poll path) combined with `log-store.mjs:54` (`r.resolve([payload])`, single-message, waiter spliced out). The audit agent reproduced the lost-seq outcome directly.
