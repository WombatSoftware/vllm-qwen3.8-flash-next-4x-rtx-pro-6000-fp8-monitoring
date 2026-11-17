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

The engine is the bare upstream v0.34.0 release image, pinned by digest. There
is no Dockerfile and no fork.

**Booted, not benchmarked.** `docs/benchmarks.md` has no throughput results.
Do not add performance claims to any file without a guarded sweep behind them.

## Hard rules

- **This repository is PUBLIC** (`WombatSoftware/...` on GitHub, visibility
  checked 2026-11-14). Never put hostnames, IPs, serials, local usernames or
  home-directory paths, credentials, or anything that identifies the
  reference box or its owner into files, comments, the dashboard JSON, or
  commit messages. Hardware is described by spec only (GPU model, RAM size,
  measured numbers) as "the reference box"; SSH examples use `<gpu-host>`.
  Before pushing, grep the diff:
  `git diff @ | grep -niE "192\.168|/Users/|<your name|hostname>"`.
- **Never commit `.env`.** It holds `GRAFANA_ADMIN_PASSWORD` and possibly
  `HF_TOKEN`. It is gitignored; keep it that way.
- **Never `docker compose down`** on a shared box. It takes Prometheus and
  Grafana with it. Recreate the single service:
  `docker compose up -d qwen-3-8-flash-next-fp8`.
- **This stack shares the four GPUs and host ports 8000/3000/9090 with the
  other engine recipe on the reference box.** Stop the other engine before
  starting this one, and do
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
- **Do not bump the vendored `chat_template.jinja` silently.** It is a
  verbatim Apache-2.0 copy pinned by hash in `NOTICE`. If you bump it, update
  the hash and commit in both `NOTICE` and `scripts/verify-chat-template.sh`,
  and run that script. Keep it in step with the single-GPU Qwen3.8 recipe.

## Restarting the engine

A boot with the weights pre-downloaded takes about 3.5 minutes to healthy
(210 s on v0.34.0, 191 s on v0.33.0). A first boot that has to download the
172.78 GiB checkpoint takes far longer, which is why `start_period` is
3600 s. Expect a pool of roughly 6,940,000 tokens on v0.34.0 (6,937,800 on
the 2026-11-17 boot; v0.33.0 reported 7,076,500 to 7,077,888 — the pool
drifts slightly between identical boots and moved about 2% with the
release).

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

- The KV pool panels read the `kv_pool_tokens` dashboard variable, a
  Prometheus `query_result` on `vllm:cache_config_info{job="vllm"}` with a
  regex on its `kv_cache_size_tokens` label. Nothing is edited after a boot.
  Do not turn it back into a constant.
- Do not reintroduce hard-coded `model_name` literals; panels bind to
  `$model_name`.
- `vllm:cache_config_info` is exported by the `vllm-rs` binary since
  v0.34.0 and carries no `model_name` label there, which is why the
  variable selects it by `job`. On an older base the variable comes back
  empty and both pool panels go blank.

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
4. Check that the dashboard's `kv_pool_tokens` variable shows the new boot's
   `GPU KV cache size` (it is read from `vllm:cache_config_info`).
5. Run a guarded sweep and compare against `docs/benchmarks.md` before calling
   the new base the baseline. Delete the old image only after that.

When blaming a release for a regression, show that a suspect commit touches a
file this model's runtime path imports. Commit-title plausibility is not
evidence.

## Style

Documentation here explains *why*, with numbers, and admits what is not known.
Match that. If you cannot measure a claim, do not make it.
