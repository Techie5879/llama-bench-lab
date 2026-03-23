# llama-server: usage and saturation

How to inspect a running **llama-server** (from [llama.cpp](https://github.com/ggml-org/llama.cpp) `tools/server`) for capacity, load, and backlog. Paths below assume default **`api_prefix`** (empty) and listen **`127.0.0.1:8080`**. If you use `--api-prefix`, prepend it to every path.

## Quick checks (same origin as OpenAI API)

OpenAI-style chat uses `http://127.0.0.1:8080/v1/...`. Operational endpoints below are on the **same host and port**, without `/v1`:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Liveness: `{"status":"ok"}` |
| `/v1/health` | GET | Same as `/health` |
| `/props` | GET | Server config: `total_slots`, `default_generation_settings.n_ctx`, model path, flags |
| `/slots` | GET | Per-slot JSON: `id`, `n_ctx`, `is_processing`, task params, `next_token` / decode state |
| `/metrics` | GET | Prometheus text (only if server started with **`--metrics`**) |

Registration and handlers live under `llama.cpp/tools/server/server.cpp` and `server-context.cpp`.

## Saturation (what to look at)

- **Parallel capacity:** `GET /props` → **`total_slots`**. This matches **`--parallel` / `n_parallel`**: each in-flight completion uses a slot (see `server-context.cpp` slot loop).
- **Busy now:** `GET /slots` → each element **`is_processing`**. Count how many are `true`. If that count equals `total_slots`, all slots are busy.
- **Backlog:** When no slot is free, new work is **deferred** (`queue_tasks_deferred` in `server-queue.h`). That depth is exposed as:
  - Prometheus gauge **`llamacpp:requests_deferred`**
  - Internal metrics JSON (used when building `/metrics`) also tracks **`processing`** vs idle; the gauge **`llamacpp:requests_processing`** is the number of slots currently processing.

**Rule of thumb:** `requests_deferred > 0` means you are **over** parallel capacity: clients are queuing. `requests_processing` at `total_slots` with `requests_deferred == 0` means fully utilized but not accumulating backlog.

## Prometheus `/metrics` (requires `--metrics`)

Default in `common/common.h` is `endpoint_metrics = false`. Start the server with **`--metrics`** (or env `LLAMA_ARG_ENDPOINT_METRICS` per `common/arg.cpp`) or the handler returns a not-supported error.

Exposed series (prefix **`llamacpp:`**), from `get_metrics` in `server-context.cpp`:

| Metric | Type | Meaning |
|--------|------|---------|
| `prompt_tokens_total` | counter | Prompt tokens processed (cumulative) |
| `prompt_seconds_total` | counter | Prompt processing time (cumulative) |
| `tokens_predicted_total` | counter | Generated tokens (cumulative) |
| `tokens_predicted_seconds_total` | counter | Generation time (cumulative) |
| `n_decode_total` | counter | `llama_decode()` calls |
| `n_tokens_max` | counter | High watermark prompt/context size observed |
| `n_busy_slots_per_decode` | counter | Average busy slots per decode (sampled) |
| `prompt_tokens_seconds` | gauge | Recent prompt throughput (tokens/s) |
| `predicted_tokens_seconds` | gauge | Recent generation throughput (tokens/s) |
| `requests_processing` | gauge | Slots currently processing |
| `requests_deferred` | gauge | Deferred (queued) requests |

Example:

```bash
curl -sS http://127.0.0.1:8080/metrics
```

Some older docs mention `kv_cache_usage_ratio` / `kv_cache_tokens`; your build may or may not include them in `all_metrics_def`—rely on the names actually returned by `/metrics`.

## `/props` fields useful for capacity

- **`total_slots`**: max concurrent completion streams.
- **`default_generation_settings.n_ctx`**: per-slot context size (runtime KV budget for that slot).
- **`endpoint_metrics`**, **`endpoint_slots`**: whether `/metrics` and `/slots` are enabled.

## `/slots` GET

Returns a **JSON array** of slot objects (`slot.to_json(...)`). Useful fields:

- **`is_processing`**: slot busy.
- **`id`**, **`n_ctx`**
- **`id_task`**, **`params`**, **`next_token`**: last or current task state (even when idle, stale task metadata can remain until cleared—use **`is_processing`** for “busy now”).

Optional query **`fail_on_no_slot`**: if set and no idle slot, server returns an error (see `server-context.cpp` `get_slots`).

## Auth

If the server uses `--api-key`, non-public routes need `Authorization: Bearer <key>` or `X-Api-Key`. Public set includes `/health`, `/v1/health`, `/models`, `/v1/models`, `/api/tags` (see `server-http.cpp`); **`/props`**, **`/slots`**, and **`/metrics`** are not in that public list, so they typically require a key when auth is enabled.

## curlReddit default base URL

This app uses `DEFAULT_LLAMA_SERVER_BASE_URL` = `http://127.0.0.1:8080/v1` for chat. Strip the **`/v1`** suffix for the operational URLs above (e.g. `http://127.0.0.1:8080/props`).

## See also

- [llama-server-m4-benchmarks.md](./llama-server-m4-benchmarks.md) — local `llama-bench` / `llama-batched-bench` throughput (one M4 Max machine).
- [llama-server-n_ctx-and-parallel-slots.md](./llama-server-n_ctx-and-parallel-slots.md) — how total context and parallel slots interact in llama.cpp.
