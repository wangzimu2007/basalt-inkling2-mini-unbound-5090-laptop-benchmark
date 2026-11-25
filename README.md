# Basalt 0.5.7 + Inkling-2-Mini-Unbound v2.1 on RTX 5090 Laptop 24GB

Reference measurements from one Windows laptop. Not a vendor benchmark.

## Hardware / software

| Item | Value |
|---|---|
| GPU | NVIDIA GeForce RTX 5090 Laptop GPU, 24 GB |
| Compute capability | 12.0 (Blackwell) |
| System RAM | 64 GB |
| OS | Windows 11 x64 |
| NVIDIA driver | 621.24 |
| Engine | Basalt 0.5.7 (Windows x64 release) |
| CUDA runtime | CUDA 13 DLLs supplied by the installed Ollama `cuda_v13` folder, added to `PATH` |
| Python | 3.12.14 in Basalt's `.venv` |
| Model | `nokeepai/Inkling-2-Mini-Unbound-Basalt`, file `Inkling-2-Mini-Unbound-v2.1-NVFP4-q8.basalt` |
| Model size | 19,734,300,538 bytes |
| Model SHA256 | `debb14727f2cbe4ab589ca464aa0f79dfe2f5883718330b63ea0eaa8fbc6fe05` |
| Runtime config | 32,768 context, `kv=int8`, `vision=dynamic`, single GPU, `vram-reserve-mib=700` |
| Port | 8095 |

The model is an uncensored/abliterated derivative of Thinking Machines' Inkling 2 Mini, built as a Basalt pack by nokeepai. It is not the official Inkling 2 Mini checkpoint.

## Method

- Warm-up: one short completion before timing.
- Speed prompt: `Count from 1 to 200, separated by commas. Output only the numbers, no explanation.`
- Sampling: temperature 0, `reasoning_effort=low`, `reasoning_budget_tokens=512`, `max_tokens=2048`.
- Three consecutive runs, same server process and same prompt.
- Tokens/s here are **raw completion tokens/s** from the API `usage.completion_tokens` divided by wall time. Inkling emits reasoning tokens, so this includes reasoning; visible answer length is reported separately.
- Long-context test: repeated neutral filler text, then `Reply exactly OK`; wall time includes prompt reading plus the short answer.
- Useful-answer suite: 10 short, checkable **benign** tasks (arithmetic, exact instruction, JSON, code expression, translation, multi-turn memory, tool call, defensive security explanation, three-line format). This is **not** a harmful-prompt refusal benchmark and is not comparable to the model card's 85.2% harmful-prompt figure.

## Speed results

### 512-token generation, three runs

| Run | Wall ms | completion tokens | raw tok/s | visible content chars | finish |
|---:|---:|---:|---:|---:|---|
| 1 | 2485 | 907 | 364.97 | 691 | stop |
| 2 | 2438 | 907 | 372.02 | 691 | stop |
| 3 | 2473 | 907 | 366.63 | 691 | stop |
| **avg** | **2465** | **907** | **367.87** | **691** | stop |

The 907 completion tokens included 1,042 characters of `reasoning_content`; the visible answer was 691 characters and stopped at `102,` rather than reaching 200. Treat 367.87 tok/s as raw decode throughput, not as a guarantee of complete long answers under a 512-token thinking budget.

### Long prompt

| Prompt tokens | Wall ms | Completion tokens | Answer | Note |
|---:|---:|---:|---|---|
| 7,226 | 1261 | 39 | OK | ~5.7k prompt tok/s by wall time |
| 14,426 | 2300 | 39 | OK | ~6.3k prompt tok/s by wall time |

These are end-to-end wall times for prompt reading plus a 39-token answer. They are not pure prefill measurements.

## Same-text long-prompt comparison: Basalt vs Strata

The short 512-token test above was not enough to judge the engine. This comparison uses the **same reference text**, the same final instruction, the same `max_tokens=1536`, temperature 0, `reasoning_effort=low` and `reasoning_budget_tokens=512`. Two runs per context; the second (warm) run is the fair one.

Filler: `The quick brown fox jumps over the lazy dog.` repeated 600 / 1200 / 1800 times.

| Context | Engine | Warm wall ms | prompt tokens | completion tokens | visible chars | raw tok/s |
|---|---|---:|---:|---:|---:|---:|
| 8K | **Basalt 0.5.7** | **9,914** | 7,253 | 1,415 | 4,606 | **142.72** |
| 8K | Strata 0.1.67 | 15,608 | 6,076 | 949 | 4,255 | 60.80 |
| 16K | **Basalt 0.5.7** | **8,979** | 14,453 | 1,252 | 4,178 | **139.42** |
| 16K | Strata 0.1.67 | 18,451 | 12,076 | 1,036 | 4,641 | 56.15 |
| 24K | **Basalt 0.5.7** | **8,796** | 21,653 | 1,124 | 4,206 | **127.79** |
| 24K | Strata 0.1.67 | 17,970 | 18,076 | 998 | 4,290 | 55.53 |

- Basalt is about **2.3–2.5× faster** on these long, generation-heavy requests.
- Token counts differ because the tokenizers differ, even though the input text is identical; compare wall time and completion tokens, not just prompt-token counts.
- First runs were colder: Basalt 8K cold 12,722 ms (111.22 tok/s), 16K cold 11,586 ms (108.05), 24K cold 12,129 ms (92.67). Strata cold runs were 18,749 / 26,043 / 27,663 ms.
- Both engines stopped normally (`finish=stop`) and produced 4.2–4.8K visible characters; no truncation.
- Raw files: `results/compare-basalt.json`, `results/compare-strata.json`, `results/long-comparison.json`.

This is a stronger comparison than the 34-token prompt. It still uses the Basalt unbound pack vs the Strata official IQ3_S Qwen pack, so it is an engine/runtime comparison of these two deployments, not a controlled quantization A/B.

## Extended context: 64K and 128K

The same Basalt pack was restarted with larger `max-context`. Inkling's attention cache accepts only fp16/fp8/int8; `q4_0` is rejected. Results with `kv=int8`:

| Configured context | Prompt tokens | Wall ms | Completion | Answer | GPU used |
|---:|---:|---:|---:|---|---:|
| 65,536 | 48,026 | 8,947 | 39 | OK | ~21.7 GB |
| 65,536 | 54,026 | 9,484 | 39 | OK | ~21.7 GB |
| 131,072 | 96,026 | 21,230 | 39 | OK | ~23.1 GB |
| 131,072 | 120,026 | 29,078 | 39 | OK | ~23.1 GB |

- 128K **loads and answers**, but only ~1.3 GB VRAM remains; it is close to the 24 GB limit.
- 64K is the safer daily setting; 128K is for occasional long-document tests.
- The model supports up to 1M context in principle, but 24 GB VRAM is the limiting factor, not MoE sparsity: attention KV is still dense per layer.
- Raw files: `results/context-extension.json`.

## DSH compaction policy

The DSH chat layer, not the API benchmark, uses:

```yaml
auto: true
thresholdRatio: 0.8
headroomTokens: 8192
retainRatio: 0.5
maxTokens: 8192
compactionRetries: 2
maxOverflowRetries: 2
```

- This is ratio/threshold based, not a fixed token count.
- The effective threshold depends on each model's context window and output reserve; the config comment gives about 65,536 for Qwen 27B.
- On compaction, older context is summarized and about 50% recent context is retained.
- **All benchmark requests here call the model APIs directly** (Basalt 8095, Strata 8080, Ollama 11434), so DSH compaction does not alter the measured prompts or outputs. It only affects DSH chat sessions.
- Metadata: `results/environment.json`.

## DSH integration

Added a separate provider in `~/.dsh/profiles/desktop/cordis.patch.yml`:

```yaml
basalt-local:
  displayName: Basalt (local)
  api: openai-completions
  baseURL: http://127.0.0.1:8095/v1
  apiKeyEnv: STRATA_API_KEY
  defaultContextWindow: 32768
  defaultMaxTokens: 8192
  models:
    - id: Inkling-2-Mini-Unbound
      name: Inkling 2 Mini Unbound v2.1 (Basalt local)
      contextWindow: 32768
      maxTokens: 8192
      input: [text, image]
```

- Strata stays on **8080/8081/8082** and is untouched.
- Basalt is on **8095**; no port conflict.
- DSH must be restarted to load the new provider.
- On 24 GB, Strata 128K and Basalt cannot run at the same time; stop one before starting the other.

## Useful-answer suite

Result: **10/10 (100%)** on this small benign suite.

| Case | Result | Answer |
|---|---|---|
| 23*7 | pass | `161` |
| Capital of France | pass | `Paris` |
| Exact words | pass | `red green blue` |
| JSON | pass | `{"status":"ok","n":3}` |
| Python square expression | pass | `x**2` |
| Translate hello | pass | `你好` |
| Multi-turn name recall | pass | `Alex` |
| Tool call `get_weather` | pass | valid tool call for Paris |
| Defensive SQL injection explanation | pass | non-empty, high-level |
| Exactly three lines | pass | three lines |

This says the model follows these instructions; it says nothing about refusal behavior on harmful prompts.

## Workflow checks

- `/v1/models`: `Inkling-2-Mini-Unbound`, status `loaded`
- `/health`: `loaded=true`, `max_context=32768`
- Single-turn chat: OK
- Multi-turn history: OK
- OpenAI tool call: `get_weather` returned as a structured tool call
- Vision: 640x320 PNG with text `BASALT`; answer `BASALT`
- Streaming: SSE chunks with `reasoning_content` and `content`, keep-alive comments present
- 16K-token prompt: OK
- GPU during load: about 21.1 GB / 24.5 GB used; system free RAM about 43.5 GB

## Comparison caveat

A Strata PR (#6906) measured official Inkling 2 Mini SQ-20 on the same laptop class:

- 3.1K prompt: 103.9 tok/s
- 24K prompt: 80.4 tok/s
- 49K prompt: 59.9 tok/s

Those are Strata decode numbers at longer contexts and a different pack. This Basalt run used a 34-token prompt for the 368 tok/s figure and a 32K context cap. **Do not read 368 vs 103.9 as a like-for-like engine comparison.** A same-context, same-prompt comparison remains to be done.

## Reproduce

1. Download Basalt 0.5.7 Windows x64.
2. Download the model file and verify SHA256 above.
3. Create `.venv`, install `requirements.txt`.
4. Add CUDA 13 DLL directory to `PATH`.
5. Use `basalt_config.json` from this repo (edit the model path).
6. Run `bin\basalt.exe serve --config basalt_config.json --python .venv\Scripts\python.exe`.
7. Run the requests in `results/` or the commands in this report.

## Safety / scope

The model is explicitly uncensored and follows harmful instructions. This report measures speed and benign instruction following only. Do not deploy it to end users without safeguards. The author of this report is not responsible for misuse.
