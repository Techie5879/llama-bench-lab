# llama-bench-lab

Practical benchmarks for running LLMs locally and on cloud GPUs. The point is to answer real questions about inference that docs and model cards don't — with actual numbers on actual hardware.

Benchmarks span different runtimes, backends, and hardware — Apple Silicon, NVIDIA CUDA, llama.cpp, MLX, vLLM, and whatever else is worth testing.

## Questions we're trying to answer

- **Offline CLI vs. server under concurrent load** — How does throughput degrade when you go from a single-request run to a server handling 4, 8, 16 concurrent requests? Where does it fall off a cliff?
- **KV cache and context length** — What's the real memory cost of scaling context from 4K to 32K to 128K? At what point does the KV cache blow past available memory and tank performance?
- **Prompt caching** — How much does prompt caching actually save on repeated/similar prompts? What's the warm-vs-cold latency gap in practice?
- **Quantization: speed vs. intelligence** — Q4_K_M vs. Q5_K_M vs. Q6_K vs. Q8_0 — what do you actually lose in output quality, and what do you gain in tok/s and memory headroom? Is Q4 good enough, or is Q6 the sweet spot?
- **Batch size and threading** — How do batch size, thread count, and GPU offload layers interact? What combination saturates memory bandwidth on a given setup?
- **Model size scaling** — 3B vs. 7B vs. 14B vs. 32B vs. 70B — where's the practical ceiling for interactive use at a given VRAM budget? Where does time-to-first-token become painful?
- **Runtime and backend comparisons** — Same model, same quant, different runtime or backend. What's actually faster, and under what conditions? Where does one pull ahead of another?
- **Slot saturation and backpressure** — How does parallel slot count affect per-request latency under load? What happens when all slots are full and requests start queuing?
