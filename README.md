# llama-bench-lab

Local [MLX](https://github.com/ml-explore/mlx) benchmarks and notes. All numbers are for the hardware below.

## Scope

- **GGUF / llama.cpp** — [llama.cpp](https://github.com/ggerganov/llama.cpp) is a submodule (`llama.cpp/`). Compared against MLX when both runs use the same model class and comparable quants.
- **Metrics** — Tokens/s, time-to-first-token, peak RAM; GGUF and MLX each get their own column or table.
- **Throughput** — Decode tokens/s and prompt-side throughput; knob settings are recorded per run so sweeps are reproducible.
- **Parameters** — Prompt length, batch size, and thread counts where the tool exposes them.
- **Concurrency** — For `llama-server`, concurrent clients and in-flight requests; aggregate throughput and per-request latency on the same rows.
- **Quantization** — Bit width, quant family (e.g. K-quants), decode vs prompt vs quality; each row states the exact GGUF type or MLX quant name.
- **llama.cpp tooling**
  - `llama-bench` — local engine microbenchmarks over a prompt/batch grid.
  - `llama-server` — HTTP API; batching, concurrent clients, KV cache under load.
  - `llama` CLI — single-process runs; interactive and small scripted workloads.

## Hardware (local)

| Item | Detail |
|------|--------|
| Machine | MacBook Pro (Mac16,5) |
| SoC | Apple M4 Max |
| CPU | 16 cores (12 performance, 4 efficiency) |
| GPU | 40 cores, Metal 4 |
| Memory | 64 GB unified |
| OS | macOS 26.3.1 (build 25D2128) |
