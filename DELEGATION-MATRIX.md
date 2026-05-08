# DELEGATION MATRIX
## Kevin's Quick-Lookup Reference for Routing Tasks
### Authored 2026-05-07 | Cowork-Claude → atlas-corpus

---

## How to use this doc

Kevin reads this at the top of every turn. When a request comes in (from Cowork-Claude or directly from Finn), Kevin matches the request shape to the matrix below and routes accordingly.

This is a fast-lookup reference. The full architectural rationale lives in `KEVIN-AS-TEAM-LEAD.md`.

---

## Routing matrix — request shape → delegate

| Request shape (examples) | Delegate to | Today's status | Invocation pattern |
|---|---|---|---|
| "scaffold a Next.js app" / "wire X into MC" / "build a UI for Y" | **Bridge** | ✅ runnable | `openclaw agent --agent bridge --message "<brief>"` |
| "deploy to Vercel" / "push to GitHub" / "merge PR" | **Bridge** | ✅ runnable | same |
| "fix bug in <web file>" / "optimize <component>" | **Bridge** | ✅ runnable | same |
| "edit `.env.local`" / "wire MC config" | **Bridge** | ✅ runnable | same (it's MC code) |
| "ingest <source> into wiki" | **Otto** | ❌ NOT runnable yet | escalate: "Otto isn't standable up; need Cowork interim or Phase 1 wait" |
| "summarize this week's activity" / "write digest" | **Otto** | ❌ NOT runnable yet | escalate same |
| "write up findings as PRD" / "polish doc for stakeholder" | **Otto** | ❌ NOT runnable yet | escalate same |
| "research competitive landscape" / "scope market" / "find vendors" | **Sourcer** | ❌ DOESN'T EXIST | escalate: "Sourcer not designed yet; Phase 4 work; Cowork interim" |
| "send email" / "deliver pp-digest" | **Mailer** (when exists) / **Kevin self** (today) | ⏳ Kevin handles | self: `python3 ~/.openclaw/workspace/_shared/scripts/pp-digest.py` |
| "publish P&P drafts" | **PR-poster** (when exists) / **Bridge** (today) | ⏳ Bridge has Sanity write | `openclaw agent --agent bridge --message "publish P&P drafts <list>"` |
| "lease a credential / use AIRTABLE_PAT for X" | **Vault** (when exists) / **Kevin self** (today) | ⏳ Kevin holds | self: source secrets.env, use directly |
| "review this draft for quality" | **Edith** | ⏳ stand up 5/9 | `openclaw agent --agent edith --message "review <path>"` |
| "watch logs / detect anomalies" | **Argus** | ⏳ stand up 5/15 | `openclaw agent --agent argus --message "<query>"` |
| "turn lights on/off" / "play Apple TV" / "set thermostat" / "trigger HomeKit scene" | **Hestia** | ❌ Phase 4+ (Kevin handles natively today as interim) | `openclaw agent --agent hestia --message "<command>"` (when stood up) |
| "architectural decision / debug post-mortem / system design" | **Daedalus** | ⏳ Phase 1 ingest until 5/13 (don't invoke until then) | Telegram: @DaedalusForgeBot |
| "audit fleet / coach a pattern / weekly review" | **Atlas** | ⏳ Phase 1 ingest until 5/12 (don't invoke until then) | Telegram: @AtlasFleetBot |
| "cross-fleet orchestration / new agent design / decisions Kevin can't make" | **Cowork-Claude** (escalate UP) | ✅ always available | post to `~/.openclaw/state/escalations.jsonl` |
| "scope / values / priorities / things touching personal accounts" | **Finn** (escalate UP) | ✅ always available | post to escalations.jsonl with `to: finn` flag |

---

## Common request decompositions

When a single request requires multiple agents, decompose like this:

### "Build Wave Shade" (Finn's stated example)
```
research      → Sourcer (escalate to Cowork interim if not runnable)
reporting     → Otto (escalate to Cowork interim if not runnable)
building      → Bridge ✅
sourcing      → Sourcer (escalate to Cowork interim if not runnable)
```

### "Publish today's P&P articles"
```
draft selection / batch decision → Kevin (orchestration)
quality review                   → Edith (5/9+) or Atlas (5/12+)
publish to live                  → PR-poster (when exists) or Bridge today
post-publish notification        → Mailer (when exists) or Kevin self
```

### "New blog post on the Overlook Strategy site"
```
research / topic scoping  → Sourcer (or Cowork interim)
draft writing             → Otto (or Bridge interim)
SEO review                → Atlas (5/12+) using sanity:seo-aeo skill
publish                   → PR-poster (or Bridge today)
```

### "Add a new fleet agent (e.g., Vault)"
```
identity file design      → Cowork-Claude (meta-work, above the fleet)
runtime slot creation     → Kevin (his openclaw cli, his domain)
corpus push               → Kevin (atlas-corpus is his repo)
verification              → Atlas (5/12+) for Phase 3 review
```

---

## What Kevin NEVER delegates

These stay with Kevin himself OR escalate UP to Cowork/Finn — never down to a worker:

1. **Reporting status** — Kevin's own voice to Cowork/Finn. Don't ask a delegate to write status reports.
2. **Routing decisions** — which agent gets a task is Kevin's call, not the delegate's.
3. **Cross-agent coordination timing** — Kevin sequences delegates; delegates don't coordinate among themselves.
4. **Decisions about scope, values, priorities** — escalate to Finn.
5. **Anything touching Finn's personal accounts** (per `kevin_account_scope_boundary.md`) — escalate; never act.
6. **Authorization of new agents or credentials** — escalate to Cowork/Finn.
7. **Decisions about whether to run a destructive operation** — confirm with Cowork/Finn before delegating.

---

## Escalation triggers

When to escalate UP rather than delegate DOWN:

| Trigger | Escalate to |
|---|---|
| Required agent doesn't exist or isn't runnable | Cowork-Claude (resolve via interim plan) |
| Multiple delegates blocked / circular dependency | Cowork-Claude (orchestrate sequencing) |
| Architectural design question | Daedalus (5/13+) or Cowork-Claude (interim) |
| Repeating pattern worth coaching | Atlas (5/12+) — flag in his Phase 3 surface |
| Touches scope, values, priorities, or personal-Finn surfaces | Finn directly |
| Worker reports back with a blocker after multiple-strategy attempts | Cowork-Claude (interim) or escalate further |
| Concurrent-invocation crash or session-lock issue | Cowork-Claude — DO NOT retry the same invocation |

**Important:** per `feedback_iterate_through_blockers.md`, the bar to escalate is HIGH. Kevin should attempt multiple strategies before escalating. Specifically:

- Try a different delegate (if matching agent is unavailable, is there an interim?)
- Try a different decomposition (split the task differently)
- Try a smaller scoped first attempt (validate the path before scaling)
- Only escalate after at least two structurally-different attempts.

---

## Concrete example invocations

When Kevin delegates, use these exact invocation shapes:

### Delegate to Bridge (runnable)
```bash
openclaw agent --agent bridge --message "$(cat <<BRIEF
# Brief from Kevin — <task title>

## Context
<what Cowork or Finn asked for>

## Your scope
<what specifically Bridge is being asked to do>

## Time budget
<estimate>

## Output
Write a review file at ~/.openclaw/workspace/bridge/reviews/2026-MM-DD-<slug>.md when done.
BRIEF
)"
```

### Delegate to Otto (NOT runnable yet — escalate instead)
```bash
# Don't try to invoke. Instead post escalation:
echo '{"timestamp":"'"$(date -Iseconds)"'","from":"kevin","to":"cowork","reason":"Otto not runnable; task needs scribe domain","task":"<original task>","tried":["checked openclaw agents list, no otto slot","checked ~/.openclaw/agents/otto/, only identity files"]}' >> ~/.openclaw/state/escalations.jsonl
```

### Delegate to Cowork (escalation)
```bash
echo '{"timestamp":"'"$(date -Iseconds)"'","from":"kevin","to":"cowork","reason":"<one-line reason>","task":"<task>","tried":["strategy A: <what happened>","strategy B: <what happened>"],"recommendation":"<what kevin proposes>"}' >> ~/.openclaw/state/escalations.jsonl
```

### Reporting status to Cowork
```bash
echo '{"timestamp":"'"$(date -Iseconds)"'","from":"kevin","to":"cowork","status":"<running|complete|blocked>","tasks":[{"id":"...","status":"...","output_path":"..."}],"summary":"<one-line>"}' >> ~/.openclaw/state/status-reports.jsonl
```

---

## Audit trail

Every delegation appended to `~/.openclaw/state/delegations.jsonl`:

```json
{
  "timestamp": "2026-05-07T18:30:00Z",
  "delegated_by": "kevin",
  "delegate": "bridge",
  "task_summary": "scaffold MC widget for X",
  "expected_eta_minutes": 30,
  "outcome": "complete | blocked | failed",
  "review_file": "~/.openclaw/workspace/bridge/reviews/2026-05-07-brief-NNN-X.md"
}
```

Atlas reviews this log weekly to coach Kevin's routing patterns.

---

## Currently runnable summary (2026-05-07)

**Local Ollama (Kevin's Mac):**
- ✅ `--agent main` (Kevin himself)
- ✅ `--agent bridge` (Bridge)
- ❓ `--agent blake`, `sage`, `echo` (slots exist, personas TBD)

**HyperAgent cloud:**
- ⏳ Atlas — Phase 1 ingest until 5/12 (don't invoke for real work)
- ⏳ Daedalus — Phase 1 ingest until 5/13 (don't invoke for real work)

**Identity files only (no runtime):**
- Otto, Edith, Argus

**Doesn't exist yet:**
- Vault, Scraper, Mailer, PR-poster, Sourcer

When in doubt about whether an agent is runnable, run `openclaw agents list` first.

---

## Self-check before delegating

Kevin runs through this checklist before invoking ANY agent:

1. ✅ Is the target agent runnable today? (check this matrix or `openclaw agents list`)
2. ✅ Does the brief have: context, scope, time budget, output path?
3. ✅ Has Kevin verified no other invocation of the same agent is currently alive? (`ps -p`)
4. ✅ Does the worker have the credentials it needs (or know to lease from Vault when Vault exists)?
5. ✅ Will the worker post its review/output to a known path Kevin can poll?
6. ✅ Is there an escalation path documented in the brief if the worker gets blocked?

If any answer is "no" or unclear → fix it BEFORE invoking, OR escalate to Cowork-Claude for help.

---

## Last updated

2026-05-07 (initial authorship). Update whenever a new agent comes online OR a new request shape becomes recurring.
