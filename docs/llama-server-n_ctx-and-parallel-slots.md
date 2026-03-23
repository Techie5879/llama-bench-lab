# llama-server: `n_ctx`, parallel slots (`-np`), and KV memory

This note summarizes how **context length** and **parallel HTTP slots** interact in **llama.cpp** `llama-server`, based on the submodule at `llama.cpp/` (paths below are relative to that tree).

## Conclusion (short)

- **`llama-server` uses one `llama_context` for the process.** Each HTTP **slot** is a `server_slot` with the **same** `llama_context *`; slots differ by **sequence id** (`slot.id`), not by separate model loads.
- **`-np` / `--parallel` maps to `llama_context_params.n_seq_max`** (`common_params.n_parallel` in `common/common.cpp`). **`total_slots` in `GET /props` is that slot count.**
- **Whether “`-c` is divided by `np`” depends on `kv_unified`:**
  - **`kv_unified == false`:** the library sets **`n_ctx_seq = n_ctx / n_seq_max`** (with padding and rounding so **`n_ctx == n_ctx_seq * n_seq_max`**). Each parallel sequence gets a **partitioned** per-sequence context budget; KV uses **`n_stream == n_seq_max`** (separate streams per sequence).
  - **`kv_unified == true`:** **`n_ctx_seq = n_ctx`**. There is **no** equal split of the user’s `-c` across slots in that constructor branch. KV uses **`n_stream == 1`** (unified layout); sequences share one physical stream as implemented in `llama_kv_cache`.
- **`llama-server` “auto” parallel (`n_parallel < 0`) sets `n_parallel = 4` and forces `kv_unified = true`**, so the **split** `n_ctx / np` path is **not** used for that default mode.
- **`--fit` does not implement “divide VRAM by number of slots.”** It fits model/context params to device memory; it may change **total** `n_ctx` when the user left **`n_ctx == 0` (auto)**. Per-sequence splitting is still governed by **`kv_unified`** and `llama-context.cpp`, not by a separate “per slot” fit.

So: **context is not “one global `-c` that every slot independently consumes in full” in the sense of N unrelated copies of the model.** It is **one context** whose **per-sequence limit** (`llama_n_ctx_seq`) and **KV layout** (`unified` vs partitioned streams) follow the rules above. **`/props` / `/slots` `n_ctx` is `llama_n_ctx_seq` (per-slot sequence budget as exposed by the API), not a second independent dial per HTTP request.**

## Wiring: server slots to the library

`common_context_params_to_llama` passes:

- **`params.n_ctx` → `cparams.n_ctx`**
- **`params.n_parallel` → `cparams.n_seq_max`**
- **`params.kv_unified` → `cparams.kv_unified`**

See `common/common.cpp` (`common_context_params_to_llama`).

After the context exists, the server sets:

- **`n_ctx = llama_n_ctx(ctx)`** (total context on that context)
- **`n_ctx_slot = llama_n_ctx_seq(ctx)`**, capped by training context, assigned to **every** `server_slot` as `slot.n_ctx`

See `tools/server/server-context.cpp` in `load_model` (slot initialization loop).

## Core math: `n_ctx_seq` vs `kv_unified`

From `src/llama-context.cpp` (after padding `n_ctx`):

- If **`kv_unified`:** **`n_ctx_seq = n_ctx`**
- Else: **`n_ctx_seq = n_ctx / n_seq_max`** (padded), and **`n_ctx`** may be **rounded** to **`n_ctx_seq * n_seq_max`**

## KV cache: streams (partitioned vs unified)

`llama_kv_cache` sets **`n_stream = unified ? 1 : n_seq_max`** and sizes cell arrays per stream (`src/llama-kv-cache.cpp`). Non-unified mode maps sequence ids to distinct streams (`seq_to_stream` when `n_stream > 1`).

## llama-server default when `-np` is omitted

`tools/server/server.cpp`: if **`params.n_parallel < 0`**, log and set **`n_parallel = 4`** and **`kv_unified = true`**.

## `GET /props` and `GET /slots`

- **`total_slots`:** `params.n_parallel`
- **`default_generation_settings.n_ctx` / per-slot `n_ctx`:** derived from **`meta->slot_n_ctx`**, i.e. the **`llama_n_ctx_seq`**-based value described above (see `tools/server/server-context.cpp` and route handlers in `tools/server/server.cpp`).

**`endpoint_props` in JSON** reflects whether **POST `/props`** is enabled (`--props`); it is **not** “whether GET `/props` works.”

## Related doc

Operational saturation and metrics: [llama-server-usage-and-saturation.md](./llama-server-usage-and-saturation.md).

## Primary source files

| Topic | Path |
|--------|------|
| `n_ctx_seq` split vs unified | `src/llama-context.cpp` |
| KV `n_stream` | `src/llama-kv-cache.cpp` |
| `n_parallel` → `n_seq_max` | `common/common.cpp` |
| Auto parallel + `kv_unified` | `tools/server/server.cpp` |
| Slots, `n_ctx_slot`, one `ctx` | `tools/server/server-context.cpp` |
| Context params API | `include/llama.h` |
