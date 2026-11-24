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
