# vLLM · Qwen3.8-Flash-Next FP8 · 4x RTX PRO 6000 · TEP4 · MTP · Monitoring

A recipe for serving the official
[Qwen3.8-Flash-Next-FP8](https://huggingface.co/Qwen/Qwen3.8-Flash-Next-FP8)
checkpoint with [vLLM](https://github.com/vllm-project/vllm) across four
**NVIDIA RTX PRO 6000 Blackwell** cards (tensor + expert parallel 4), with
MTP-3 speculative decoding, the Rust frontend, and a bundled local
**Prometheus + Grafana** stack.

The engine flags are the vLLM recipe's rendered command for
[`rtx_pro_6000_4x` / `fp8` / tool calling + reasoning + spec decoding](https://recipes.vllm.ai/Qwen/Qwen3.8-Flash-Next?hardware=rtx_pro_6000_4x&variant=fp8&features=tool_calling%2Creasoning%2Cspec_decoding),
unchanged. The packaging (compose, digest pin, monitoring, benchmark guard)
and the 30-panel Grafana dashboard come from the sibling
[GLM-5.3-Flash recipe](https://github.com/WombatSoftware/vllm-glm5.3-flash-4x-rtx-pro-6000-dflash2-monitoring)
for the same four cards.

## Status

**Not yet booted or benchmarked on this hardware.** Every number below is
either from the vLLM recipe page or a property of the checkpoint; none is a
measurement from this repo. `docs/benchmarks.md` has the sweep to run and no
results yet. The GLM-5.3 and single-GPU Qwen3.8 recipes this is modelled on
make claims only from measurements; this one will once it has some.

What was checked, inside the pinned image and without a GPU: the `vllm-rs`
Rust frontend binary is present, `vllm serve` (the image ENTRYPOINT) honours
`VLLM_USE_RUST_FRONTEND`, and `--attention-config.indexer_kv_dtype`, the
`qwen3_coder` tool parser, the `qwen3` reasoning parser and the `qwen4_exp`
model are all in v0.33.0.

### Opinionated about what, exactly

- **The official FP8 checkpoint** — `Qwen/Qwen3.8-Flash-Next-FP8`, 172.78 GiB,
  weights served as-is.
- **The Qwen-Sharp chat template** — the vendored
  [Qwen-Sharp v22.5.0](https://huggingface.co/peculiar-ragdoll/Qwen-Sharp-Chat-Templates),
  the same pinned copy as the single-GPU recipe, instead of the checkpoint's
  template. See [The chat template](#the-chat-template).
- **The recipe's command otherwise verbatim** — only `--served-model-name`,
  `--host`, `--port` and `--chat-template` are added.
- **A pinned release engine** — the bare upstream vLLM v0.33.0 image, pinned
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
| Driver | CUDA 13 capable (v0.33.0 images are CUDA 13 only). |
| Host RAM | **≥ 51 GB free plus headroom** for the N-gram embedding table, which this image offloads to host RAM by default (recipe). |
| Disk | ~173 GiB for the checkpoint in the Hugging Face cache. |
| Software | Docker with Compose v2 and the NVIDIA Container Toolkit. |

This stack binds the same four GPUs and the same host ports (8000, 3000,
9090) as the GLM-5.3-Flash recipe. Run one at a time.

## Quick start

```bash
git clone git@github.com:WombatSoftware/vllm-qwen3.8-flash-next-4x-rtx-pro-6000-fp8-monitoring.git
cd vllm-qwen3.8-flash-next-4x-rtx-pro-6000-fp8-monitoring
cp .env.example .env && $EDITOR .env      # set a Grafana password
```

**1. Pull the engine image** on the GPU host:

```bash
docker pull vllm/vllm-openai:v0.33.0@sha256:d825baf9ce4b1b6bfc5ecc7b79880c7e6748c65a72d697ce3d47607aafa84c6e
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
`kv_cache_dtype: fp8`. Put the `GPU KV cache size` token count into the
dashboard's `kv_pool_tokens` variable (see [Monitoring](#monitoring)).

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
| KV cache | fp8, plus fp8 for the sparse-attention indexer cache; pool size known after first boot |
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
against the Python frontend, so four things are unverified until the first
boot:

- **Engine metrics.** The binary contains every `vllm:*` metric name the
  dashboard queries except `vllm:cache_config_info`. Not yet seen on a live
  `/metrics`.
- **HTTP metrics.** The "HTTP requests and errors" panel queries
  `http_requests_total`. The binary has a counter described as "Total number
  of HTTP requests by method, status, and handler", but its exact metric
  name is not visible in the binary's strings.
- **`vllm:cache_config_info` is not in the binary**, so the "KV cache pool
  (boot-time config)" table will most likely show No data. The boot log's
  `GPU KV cache size` line has the same information.
- **Access-log format.** `scripts/bench-guard.sh` detects foreign clients by
  parsing uvicorn-style access-log lines. If the Rust frontend logs
  differently, that check sees nothing; the guard then records
  `VERDICT: PASS-BLIND` instead of `PASS`, so a blind run cannot pass as a
  guarded one.

To run the Python frontend instead, delete `VLLM_USE_RUST_FRONTEND` from
`compose.yaml`. That is a departure from the recipe.

## Monitoring

The dashboard (`monitoring/grafana/dashboards/vllm-qwen-3-8-flash-next-fp8.json`)
is the GLM-5.3-Flash dashboard with the model-specific parts changed:

- the speculative-decoding row tracks MTP-3 instead of DFlash2;
- the "KV cache tokens in use" panel multiplies usage by a
  `kv_pool_tokens` dashboard variable instead of a hard-coded pool size,
  because this config's pool is not known yet. It ships as `0` and the panel
  stays blank until you set it. The dashboard is provisioned read-only, so
  persist the value by editing the variable's `query` in the JSON.

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

Boot it, check the lines from Quick start step 3, update `kv_pool_tokens`,
and run a guarded sweep before trusting it. Keep the previous image until
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
