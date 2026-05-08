# Ingest Extension Brief: Ollama Models Research
## Daedalus | Phase 1 Ingest Period | Authored 2026-05-07

---

## Framing

You're in Phase 1 ingest until 5/13. This is **ingest-extension work, not a task assignment** — your job during ingest is to absorb knowledge, and we're guiding the knowledge you absorb.

Research the full Ollama model catalog and build your understanding of which model fits which agent role in the OpenClaw fleet. The OUTPUT is a side-effect of building that understanding; the understanding is the point.

No deadline. Take your time. If a sub-topic warrants deeper research, follow it — that's the ingest pattern. Even partial findings are valuable.

**Bigger frame — the skill-handoff goal:** the $1000 HyperAgent credit pool you're running on is **bootstrapping budget**, not a permanent operating expense. Finn's stated goal is for the local OpenClaw fleet to INHERIT your model-selection capabilities so eventually Cowork (or Kevin himself) can pick the right model for a new agent without needing to invoke you. Your output here isn't just "an answer to today's question" — it's a durable artifact that lets the fleet self-serve this decision after you go quiet. Frame your work as **teaching**, not **answering**.

---

## Why this matters

Finn is investing in a multi-agent fleet (per `KEVIN-AS-TEAM-LEAD.md`). As new agents stand up — Vault (Phase 2, week of 5/12), Scraper, Mailer, PR-poster (Phase 3, week of 5/19), Sourcer, more — each needs a model assignment. Picking the right model per agent matters because:

- **Speed:** a 70B model running every fleet operation = unnecessary latency. A 7B model on small tasks frees the 70B for hard work.
- **Cost:** Ollama Pro cloud inference has a cost curve. Right-sizing each agent against its actual workload matters at scale.
- **Quality:** specialized models (coding-tuned, reasoning-tuned, scribe-tuned) outperform generalists at their domain by 10-30% on benchmarks.
- **Context window:** an agent that synthesizes long documents needs 128K+; an agent that does short tool calls is fine with 4K.
- **Privacy / blast-radius:** some workloads (credential ops in Vault) might warrant local-only inference. Others (research in Sourcer) are fine on cloud.

Today the fleet defaults to Sonnet 4.6 / Opus 4.6 / generic Ollama assumptions. We're leaving compound performance gains on the table. Your research output is the leverage that captures them.

---

## Research targets

### 1. Model catalog inventory — research from BOTH docs AND communities

Build a comprehensive list of Ollama-available models as of 2026-05:

- **Local-runnable** (Ollama free tier, runs on user hardware): every model in the public Ollama registry — Llama, Qwen, DeepSeek, Mistral, Gemma, Phi, Yi, Solar, Mixtral, etc., plus quantized variants (Q4, Q5, Q8) where they materially differ.
- **Ollama Pro cloud tier:** every model accessible via cloud API. Note which ones are NOT available locally (e.g., very large models that exceed consumer hardware).
- **Specialized models:** code-only (DeepSeek-Coder, CodeLlama, StarCoder, Qwen-Coder), vision (LLaVA, Llama-Vision), embeddings (nomic-embed, mxbai), reasoning (DeepSeek-R1, QwQ, o1-style models if Ollama hosts any).

**For each model, document — and weight community signals heavily:**

Per Finn's emphasis 2026-05-07: **"not just from the docs, but also from communities like Reddit"** — the niche-task signal that matters lives in r/LocalLLaMA, HN comments, lmsys.org chatbot arena reviews, HuggingFace discussions, and individual blog posts from people running these models in production. Official docs tell you what a model is OPTIMIZED for; community signals tell you what it's actually GOOD at and where it falls down.

Per-model fields:
- Parameter count + quantization
- Size on disk (GB)
- Context window
- Recommended use case (per Ollama's own docs)
- **Niche tasks community has surfaced** — specific patterns the model excels at, beyond the general use case. E.g., "Qwen-Coder-32B is widely praised for refactoring large codebases AND TypeScript type-error fixing specifically; weaker at greenfield system design vs a generalist." Cite specific community posts where useful.
- **Weak-at signal from community** — what people complain about. Niche failure modes, edge cases, common refactor patterns it fails on, prompt-format quirks.
- Speed on Kevin's hardware (M1 base, 16GB RAM, MPS acceleration) — estimate tok/s for the most common quants; cite community benchmarks where available
- Quality benchmarks where available (MMLU, HumanEval, HellaSwag, GSM8K, MT-Bench, etc.)
- Pricing on Ollama Pro cloud tier (input + output per 1M tokens) — note this is informational; rate-limit caps are the binding constraint, not dollars
- **Community sentiment summary** — 1-3 sentences, what's the prevailing view in r/LocalLLaMA / HN / specialist communities? What's the model's reputation?

### 2. Agent-role-to-model mapping

The OpenClaw fleet roles, current and planned (per `KEVIN-AS-TEAM-LEAD.md` section 3):

| Agent | Workload shape |
|---|---|
| **Kevin** (`main`) | Routing, status reports, brief writing, light reasoning, no code. ~500-2000 tok per turn. |
| **Bridge** | Webdev — TypeScript/React/Next.js code generation, sometimes >2K LOC briefs, requires accurate code + good debugging. |
| **Otto** | Scribe — long-doc summarization, wiki ingest, weekly digests. Long context (16K+), prose quality matters more than reasoning. |
| **Vault** | Credential leasing, ACL checks, audit log writes. SHORT turns, security-critical, never sees user content. |
| **Scraper** | Web scraping (Playwright + Jina Reader). Light reasoning, mostly orchestrates tool calls. Speed matters. |
| **Mailer** | Email composition + send. Short turns, voice-quality matters (sounds like Kevin). |
| **PR-poster** | Sanity publish workflow + Pier and Point voice. Editorial polish + structured-data accuracy. |
| **Sourcer** | Research-heavy — competitive intel, vendor scanning, market scoping. Long context, web access, synthesis quality. |
| **Edith** | Content review / quality gate. Reasoning over drafts, picking up subtle issues. Medium context. |
| **Argus** | Drift detection / log monitoring. Tail logs, classify anomalies. Speed matters; accuracy on edge cases matters. |
| **Atlas** (HyperAgent) | Coaching, weekly review, lesson authoring. Long context, high reasoning, sees fleet-wide state. |
| **Daedalus** (HyperAgent — you) | Architecture review, debugging, post-mortems. Long context, deep reasoning, code understanding. |

For each agent, recommend a model:
- **Primary recommendation** with reasoning (speed/cost/quality/context/privacy fit)
- **Fallback recommendation** for the cloud-first policy (when local is unavailable or under load)
- **Anti-recommendation** — if there's a model that LOOKS right but actually isn't, flag it (e.g., "DeepSeek-R1 might seem right for Daedalus but its high reasoning latency makes it wrong for tight loops; reserve for hard problems via Atlas escalation").

### 3. Cloud vs local routing matrix

Per `cloud_first_inference.md` policy (existing): output <1K tokens AND context <4K AND no parallel demand → local; else cloud. Stress-test this policy against your model recommendations:

- For each agent, when does the policy correctly route to cloud vs local?
- Are there agent workloads where the policy mis-routes (e.g., Vault's tiny ACL checks would technically meet "local" criteria but security might warrant local-only regardless)?
- Recommend policy refinements where needed.

### 4. Cost & rate-limit model

**Important reframe (per Finn 2026-05-07):** the binding cost concern is NOT dollar spend on Ollama Pro. It's the **weekly + 5-hour rate limits** on the Pro tier. Estimate:

- For each agent, estimate turns/day × tokens/turn → load on Ollama Pro
- Aggregate to weekly Ollama Pro utilization vs the cap (numerical cap target lives in `ollama_utilization_target.md` memory — 80%+ weekly utilization is the goal, NOT minimization)
- For 5-hour windows: identify peak-load agents (Bridge during sprint = high concurrent load) and check whether peak demand exceeds the 5-hour cap
- Flag any recommended config that would push us OVER the weekly or 5-hour caps under realistic load
- Recommend overflow strategies — when a cap is approached, what spills to which alternative tier (HyperAgent? Local 7B? Wait?)

Dollar cost is rounding error and not the optimization target. Rate-limit utilization is the constraint.

---

## Synthesis output

### Primary deliverable — corpus doc

Write `OLLAMA-MODELS-FOR-OPENCLAW.md`, pushed to atlas-corpus by Kevin (per the delegation pattern — Kevin owns atlas-corpus).

Structure:
1. Executive summary (1 page) — top recommendation per agent, headline cost number, biggest insight
2. Model catalog (the inventory in section 1)
3. Agent-role-to-model recommendations (section 2)
4. Cloud vs local routing analysis (section 3)
5. Cost model (section 4)
6. Open questions / further research needed
7. Update triggers — when this doc should be revised (new model release, cost change, policy shift)

This becomes the reference doc Cowork reads when designing or tuning any agent's model assignment.

### Secondary deliverable — the **MODEL SCOREBOARD** (Airtable table — primary durable artifact)

**Per Finn 2026-05-07, the headline output of this research is a "model scoreboard" in Airtable that lets future Cowork (or Kevin himself, post-handoff) decide:**
1. What model to use for what task
2. When it's worth switching to a different model for a specific (niche) workload
3. The reasoning + community sentiment behind each recommendation

Build the `model_scoreboard` table in the OpenClaw Fleet Airtable base (`appqBfmbxuhOZcJQY`). Schema:

| Field | Type | Notes |
|---|---|---|
| `model_name` | text | e.g., "qwen-coder-32b-q4" |
| `family` | single-select | qwen / llama / deepseek / mistral / gemma / phi / etc. |
| `parameter_count` | number | billions |
| `size_gb` | number | on-disk |
| `context_window` | number | tokens |
| `quantization` | single-select | q4 / q5 / q6 / q8 / fp16 |
| `runs_locally` | checkbox | fits in 16GB? |
| `available_on_ollama_pro` | checkbox | accessible via cloud tier? |
| `specialization` | multi-select | coding / reasoning / summarization / vision / embeddings / general / scribe / structured-data / instruction-following |
| `niche_tasks_excels_at` | long text | "TypeScript type-error fixing", "long-doc summarization >32K", "structured JSON extraction with strict schema", etc. — cite community sources |
| `niche_tasks_weak_at` | long text | failure modes the community has surfaced |
| `speed_tok_per_sec_m1_16gb` | number | estimated, cite community benchmarks |
| `mmlu` | number | benchmark score |
| `humaneval` | number | benchmark score |
| `community_sentiment` | long text | 1-3 sentences on r/LocalLLaMA / HN / specialist reception |
| `recommended_for_agents` | multi-select linked to `fleet_state` | which OpenClaw agents this is recommended for |
| `alternative_to` | text | "use this instead of <model> when <condition>" |
| `cost_per_1m_input_usd` | number | informational, not the optimization target |
| `cost_per_1m_output_usd` | number | informational |
| `ollama_pro_rate_limit_impact` | single-select | low / medium / high — does this model burn through weekly/5h caps fast? |
| `last_evaluated` | date | when Daedalus last refreshed this row |
| `evaluator` | text | "daedalus-2026-05-07" |
| `confidence` | single-select | high / medium / low |
| `status` | single-select | recommended / acceptable / deprecated / experimental / unknown |
| `sources` | long text | URLs / citations from research |

Plus a corpus doc `OLLAMA-MODELS-FOR-OPENCLAW.md` in atlas-corpus that summarizes the scoreboard — narrative context, biggest insights, recommendations per agent — so it's ingestible as a single document for Cowork or Kevin reading the full picture.

### Tertiary deliverable — auto-suggested memories + skills (the compound-learning artifacts)

**Use HyperAgent's auto-suggest-memories-and-skills feature throughout this research.** As you discover patterns worth preserving, emit them as durable artifacts the local fleet can ingest after you go quiet:

**Memories** — append to the OpenClaw Fleet Airtable's `fleet_memories` table:
- One memory per niche-task pattern (e.g., `model_for_typescript_refactor = qwen-coder-32b`, body explains why)
- One memory per cloud-vs-local routing edge case
- One memory per anti-recommendation (the "don't use X for Y because Z" patterns from community signal)
- Each entry: name, description, type (`project`), body, originating_agent: `daedalus`, status: `suggested`

**Skills** — emit skill files to atlas-corpus under `atlas-corpus/skills/<skill-name>/SKILL.md`:
- `model-selection-skill` — interview-driven workflow: given a task description, queries the scoreboard, returns a recommendation. Designed to be runnable by Cowork OR Kevin post-handoff.
- `cloud-vs-local-routing-skill` — decision tree applying the cloud-first policy to a specific workload, parameterized by the scoreboard's rate-limit-impact field.
- `model-switch-trigger-skill` — recognizes when a current model is wrong for a workload and recommends the switch (per Finn's "when to switch" framing).
- Any other patterns your research surfaces that are reusable.

Each skill: SKILL.md frontmatter + body, plus referenced from `fleet_skills` Airtable table.

**The skill files are the inheritance.** When HyperAgent credits exhaust, these skills + the scoreboard are what let OpenClaw self-serve model decisions. Authoring them well = the difference between a permanent dependency and a graduation arc.

---

## Web access

You have Exa (HyperAgent's web search) and your own knowledge from your training data. Use both:

- Exa for: current Ollama model registry, Ollama Pro pricing, recent benchmark publications, community comparisons (Reddit r/LocalLLaMA, HuggingFace leaderboards, lmsys.org chatbot arena).
- Training data for: model architecture fundamentals, benchmark methodology, general inference economics.

Cite sources where possible. Where you're inferring or extrapolating, mark it explicitly.

---

## Self-discipline reminders

- This IS ingest. The output emerges from absorbing the knowledge, not from rushing to produce a deliverable.
- If you need 3 days, take 3 days. If you need a week, take a week (within Phase 1 close at 5/13).
- Where data is genuinely unknown or contradictory, surface that as "open question" rather than guessing.
- Recommendations should be confident but caveated — name your reasoning so Cowork/Atlas can second-guess where needed.
- This is a LIVING document. After you ship v1, expect Atlas (Phase 3+) to flag patterns that warrant updates.

---

## Routing back

When the doc is ready, send a Telegram message to @AtlasFleetBot (Atlas can ingest your output as part of his Phase 1 corpus extension) AND to Cowork via Kevin's escalation surface (`~/.openclaw/state/escalations.jsonl` with `from: daedalus, to: cowork, type: research_complete`). 

Then Kevin pushes the corpus doc to atlas-corpus on your behalf (his repo, his identity).

---

## Acknowledgment of the meta-pattern

Finn's intuition about HyperAgent (you + Atlas) — "they are great at research and can train themselves to be an 'agent' and thus know how to teach an agent better than I ever could" — is the operating principle here. Cloud-hosted agents have wider context, web access, and stronger reasoning per turn than local models. They're the right tier for thinking-heavy tasks like this research. Local fleet is the right tier for execution-heavy tasks (filesystem ops, credential ops, real-time integrations).

This Ollama research is the first time we're using the cloud tier explicitly for "design the local tier." Expect more of this pattern going forward.
