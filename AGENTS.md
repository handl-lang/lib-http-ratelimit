# lib-http-ratelimit

`http-ratelimit` — rate limiting middleware for Handl HTTP servers, in
pure handl. Implements the `lib-http-middleware` onion protocol:
token-bucket and fixed-window algorithms, per-key isolation, `429` +
`Retry-After` + `X-RateLimit-*` headers, injectable clock and
`{get,set}` store port.

## Layout

| path | role |
| --- | --- |
| `src/index.hnl` | the middleware — `(ctx, next) ~> { resp, state }` |
| `src/index.test.hnl` | tests |
| `pkg.hnl` | manifest (deps via host chain) |
| `scripts/hnl-gate.sh`, `ci.sh` | gate + bupha entry |
| `workflows/gate.yaml` | CI — shared `handl-gates` campaign |

## Contract

- Protocol: `ctx := { req, state, ... }`, `next := (ctx) ~> { resp,
  state }` — same as `lib-http-middleware`; clock and store are
  injected so tests stay pure.
- Pure `.hnl`, no Rust crate.

## Verification

```bash
HNL=../host-cli/target/debug/hnl scripts/hnl-gate.sh
```

<!-- myco:agent-awareness -->

## Agent awareness

- Session start / any work discussion: run `myco status --prompt`
  before proposing next steps — know the board first.
- User describes new work → offer to capture it (`myco todo add`,
  `issue new`, `proposal new`). Don't wait to be asked.
- Picking up a task: `myco resume <ID>` is the briefing.
- Wrapping up: `myco handoff <ID>` anything resumable; keep statuses
  honest (claimed → in-progress → blocked/done).

<!-- /myco:agent-awareness -->
