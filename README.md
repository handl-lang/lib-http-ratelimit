# lib-http-ratelimit

`http-ratelimit` — rate limiting middleware for Handl HTTP servers,
in pure handl. It implements the `lib-http-middleware` onion
protocol exactly:

```text
ctx  := { req, state, ... }
next := (ctx) ~> { resp, state }
middleware := (ctx, next) ~> { resp, state }
```

so it composes with `lib-http-middleware`'s `use`/`compose` and with
any handler built on the same `{ resp, state }` convention
(`lib-http-route` routers included).

```handl
from "http-ratelimit" import { rate_limit }
from "http-middleware" import { compose, use } -- or wire manually

-- token bucket: 100 req/s sustained, bursts of 200
limiter := rate_limit({ limit: 100, window: 1000, burst: 200 })

-- fixed window: 60 requests per minute per client IP
windowed := rate_limit({ algorithm: "window",
                         limit: 60, window: 60 })
```

## Surface

- `rate_limit(opts?)` → `(ctx, next) ~> { resp, state }` middleware.
- `default_key(ctx)` → `String` — the default `key_fn`; takes the
  first `x-forwarded-for` hop, then `x-real-ip`, then `"unknown"`.

`opts` (all optional):

- `algorithm` — `"bucket"` (default, token bucket with `burst`
  capacity) or `"window"` (fixed window counter).
- `limit` — positive number; requests per `window` (bucket refill
  rate) or per fixed window (window counter).
- `window` — milliseconds as a number, or a duration record
  `{ ms }`/`{ seconds }`/`{ minutes }`.
- `burst` — bucket capacity, default `= limit`.
- `key_fn` — `(ctx) ~> String`; default `default_key`.
- `store` — `{ get: (k) ~> v, set: (k, v) ~> () }` port for external
  counter state, or `null` (default: counters live in
  `ctx.state[state_key]`).
- `now` — `() ~> ms` clock, default wall-clock `now_ms` (built-in
  `clock_gettime` via `ffi`); inject a fake for tests.
- `state_key` — `ctx.state` slot name, default `"rate_limit"`.

## Semantics

- **Allow path** — the wrapped `next(ctx)` runs; its response headers
  gain `x-ratelimit-limit`, `x-ratelimit-remaining`, and
  `x-ratelimit-reset` (epoch seconds) when the response is an object
  with a `headers` record.
- **Deny path** — no `next` call; returns
  `{ status: 429, headers: { retry-after, x-ratelimit-* } }` with
  `retry-after` in whole seconds (ceil).
- **State** — Handl has no mutable cells, so counters are explicit:
  with no `store`, the middleware reads/writes
  `ctx.state[state_key][key]` and returns the updated `state` merged
  right-wins over the incoming one (middleware protocol). With `store`,
  `store.get(key)`/`store.set(key, entry)` are used instead and
  `state` passes through unchanged.
- **Algorithms** — token bucket refills at `limit / window` tokens
  per ms up to `burst`, deducting 1 per allowed request; fixed window
  counts requests in `[window_start, window_start + window)` and
  resets when `now` crosses the boundary.
- **Keys** — per-key isolation is by whatever `key_fn` returns;
  default keys on client IP headers, so all clients share `"unknown"`
  unless the edge sets forwarding headers or `key_fn` is injected
  (e.g. from `req.params` for per-route limits).
- **Single-process** — state is in-process (`ctx.state` or an
  injected store). There is no shared store across workers; for
  multi-process deployments inject a `store` backed by a shared KV
  (the `{ get, set }` shape matches `lib-kv`/`lib-kv-map` ports).

## Layout

- `src/index.hnl` — helpers, `default_key`, clock, both evaluators,
  `rate_limit`.
- `src/index.test.hnl` — `hnl test` suite: allow→429, retry-after,
  per-key isolation, window reset, bucket burst/refill, store port,
  key_fn, header merge, algorithm validation.
- `pkg.hnl` — package manifest (`http-ratelimit`, no deps).
- `scripts/hnl-gate.sh`, `workflows/gate.yaml`, `ci.sh`,
  `bupha.toml`, `LICENSE` (MIT), `.gitignore`, `AGENTS.md`.

## Gate

```bash
HANDL_EXT_DIR=<path-to-dogfood-debug> \
  <path-to-dogfood>/hnl test src/index.test.hnl

HANDL_EXT_DIR=<path-to-dogfood-debug> bash scripts/hnl-gate.sh
```

## License

MIT — see `LICENSE`.
