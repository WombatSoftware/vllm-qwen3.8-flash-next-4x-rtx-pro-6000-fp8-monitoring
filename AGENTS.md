# Notes for coding agents

Instructions for AI coding agents working in this repository. Humans may find
the hazards section useful too.

## What this repo is

A four-GPU (TEP4) vLLM deployment recipe. It is **configuration, not an
application** -- there is no build, no test suite, and no code to refactor. The
deliverable is `compose.yaml` plus the documentation that justifies it.

The engine flags are the vLLM recipe's rendered command for
`rtx_pro_6000_4x` / `fp8` / tool_calling + reasoning + spec_decoding, plus
`--served-model-name`, `--host` and `--port`. The comments in `compose.yaml`
are load-bearing. Do not "tidy" them away.

The engine is the bare upstream v0.33.0 release image, pinned by digest. There
is no Dockerfile and no fork.

**Nothing here is measured yet.** `docs/benchmarks.md` has no results. Do not
add performance claims to any file without a guarded sweep behind them.

## Hard rules

- **Never commit `.env`.** It holds `GRAFANA_ADMIN_PASSWORD` and possibly
  `HF_TOKEN`. It is gitignored; keep it that way.
- **Never `docker compose down`** on a shared box. It takes Prometheus and
  Grafana with it. Recreate the single service:
  `docker compose up -d qwen-3-8-flash-next-fp8`.
- **This stack and the GLM-5.3-Flash stack share the four GPUs and host ports
  8000/3000/9090.** Stop the other engine before starting this one, and do
  not stop someone else's engine without asking: on the reference box the
  engine on these cards may be the one serving you.
- **Never change a flag and a benchmark claim in the same commit** without
  re-running the sweep.
- **Never run a sweep outside `scripts/bench-guard.sh`**, and never while an
  agent session is using this engine. `VERDICT: PASS-BLIND` means the
  access-log check saw nothing (likely under the Rust frontend); say so
  wherever such a result is recorded.
- **After any edit near `command:` in `compose.yaml`, re-check the flag list**
  against the `non-default args` line of the next boot. Do not eyeball the
  diff: a dropped `--speculative-config` still boots and still answers.
- **Pass bare args.** The image ENTRYPOINT is `vllm serve`; prefixing
  `command:` with `vllm serve` breaks it.
- **`VLLM_USE_RUST_FRONTEND=1` is part of the recipe.** Removing it is a
  deliberate departure; document it if you do.

## Restarting the engine

Boot time on this config is not measured yet. A first boot downloads the
172.78 GiB checkpoint first, which is why `start_period` is 3600 s.

```bash
docker compose up -d qwen-3-8-flash-next-fp8
until curl -sf -o /dev/null http://127.0.0.1:8000/health; do sleep 10; done
docker logs qwen-3-8-flash-next-fp8 2>&1 | grep -E \
  "non-default args|GPU KV cache size|speculative_config=" | cut -c1-900
```

Expect `SpeculativeConfig(method='mtp', ..., num_spec_tokens=3)`,
`kv_cache_dtype: fp8`, `tensor_parallel_size: 4` and
`enable_expert_parallel: True`. Smoke with `max_tokens` of at least 1024; the
model reasons first and a small budget can return `content: null` from a
healthy engine.

If it does not come up, look for a crash loop before assuming slowness:
`docker inspect -f '{{.RestartCount}}' qwen-3-8-flash-next-fp8`. Each cycle
re-reads the weights and re-pins the N-gram table in host RAM.

## Dashboard

- The "KV cache tokens in use" panel reads the `kv_pool_tokens` dashboard
  variable. After every boot that can move the pool (image, TP,
  gpu-memory-utilization, batch budget, speculative config), set the
  variable's `query`/`current` in the JSON to the boot's `GPU KV cache size`.
- Do not reintroduce hard-coded `model_name` literals; panels bind to
  `$model_name`.
- `vllm:cache_config_info` is a Python-frontend metric and is absent from the
  `vllm-rs` binary. Do not "fix" its empty panel by guessing values.

## Benchmarking

`./scripts/benchmark.sh <id>` runs the full sweep under the isolation guard.
Requirements:

- The engine must already be **healthy**.
- `llama-benchy` buffers stdout until exit. Poll `pgrep -f llama-benchy` and
  the `.done` file; do not watch the log for growth.
- Nothing else may use the GPUs or the engine during a run. Exit 42 and
  `VERDICT: FAIL` mean the numbers are void, not that the build is slow.
- The harness does **not** abort on HTTP errors; it drops failed requests
  from aggregation. Check the log and the server log before trusting a result.

## Moving the image pin

1. Pull the new image and pin it as `vllm/vllm-openai:<tag>@sha256:<digest>`.
2. Check it contains PR #59621 (README "Moving the pin" has the grep), the
   `vllm-rs` binary, and the flags in `command:`. `vllm serve --help` needs a
   visible GPU in this image; grep the installed source instead.
3. Keep the old image. Recreate the engine and verify the boot lines above.
4. Update `kv_pool_tokens` in the dashboard.
5. Run a guarded sweep and compare against `docs/benchmarks.md` before calling
   the new base the baseline. Delete the old image only after that.

When blaming a release for a regression, show that a suspect commit touches a
file this model's runtime path imports. Commit-title plausibility is not
evidence.

## Style

Documentation here explains *why*, with numbers, and admits what is not known.
Match that. If you cannot measure a claim, do not make it.
