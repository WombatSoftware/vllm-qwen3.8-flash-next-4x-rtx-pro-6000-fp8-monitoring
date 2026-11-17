# Benchmarks

**No throughput results recorded yet.**

Pin moved to v0.34.0 (2026-11-17), boot and function checks only, no sweep:

| | v0.34.0 | v0.33.0 |
|---|---|---|
| Boot to `/health`, weights cached | 210 s | 191 s |
| `GPU KV cache size` | 6,937,800 tokens (26.47x) | 7,077,888 (27.00x) |
| KV memory per card | 55.15 GiB | 53.79 GiB |
| Weights + non-torch per card | 34.07 GiB | 34.85 GiB |
| Engine container host RAM | 76.5 GiB (`docker stats`) | 83 GiB |

On v0.34.0, one request each at temperature 0 with thinking off (a list of
numbered records, "output the code of record N, then write an essay"):
2,855 / 89,414 / 223,286 / 261,200 prompt tokens gave MTP acceptance lengths
of 2.32 / 2.38 / 2.42 / 2.34 from `/metrics` counter deltas, and the right
record code every time. A prompt over 262,144 tokens is rejected with HTTP
400. Chat, the reasoning split, and tool calls (streaming and not) passed;
all 29 `vllm:*` metric names the dashboard queries are exported. Four
requests are not a benchmark.

The rest of this section is the v0.33.0 history. First boot on the reference box
(2026-11-10): healthy in 191 s with the weights pre-downloaded, KV pool
7,076,500 tokens (26.99x at 262,144), so four 131k requests (524,288 tokens)
fit in the pool by arithmetic (preemption not yet observed either way). The Rust frontend writes no access log, so the guard
can only report `PASS-BLIND`. Results go here once a sweep runs. The
2026-11-12 and 2026-11-13 boots of the same config report 7,077,888 tokens
(27.00x): the pool moves slightly between identical boots, so re-read the
boot line before setting `kv_pool_tokens`.

MTP-3 spot check (2026-11-10, one ~1,000-token prose request each, from
`/metrics` counter deltas): **1.82 accepted tokens per draft at temperature 0**
(mean acceptance length 2.82 including the bonus token; the single-GPU
NVFP4 recipe measured 2.695) and **1.61 at temperature 0.7**. Two requests,
not a benchmark. The boot log's `Qwen4ExpMTP does not support external
multimodal embeddings ... using text-only draft inputs` warning does not
show up as an acceptance loss on text. An earlier 0.88 reading came from a
handful of short creative requests right after boot and is not
representative.

## Test machine

| | |
|---|---|
| GPUs | 4x NVIDIA RTX PRO 6000 Blackwell Max-Q Workstation Edition, 96 GB each, 300 W limit, PCIe (no NVLink) |
| Checkpoint | `Qwen/Qwen3.8-Flash-Next-FP8` |
| Engine | vLLM v0.34.0 (the digest pin in `compose.yaml`), bare upstream image, Rust frontend |
| Tool | llama-benchy, 3 runs per cell, under `scripts/bench-guard.sh` |

The same box hosts the other engine recipe for these cards; the two cannot
share the cards.

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

This is the sweep the single-GPU Qwen3.8 recipe reports, so results compare
cell for cell. Reading the labels: `pp` is prefill, `tg`
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
- Host RSS of the engine container with the N-gram table offloaded —
  recorded 2026-11-14: 83 GiB for the engine container (host total 122 GiB).
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
