# CLOUD-FIRST INFERENCE POLICY — fleet-wide architectural decision

_Locked-in 2026-05-06 by Finn. Applies to every agent in the OpenClaw fleet (Otto, Edith, Argus, Bridge, future agents). Encoded in each agent's BOOTSTRAP/AGENTS file under a "Cloud-first inference" heading. This document is the canonical Atlas-corpus reference for the policy, its scope, and its rationale._

## The rule

```
Output < 1K tokens AND context < 4K AND no parallel demand → local OK
Everything else → cloud
```

Default inference target: **Ollama Pro cloud endpoint** (Anthropic-hosted via the runtime's standard inference path). Local Ollama is permitted only when **all three** of these conditions hold simultaneously:

1. **Output < 1K tokens** — the response Argus is asked to produce, the summary Otto is composing, the review note Edith is writing. If the inference is expected to generate more than ~1K tokens of output, route cloud.
2. **Context < 4K** — the input prompt plus retrieved context plus tool results combined. If the agent is loading a long document, a multi-file diff, or anything else that pushes input above 4K, route cloud.
3. **No parallel demand** — no other inference is queued or running on the M1 at the same time. If a cron is firing while a heartbeat is also active, both go cloud — even if individually they would qualify as local.

For everything else — sub-agent spawns, validators, rubric judges, content generation, multi-source summarization, deep-dive analysis, anything Atlas does — **use cloud, no exceptions**.

## Implementation

The inference helper at `~/.openclaw/workspace/_shared/inference.py` (or wherever the canonical wrapper lives at the time of read) is the single point of routing. It:

1. Sets `OLLAMA_HOST` to the cloud endpoint by default.
2. Reads an environment override `OLLAMA_HOST_OVERRIDE_LOCAL` for the small-job exception path. When set, AND when the three conditions above are all satisfied, the wrapper will resolve to the local Ollama socket. Otherwise it ignores the override.
3. Logs the routing decision into `~/.openclaw/logs/inference-routing.jsonl` so post-hoc audits can confirm which jobs went where.

Each agent's BOOTSTRAP/AGENTS file restates the policy in agent-specific terms (which models to default to, which to escalate to, what to do when cloud is unavailable). The wrapper is the enforcement layer; the agent files are the operational layer.

## Why

Three reasons, in order of weight.

**1. Paid Ollama Pro is a paid monthly subscription.** Unused capacity expires worthless at the end of the billing cycle. Routing default-cloud means we get value out of capacity already paid for. Routing default-local means we're paying for unused cloud while burning M1 thermal budget on inference that has no marginal cost benefit.

**2. M1 thermal/memory stability under parallel load is a real constraint.** Running two non-trivial Ollama jobs simultaneously on Kevin's MacBook Pro (M1 Max, 64GB) thrashes the memory subsystem and causes both to slow significantly — sometimes triggering watchdog timeouts on cron jobs that would otherwise complete cleanly. Cloud routing eliminates this entirely. The constraint isn't "the M1 can't run inference" — it's "the M1 can't run two of them while a heartbeat is also dispatching tool calls and a Sanity GROQ query is in flight." Cloud-first routes around the contention.

**3. Quality is load-bearing, especially for Edith and Argus.** Edith's reviews inform publish decisions across P&P, the wiki, and OS agency content. Argus's digests inform Atlas's coaching. A wrong review or a wrong digest cascades downstream. Cloud models (claude-sonnet-4-6 default, claude-opus-4-6 for edge cases, claude-haiku-4-5-20251001 for high-volume structured summarization) are stronger than what the local Ollama install can run, and the cost-per-call at Edith/Argus volumes is small enough that quality dominates the cost-quality tradeoff at every comparison point we've checked.

## Scope and exceptions

**In scope** (the cloud-first default applies):
- All sub-agent spawns by any agent
- All validator and rubric-judge calls
- All content generation (P&P drafts, wiki ingest summaries, OS marketing copy, Otto diary entries)
- Edith's review work
- Argus's digest summarization and anomaly classification
- Atlas's coaching and lesson-promotion work
- Anything Bridge writes to external systems (proper context to format the write correctly comes from cloud inference)

**Permitted local** (when the three conditions all hold):
- Quick parsing tasks: parsing a small JSON response, classifying a short string, deciding which of two tools to call.
- Heartbeat status emissions when an agent has nothing to report (these are template-generated and don't need real inference at all, but if they do invoke Ollama, local is fine).
- Argus's heuristic layer is **not inference at all** — threshold checks, statistical deltas, exit-code parsing run in plain Python. The cloud-first policy doesn't apply because no model is invoked. Argus's inference layer (digest narrative summaries, ambiguous classifications) is in scope and routes cloud.

**Hard local-prohibited** (cloud always, regardless of size):
- Any work that determines a publish decision, a money decision, a write to an external system, or any output Atlas will coach from. Even if the inference is short and context is small, route cloud — the consequence-cost is too high for the marginal compute savings.

## Failure mode

If cloud inference is unavailable (Anthropic outage, gateway error, API key revoked, rate limit hit), the policy says: **stop. Do not silently degrade to local.** File an Atlas ask in the appropriate channel. The fleet's correctness depends on cloud routing for the load-bearing inferences; quietly running them locally produces output that Atlas may coach from without realizing it was lower-quality.

The only inferences that may proceed on local during a cloud outage are the explicit small-job exception cases (output < 1K AND context < 4K AND no parallel demand) — and even those should log the fallback for post-incident review.

---

_End policy. The exact rule is reproduced verbatim in each agent's BOOTSTRAP/AGENTS file so it survives even if this corpus document is unavailable. This file is the canonical reference Atlas should cite when coaching about routing decisions._
