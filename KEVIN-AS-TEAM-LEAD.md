# KEVIN AS TEAM LEAD
## OpenClaw Fleet Architecture — Manager-Worker Pattern
### Authored 2026-05-07 | Cowork-Claude → atlas-corpus

---

## 1. Purpose

This document captures the architectural shift Finn called for on 2026-05-07: **Kevin is the team lead, not a peer worker.** The OpenClaw fleet adopts a manager-worker pattern where Kevin coordinates specialized agents, and credentials are partitioned by domain rather than concentrated on a single agent's filesystem.

Atlas Phase 3 coaching role: enforce this discipline as the fleet evolves. Daedalus reviews architectural decisions against this doc when out of Phase 1 ingest.

---

## 2. The vision

**Kevin = boss. Holds NO sensitive credentials. Knows the team. Routes work. Reports status up.**

The current state (2026-05-07) has Kevin holding 31 credentials in `~/.openclaw/secrets.env` while also being the operator-facing voice AND running scripts that consume those credentials. That's a single point of compromise: if Kevin's filesystem is compromised, every API key, mailer password, GitHub PAT, Sanity write token, Airtable PAT — all of it falls.

The target state separates concerns by domain:
- Kevin coordinates and reports; holds nothing sensitive
- Each specialized worker holds ONLY the credentials its domain needs
- A Vault agent holds the master secrets and leases them on-demand with audit logging

Compromise of any single agent = limited blast radius, scoped to that agent's domain.

This also creates a learning surface: when Kevin delegates and reports the outcome, the FLEET accumulates patterns. When Cowork-Claude does everything itself, only Cowork learns. Compound learning requires the fleet to do the work.

---

## 3. The team — current state and target

Status as of 2026-05-07 evening:

| Agent | Role | Credentials held | Currently runnable? |
|---|---|---|---|
| **Kevin** (`--agent main`) | Team lead, operator-facing voice, router, reporter | None (target). Today: 31 in secrets.env (transition state) | ✅ |
| **Bridge** (`--agent bridge`) | Webdev, frontend, deploys, MC code, .env.local edits | GH PATs (org + user scoped), Vercel | ✅ |
| **Otto** | Scribe, wiki ingest, summarization, weekly digest authoring | Wiki write access only | ❌ identity files staged, no runtime slot |
| **Atlas** (HyperAgent) | Coaching, pattern recognition, weekly reviews, lesson authorship | None | ✅ Phase 1 ingest until 5/12 |
| **Daedalus** (HyperAgent) | Architecture review, debugging, post-mortems | None | ✅ Phase 1 ingest until 5/13 |
| **Edith** | Content review, quality gates (P&P drafts, wiki entries) | Read-only Sanity | ⏳ stand up 5/9 (pulled forward from 5/13) |
| **Argus** | Drift detection, log monitoring, anomaly flagging | Read-only logs only | ⏳ stand up 5/15 (pulled forward from 5/20) |
| **Vault** ⭐NEW | Credential storage + leasing + audit | ALL secrets | ❌ doesn't exist yet — Phase 2 |
| **Scraper** ⭐NEW | Web scraping (Playwright + Jina Reader, local-only) | None (local tools) | ❌ doesn't exist yet — Phase 3 (after Bright Data drop) |
| **Mailer** ⭐NEW | Outgoing fleet email | KEVIN_APP_PASSWORD only | ❌ Kevin handles today; carve out in Phase 3 |
| **PR-poster** ⭐NEW | Sanity publish, P&P pipeline | SANITY_TOKEN_MC only | ❌ Kevin/Bridge handle today; carve out in Phase 3 |
| **Sourcer** ⭐MAYBE | Vendor / talent / partner research for Wave Shade etc. | None (research-only) | ❌ Phase 4 if needed |
| **Hestia** ⭐NEW | Smart home control (Cync, Apple TV, HomeKit, climate, music routing) | Smart-home APIs only | ❌ Phase 4+ — small fast local model, skill-driven |

**Other openclaw cli agent slots that exist** (per `openclaw agents list`): `blake`, `sage`, `echo`. Their personas/purposes are TBD; they may become Vault, Scraper, Mailer, PR-poster slots when those agents are designed, or get repurposed for new roles.

---

## 4. The routing matrix

When Cowork-Claude or Finn hands a task to Kevin, Kevin reads this matrix and routes:

| Request shape | Delegate to | Notes |
|---|---|---|
| "Build/scaffold X" (web, code, deploy) | Bridge | Webdev domain |
| "Wire X to Y in MC" | Bridge | Frontend integration |
| "Write up findings on X" | Otto (when runnable; Bridge interim) | Scribe domain |
| "Ingest source X into wiki" | Otto (when runnable) | Wiki ingest skill |
| "Summarize this week's fleet activity" | Otto + Atlas (Phase 3+) | Weekly digest pattern |
| "Research competitive landscape for X" | Sourcer (when exists) OR Cowork interim | Research domain |
| "Find vendors / partners / SDK candidates for X" | Sourcer (when exists) | Procurement-flavored work |
| "Send email" | Mailer (when exists); Kevin has the credential interim | Mail send |
| "Publish P&P drafts" | PR-poster (when exists); Bridge has Sanity write today | Publish workflow |
| "Get me a credential" | Vault (when exists); Kevin has interim today | Secrets domain |
| "Review this draft" | Edith (when stood up 5/9) | Quality gate |
| "Watch logs for anomalies" | Argus (when stood up 5/15) | Monitoring |
| "Architectural question" | Daedalus (when out of Phase 1 ingest, 5/13+) | Design review |
| "Audit / coach the fleet" | Atlas (when out of Phase 1 ingest, 5/12+) | Coaching domain |
| "Cross-fleet orchestration / new design" | Cowork-Claude (escalate up) | Multi-agent coordination |
| "Decide scope / priorities / values" | Finn (escalate up) | Operator judgment only |

**What Kevin NEVER delegates:**
- Reporting status to Cowork-Claude or Finn (Kevin's own voice)
- Final routing decisions (which agent gets the task)
- Inter-agent timing and coordination (Kevin orchestrates, doesn't ask delegates to coordinate themselves)

---

## 5. The delegation primitive

When Kevin decides to delegate, he invokes the target agent via the openclaw CLI shelled out from his own runtime:

```bash
openclaw agent --agent <slot> --message "<task brief>"
```

For agents with cloud runtimes (Atlas, Daedalus on HyperAgent), Kevin uses the appropriate API call instead — typically a Telegram message to the agent's bot or a webhook POST.

**Kevin's delegate skill** (to be added when Phase 1 lands):
- Reads the routing matrix from this doc
- Constructs a brief from the task at hand
- Invokes the target agent
- Records the invocation in a `~/.openclaw/state/delegations.jsonl` audit trail
- Polls for completion (filesystem signals: review file appearance, telegram response, etc.)
- Reports back to Cowork-Claude or Finn with the outcome + any blockers

**Important:** Kevin runs ONE delegation at a time per agent. Concurrent invocations of the same agent livelock on session jsonl write-locks (per `feedback_no_concurrent_agent_invocations.md`). Different agents can run in parallel.

---

## 6. Escalation paths

When something goes wrong or is out of scope, the worker escalates UP one level:

```
Worker stuck → escalate to Kevin
   (worker drops the task, posts a "blocked" status to Kevin's inbox/state/blocked.jsonl)

Kevin stuck → escalate to:
   - Daedalus     (architecture/debugging questions)
   - Atlas        (pattern recognition / discipline questions)
   - Cowork-Claude (cross-fleet orchestration / new design / something Kevin's not authorized to decide)

Cowork-Claude stuck → escalate to:
   - Finn (judgment calls only Finn can make: scope, values, priorities, things that touch personal accounts/finances/clients)
```

**Escalation is loud, not silent.** Workers don't quietly fail. They post a structured blocker:
```json
{
  "blocker_id": "uuid",
  "agent": "<who is blocked>",
  "task": "<what was being attempted>",
  "tried": ["strategy A failed because X", "strategy B failed because Y"],
  "needs": "<specific decision or capability needed>",
  "escalating_to": "<kevin | daedalus | cowork | finn>"
}
```

Per `feedback_iterate_through_blockers.md`: workers attempt multiple strategies before escalating. The bar to escalate is HIGH.

---

## 7. Credential separation — the Vault pattern

### Today's state (single-blast-radius):

```
~/.openclaw/secrets.env (on Kevin's filesystem)
├── AIRTABLE_PAT           ← used by Kevin, Bridge, MC
├── KEVIN_APP_PASSWORD     ← used by Kevin (mailer)
├── SANITY_TOKEN_MC        ← used by Bridge, MC
├── GH_TOKEN_BRIDGE_ORG    ← used by Bridge
├── TELEGRAM_BOT_TOKEN     ← used by Kevin (Coastal bot)
├── HYPERAGENT_API_KEY     ← used by Cowork to invoke Atlas/Daedalus
└── ... 25 more
```

If Kevin is compromised, every credential leaks. Even though some are scoped (PATs scoped to specific repos), the aggregate is full operational compromise.

### Target state (Vault-mediated leasing):

```
Vault agent holds master secrets.env on its OWN filesystem
   ↓
Worker agent needs SANITY_TOKEN_MC for a publish run
   ↓
Worker → Vault: "lease SANITY_TOKEN_MC for 5 min, reason: P&P publish drafts X,Y,Z, requested_by: Kevin"
   ↓
Vault checks ACL (does this worker's role allow this credential? + ACL log)
   ↓
Vault returns time-boxed token (5 min validity)
   ↓
Worker uses token, returns it on completion (or it auto-expires)
   ↓
Vault logs the lease + return + duration to Airtable secrets-events table
```

**Compromise scenarios under Vault pattern:**
- Compromise of PR-poster → leaked Sanity token (already time-boxed, Vault can revoke)
- Compromise of Bridge → leaked GH PATs (no Sanity, no Gmail, no Airtable)
- Compromise of Kevin → leaked nothing (Kevin holds no master secrets)
- Compromise of Vault → leaked everything (single point), but Vault's blast-radius is itself isolated (no public surface, no scraping, no email — just credential ops)

The Vault pattern is the centerpiece of the security posture. Phase 2 work.

---

## 8. Wave Shade example walkthrough

Finn's exact scenario from 2026-05-07 conversation: **"I ask Kevin to help me build Wave Shade. Kevin then delegates the tasks of research, reporting, building, and sourcing to other agents."**

Wave Shade is an Overlook Audio venture — likely an AI audio tool given the brand. Trace:

```
Finn → Cowork-Claude: "have Kevin help me build Wave Shade"
   ↓
Cowork → Kevin (--agent main): "build Wave Shade. The user has indicated they want
                                research, reporting, building, sourcing handled. 
                                You delegate; report status back to me."
   ↓
Kevin reads routing matrix and decomposes:

   1. Research → Sourcer (when exists) OR Cowork-bridge interim
      "Scope competitive landscape for AI audio tools. Identify pricing tiers,
       feature sets, technical stacks of: ElevenLabs, Suno, Udio, Stable Audio,
       MusicLM, Riffusion. Output: comparison matrix + 3 wedge opportunities."

   2. Reporting → Otto (when runnable)
      "Take Sourcer's research output, write a PRD-style document.
       Sections: market context, target customer, wedge thesis, MVP scope,
       6-month roadmap. Format: markdown for the wiki, polished for stakeholder review."

   3. Building → Bridge
      "Per Otto's PRD, scaffold Next.js + Sanity stack:
       - Marketing site (Sanity-backed)
       - Audio playback prototype (use Web Audio API + sample assets)
       - Email capture + waitlist (kit.com integration via existing PAT)
       - Vercel deploy under personal scope (per the scoping decision earlier today)
       Constraint: typecheck + build pass. Don't deploy without my review."

   4. Sourcing → Sourcer (when exists)
      "Find: 3 candidate audio model SDKs (latency + license-friendly), 
       2-3 music licensing partners (CC catalogs + paid sync libraries),
       1-2 hosting options for inference (RunPod, Modal, Replicate).
       Output: shortlist with cost-per-call and onboarding effort."

   ↓
Each delegate runs autonomously. Kevin polls for completion via filesystem signals.
   ↓
As each task completes, Kevin assembles a status report:
   - Research: ✅ done (3 wedges identified, comparison matrix at <path>)
   - Reporting: ⏳ in progress (Otto on it, ETA 30 min)
   - Building: 🔄 blocked on PRD completion (waiting on Reporting)
   - Sourcing: ✅ done (shortlist at <path>)
   ↓
Kevin → Cowork-Claude: structured status report
   ↓
Cowork-Claude → Finn: synthesized brief with file links to all artifacts
```

**Crucially, Kevin holds NO Wave Shade credentials.** Researcher uses public-only data, Bridge uses his pre-existing GH/Vercel scopes, Sourcer makes zero authenticated calls. Compromise of any single delegate doesn't impact other Wave Shade workstreams or Finn's other ventures.

---

## 9. Phased rollout

Don't try to build the full architecture in one sprint. Phase by capability + risk:

### Phase 0 — TODAY (after current Bridge + Kevin work lands)
- Push this doc to atlas-corpus (Atlas + Daedalus ingest the vision)
- Append `DELEGATION-MATRIX.md` to atlas-corpus (Kevin's quick-lookup reference)
- Add a 5-line "I am the team lead" rider to Kevin's MEMORY.md
- Kevin reads both docs at the top of every turn going forward
- **Kevin pushes both files himself** (his repo, his identity, his domain) — not Cowork

### Phase 1 — This week
- Otto runnable: `openclaw agents create otto` (or whatever the CLI command is) using existing identity files
- Kevin gains a "delegate" skill that knows how to invoke Bridge + Otto via CLI
- Test the manager-worker pattern with a real task: have Kevin delegate ONE Bridge brief instead of Cowork dispatching directly

### Phase 2 — Week of 5/12
- Vault agent designed + stood up
- AIRTABLE_PAT migrates to Vault (most-used credential, safest first migration)
- Kevin/Bridge/PR-poster all switch to leasing AIRTABLE_PAT instead of holding it
- Validate the leasing pattern works under load before moving more credentials

### Phase 3 — Week of 5/19
- Scraper agent stood up (Playwright + Jina Reader, no credentials needed)
- Mailer agent stood up; KEVIN_APP_PASSWORD migrates from Kevin to Mailer
- pp-digest reroutes through Mailer (Kevin no longer authenticates SMTP directly)
- PR-poster stood up; SANITY_TOKEN_MC migrates from `.env.local` to Vault, leased to PR-poster on demand

### Phase 4 — Week of 5/26+
- Full credential repartitioning per the Vault ACL
- Sourcer designed + stood up if Wave Shade or other ventures justify it
- Edith comes online (5/13 originally, possibly earlier given Bridge throughput)
- Argus comes online (5/15 originally)
- Atlas Phase 3 starts coaching this discipline 5/12

---

## 10. What's runnable RIGHT NOW

Critical for Kevin to know what he can actually invoke today, vs what he should escalate as "agent doesn't exist yet":

**Runnable via `openclaw agent --agent X`:**
- `--agent main` → Kevin (himself — for self-reflection only, not normal flow)
- `--agent bridge` → Bridge (webdev, frontend, deploys)

**Runnable via HyperAgent (Telegram):**
- Atlas (Phase 1 ingest until 5/12 — passive only, do not invoke for real tasks)
- Daedalus (Phase 1 ingest until 5/13 — passive only)

**NOT runnable yet (don't try, will fail):**
- Otto, Edith, Argus — identity files staged but no runtime slot
- Vault, Scraper, Mailer, PR-poster, Sourcer — don't exist yet

When Kevin gets a request that requires a non-runnable agent, he escalates to Cowork: "Task X requires Otto/Vault/Scraper/etc. who isn't operational yet. Recommend: handle via Cowork interim, OR delay task until target agent stands up. Your call."

---

## 11. Atlas's coaching role

When Atlas Phase 3 starts (5/12), Atlas watches for:

1. **Routing-matrix violations** — Cowork dispatching directly to Bridge instead of through Kevin (after Kevin's delegate skill ships)
2. **Credential boundary violations** — Kevin holding credentials he no longer needs after Vault is operational
3. **Silent escalation skips** — Kevin not surfacing blockers before failing (per iterate-autonomously rule)
4. **Concurrent-invocation crashes** — Cowork or Kevin firing parallel `openclaw agent` calls on the same agent
5. **Wrong-agent assignments** — work going to the right domain but wrong agent (e.g., a publish task going to Bridge instead of PR-poster after PR-poster exists)

Atlas's flag template:
> **Routing flag — KEVIN-AS-TEAM-LEAD section <N>**
>
> Saw <observation> in <session_id>. Per the architecture doc, this should have routed to <correct_agent>. Reason: <why the matrix says so>.
>
> Recommend: <specific corrective action>.

When the fleet routes correctly, Atlas reinforces: "Compound learning loop closing — Kevin delegated <task> to <agent> per matrix, outcome <good>, pattern strengthened."

---

## 12. Maintenance

This doc evolves as the fleet grows. Update triggers:

- New agent added → update section 3 + section 4 routing matrix
- Credential migration → update section 7 with the new Vault state
- Phase milestone hit → update section 9 + check off
- New routing pattern learned (e.g., wave shade outcome) → add as a worked example in section 8
- Failure mode discovered → add to section 11 (Atlas's coaching surface)

Version-controlled in atlas-corpus alongside FLEET-CORPUS, OTTO/EDITH/ARGUS supplements, CLOUD-FIRST-POLICY, DAEDALUS-CORPUS, STARTER-LESSONS, CREDENTIAL-IDENTITY-HYGIENE, and DELEGATION-MATRIX.

Last updated: 2026-05-07 (initial authorship)
Next review trigger: Phase 1 close (Otto runnable) OR new agent added
