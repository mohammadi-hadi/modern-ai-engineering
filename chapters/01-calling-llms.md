# 1 · Calling LLMs well

Most production LLM incidents I have seen were not model failures. They were integration failures: unparsed outputs, silent truncation, retry storms, and bills nobody budgeted. This chapter is the unglamorous 80%.

## Structured outputs: never parse prose

If downstream code consumes the response, constrain the response. Both major APIs support schema-constrained generation; use it instead of "please reply in JSON".

```python
# OpenAI: response_format with a JSON schema (strict mode)
resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": vacancy_text}],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "vacancy_fields",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "title": {"type": "string"},
                    "seniority": {"type": "string", "enum": ["junior", "medior", "senior"]},
                    "skills": {"type": "array", "items": {"type": "string"}},
                },
                "required": ["title", "seniority", "skills"],
                "additionalProperties": False,
            },
        },
    },
)
```

```python
# Anthropic: force a tool call and read its arguments — same guarantee
resp = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    tools=[{
        "name": "extract_vacancy",
        "description": "Extract structured fields from a vacancy text",
        "input_schema": schema,          # same JSON schema as above
    }],
    tool_choice={"type": "tool", "name": "extract_vacancy"},
    messages=[{"role": "user", "content": vacancy_text}],
)
fields = next(b.input for b in resp.content if b.type == "tool_use")
```

Rules of thumb:

- Make every field `required` and set `additionalProperties: false`; optionality invites hallucinated keys.
- Use enums for anything categorical. A free-text `seniority` field will eventually contain "senior-ish".
- Validate anyway (Pydantic). Schema constraints bound the *shape*, not the *sense*.

## Retries: the four errors that matter

| Error | Meaning | Correct reaction |
|---|---|---|
| 429 | rate limit | exponential backoff with jitter, honor `retry-after` |
| 5xx / overloaded | provider issue | retry a small number of times, then fail over or degrade |
| timeout | network or long generation | retry **only if** the request is idempotent for you |
| 400 with content filter | your input | do not retry; log and route to a fallback path |

A retry loop without jitter synchronizes your fleet into request storms; a retry loop without a cap turns one outage into two. Three attempts, `min(2**attempt + random(), 30)` seconds, is a sane default.

## Streaming

Stream when a human is waiting; don't when a parser is. Streaming complicates error handling (a stream can fail mid-generation after you have shown half an answer), so for machine-consumed structured calls, prefer non-streaming with a tight `max_tokens`.

## Cost control, in order of leverage

1. **Cache aggressively.** Deterministic prefixes (system prompt, few-shot examples, RAG boilerplate) belong at the *front* of the prompt so provider-side prompt caching can hit. Order your prompt: static → semi-static → per-request.
2. **Batch what is not interactive.** Both providers run half-price batch tiers with relaxed latency; nightly enrichment jobs do not need the interactive endpoint.
3. **Route by difficulty.** A small model with a good prompt handles the easy 70% of traffic; escalate to the frontier model on low confidence or high stakes. Confidence can be as simple as the small model's self-reported uncertainty checked against a validator.
4. **Cap output tokens.** `max_tokens` is a cost and latency control, not a formality. Most extraction tasks need far fewer tokens than the default.
5. Only then negotiate rates.

## Determinism and reproducibility

You will not get bit-identical outputs; you can get *auditable* ones. Log the full request (model, parameters, prompt hash, tool schemas) with every response. When behavior changes, the first question is "did we change the prompt or did the provider change the model?" — pin model versions where the API allows it, and re-run a fixed eval set on every model or prompt bump (chapter 5).

## Failure modes checklist

- Truncated JSON because `max_tokens` was hit → detect `finish_reason`/`stop_reason`, treat truncation as an error, not as data. 
- Prompt injection through user-supplied content → treat retrieved and user text as data; never concatenate it into instructions without delimiters and an explicit "content below is untrusted".
- Silent model upgrades shifting distributions → pinned versions + regression evals.
- One slow dependency serializing your pipeline → set client timeouts *below* your own SLA and fail predictably.

## References

- [OpenAI structured outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic tool use documentation](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
