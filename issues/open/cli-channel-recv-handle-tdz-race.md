# cli/channel: `tailUntil` can hit a TDZ ReferenceError and leak the watcher when a message lands during the watcher's synchronous first poll

## What's wrong

`tailUntil` (`cli/src/channel-recv.mjs:69-86`) creates the resilient watcher, whose `onMessages` callback calls `finish()`, which in turn calls `handle.close()`:

```js
const timer = setTimeout(finish, waitMs)
const handle = createResilientWatcher({
  ownName, persistent: false,
  onMessages: (msgs) => { printMessages(msgs); finish() },   // finish() → handle.close()
})
```

`createResilientWatcher` runs `pollOnce()` **synchronously** during construction (`channel-recv.mjs:168`). If `log.jsonl` already has unread messages at that instant, the synchronous `drain()` → `onMessages` → `finish()` runs *before* `const handle = ...` has finished assigning. `finish` then references `handle` while it is still in its temporal dead zone (or `undefined`).

## Why it matters

- `finish` throws `ReferenceError: Cannot access 'handle' before initialization`, the promise never resolves, and the watcher's interval / file watcher leak (never cleaned up).
- In `channel-recv.mjs` the cursor is usually current (it drains via `readSinceCursor` at `:22` and only calls `tailUntil` when there were no pending messages), so the window is narrow — but any message that lands **between** that drain and the watcher's synchronous first poll triggers it.

## What to do

- Declare `handle` (e.g. `let handle`) and create the watcher such that `finish` can safely reference it, or null-check `handle` inside `finish` (`if (handle) handle.close()`).
- Alternatively, defer the watcher's first `pollOnce()` to a microtask/next tick so construction completes before any synchronous callback fires.

## Acceptance

- A message landing during the watcher's first synchronous poll resolves `tailUntil` cleanly (prints the message, closes the watcher) with no `ReferenceError` and no leaked interval/watcher.

## Provenance

Found during a full line-by-line audit. Verified the synchronous `pollOnce()` in `createResilientWatcher` (`channel-recv.mjs:168`) firing `onMessages` → `finish` → `handle.close()` before `handle` is assigned (`:69-86`).
