# Benchmarks

**No results recorded yet.** This configuration has not been booted on the
reference box. This file holds the method; results go here once a guarded
sweep passes.

## Test machine

| | |
|---|---|
| GPUs | 4x NVIDIA RTX PRO 6000 Blackwell Max-Q Workstation Edition, 96 GB each, 300 W limit, PCIe (no NVLink) |
| Checkpoint | `Qwen/Qwen3.8-Flash-Next-FP8` |
| Engine | vLLM v0.33.0 (the digest pin in `compose.yaml`), bare upstream image, Rust frontend |
| Tool | llama-benchy, 3 runs per cell, under `scripts/bench-guard.sh` |

The same box runs the GLM-5.3-Flash recipe; the two cannot share the cards.

## Command

`./scripts/benchmark.sh <id>` runs:

```
llama-benchy \
  --base-url http://127.0.0.1:8000/v1 \
  --model Qwen/Qwen3.8-Flash-Next-FP8 \
  --served-model-name Qwen3.8-Flash-Next-FP8 \
  --pp 2048 --tg 128 --depth 0 8192 32768 131072 --concurrency 1 4 --runs 3 \
  --latency-mode generation --enable-prefix-caching \
  --format md
```

This is the sweep the GLM-5.3-Flash and single-GPU Qwen3.8 recipes report,
so results compare cell for cell. Reading the labels: `pp` is prefill, `tg`
is token generation, `d<N>` is the context depth, `c1`/`c4` is concurrency;
`ctx_pp` / `ctx_tg` are the context-loading phase.

The vLLM recipe advises benchmarking MTP with SPEED-Bench
(`vllm bench serve --dataset-name speed_bench`) rather than random tokens:
draft acceptance, and so decode speed, depends on how predictable the text
is. Treat llama-benchy decode numbers as a lower bound on what MTP gives
real text, and read acceptance from the dashboard's speculative-decoding row
while the sweep runs.

## Things to record from the first boot

- Warm and cold boot time to `/health`.
- `GPU KV cache size` and the max-concurrency line (and set the dashboard's
  `kv_pool_tokens` from it). Four 131,072-token requests need 524,288
  tokens; whether the deep c4 cells preempt depends on this number.
- Host RSS of the engine container with the N-gram table offloaded.
- Whether the Rust frontend's `/metrics` fills every dashboard panel, and
  what its access log looks like (README "Rust frontend").

## Benchmark isolation

A sweep is only evidence if nothing else used the engine while it ran.
`scripts/bench-guard.sh` wraps the harness: it waits for the engine to go
idle, then aborts with exit 42 on any request from a client outside the
allowlist or on concurrency above the bench's own maximum. Preemptions are
logged as a note.

The foreign-client check parses uvicorn-style access-log lines. If none
match during the run — possible under the Rust frontend — the verdict is
`PASS-BLIND`: only the concurrency check guarded it. Do not record a
`PASS-BLIND` sweep as evidence without saying so.

If the engine also serves agents, run the sweep only when none of them are
active. A concurrency-1 cell with a standard deviation several times its
usual size is contention until proven otherwise: look at
`vllm:num_requests_running` and the access log before looking at the build.
