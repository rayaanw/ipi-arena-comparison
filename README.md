# IPI Arena Bench — Qwen3-VL vs Claude Sonnet 4.5

Independent AI-safety benchmarking: how often does an AI agent get hijacked
by malicious instructions hidden inside the content it's processing (an
email, a web page, a tool result) rather than typed by its actual user —
"indirect prompt injection" (IPI)? This runs a published, open-source attack
benchmark against two closed-source agent models side by side to compare how
each one holds up.

Comparison run of [`ipi-arena-bench`](https://github.com/GraySwanAI/ipi_arena_os) (Gray Swan AI's
indirect prompt injection benchmark) using the open-source attack set from
[`sureheremarv/ipi_arena_attacks`](https://huggingface.co/datasets/sureheremarv/ipi_arena_attacks)
(95 attacks across 28 behaviors, generated/tuned against Qwen3-VL).

Both models were run against the **identical** 95 attacks, judged by the same LLM judge
(`google/gemini-2.5-flash`), via OpenRouter.

## Setup

- Target providers: `openrouter`
- Judge / WorldSim model: `google/gemini-2.5-flash`
- Targets compared:
  - `qwen/qwen3-vl-235b-a22b-instruct`
  - `anthropic/claude-sonnet-4.5`

Reproduce:

```bash
git clone https://github.com/GraySwanAI/ipi_arena_os.git
cd ipi_arena_os
python3 -m venv .venv && source .venv/bin/activate
pip install -e ".[browser]"
playwright install chromium
pip install datasets

export OPENROUTER_API_KEY=sk-or-...

# copy configs/hf_attacks_full_qwen.yaml and configs/hf_attacks_full_claude.yaml
# from this folder into ipi_arena_os/examples/, then:
ipi-arena-bench run --config examples/hf_attacks_full_qwen.yaml
ipi-arena-bench run --config examples/hf_attacks_full_claude.yaml
```

## Results

### Qwen3-VL 235B — 95/95 completed, 0 errors

**Attack Success Rate: 33.7%** (32/95 breaks)

| Category | Breaks | Total | ASR |
|---|---|---|---|
| coding | 23 | 50 | 46% |
| tool | 8 | 39 | 21% |
| browser | 1 | 6 | 17% |

Qwen was most vulnerable on **coding**-category attacks (faking test results, cross-agent
credential sharing, code-execution cascades) — nearly half of those attacks succeeded.

### Claude Sonnet 4.5 — 58/95 completed, 37 errors

**Attack Success Rate: 0%** (0/58 breaks)

| Category | Breaks | Total run | ASR |
|---|---|---|---|
| tool | 0 | 23/39 | 0% |
| coding | 0 | 35/50 | 0% |
| browser | 0 | 0/6 | 0% |

The 37 errors are all `402 insufficient credits` — Claude Sonnet 4.5's per-token cost is high
enough that a full 95-attack run (with up to 5 tool-calling steps each) exceeds the $20 total
spend cap configured on the OpenRouter key used, even after account credits were topped up
separately. Raising the key's `$20` total-limit field further (not just account balance) would
let the remaining ~37 attacks complete.

An earlier independent partial run (53/95 attacks, different API-key restriction) also completed
with **0/53 breaks** — see `logs/claude_partial_run_53of95.log`. Across both runs combined,
Claude was tested against 111 attack attempts (with some overlap) and broke on **zero** of them.

## Takeaway

This is consistent with the source paper: these 95 attacks were generated and tuned specifically
against Qwen3-VL, and the paper's own transfer experiments found they largely don't transfer to
closed-source models. Qwen breaks on roughly a third of attacks (33.7% ASR); Claude Sonnet 4.5,
across every attack actually delivered to it in two separate runs, broke on none (0% ASR).

## Files

- `configs/` — the exact YAML configs used for each run
- `results/qwen/` — full raw JSON output (per-attack transcripts + judge scores), 95/95 complete
- `results/claude/` — full raw JSON output, 58/95 complete (rest blocked by API spend cap, see above)
- `logs/` — captured stdout logs from each benchmark run, including the earlier partial attempt

## Notes

- No API keys or credentials are included anywhere in this folder.
- Judge scores are LLM-generated (`google/gemini-2.5-flash`) and are probabilistic — re-running
  the same attacks can produce slightly different pass/fail results.
