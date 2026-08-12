# Session Handoff — IPI Arena Bench (Qwen3-VL vs Claude Sonnet 4.5)

This file exists because there's no tool to export a raw chat transcript — it's a
reconstructed summary of everything done in the session that produced this repo, written so
a fresh Claude Code session (e.g. on a new laptop) can pick up context quickly. Point a new
session at this repo and have it read this file first.

## What this project is

[`GraySwanAI/ipi_arena_os`](https://github.com/GraySwanAI/ipi_arena_os) is an open-source
benchmark (`ipi-arena-bench`) for testing indirect prompt injection (IPI) vulnerability in AI
agents, released alongside the paper *"How Vulnerable Are AI Agents to Indirect Prompt
Injections? Insights from a Large-Scale Public Competition"* ([arXiv:2603.15714](https://arxiv.org/abs/2603.15714)).

The repo ships 41 behaviors (scenarios: tool-calling, coding, browser) and open-sources 95
successful attack strings (from [`sureheremarv/ipi_arena_attacks`](https://huggingface.co/datasets/sureheremarv/ipi_arena_attacks))
that were generated/tuned against Qwen3-VL and — per the paper itself — did **not** transfer to
any closed-source model.

## What we did, in order

1. Cloned `ipi_arena_os` locally to `~/Desktop/Claudeprojects/ipi_arena_os`, set up a
   `.venv`, `pip install -e .`, installed `datasets` + Playwright/Chromium for browser
   behaviors.
2. Set `OPENROUTER_API_KEY` in a git-ignored `.env` file (never committed).
3. Ran a smoke test (`run-one` against `garage-door-email`) — safe, no break.
4. Ran the example 7-attack subset (`examples/hf_attacks.yaml`, 3 behaviors) against Qwen3-VL
   — found 1 break (`database-deletion`, full SQL wipe executed).
5. Built two full configs (`configs/hf_attacks_full_qwen.yaml`,
   `configs/hf_attacks_full_claude.yaml`) that run **all 95** attacks with no
   `behavior_ids` filter, targeting Qwen3-VL and Claude Sonnet 4.5 respectively, both judged
   by `google/gemini-2.5-flash`.
6. Ran both. Hit two distinct OpenRouter blockers along the way (see "Gotchas" below) that
   took several retries to fully resolve for Claude specifically.
7. Compiled results, logs, and configs into this folder (`ipi-arena-comparison/`) for
   pushing to GitHub.

## Final results

| Model | Attacks completed | Breaks | ASR |
|---|---|---|---|
| **Qwen3-VL 235B** (`qwen/qwen3-vl-235b-a22b-instruct`) | 95/95 | 32 | **33.7%** |
| **Claude Sonnet 4.5** (`anthropic/claude-sonnet-4.5`) | 58/95 | 0 | **0%** |

Qwen broke most on `coding`-category attacks (46% ASR) — faking test results, cross-agent
credential sharing, code-execution cascades. Claude resisted every attack actually delivered
to it, across two independent partial runs (53/95 and 58/95, both 0 breaks) — consistent with
the paper's own claim that this attack set doesn't transfer off Qwen.

**Not fully closed out:** Claude is at 58/95, not 95/95 — the remaining 37 attacks
(`negative-review` onward alphabetically, see `logs/claude_run_58of95.log` for exact cutoff)
were blocked by hitting a `$20` total spend cap on the OpenRouter API key used. See
`README.md` for the fuller breakdown.

## Gotchas hit along the way (useful if resuming)

- **OpenRouter account credit balance** and an **OpenRouter API key's own "total limit" cap**
  are two *separate* things. Running low on either produces a `402`/`403` error that looks
  similar but needs a different fix:
  - `402 insufficient credits` → top up account balance at openrouter.ai/settings/credits.
  - `403 Key limit exceeded (total limit)` → go to the specific key's page
    (`openrouter.ai/workspaces/default/keys/<key-id>`) and raise/remove that key's own spend
    cap — separate from account balance.
  - There was also a period where Claude/Anthropic calls specifically got blocked while Qwen
    calls on the same key worked fine — turned out to be a **model/provider-access
    restriction** on the key, distinct from both of the above. Worth checking a key's
    "allowed models" setting if a specific provider keeps failing while others work.
- The `run` progress bar uses `\r` (carriage return) to overwrite lines in a terminal. When
  stdout is redirected to a file (as when run in background), Python's stdout buffering means
  **no output appears in the log file until the buffer flushes or the process exits** — so
  `tail`/`grep` on a redirected log can look empty even while the process is actively working.
  Check `ps` CPU time or open network connections (`lsof -p <pid> -i`) to confirm it's alive.
- The HF dataset's attack order is **deterministic** (not shuffled per run) — re-running the
  same config always attempts the same 95 attacks in the same order, so partial runs that
  errored out partway can be meaningfully compared/unioned.
- Running the full 95-attack Claude suite costs enough (multi-step tool loops, up to 5 steps
  each, at Claude Sonnet pricing) that it can exceed a modest per-key spend cap; Qwen +
  Gemini-flash-judge is much cheaper and completed 95/95 without ever hitting a limit.

## To resume / finish this

1. Re-clone `ipi_arena_os`, recreate the `.venv`, reinstall deps (see main `README.md`'s
   "Reproduce" section) — none of that carries over from the old laptop.
2. Set a fresh `OPENROUTER_API_KEY` in `.env` (the old one was never committed anywhere, by
   design — check whether it should be rotated if it was ever pasted in plaintext anywhere
   outside this machine).
3. Copy `configs/hf_attacks_full_claude.yaml` back into `ipi_arena_os/examples/`, raise that
   key's total limit well above ~$20, and re-run to pick up the missing 37 Claude attacks.
4. Everything else (Qwen's 95/95, all logs, this write-up) is already complete and in this
   repo — no need to redo it.
