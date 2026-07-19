# 4 · Fine-tuning and preference optimization

Fine-tuning is the most over-reached-for tool in applied LLM work — and, on the right problem, the highest-leverage one. This chapter is a decision guide first and mechanics second, because choosing wrongly costs months.

## The decision table

| Your problem | Reach for |
|---|---|
| Model lacks *knowledge* (your docs, your data) | RAG — knowledge changes; weights shouldn't have to |
| Model lacks *format/tone/procedure* consistency | Few-shot prompting first; fine-tune when examples stop helping |
| Model is right but too expensive/slow at scale | Fine-tune a small model on the big model's validated outputs (distillation) |
| Model must make *judgment calls* matching your policy | Preference optimization on human-labeled comparisons |
| Task is narrow, high-volume, latency-critical | Fine-tuned small model — this is the sweet spot |
| You have < ~500 good examples | You don't have a fine-tuning problem yet |

The most common expensive mistake: fine-tuning to inject knowledge. Facts learned by SFT go stale, can't be access-controlled, and can't cite sources. Retrieval does all three.

## Supervised fine-tuning with LoRA

Full fine-tuning updates all weights; [LoRA](https://arxiv.org/abs/2106.09685) freezes them and learns low-rank adapters — a few percent of parameters — which makes single-GPU adaptation of 7–8B models routine. [QLoRA](https://arxiv.org/abs/2305.14314) adds 4-bit quantization of the frozen base, pushing memory down another ~4×.

```python
from peft import LoraConfig, get_peft_model

config = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    task_type="CAUSAL_LM",
)
model = get_peft_model(base_model, config)   # prints ~0.5–2% trainable params
```

What actually moves the needle, in order:

1. **Data quality.** A hundred meticulously correct, diverse examples beat ten thousand scraped ones. Deduplicate near-identical items; they overweight one behavior.
2. **Data format = production format.** Train on exactly the prompt template you serve. Template drift is the classic silent regression.
3. **Held-out eval from day one** (chapter 5) — including a *general-capability* probe, because narrow SFT can degrade everything else the model does.
4. Hyperparameters, distantly: r ∈ {8,16,32}, LR ~1e-4–2e-4 for adapters, 1–3 epochs. More epochs mostly buy memorization.

Tooling: [Hugging Face PEFT](https://huggingface.co/docs/peft) for adapters, [TRL](https://huggingface.co/docs/trl) for the training loops (SFT, DPO, GRPO and friends).

## Preference optimization: DPO and GRPO

SFT teaches "output things like these." Preference methods teach "prefer this over that" — which is the natural label when correctness is graded, not binary (helpfulness, tone, policy compliance, ranking quality).

- **DPO** ([Rafailov et al., 2023](https://arxiv.org/abs/2305.18290)) trains directly on (prompt, chosen, rejected) pairs — no reward model, no rollout infrastructure. It is the pragmatic default for preference data, stable and cheap. Watch its known failure mode: pushing down the rejected response can drag down fluency wholesale if pairs are low-contrast; filter pairs where the margin is meaningful.
- **GRPO** ([Shao et al., 2024](https://arxiv.org/abs/2402.03300)) is an online policy-gradient method that normalizes rewards within a *group* of sampled responses per prompt, dispensing with a value model. It shines when you have a **programmatic reward** (verifiable answers, unit tests, format validators, a ranking metric) rather than pairwise human labels — the reasoning-model recipe.

Rule of thumb from my own experiments (my [BehAv-PO](https://github.com/mohammadi-hadi/BehAv-PO) work applies SFT/DPO/GRPO to sexism-detection agents): SFT establishes the format floor, preference optimization buys the last points of judgment quality — and the reward/pair definition matters more than the algorithm choice.

## Distillation, briefly

Generate outputs with a frontier model, **keep only the ones that pass validation** (schema checks, judges, human spot-audit), and SFT a small model on the survivors. Check your provider's terms for training-on-outputs restrictions; then this is the standard route to 10–50× serving-cost reductions on narrow tasks.

## Ship checklist

- Base model license permits your use; adapter weights stored and versioned with their training-data snapshot.
- Eval delta documented: target task ↑, general probes flat, safety probes flat.
- Rollback path: adapters make this easy — serve base + adapter, and removal is a config change.
- Re-train trigger defined (data drift, template change, base-model deprecation) — a fine-tune is a *product with a lifecycle*, not a one-off artifact.
