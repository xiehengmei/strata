# Community benchmark on NVIDIA GeForce RTX 4060 Ti

Measured on 2026-10-12 with `tools/strata_bench.py` (suite 1, 3 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA GeForce RTX 4060 Ti, 16 GiB VRAM
- CPU: Intel(R) Core(TM) i7-14700K (20 cores, 28 threads); installed RAM: 31.7 GiB; storage: Samsung SSD 990 EVO Plus 2TB SSD/NVMe
- OS: Windows 11 (10.0.26200); GPU driver: 616.92; CUDA: 13.4
- Strata: project 0.1.43, commit 7695859a; engine 0.1.42
- GPU detail: vram_mib=16380, driver=616.92, power_limit_w=165.00, power_draw_w=81.53, pcie=gen4 x8, pstate=P2, temp_c=54
- Background workloads: not recorded (GPU power limit/draw read above)

## Model and configuration

- Model and quantization: qwen3.8-flash-next-coder-iq1_m
- Context: 65536 tokens; parallel slots: 1
- Settings: the install's own configuration, unchanged; thinking off for the suite; temperature 0

```text
strata.exe --pack coder-iq1_m --native Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf --ple-gguf Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf --expert-profile expert-profile-coder.bin --expert-cache auto --prefill auto --spec 4 --mtp rt --max-context 65536 --kv int8 --resident-experts --spec-min-p 0.5 --pool-workers 10 --adapt-every 1 --adapt-swaps 160 --adapt-decay 0.97
```

## Method

Run by `python tools/strata_bench.py --long` against the HTTP API of a Strata server (started by the benchmark from the install configuration); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 3 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 3 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 3 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 46 (43-51) | 0 | 100 (100-100) | 9 | not measured | 30.6 (22.0-35.2) | 0.98 (0.68-1.52) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 3 | 975.8 (952.8-1047.3) | not measured | 4.20 (3.89-4.25) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 3 | 1223.8 (1136.8-1235.4) | not measured | 13.17 (13.08-14.20) |
| 32K prompt, fresh | 32049 (32049-32049) | 0 (0-0) | 24 cap | 3 | 1245.3 (1229.7-1271.3) | not measured | 25.81 (25.27-26.16) |
| agent session, follow-up turns | 14400 (14367-14410) at the last turn | see below | up to 64 | 3 | see below | 47.1 (20.4-71.5) | 2.93 (2.06-3.79) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 68.5 % (68.5-68.6); earlier-turn tokens read again per session: 74 (5-75) (earlier turns that the cache did not hold; the 13634 (13633-13634) new tokens of the files are read once); first turn (nothing cached) first token: 1.75 s (1.58-2.13).

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 293.9 | 2.127 | 15.6 |
| 1 | 3711 | 630 | 3081 | 892.2 | 3.529 | 20.4 |
| 2 | 5244 | 3704 | 1540 | 638.0 | 2.472 | 20.8 |
| 3 | 8657 | 5277 | 3380 | 908.0 | 3.789 | 24.4 |
| 4 | 11541 | 8681 | 2860 | 859.0 | 3.427 | 24.0 |
| 5 | 14400 | 11554 | 2846 | 896.1 | 3.217 | 38.1 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load is not recorded. Total time of the suite: 261 s.
