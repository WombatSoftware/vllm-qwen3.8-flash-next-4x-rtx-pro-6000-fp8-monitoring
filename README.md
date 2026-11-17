# vLLM · Qwen3.8-Flash-Next FP8 · 4x RTX PRO 6000 · TEP4 · MTP · Monitoring

A recipe for serving the official
[Qwen3.8-Flash-Next-FP8](https://huggingface.co/Qwen/Qwen3.8-Flash-Next-FP8)
checkpoint with [vLLM](https://github.com/vllm-project/vllm) across four
**NVIDIA RTX PRO 6000 Blackwell** cards (tensor + expert parallel 4), with
MTP-3 speculative decoding, the Rust frontend, and a bundled local
**Prometheus + Grafana** stack.

The engine flags are the vLLM recipe's rendered command for
[`rtx_pro_6000_4x` / `fp8` / tool calling + reasoning + spec decoding](https://recipes.vllm.ai/Qwen/Qwen3.8-Flash-Next?hardware=rtx_pro_6000_4x&variant=fp8&features=tool_calling%2Creasoning%2Cspec_decoding),
plus one local addition (`--gpu-memory-utilization 0.95`, which no recipe
render carries — see the flag notes in `compose.yaml`). The packaging
(compose, digest pin, monitoring, benchmark guard) and the 30-panel Grafana
dashboard follow an earlier recipe written for the same four cards.

## Status

**Booted, not yet benchmarked.** The engine pin moved from v0.33.0 to
v0.34.0 on 2026-11-17: healthy in 210 s, KV pool **6,937,800 tokens =
26.47x** one 262,144-token request, chat, reasoning split, tool calls
(streaming and not) and every dashboard metric checked, MTP acceptance
length 2.32–2.42 on four single requests from 2,855 to 261,200 prompt
tokens. No sweep was run on either release. The history below is v0.33.0.

First boot on the reference box
(2026-11-10, 4x RTX PRO 6000 Max-Q, weights pre-downloaded, empty compile
caches) reached `/health` in **191 s** and reported a KV pool of
**7,076,500 tokens = 26.99x** one 262,144-token request. A tool call came
back parsed into `tool_calls` (`finish_reason: tool_calls`), a plain chat
answered correctly, and every dashboard panel returned data against live
Prometheus. There are no throughput numbers yet; `docs/benchmarks.md` has
the sweep to run.

### Opinionated about what, exactly

- **The official FP8 checkpoint** — `Qwen/Qwen3.8-Flash-Next-FP8`, 172.78 GiB,
  weights served as-is.
- **The Qwen-Sharp chat template** — the vendored
  [Qwen-Sharp v22.5.0](https://huggingface.co/peculiar-ragdoll/Qwen-Sharp-Chat-Templates),
  the same pinned copy as the single-GPU recipe, instead of the checkpoint's
  template. See [The chat template](#the-chat-template).
- **The recipe's command otherwise verbatim** — only `--served-model-name`,
  `--host`, `--port`, `--chat-template` and `--gpu-memory-utilization 0.95`
  are added; the last is a local choice, not part of the render (the recipe
  leaves the flag unset).
- **A pinned release engine** — the bare upstream vLLM v0.34.0 image, pinned
  by digest. The recipe asks for ≥ 0.30.0 and is tuned on nightly; a digest
  pin is preferred over a moving `nightly` tag (see
  [Moving the pin](#moving-the-pin)).
- **TEP4** — tensor parallel 4 with expert parallelism, which the recipe
  recommends over plain TP for this model.
- **MTP-3** — the checkpoint's own MTP head, 3 draft tokens.
- **The Rust frontend** — `VLLM_USE_RUST_FRONTEND=1`, from the recipe. It
  has side effects on monitoring; see [Rust frontend](#rust-frontend).
- **Monitoring is not optional** — Prometheus and Grafana come up with the
  engine, with a 30-panel dashboard in 8 rows provisioned, including per-card
  power and thermals.

---

## Requirements

| | |
|---|---|
| GPUs | **4x** RTX PRO 6000 Blackwell, 96 GB each. The reference box has the 300 W Max-Q edition over PCIe. |
| Driver | CUDA 13 capable (v0.34.0 images are CUDA 13 only). Booted on driver 615.71.09. |
| Host RAM | **≥ 51 GB free plus headroom** for the 51B N-gram embedding table, which vLLM offloads to pinned host RAM by default (recipe; this stack's boot logs confirm `EngramConfig(cpu_offload=True)` and a pinned CPU-resident table, and the running engine container holds ~83 GiB of host RAM on the 122 GiB reference box). |
| Disk | ~173 GiB for the checkpoint in the Hugging Face cache. |
| Software | Docker with Compose v2 and the NVIDIA Container Toolkit. |

This stack binds the same four GPUs and the same host ports (8000, 3000,
9090) as the other engine recipe on the reference box. Run one engine at a
time.

## Quick start

```bash
git clone git@github.com:WombatSoftware/vllm-qwen3.8-flash-next-4x-rtx-pro-6000-fp8-monitoring.git
cd vllm-qwen3.8-flash-next-4x-rtx-pro-6000-fp8-monitoring
cp .env.example .env && $EDITOR .env      # set a Grafana password
```

**1. Pull the engine image** on the GPU host:

```bash
docker pull vllm/vllm-openai:v0.34.0@sha256:20fdb089454afbb324efa4110a2c025ff7b27cc1a086e07aa580b18df8ad1ce1
```

`compose.yaml` uses `pull_policy: never`, so this step is not optional.

**2. Start everything:**

```bash
docker compose up -d
until curl -sf -o /dev/null http://127.0.0.1:8000/health; do sleep 10; done
```

The first boot downloads the 172.78 GiB checkpoint into `HF_CACHE_DIR`
before it can answer; `start_period` allows an hour.

**3. Confirm the boot is the config you meant:**

```bash
docker logs qwen-3-8-flash-next-fp8 2>&1 | grep -E \
  "non-default args|GPU KV cache size|speculative_config=" | cut -c1-900
```

Expect `SpeculativeConfig(method='mtp', ..., num_spec_tokens=3)`,
`tensor_parallel_size: 4`, `enable_expert_parallel: True` and
`kv_cache_dtype: fp8`. The dashboard picks the `GPU KV cache size` up by
itself (see [Monitoring](#monitoring)).

**4. Check it:**

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen3.8-Flash-Next-FP8",
       "messages":[{"role":"user","content":"What is 17 times 23?"}],
       "max_tokens":1024}' | jq -r '.choices[0].message.content'
```

Keep `max_tokens` generous: the model reasons first, and a small budget can
end inside the reasoning with an empty `content`.

Grafana is at <http://127.0.0.1:3000>, Prometheus at <http://127.0.0.1:9090>.

> **Security:** the vLLM endpoint has no authentication and Prometheus has
> none either. Everything binds to `127.0.0.1` by default. To reach it from
> elsewhere, tunnel rather than rebinding:
> `ssh -L 8000:127.0.0.1:8000 -L 3000:127.0.0.1:3000 <gpu-host>`

## What you get

| | |
|---|---|
| Endpoint | OpenAI-compatible, `http://127.0.0.1:8000/v1`, served as `Qwen3.8-Flash-Next-FP8` |
| Context | 262,144 tokens (checkpoint native; `--max-model-len` is left unset as in the recipe) |
| Concurrency | `--max-num-seqs 16`, `--max-num-batched-tokens 8192` |
| Tool calling | Auto tool choice, `qwen3_coder` parser |
| Reasoning | Split into `reasoning_content` (`qwen3` reasoning parser) |
| Speculative decoding | MTP, 3 draft tokens |
| KV cache | fp8, plus fp8 for the sparse-attention indexer cache; 6,937,800-token pool (26.47x) on v0.34.0; v0.33.0 reported 7,077,888 (27.00x) |
| Monitoring | Prometheus scraping vLLM and a GPU exporter (all four cards), Grafana with a 30-panel dashboard in 8 rows, provisioned |

## The chat template

`chat_template.jinja` is a **verbatim, unmodified copy** of Qwen-Sharp
v22.5.0, pinned by hash (`./scripts/verify-chat-template.sh` checks it
against upstream). It gives per-request control through
`chat_template_kwargs`: `enable_thinking`, `reasoning_effort`
(`none`…`high`), `preserve_thinking`, `auto_disable_thinking_with_tools`,
`max_tool_response_chars` and more.

Its default `tool_call_format: "xml"` renders
`<tool_call><function=…><parameter=…>` blocks, which is what the recipe's
`qwen3_coder` parser reads. Do not set `tool_call_format: "json"`: the calls
would then arrive as plain text in `content`.

## Rust frontend

`VLLM_USE_RUST_FRONTEND=1` makes `vllm serve` run the `vllm-rs` binary as
the HTTP server instead of the Python API server. The dashboard was built
against the Python frontend:

- **Engine and HTTP metrics are there.** Every `vllm:*` metric the dashboard
  queries is exported, and `http_requests_total{method,status,handler}` uses
  the same grouped status labels (`2xx`, `4xx`) as the Python instrumentator.
- **`vllm:cache_config_info` is exported since v0.34.0**, without a
  `model_name` label. The dashboard's `kv_pool_tokens` variable reads the
  pool from its `kv_cache_size_tokens` label. On v0.33.0 the Rust frontend
  did not export the metric and the value had to be copied from the boot
  log.
- **There is no per-request access log.** `scripts/bench-guard.sh` cannot see
  foreign clients and records `VERDICT: PASS-BLIND`; only its concurrency
  check guards a sweep.

To run the Python frontend instead, delete `VLLM_USE_RUST_FRONTEND` from
`compose.yaml`. That is a departure from the recipe.

## Monitoring

The dashboard (`monitoring/grafana/dashboards/vllm-qwen-3-8-flash-next-fp8.json`)
follows the earlier 4-card recipe's dashboard with the model-specific parts
changed:

- the speculative-decoding row tracks MTP-3;
- the "KV cache tokens in use" panel and the "KV cache pool" stat read a
  `kv_pool_tokens` dashboard variable, which is a Prometheus query on
  `vllm:cache_config_info` (label `kv_cache_size_tokens`), so it follows
  whichever engine is running; the reference boot of this recipe reports
  6,937,800. The query was checked against live Prometheus on the reference
  box; the variable is empty, and both panels blank, while the engine is
  down.

The GPU row is unchanged: per-card power against the enforced limit, core
temperature against the live HW-slowdown threshold, fan speed.

## Moving the pin

Pin by digest (`tag@sha256:...`): release and nightly tags can be re-pushed.
A newer base must still contain PR #59621, or it dies at boot with
`ValueError: Invalid layer_type indexed_attention`:

```bash
docker run --rm --entrypoint bash vllm/vllm-openai:<tag>@<digest> -c \
  'grep -rn indexed_attention /usr/local/lib/python3.12/dist-packages/vllm/models/qwen4_exp/'
```

Boot it, check the lines from Quick start step 3 (the dashboard reads the
new pool by itself), and run a guarded sweep before trusting it. Keep the previous image until
the sweep passes.

## Documentation

- [`docs/benchmarks.md`](docs/benchmarks.md) — the sweep, benchmark
  isolation, and (once measured) the results.
- [`AGENTS.md`](AGENTS.md) — conventions and hazards for coding agents (and
  humans) changing this repo.
- `compose.yaml` — every flag is explained inline.

## Credits

- **[Qwen team, Alibaba](https://huggingface.co/Qwen)** — Qwen3.8-Flash-Next
  and the official FP8 checkpoint.
- **[vLLM project](https://github.com/vllm-project/vllm)** — the inference
  engine, the Rust frontend, and the
  [recipe](https://recipes.vllm.ai/Qwen/Qwen3.8-Flash-Next) this serve
  command comes from.
- **[Prometheus](https://prometheus.io/)**, **[Grafana](https://grafana.com/)**
  and **[nvidia_gpu_exporter](https://github.com/utkuozdemir/nvidia_gpu_exporter)**
  — the monitoring stack.
- **[llama-benchy](https://pypi.org/project/llama-benchy/)** — the benchmark
  harness.

None of the above endorse this repository.

## Licences

[Apache-2.0](LICENSE) for the configuration, documentation and scripts.

Apart from `chat_template.jinja` (Apache-2.0, peculiar-ragdoll / froggeric,
unmodified), nothing third-party is redistributed here; everything is pulled
at run time and stays under its own licence. `Qwen/Qwen3.8-Flash-Next-FP8` is published
under `qwen-community-1.0` ("other" on Hugging Face); read its terms before
you deploy it. See [`NOTICE`](NOTICE) for the full list.
