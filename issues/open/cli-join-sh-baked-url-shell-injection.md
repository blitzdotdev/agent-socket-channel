# cli/channel: served `join.sh` interpolates the baked URL into a bash string unescaped — command injection into joiners

## What's wrong

`renderJoinScript` (`cli/src/join-sh-template.mjs:7-8`) drops the baked URL straight into a double-quoted bash assignment with no shell escaping:

```js
const urlBlock = bakedUrl
  ? `URL="\${1:-${bakedUrl}}"`
  : `URL="\${1:-}"`
```

`bakedUrl` is `publicTokenUrl` (`cli/src/channel-host.mjs:128`), derived from the relay-returned `link.url` or the operator's `--public-base` value. The advertised way to join is:

```
bash <(curl -s URL/join.sh) "" "<your-name>"
```

Every friend runs the served script. If the baked value contains a `"` plus shell metacharacters, it breaks out of the string. For example a baked value of:

```
http://x/t/ABC"; touch /tmp/PWNED; echo "
```

produces `URL="${1:-http://x/t/ABC"; touch /tmp/PWNED; echo "}"`, which executes `touch /tmp/PWNED` on every joiner's machine.

## Why it matters

The served `join.sh` becomes a client-side RCE vector against everyone who joins via the one-liner. The injected value comes from the **relay** (`link.url`) or the **operator** (`--public-base`), not from an arbitrary peer — so this is distinct from the closed `channel-prompt-injection-threat-model` (peer-controlled content). It is a relay/operator-controlled code-injection path: a compromised, malicious, or simply mis-pointed relay (or an operator who pastes a hostile `--public-base`) turns "share this join command" into RCE on joiners.

In the normal case the real relay returns a token URL built from a fixed charset (Crockford base32 + base64url), which can't contain shell metacharacters — so this is latent under a trusted relay. It should still be hardened, because the whole point of the one-liner is that joiners run code from a URL they were handed.

## What to do

- Shell-escape the baked value before interpolation: single-quote it and escape embedded single quotes (`'\''`), instead of dropping it into a double-quoted string.
- Validate that `publicTokenUrl` / `--public-base` is a well-formed `http(s)` URL with no shell metacharacters before baking it in; reject otherwise.

## Acceptance

- A baked URL containing `"`, `;`, `$(...)`, or backticks is rendered inert in the served `join.sh` (no command execution on the joiner).
- The normal join flow (clean token URL, or a clean `--public-base`) still works.

## Provenance

Found during a full line-by-line audit. Verified the unescaped interpolation (`join-sh-template.mjs:8`) and the `publicTokenUrl` source (`channel-host.mjs:127-130`). Calibrated MEDIUM: real injection, but the exploit requires controlling the relay's returned URL or the `--public-base` value rather than just a peer name.
