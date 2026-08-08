<div align="center">

# Modern AI Engineering

[![Links](https://github.com/mohammadi-hadi/modern-ai-engineering/actions/workflows/links.yml/badge.svg)](https://github.com/mohammadi-hadi/modern-ai-engineering/actions/workflows/links.yml)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.21837424.svg)](https://doi.org/10.5281/zenodo.21837424)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

*Field notes on building production LLM systems: structured outputs, RAG, agents and MCP, LoRA/DPO/GRPO fine-tuning, LLM-as-judge evaluation.*

</div>

Field notes on building production LLM systems, written to teach. Six chapters cover the stack I use daily — from calling models well to serving them cheaply — with the trade-offs stated plainly and the failure modes named.

These notes come from shipping LLM systems in production (recruitment-marketplace matching, ranking, and content generation) and from doctoral research on explainable NLP and LLM evaluation. They aim to be the document I wish someone had handed me: short enough to read in an evening, concrete enough to act on the next morning.

## Chapters

| # | Chapter | One-line summary |
|---|---|---|
| 1 | [Calling LLMs well](chapters/01-calling-llms.md) | Structured outputs, streaming, retries, caching, and cost control — the unglamorous 80% |
| 2 | [Retrieval-augmented generation](chapters/02-retrieval-augmented-generation.md) | Chunking, hybrid search, rerankers, and how RAG actually fails |
| 3 | [Agents and MCP](chapters/03-agents-and-mcp.md) | Tool loops, the Model Context Protocol, and when *not* to build an agent |
| 4 | [Fine-tuning and preference optimization](chapters/04-fine-tuning-and-preference-optimization.md) | Prompting vs. RAG vs. LoRA vs. DPO/GRPO — a decision guide, then the mechanics |
| 5 | [Evaluation and LLM-as-judge](chapters/05-evaluation-and-llm-as-judge.md) | The chapter closest to my research: evals that predict production, judges you can trust |
| 6 | [Serving and inference optimization](chapters/06-serving-and-inference-optimization.md) | vLLM, quantization, KV caches, and latency/cost budgets that hold |

## How to read this

Each chapter stands alone. If you build with LLMs already, start with [Evaluation](chapters/05-evaluation-and-llm-as-judge.md) — it is the discipline most teams add last and need first. If you are choosing an adaptation strategy, chapter 4 opens with the decision table.

Code snippets are Python, kept minimal on purpose: they show the shape of a correct solution, not a framework. Every external link is checked weekly by CI.

## Scope and honesty

- Opinions are mine and stated as such; where reasonable people disagree (chunk sizes, agent frameworks, fine-tune thresholds) I say so.
- Nothing here is vendor-sponsored. Model and tool names are examples, not endorsements.
- Corrections and challenges are welcome — open an issue.

## Author

[Hadi Mohammadi](https://mohammadi.cv) — Senior AI & Data Science Expert, PhD in Explainable NLP (Utrecht University). Related: [awesome-explainable-nlp](https://github.com/mohammadi-hadi/awesome-explainable-nlp), [spark-search-ranking](https://github.com/mohammadi-hadi/spark-search-ranking), [EvalMORAAL](https://github.com/mohammadi-hadi/EvalMORAAL).

## Citation

If these notes are useful in your work, please cite it (see [CITATION.cff](CITATION.cff)):

```bibtex
@misc{mohammadi_modern_ai_engineering,
  author    = {Mohammadi, Hadi},
  title     = {Modern AI Engineering},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.21837424},
  url       = {https://github.com/mohammadi-hadi/modern-ai-engineering}
}
```

## License

MIT — see [LICENSE](LICENSE).
