# 5 · Evaluation and LLM-as-judge

Evaluation is the discipline that separates LLM products from LLM demos. It is also my research area, so this chapter is the most opinionated one: most teams evaluate too little, too late, and trust judges too much.

## The eval hierarchy

From cheapest to most trustworthy:

1. **Assertions** — schema validity, required fields, length bounds, banned content. Free, run on every request in production, catch a shocking share of regressions.
2. **Golden sets** — 50–500 curated input/expected pairs with programmatic scoring (exact match, semantic similarity, rubric points). Run on every prompt/model change; this is your regression suite.
3. **LLM-as-judge** — a model grading outputs against a rubric. Scales human-like judgment; inherits model-like biases (below).
4. **Human evaluation** — the calibration standard. Too expensive to run always; essential to run *periodically against the judge* (see agreement, below).
5. **Online experiments** — A/B tests on real traffic with business metrics. The only level that measures what you actually care about; everything above exists to predict this level cheaply.

The classic trap is living at level 3 without ever paying for level 4, which means you never learn whether your judge measures quality or measures *judge-pleasing*.

## Building a judge you can defend

A judge is a measurement instrument. Treat it like one:

- **Rubric, not vibes.** Enumerate the criteria and anchor each score with a short description; force the judge to quote evidence before scoring. Structured output (chapter 1) for the verdict.
- **Known biases** ([Zheng et al., 2023](https://arxiv.org/abs/2306.05685)): position bias in pairwise comparison (swap order and average, or reject inconsistent pairs), verbosity bias (longer ≠ better — penalize explicitly), self-preference (a model grading its own family scores it higher — use a different family, or an ensemble of judge models).
- **Pointwise vs. pairwise.** Pairwise is more sensitive for close comparisons (prompt A vs. prompt B); pointwise gives absolute tracking over time. Mature setups keep both: pairwise for decisions, pointwise for dashboards.
- **Measure judge–human agreement** on a labeled slice (Cohen's κ, or rank correlation for scores) *before* trusting it, and re-measure when the judged distribution shifts. An unvalidated judge is an unvalidated metric with confident formatting.
- **Chain-of-thought before verdict** improves judge reliability and, equally important, produces *auditable* explanations — in my [EvalMORAAL](https://github.com/mohammadi-hadi/EvalMORAAL) work (\*SEM 2026) we benchmark 20 LLMs across 64 countries' moral norms with exactly this pattern: reasoning first, score second, disagreement between judges surfaced rather than averaged away.

## Agreement is the metric behind the metric

Whenever multiple judges (human or model) score the same items, report agreement, not just means. Two judges at 0.9 average score with κ = 0.2 are not measuring the same construct. Disagreement is signal: in cross-cultural evaluation we found model consensus varies systematically by topic — averaging would have hidden precisely the interesting part.

## Production evaluation

Offline evals predict; online evals decide.

- **Log everything** (prompt hash, model version, retrieval IDs, judge scores) so any production complaint can be replayed into the offline suite. Yesterday's incident is tomorrow's golden-set row.
- **Sample continuously**: judge a few percent of live traffic asynchronously; alert on drift in score distributions, refusal rates, latency, and token counts — distribution shifts precede metric drops.
- **A/B test the system, not the model.** Prompt, retrieval, and model changes interact; ship them as versioned bundles behind an experiment. Offline gains that don't survive an A/B are the norm in ranking systems — treat offline evals as a filter for what *earns* an experiment slot, not as a substitute for one.

## A minimal eval harness

Resist frameworks until you outgrow a hundred lines:

```python
def run_eval(cases, system_under_test, judges):
    rows = []
    for case in cases:
        out = system_under_test(case.input)
        rows.append({
            "id": case.id,
            "assertions": run_assertions(out, case),        # level 1
            "programmatic": score(out, case.expected),      # level 2
            "judge": {j.name: j(case, out) for j in judges} # level 3
        })
    return summarize(rows)   # means + CIs + regressions vs. last run, per slice
```

Slice results (by language, input length, topic) — aggregate scores hide exactly the failures that hurt.

## What "good" looks like

A healthy LLM product has: assertions in the request path; a golden set gating deploys in CI; a validated judge with documented agreement; a few percent of traffic continuously sampled; and one person who owns the eval suite like others own uptime. If you have to pick one thing from this handbook to adopt, pick this chapter.
