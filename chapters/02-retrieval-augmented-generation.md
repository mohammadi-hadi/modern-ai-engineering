# 2 · Retrieval-augmented generation

RAG is a search problem wearing an LLM costume. Teams that treat it as prompt engineering plateau at demos; teams that treat it as information retrieval ship. The generation step is rarely the bottleneck — retrieval quality is.

## The pipeline, and where it actually breaks

```
ingest → chunk → embed → index → [query] → retrieve → rerank → assemble context → generate → (cite)
```

In failure-analysis sessions, the distribution of blame is roughly: chunking and query mismatch first, missing content second, reranking absent third, generation last. Measure each stage separately before touching prompts.

## Chunking

- **Split on structure, not on character counts.** Headings, paragraphs, table boundaries. A chunk that starts mid-sentence retrieves mid-sentence.
- 200–500 tokens is a reasonable starting range for prose; tables and code deserve their own strategy (keep a table whole, or serialize rows with their header).
- **Attach context to every chunk**: document title, section path, date. A paragraph that says "this policy replaces the previous one" is useless without knowing which document and which date.
- Overlap is a band-aid for bad boundaries. Prefer better boundaries; add 10–15% overlap only if you cannot.

## Embeddings and indexes

- Pick an embedding model off the [MTEB leaderboard](https://huggingface.co/spaces/mteb/leaderboard) *for your language and domain*, then verify on your own retrieval eval — leaderboard rank does not survive contact with domain jargon.
- Index choice is a scale decision: brute-force cosine is exact and fine to ~1M vectors ([FAISS](https://github.com/facebookresearch/faiss) flat); beyond that, HNSW gives sub-linear search with high recall. Managed options ([Qdrant](https://qdrant.tech/), [Chroma](https://www.trychroma.com/), pgvector) trade control for operations you don't have to own — pgvector is underrated when your data already lives in Postgres.
- Store the *text and metadata with the vector*. Re-fetching from a second system doubles your failure modes.

## Hybrid search and reranking

Dense retrieval misses exact identifiers, rare names, and codes; lexical (BM25) misses paraphrases. Production systems run both and fuse:

- **Reciprocal rank fusion** is a robust, tuning-free fusion baseline: `score(d) = Σ 1/(k + rank_i(d))` with k≈60.
- Then **rerank the top 50–100 with a cross-encoder**. This is the highest-ROI component in most RAG stacks: bi-encoders buy recall cheaply, the cross-encoder buys precision where it counts.

## Query understanding

The user's query is usually not the best retrieval query.

- Rewrite conversational queries into standalone ones (resolve "what about the second one?").
- For complex questions, decompose into sub-queries and retrieve per sub-query.
- HyDE (embedding a hypothetical answer instead of the question) helps when questions and documents live in different registers — worth an experiment, not a default.

## Context assembly

- Order retrieved chunks by document and position, not by score — models read documents better than shuffled fragments.
- Deduplicate near-identical chunks; five copies of the same paragraph crowd out the answer.
- Demand citations: instruct the model to answer *only* from provided context and cite chunk IDs. Refusals ("not in the provided documents") are a feature; measure them separately from wrong answers.

## Evaluating RAG (before users do)

Build a small gold set of (question, expected source, expected answer) triples — 50 is enough to start. Track:

- **Retrieval**: recall@k of the expected source; MRR if order matters to your context window.
- **Generation**: faithfulness (is every claim supported by retrieved context?) and answer relevance — LLM-as-judge works here, with the caveats of chapter 5.
- **End-to-end**: exact/semantic match against expected answers, plus refusal rate on unanswerable questions you deliberately include.

The original RAG formulation is [Lewis et al., 2020](https://arxiv.org/abs/2005.11401); modern practice diverges from the paper but the framing — parametric memory plus non-parametric memory — still clarifies design debates.

## When RAG is the wrong tool

- The knowledge is small and stable → put it in the prompt; a 2,000-token policy does not need a vector database.
- The task needs *behavioral* change (tone, format, procedure) → fine-tune (chapter 4).
- The corpus updates faster than you can re-index coherently → fix the pipeline before blaming the model.
