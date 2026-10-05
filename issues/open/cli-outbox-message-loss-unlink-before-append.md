# cli/channel: local `channel send` messages can be silently lost — outbox file unlinked before append, no temp-rename, no size check

## What's wrong

`channel send` (`cli/src/channel-send.mjs:32`) writes a message straight into the host's `outbox/` directory with `fs.writeFileSync` (no temp-file-then-rename). The host watches that dir and ingests on file creation (`cli/src/channel-host.mjs:87-101`):

```js
try { parsed = JSON.parse(fs.readFileSync(full, "utf8")) }
catch { try { fs.unlinkSync(full) } catch {}; continue }   // (a) partial read → drop
try { fs.unlinkSync(full) } catch {}                        // (b) unlink BEFORE append
if (...) {
  try { store.append({ from: parsed.name, text: parsed.text }) }
  catch (e) { console.error(...) }                          // (c) oversize → file already gone
}
```

Two loss paths:

- **(b)+(c) deterministic for oversize text:** the file is unlinked *before* `store.append`, and `channel send` does not cap text size. A message whose text exceeds `MAX_TEXT_BYTES` (64 KB) makes `append` throw (`log-store.mjs:23`); the throw is caught and logged, but the outbox file is already deleted, so the message is gone.
- **(a) TOCTOU on partial read:** because the writer doesn't write-to-temp-then-rename, a watch event can fire before the write is fully flushed; `JSON.parse` of a truncated file throws and the host unlinks-and-drops. Narrow on local Linux for small payloads, but the create-triggered-read of a non-atomically-written file is the classic race, and the catch-and-unlink masks it.

## Why it matters

Locally-sent messages disappear with no error visible to the sender (the sender's `channel send` already returned). For oversize text it is deterministic; for the partial-read race it is FS/timing-dependent but real for larger payloads.

## What to do

- `channel send` should write to a temp filename and `rename()` it into `outbox/` (rename is atomic on the same filesystem), so the host never reads a half-written file.
- The host should unlink the outbox file **only after** a successful `store.append`, not before.
- Validate text size in `channel send` before writing, returning a clear error to the sender, rather than letting the host silently drop oversize messages.

## Acceptance

- An oversize `channel send` returns an error to the sender and does not silently vanish.
- The host never reads a partially-written outbox file (temp-then-rename), and never deletes a message it failed to append.

## Provenance

Found during a full line-by-line audit. Verified the ingest order in `channel-host.mjs:95-101` (unlink precedes append), the direct `fs.writeFileSync` in `channel-send.mjs:32`, and the `MAX_TEXT_BYTES` throw in `log-store.mjs`.
