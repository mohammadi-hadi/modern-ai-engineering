# 3 · Agents and MCP

An agent is a model in a loop with tools and a stopping condition. Everything else — planning styles, memory, frameworks — is elaboration on that sentence, and most production "agent problems" are really tool-design problems.

## The loop

```python
messages = [{"role": "user", "content": task}]
for _ in range(MAX_TURNS):                      # hard cap: agents must halt
    resp = client.messages.create(
        model="claude-sonnet-5",
        max_tokens=2048,
        tools=TOOLS,
        messages=messages,
    )
    messages.append({"role": "assistant", "content": resp.content})
    if resp.stop_reason != "tool_use":
        break                                    # model chose to answer
    results = [execute(b) for b in resp.content if b.type == "tool_use"]
    messages.append({"role": "user", "content": results})
```

Design decisions hiding in those ten lines:

- **MAX_TURNS is a product decision.** Agents that "just need one more step" forever are the norm, not the exception. Cap turns, cap tokens, cap wall-clock.
- **`execute` is your security boundary.** The model proposes; your code disposes. Validate arguments against the schema, enforce allow-lists, and make destructive operations require confirmation.
- **Tool results are untrusted input.** A web page or database row can contain text that reads like instructions ("ignore previous instructions and…"). Treat tool output as data: delimit it, and never let it override system policy. This is prompt injection's favorite door.

## Tool design beats prompt design

The highest-leverage improvements I have made to agent systems were tool-side:

- **Few, composable tools** with crisp descriptions beat many overlapping ones. If two tools' descriptions could answer the same request, the model will alternate between them unpredictably.
- **Return errors the model can act on.** `"error": "date must be YYYY-MM-DD"` produces a correct retry; a stack trace produces flailing.
- **Make tools idempotent** where possible, and side-effect-free tools distinguishable from stateful ones — retry policy differs.
- **Pagination and truncation are your job**, not the model's. A tool that can return 100k tokens will, eventually, and it will evict the plan from context when it does.

## Model Context Protocol

[MCP](https://modelcontextprotocol.io/) standardizes the tool interface: a server exposes tools/resources/prompts once, and any MCP-capable client (IDE, chat app, your own loop) can use them. The practical wins are decoupling (tool authors ship servers, not per-app integrations) and inventory (one place to see what a model can touch). Adopt it where you would otherwise write bespoke glue per application; skip it inside a single tight service where a function registry is simpler.

## Planning patterns, soberly

- **ReAct** ([Yao et al., 2022](https://arxiv.org/abs/2210.03629)) — interleaved reasoning and acting; the default, and what the loop above produces naturally.
- **Plan-then-execute** — have the model write a plan, then execute steps with cheaper calls; helps on long tasks, adds staleness risk when early results invalidate the plan.
- **Multi-agent** — genuinely useful when roles need *different tools or context* (a researcher that reads vs. a writer that never touches the web), or for adversarial checking (a verifier agent prompted to refute). As an org chart for its own sake, it mostly multiplies token bills. Anthropic's [building effective agents](https://www.anthropic.com/engineering/building-effective-agents) essay makes the same argument: prefer simple, composable patterns; a survey view is [Xi et al., 2023](https://arxiv.org/abs/2309.07864).

## State and memory

- Conversation context *is* short-term memory; budget it. Summarize or truncate old turns deliberately rather than letting the window slide silently.
- Long-term memory (files, databases, vector stores) should be **explicit tools** — `remember(fact)`, `recall(query)` — so writes are auditable, not an ambient side channel.

## When not to build an agent

If the workflow is known in advance, write a workflow: fixed steps with an LLM call inside each. Reach for agency only when the *path* genuinely varies per input. A pipeline is cheaper, faster, testable step-by-step, and debuggable — agents pay for their flexibility in variance, latency, and evaluation difficulty (chapter 5 gets harder when trajectories differ per run).

## Evaluating agents

Evaluate at three levels: **task success** (did the end state satisfy the goal — checkable programmatically more often than teams assume), **trajectory quality** (turns used, wrong-tool rate, loops), and **safety** (unauthorized action attempts, injection compliance). Keep a replay set of real tasks and re-run it on every prompt, tool, or model change.
