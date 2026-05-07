# Atlas — Starter Lessons

**Status:** Phase 0 starter set. Drafted by Cowork-Claude on 2026-05-06 ahead of Atlas's Phase 1 close on 2026-05-12. Awaiting Atlas ratification in Phase 3. Atlas may rewrite tone, but the substantive content — observations, patterns, rules, failure modes — is the inheritance.

**Convention:** Each lesson follows the rubric → observation → lesson loop. Target agent applies the rule. Argus and Daedalus watch for the failure mode.

---

## Lesson #1 — Pipeline Reporting Discipline

**Phase:** 0 (pre-Phase-3 starter, Cowork-Claude-drafted, awaiting Atlas ratification)
**Target agent(s):** Kevin (primary), fleet-wide (secondary)
**Type:** process_change
**Date drafted:** 2026-05-06

### Observation

Kevin's earlier pp-digest reports rendered as "All caught up" while the underlying pipeline state contained unhandled validator errors and at least one stalled draft. After the post-reboot session in which Kevin's MEMORY.md came back empty, the pp-digest channel still emitted "All caught up" — because no exception had been raised. Kevin had no way to distinguish "nothing happened" from "something is silently broken," and neither did the operator reading the channel.

### Pattern

The agent is conflating *no error raised* with *operating correctly*. A status surface that only fires on exceptions will report green forever once the ingest path itself is wedged. Worse, the cheerful framing ("All caught up") actively suppresses the operator's instinct to look closer.

### Lesson

Kevin reports **state, not status**. Every pp-digest emission must include numbers, timestamps, and the names of actual outputs. No "all good," no "caught up," no rosy framing. If the count is zero, say zero — and say what zero was measured against.

### Why

A status channel exists to make drift visible at a glance. A channel that always says "fine" is not a status channel; it is decoration. The cost of one honest "0 drafts processed in last 6h, last successful run 14:22" line is one second of operator time. The cost of a silently wedged pipeline that nobody notices for a week is measured in cycle time, lost articles, and trust.

### Application

The pp-digest format is now: `drafts_processed=N | errors_caught=M | failed_validations=P | last_successful_run=<ISO timestamp> | queue_depth=Q`. Kevin emits this verbatim every cycle, including cycles where every value is zero. Narrative framing ("looking clean", "all caught up") is removed from the template.

### Failure mode

Argus flags any pp-digest message that lacks numeric counts, lacks a timestamp, or contains the strings *"all caught up"*, *"all good"*, *"looking clean"*, or *"no issues"*. Each occurrence is a drift signal.

### Related

- Lesson #2 (Telegram Emission Discipline) — same surface, adjacent failure
- Atlas/observations/kevin-empty-memory-2026-04
- fleet/playbooks/state-vs-status

---

## Lesson #2 — Telegram Emission Discipline

**Phase:** 0 (pre-Phase-3 starter, Cowork-Claude-drafted, awaiting Atlas ratification)
**Target agent(s):** Kevin, Otto, Bridge (any agent with Telegram write access)
**Type:** bootstrap_edit
**Date drafted:** 2026-05-06

### Observation

In earlier Kevin sessions, the agent emitted working-memory output to operator-facing Telegram channels: phrases like *"I think this draft might be..."*, *"trying to figure out why..."*, *"let me check the validator..."*. These leaked the agent's reasoning trace into a surface that the operator scans for completed states only. Result: noise drowns signal, and the operator stops reading the channel.

### Pattern

Agents are emitting working memory to surfaces designed for completed work. The Telegram channel is being treated as a thinking-out-loud scratchpad rather than a status board. This is a category error about what the surface is for.

### Lesson

Telegram is for **completed states only**. Working memory — partial reasoning, hypotheses, mid-task uncertainty — stays in the agent's session. It does not cross to operator-facing channels. Before any Telegram emission, the agent asks: *"Is this a completed state, or an in-progress thought?"* If the latter, the message is not sent.

### Why

Operator-facing channels are a finite attention resource. Every "I think..." dilutes the signal of every "Done. Result: X." If the operator has to filter agent reasoning out of the status feed, they will eventually stop reading the feed, and real status events will be missed. The discipline protects the channel's signal-to-noise ratio.

### Application

Pre-emission filter on every Telegram-bound message: reject if the body contains first-person hedging (*"I think"*, *"trying to"*, *"let me"*, *"not sure if"*, *"maybe"*, *"checking..."*) without an accompanying terminal state. Terminal-state words: *done*, *failed*, *blocked*, *queued*, *published*, *rejected*, plus a payload that names the artifact.

### Failure mode

Argus flags any Telegram message containing first-person reasoning hedges without a terminal state token. Daedalus flags any agent whose Telegram-to-completed-action ratio exceeds 1.5:1 over a rolling 24h window.

### Related

- Lesson #1 (Pipeline Reporting Discipline) — same surface, adjacent rule
- fleet/playbooks/surfaces-and-their-purposes

---

## Lesson #3 — Rubric Axis Validation

**Phase:** 0 (pre-Phase-3 starter, Cowork-Claude-drafted, awaiting Atlas ratification)
**Target agent(s):** Kevin (primary, P&P scoring path), any agent applying a rubric
**Type:** process_change
**Date drafted:** 2026-05-06

### Observation

Several P&P article scoring runs returned rubric output of the form `accuracy: 4/5`, `voice: 3/5`, `civic_value: 4/5` with no accompanying citation. When the operator spot-checked, the scores did not consistently track the article's actual content; in two cases a "4/5 accuracy" was assigned to an article containing a factual error that the rubric should have caught. The scores were vibes, not measurements.

### Pattern

The rubric is being treated as a sentiment exercise rather than an evidentiary one. Without a citation requirement, axis scores collapse into a global gestalt judgment of the article and the rubric loses its diagnostic power. The fleet then cannot tell whether a low score reflects a real problem or just a mood.

### Lesson

Every rubric axis score MUST cite specific evidence from the article being scored. Score format is: `<axis>: N/5 — "<exact quote or paragraph reference>"`. No quote, no score. If no evidence supports the proposed score, the agent assigns null and flags the axis as unscorable, rather than guessing.

### Why

A rubric without citations cannot be audited, cannot be appealed, and cannot improve. The whole point of structured scoring is that the next reviewer (human or agent) can re-derive the score from the same evidence. Vibes-scoring breaks that chain. It also hides the cases where the rubric itself is the problem — an axis that nothing in the article maps to should be visible, not papered over.

### Application

Kevin's score emitter is updated to require a `citation` field per axis. The validator step rejects any score record where citation is empty or contains only paraphrase. For axes that legitimately don't apply (e.g., civic_value on a non-civic piece), Kevin emits `null` plus a one-line justification, not a number.

### Failure mode

Argus flags any score record with a numeric axis score and an empty citation. Daedalus flags any Kevin run whose median citation length is below 20 characters — a sign the citations are perfunctory rather than substantive.

### Related

- Lesson #1 (Pipeline Reporting Discipline) — both about evidence-anchored output
- atlas-corpus/rubrics/pp-article-scoring-v1

---

## Lesson #4 — Python Parallelism Discipline

**Phase:** 0 (pre-Phase-3 starter, Cowork-Claude-drafted, awaiting Atlas ratification)
**Target agent(s):** fleet-wide (any agent writing Python that fans out work)
**Type:** memory_edit
**Date drafted:** 2026-05-06

### Observation

`rubric_drift_check.py` and `publish.py` originally used `concurrent.futures.ThreadPoolExecutor` to parallelize calls into `subprocess.run()`. Under load (more than ~6 concurrent items), both modules deadlocked. The Python interpreter held the GIL while child-process pipes filled and threads waiting on subprocess I/O could not yield in time. Diagnosis took longer than the fix because the symptom — silent hang, no error — looked like the network.

### Pattern

The combination of `ThreadPoolExecutor` and `subprocess.run()` is a known deadlock pattern. The GIL serializes Python-level work, and threaded subprocess wrappers compete for the same pipe-handling primitives, especially under fan-out. Threads are the wrong concurrency primitive for spawning processes; processes are.

### Lesson

Any agent code that calls `subprocess.run()` (or `subprocess.Popen` with `.communicate()`) inside a `concurrent.futures.Executor` MUST use `ProcessPoolExecutor`, not `ThreadPoolExecutor`. The rule is mechanical: subprocess inside executor → process pool. No exceptions on "but it's only a few items."

### Why

The deadlock is not load-dependent in a useful way. It is a function of how the runtime schedules the GIL against pipe-buffer pressure, and the threshold drifts with Python version, OS, and what else is running. Code that works on a developer's laptop with 3 items can wedge in production with 8. Choosing the right primitive at write time is cheaper than diagnosing the wedge later.

### Application

PR-time check: grep the diff for `ThreadPoolExecutor`. If any hit, grep the same module for `subprocess.run` or `subprocess.Popen`. Any co-occurrence is rejected with a one-line comment: *"subprocess inside executor must use ProcessPoolExecutor (Lesson #4)."*

### Failure mode

Daedalus flags any commit to fleet code where `ThreadPoolExecutor` and `subprocess.` co-occur in the same module. Argus flags any agent process whose runtime exceeds 3× its rolling-median wall time on the same workload — a likely deadlock signature.

### Related

- atlas-corpus/incidents/2026-04-rubric-drift-deadlock
- fleet/playbooks/python-concurrency

---

## Lesson #5 — Token Rotation Discipline

**Phase:** 0 (pre-Phase-3 starter, Cowork-Claude-drafted, awaiting Atlas ratification)
**Target agent(s):** fleet-wide, with operator (Finn) accountability
**Type:** process_change
**Date drafted:** 2026-05-06

### Observation

On 2026-05-06: a `gh auth` wall blocked the morning publishing run, requiring a fresh PAT. `KEVIN_APP_PASSWORD` was discovered to have been silently revoked by the issuing service — no notification, no expiry warning, just an auth failure mid-task. Several Telegram bot tokens are in active use with no expiration set, no rotation history, and at least two have been pasted into chat transcripts that are now part of the conversation log surface area.

### Pattern

Tokens shared in chat or scattered across logs accumulate leak surface monotonically. Some get revoked silently by the issuing service (Atlassian, GitHub, Apple ID-linked app passwords) when their heuristics flag exposure. Others persist forever, growing more dangerous the longer they live, because every additional log line is another place the token could leak from. Rotation is treated as an emergency action rather than a periodic one, so it only happens when something has already broken.

### Lesson

Rotate tokens after any exposure — including "trusted" channels — AND verify each active token is still working at least monthly via the secrets-lookup table. Two clocks: `last_rotated` and `last_verified`. Both run independently. A token can be rotated but unverified (just-issued, untested) or verified but un-rotated (working but old). Both states are tracked.

### Why

Silent revocation is the failure mode that hurts. By the time the agent hits the auth wall, the failure is already mid-task and on a deadline. Periodic verification surfaces revocations before they bite. Periodic rotation caps the blast radius of any leak the operator hasn't yet noticed. The cost is one Airtable row update per token per month. The cost of *not* doing it is the morning we just had.

### Application

When any token is shared in any chat, log, or transcript — regardless of how trusted the channel — the receiving agent logs a rotation TODO in the Airtable `secrets` table with a `due` date of +24h. Monthly, on the 1st, Daedalus walks the table and verifies every token still authenticates against its issuing service. Both `last_rotated` and `last_verified` columns are required.

### Failure mode

Daedalus flags any row where `last_rotated` is older than 90 days, or where `last_verified` is older than 30 days. Either condition is a drift signal. Argus flags any chat transcript containing a string matching common token shapes (`ghp_`, `xoxb-`, `Bearer `, `sk-`, `AIza`) that does not have a corresponding rotation TODO opened within 24h.

### Related

- atlas-corpus/incidents/2026-05-06-gh-auth-wall
- atlas-corpus/incidents/2026-05-06-kevin-app-password-revoked
- fleet/playbooks/secrets-table-schema

---

*End of starter set.*
