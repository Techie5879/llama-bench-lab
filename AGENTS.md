# Agents

## Scope

This repo is for benchmarking various small experiments with local LLMs to find the optimal params. Local MLX benchmark notes and comparisons against **llama.cpp** (Git submodule at `llama.cpp/`).

## Hardware (local)

| Item | Detail |
|------|--------|
| Machine | MacBook Pro (Mac16,5) |
| SoC | Apple M4 Max |
| CPU | 16 cores (12 performance, 4 efficiency) |
| GPU | 40 cores, Metal 4 |
| Memory | 64 GB unified |
| OS | macOS 26.3.1 (build 25D2128) |

- llama.cpp is locally installed via Homebrew

## References

- [docs/](docs/) — Markdown notes for bench and llama-server behavior in this repo.
- [docs/llama-server-usage-and-saturation.md](docs/llama-server-usage-and-saturation.md) — Operational endpoints, slot saturation, deferred backlog, and Prometheus `/metrics`.
- [docs/llama-server-n_ctx-and-parallel-slots.md](docs/llama-server-n_ctx-and-parallel-slots.md) — How `n_ctx`, parallel slots (`-np`), and KV memory interact in llama-server.


# How to work in Current repo
- First step before any task: determine whether you are on macOS (Metal) or Linux (CUDA) — this dictates available backends, flags, and tooling.

Treat `llama.cpp/` as upstream: do not edit it for one-off experiments; bump the submodule commit when you intentionally pin a new version.

- llama.cpp/docs is a useful resource
- llama.cpp source code in general is useful and should be explored when in doubt to get concrete information. THIS IS IMPORTANT.
    - DO NOT guess key names, param names, unless you are very sure that they are exactly as you understood/documented in llama.cpp
    - DO NOT MAKE UP PARAM/KEY/ARGUMENT names. Search and explore with subagents when in doubt to fetch exact names.
- You should also be able to spin up llama-server and llama-cli or llama-bench as required for your experiments.
- Always clean up all severs/processes spun up after done
- Logging is a first class citizen of the repo. Always log raw responses, parsed responses, and performance metrics.
- Always run benchmarking scripts/long running scripts with `tee` so that logs can be preserved and inspected later on.
- Any new information that you had to explore for in llama.cpp git submodule - that you think is useful/explicitly told by the user as useful - needs to be documented in docs/ and the references here updated for next time easy use and acces.
- NO FALLBACK behavior. No catching of errors, no silencing of errors. When porting anything to something else/new data format/breaking changes -- DO NOT try to fallback to old behavior/keeping legacy support.
- If you really think something needs to be fallen back to/you want to try to catch something and fallback -- ask the user using the `question` tool. DO NOT fallback. Raising full errors and stack trace is much better than falling back.
- Raising errors at EXACTLY where they occur is very important. Exception handling is not needed.
- This is because silent behaviors are not good.
- Use llama.cpp help commands in shell to understand what params to start/use servers with/benches with/to find supported commands.