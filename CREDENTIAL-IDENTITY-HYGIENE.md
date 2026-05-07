# CREDENTIAL IDENTITY HYGIENE
## Atlas Corpus Document — Phase 1 Ingest
### Authored 2026-05-07 | Cowork-Claude → atlas-corpus

---

## Purpose

A discipline for managing credentials — keys, tokens, passwords, PATs — across the OpenClaw multi-agent fleet without identity mismatches. The 2026-05-07 incident with `KEVIN_APP_PASSWORD` (which 535'd because the test script hardcoded the wrong auth username) is the prototype bug class this discipline prevents.

Atlas's job in Phase 3: coach the fleet to maintain this discipline. This document is the reference standard.

---

## The three facets of every credential

Every credential has three identity facets that must align. Most credential bugs are misalignments between these.

1. **Account-of-origin** — which Google/GitHub/Airtable/etc. account *minted* the credential
2. **Scope-of-access** — what resources the credential can *read/write*
3. **Consumer-identity** — which agent/script will use it AS its own identity to take actions

Example, done correctly:
- `KEVIN_APP_PASSWORD`
  - Account-of-origin: `kevin@overlookstrategy.com` (Workspace)
  - Scope-of-access: full Gmail mailbox for `kevin@overlookstrategy.com`
  - Consumer-identity: Kevin (the agent), authenticating AS `kevin@overlookstrategy.com` to send fleet email

Example, done incorrectly (the 2026-05-07 prototype bug):
- `KEVIN_APP_PASSWORD`
  - Account-of-origin: `kevin@overlookstrategy.com` ✓
  - Scope-of-access: full Gmail mailbox for `kevin@overlookstrategy.com` ✓
  - Consumer-identity: my test script wrote `s.login("finlaybennett@gmail.com", pw)` ❌ — hardcoded the WRONG identity

The credential was correct, the consumer-identity declaration was wrong, the SMTP server returned `535 BadCredentials`. Cost: ~30 minutes of debug, two unnecessary App Password mints, false architectural concern that personal Gmail credentials were on Kevin's filesystem.

---

## The bug class — variable-name vs auth-identity mismatches

Look for these patterns. They all reduce to the same root cause: a credential's *declared* identity (variable name, comment, doc) doesn't match its *actual* identity (the email/username/account it authenticates as).

### 1. Variable name says one identity; consumer code uses another
```python
# Variable name implies kevin's identity
pw = os.environ["KEVIN_APP_PASSWORD"]
# But login uses someone else's
s.login("finlaybennett@gmail.com", pw)  # ❌
```

### 2. Test scripts hardcode usernames instead of deriving from env
```python
# ❌ Hardcoded — silently wrong if the account changes
s.login("kevin@overlookstrategy.com", pw)

# ✓ Derived from env — explicit, single source of truth
s.login(os.environ["KEVIN_AUTH_USERNAME"], pw)
```

### 3. FROM/sender addresses don't match auth identity
```python
# ❌ Auth as kevin@, send AS someone else
s.login("kevin@overlookstrategy.com", pw)
msg["From"] = "finlaybennett@gmail.com"  # silently mismatched
```

### 4. Git commit user.email doesn't match SSH/HTTPS auth
```bash
# Auth via Bridge's GH PAT (org-scoped), commits authored as personal Gmail
git -c user.email="finlaybennett@gmail.com" commit  # ❌
```

### 5. API tokens passed where identity-bound tokens are expected
- Some APIs treat the token as the identity (Slack bot tokens, GH PATs)
- Others require the identity AND the token (Gmail SMTP, AWS IAM users)
- Mixing these = silent failures or wrong-actor attribution

### 6. Hardcoded recipients on outgoing fleet messages
- pp-digest.py originally had `TO_ADDR = "finlaybennett@gmail.com"` — the personal Gmail of Kevin's operator. After sanitization: `TO_ADDR = "finn@overlookstrategy.com"` (Workspace).

---

## The discipline — five rules

### Rule 1: Variable names should encode the auth identity unambiguously

When the consumer-to-account mapping is 1:1 and obvious, the consumer name is enough:
- ✓ `KEVIN_APP_PASSWORD` — Kevin's only Workspace identity is kevin@overlookstrategy.com, no ambiguity
- ✓ `BRIDGE_GIT_EMAIL` — Bridge has one git identity

When ambiguity is possible, name the account explicitly:
- ❌ `GMAIL_PASSWORD` — which Gmail?
- ✓ `KEVIN_OVERLOOKSTRATEGY_APP_PASSWORD` — explicit
- ✓ `FINN_GMAIL_APP_PASSWORD` — explicit (Finn's personal, NOT for fleet use)

### Rule 2: Never hardcode auth identities in code

Every place a credential is consumed, the matching identity must come from env or config:

```python
# ✓ Right
s.login(os.environ["KEVIN_AUTH_USERNAME"], os.environ["KEVIN_APP_PASSWORD"])

# ❌ Wrong
s.login("kevin@overlookstrategy.com", os.environ["KEVIN_APP_PASSWORD"])
```

This makes account changes a one-line edit in `secrets.env` rather than a multi-file grep.

### Rule 3: Document every credential's three facets in Airtable inventory

Every secrets-inventory row must carry:
- `env_var_name`
- `auth_identity` (the exact email/username it authenticates as)
- `scope` (what resources it grants access to)
- `consumer` (which agent/script uses it)
- `account_of_origin` (where it was minted — useful for rotation)

If any field is empty or ambiguous, that's a finding to flag.

### Rule 4: When asking the operator for a credential, name the three facets up front

Before pasting a credential, the asker (Cowork-Claude or fleet agent) must declare:

> "I need `<credential_type>` for `<exact-email-or-username>`, stored as `<VAR_NAME>`, used by `<consumer>` to `<purpose>`. Mint at `<URL>` while signed in as `<exact-email-or-username>` — verify the avatar in the top-right reads `<exact-email-or-username>` before generating."

This catches the "minted from the wrong signed-in browser tab" failure mode that almost burned us on 2026-05-07.

### Rule 5: HARD blast-radius rule — credentials whose REACH includes personal surfaces are off-limits on fleet filesystems

This is the boundary rule from `kevin_account_scope_boundary.md`:
- Account-scope credentials (full Gmail access, full Apple ID access) → must be on Kevin's email accounts only (kevin@overlookstrategy.com or finn@overlookstrategy.com)
- Resource-scope credentials (PATs scoped to specific bases/repos, bot tokens) → fine regardless of account-of-origin, AS LONG AS the scope is fleet-only

The 2026-05-07 false alarm: `AIRTABLE_PAT` is full-account-scope BUT minted from an Airtable account used exclusively for OpenClaw. Reach = fleet-only. Acceptable. The instant a personal base joins that account, the PAT becomes a hard violation.

---

## The audit pattern — how to find these bugs

### Step 1: Inventory
List every credential file/var on the agent's filesystem:
```bash
grep -hE "^export [A-Z_]+=" ~/.openclaw/secrets.env | sed 's/=.*//; s/^export //'
```

### Step 2: For each credential, identify
- Variable name
- Where it's consumed (grep the codebase for the variable name)
- What auth identity each consumer declares (the email/username next to the credential in `s.login`, `git -c user.email`, API headers, etc.)

### Step 3: Compare
For each consumer:
- Does the consumer's declared identity match what the variable name implies?
- Is the consumer's declared identity even an env var, or hardcoded?
- Does the recipient/FROM address (if applicable) match the auth identity?

### Step 4: For each mismatch, classify
- 🔴 **Critical** — credential authenticates as the wrong account (causes auth failures or wrong-actor actions)
- 🟠 **High** — variable name misleads operators into wrong-account assumptions
- 🟡 **Medium** — hardcoded identity creates one-line-edit risk on account changes
- 🟢 **Low** — operator-distinction prose that doesn't trigger auth, but should be paraphrased for clarity

### Step 5: Fix per severity
- Critical → fix immediately, retest with the live consumer
- High → rename variable or update doc
- Medium → refactor to env-derived
- Low → batch into next maintenance pass

---

## Atlas's role — Phase 3 coaching

### What to flag in weekly fleet review

When reviewing the past week of fleet activity, watch for:

1. **A new credential staged without a matching Airtable inventory row** — unrecorded credentials are unauditable
2. **A test script hardcoding usernames** — even temporarily; even in scratch
3. **A variable name that doesn't match its actual auth identity** — surface the bug class link
4. **A FROM/sender address that doesn't match the auth identity** — silent identity mismatches
5. **A new agent's identity files that reference the wrong email** — Kevin's USER.md said `finlaybennett@gmail.com` until 2026-05-07; future identity files for Otto/Bridge/Daedalus/Edith/Argus need the same audit

### How to flag

Reference this corpus document by name. Quote the relevant rule. Cite the 2026-05-07 prototype bug as the example. Recommend the specific fix from the discipline section.

Format:
> **Identity hygiene flag — Rule 2 (no hardcoded auth identities)**
>
> Found in `<file>:<line>`: `s.login("hardcoded@example.com", pw)`
>
> Per CREDENTIAL-IDENTITY-HYGIENE Rule 2, auth identities should be env-derived. The 2026-05-07 prototype bug shows what happens otherwise: ~30 min debug + two unnecessary credential rotations because the env had the right value but the code hardcoded the wrong username.
>
> Fix: replace with `s.login(os.environ["<VAR>_AUTH_USERNAME"], pw)` and add `<VAR>_AUTH_USERNAME` to `secrets.env`.

### When the fleet does it right — positive reinforcement

If a fleet agent stages a credential AND adds the proper Airtable row AND uses env-derived auth AND the variable name matches the identity, call it out:
> Atlas weekly note — Bridge's brief #N followed CREDENTIAL-IDENTITY-HYGIENE rules 1-5 cleanly. New credential `<VAR>` was added with full Airtable inventory, auth derived from env, variable name matches account. Compound learning loop closing.

Positive reinforcement keeps the discipline alive. Without it, agents drift toward "good enough" and the bug class returns.

---

## How this fits the broader hygiene picture

The OpenClaw fleet has multiple identity-related disciplines that interlock:

| Document | Domain | Trigger |
|---|---|---|
| `kevin_account_scope_boundary.md` | Which accounts Kevin can hold credentials for | Before staging any credential on Kevin's fs |
| `feedback_credential_account_specificity.md` | How to ask the operator for credentials | When Cowork or fleet agent needs a new credential |
| **CREDENTIAL-IDENTITY-HYGIENE.md** (this doc) | The full discipline of variable-name/auth-identity alignment | When auditing or maintaining credentials |
| `feedback_delegate_to_fleet_agents.md` | Who does the work | When deciding whether to delegate or self-execute |

Atlas references the right document for the situation. They don't conflict; they layer.

---

## The 2026-05-07 incident — full case study for Atlas

### Timeline

- **Earlier in week**: Kevin had `KEVIN_APP_PASSWORD` set, working, sending pp-digest emails. (At the time, the password authenticated as `finlaybennett@gmail.com` — silently wrong but functional.)
- **2026-05-06 ~7am**: Old App Password was revoked (security event, separate context). Kevin's pp-digest started failing with 535.
- **2026-05-07 morning**: Finn rotated, minted new App Password. Pasted to Cowork. I saved it.
- **2026-05-07 first SMTP test**: 535 BadCredentials. I diagnosed length anomaly (15 chars). Asked for re-paste.
- **2026-05-07 second paste**: 16-char value. SMTP test still 535'd.
- **2026-05-07 false alarm**: I framed this as "format right, account-level wall" — listed possible causes (rate limit, 2FA off, Workspace policy). Recommended Finn check Google Account settings.
- **2026-05-07 boundary discovery**: Finn corrected me — Kevin shouldn't have access to `finlaybennett@gmail.com` at all. We sanitized the filesystem.
- **2026-05-07 plot twist**: Finn clarified the App Passwords were minted from `kevin@overlookstrategy.com`, not `finlaybennett@gmail.com`. The 535 was caused by my test script hardcoding the wrong username — the password was correct.
- **2026-05-07 fix**: corrected `s.login("kevin@overlookstrategy.com", pw)`. First try succeeded. Test email landed in Finn's Workspace inbox.

### Lessons embedded

1. **Length anomaly was a red herring.** I caught a real format issue on the first paste (15 chars vs 16) but it gave me false confidence that the second paste's 535 was a Google-side issue. Should have checked auth identity before declaring "account-level wall."
2. **My test script hardcoded the wrong username.** Rule 2 violation in my own tooling. The discipline starts with me.
3. **Variable name didn't help.** `KEVIN_APP_PASSWORD` was ambiguous — could have meant "Kevin's password for any account he holds" or "the password for the kevin@overlookstrategy.com account specifically." When the variable doesn't disambiguate, the consumer code has to.
4. **The boundary memory + the specificity memory each caught half of the issue.** Neither was enough alone. The full discipline needed all five rules.
5. **30 minutes of debug = the cost of one prototype bug.** Across a 7-agent fleet over 6 months, this bug class compounds. Atlas's Phase 3 coaching is the leverage that prevents it from happening 50 times.

---

## Acceptance criteria for Atlas Phase 1 ingest

By 2026-05-12 (Phase 1 close), Atlas should be able to:

1. **Recite the three facets** of any credential (account-of-origin, scope-of-access, consumer-identity) when shown a credential reference
2. **Spot the bug class** in code (variable-name vs auth-identity mismatch, hardcoded auth, FROM-mismatch)
3. **Cite the right rule** (1 through 5) when flagging a finding
4. **Reference the 2026-05-07 prototype bug** when explaining why the discipline matters
5. **Recommend env-derived fixes** instead of hardcoded ones

When asked "what's the credential identity hygiene discipline?", Atlas should be able to summarize all 5 rules in 2-3 sentences and point a flagged team member to this document.

---

## Maintenance

This document evolves. When Atlas catches a NEW class of credential identity bug, append it to "The bug class" section. When the fleet adopts a new credential pattern, update the rules. Version-control this in atlas-corpus alongside the FLEET-CORPUS, OTTO/EDITH/ARGUS supplements, CLOUD-FIRST-POLICY, DAEDALUS-CORPUS, and STARTER-LESSONS.

Last updated: 2026-05-07 (initial authorship)
Next review trigger: any new credential identity bug found by audit OR any new agent added to fleet
