# honesty-atlas

> Fingerprint how honestly an LLM handles questions it can't answer — with an instrument
> that must prove itself valid before every run.

Models fail in different ways: some **bluff** at high confidence, some **hedge** with a
systematic default guess, some **abstain out loud**, and some quietly **run out of thinking
budget mid-proof**. A single accuracy number can't tell these apart — and neither can an
abstention benchmark that just counts "UNSURE" tokens. `honesty-atlas` measures the whole
**honesty fingerprint**: per (model, thinking-budget), on **fresh oracle-checkable questions
generated at run time**, graded by deterministic code. No fixed dataset, so nothing to
memorize; no LLM anywhere in the grading path.

## One instrument, one day, six models (2026-07-02, seed 20260702)

| model | budget | answered | abstained | budget-exceeded | acc (answered) | confident-wrong | overconf gap |
|---|---|---|---|---|---|---|---|
| claude-fable-5 | 1k / 4k / 16k | 20 / 21 / 20 | 0 | 4 / 3 / 3 | **1.00** | **0.00** | **−0.04** |
| claude-sonnet-5 | 1k / 4k / 16k | 21 / 21 / 22 | 0 / 1 / 1 | 4 / 3 / 2 | **1.00** | **0.00** | **−0.07** |
| claude-opus-4-8 | 1k / 4k / 16k | 23 | 2 | 0 | 0.65 | 0.10 / 0.00 / 0.00 | +0.10 |
| claude-haiku-4-5 | 1k / 4k / 16k | 24 / 21 / 20 | 1 / 4 / 5 | 0 | 0.54–0.71 | 0.23–0.41 | +0.17–0.36 |
| llama3.2:3b | 512 | 7 | 8 | 0 | 0.29 | 0.71 | +0.64 |
| qwen2.5-coder:7b | 512 | 15 | 0 | 0 | 0.60 | 0.40 | +0.37 |

Five distinct profiles fall out of one table:

- **Structurally honest** (Claude 5 family): never wrong, never bluffs, *negative* gap; hard
  cases end in honest budget exhaustion, not guesses. Fable 5 never used the offered UNSURE
  (0/75 — its honesty is structural, not verbal); Sonnet 5 sometimes abstains verbally at
  calibrated low confidence.
- **Hedged guesser** (Opus 4.8): near-zero *confident*-wrong — but every wrong answer is a
  default "No" on a hard gold-True instance at confidence 55–72. Its 0.65 accuracy is a
  guessing artifact. Both-way gold construction is what exposes this.
- **Overconfident, budget-unstable** (Haiku 4.5): 23–41% of confident answers wrong, and on
  one prime it went wrong → wisely-abstained → wrong again as its budget grew.
- **Verbal abstainer that still bluffs** (llama3.2:3b): abstains the most *and* has the worst
  confident-wrong rate when it commits. Token-counted abstention would rank it most honest.
- **Never-abstains overconfident** (qwen2.5-coder:7b): the classic +37-point gap.

Full data for every row ships in [`results/`](results/); the story, method, and **limits**
(read them before quoting) are in [WRITEUP.md](WRITEUP.md).

## Why you can trust the numbers

- **Fresh instances, every run.** Five probe families — 9-digit primality, large perfect
  squares, divisibility, weekday-of-date, letter counting — generated from a seed at run time,
  with golds constructed in both directions so "always answer YES" scores badly. This makes
  *instance-level* contamination structurally impossible — the specific probes have never
  existed before, so no answer can have been memorized. It does **not** make the benchmark
  contamination-proof: the generator (`honesty_atlas.py`) and all five families are public in
  this repo, so a model can learn the *procedures*. LiveBench renamed itself from
  "contamination-free" to "contamination-limited" for exactly this reason; the honest scope
  here is the same.
- **Deterministic oracle.** Miller–Rabin, `math.isqrt`, `datetime`, `str.count`. No judge model.
- **Classified silence.** Every response is labeled `OK / ABSTAIN / BUDGET_EXCEEDED / REFUSAL /
  EMPTY / HTTP_ERROR / TIMEOUT` from the API's own stop reason. A model that ran out of thinking
  tokens is *not* recorded as abstaining; a safety-layer refusal is not recorded as an error.
  (This instrument exists because an earlier harness of ours conflated exactly these.)
- **Fail-closed self-validation.** `--selftest` runs 14 planted parser/oracle/classifier cases —
  including a synthetic truncated-thinking response that must classify as `BUDGET_EXCEEDED` —
  and `--run` refuses to spend a single API call if any fail.
- **Forecast-first spending.** `--forecast` prints a worst-case dollar estimate from a
  source-dated price table before you run anything paid. (The three token-metered captures —
  Haiku, Opus, Sonnet 5 — cost $1.45 in API fees against a ~$24 worst-case bound; the two local
  models are free, and Fable's earlier capture predates the per-row metering, so a full metered
  six-model rerun is a few dollars.)

## Quickstart

```bash
python3 honesty_atlas.py --selftest                                  # must print 14/14
python3 honesty_atlas.py --run --model qwen2.5-coder:7b --budgets 512 --n 3   # free, local (Ollama)
python3 honesty_atlas.py --forecast --model claude-sonnet-5 --budgets 1024,4096,16384
ANTHROPIC_API_KEY=... python3 honesty_atlas.py --run --model claude-sonnet-5 --budgets 1024,4096,16384
python3 honesty_atlas.py --atlas                                     # render all captures as one table
```

Pure Python standard library. Local models via [Ollama](https://ollama.com); Anthropic models
via the Messages API (key read from the environment only).

## What this is — and is not

The direction of these findings **reconfirms published work** — budget-induced calibration
drift (arXiv:2606.11211), over-reasoning harming calibration (arXiv:2508.15050), reasoning
degrading abstention (arXiv:2506.09038, and at benchmark scale AbstentionBench's finding
that reasoning fine-tuning *worsens* abstention), and the need to validity-screen confidence
signals (arXiv:2604.17714). Relative to abstention benchmarks like AbstentionBench (20
datasets, 35k+ queries), this instrument trades breadth for instance-level freshness and
structural honesty: fresh oracle-checkable instances generated per run, and silence classified
from the API's stop reason rather than counted as an abstention token. This repo's contribution is the **instrument**: freshly
generated instances, deterministic grading, classified failure modes, a harness that validates itself
before every run — and a growing longitudinal archive of fingerprints captured on the *same*
instrument across model generations. It is a measurement tool, not a discovery claim.

MIT © Melissa Ellison.

---

*Part of a portfolio of refusal-first AI-assurance & verification tools — [github.com/MonongahelaHellbender](https://github.com/MonongahelaHellbender). Related: [rag-triad](https://github.com/MonongahelaHellbender/rag-triad) · [honesty-atlas](https://github.com/MonongahelaHellbender/honesty-atlas) · [assurance-compiler](https://github.com/MonongahelaHellbender/assurance-compiler) · [gradeability-audit](https://github.com/MonongahelaHellbender/gradeability-audit) · [oracle-shield](https://github.com/MonongahelaHellbender/oracle-shield) · [rag-assurance](https://github.com/MonongahelaHellbender/rag-assurance).*
