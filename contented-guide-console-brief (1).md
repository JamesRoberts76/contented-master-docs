# contented.guide — Console Architecture & Build Brief (v1.1)

**For:** Perplexity, to build autonomously against, then return for review before
any GitHub/Cloudflare/Stripe production step.
**Status:** Revised after independent review from Perplexity and Gemini.
Six new fixes in this revision: a full auth/session lifecycle (§5.1), a fully
specified encryption approach (§4.3), an atomic Stripe webhook process
(§5.2), corrected deletion wording that doesn't overpromise (§8), a hardened
schema with real constraints and an audit table (§4.1), and a required
failure/recovery matrix (§12).
**Do not deploy, connect domains, activate Stripe live mode, or run production
D1 migrations without James's explicit sign-off, per Master §20-21.**

---

## 0. What changed from the prior draft, and why

1. **The contentment claim is corrected.** "Every man can achieve contentment"
   is gone. Replaced with language matching what's actually true and already
   established: contentment is a genuine, hard-won possibility — not a
   guarantee, and never something the product promises.
2. **The "minimum five nodes" claim is removed.** It was invented precision
   with no basis, and it quietly decided an open architecture question
   (whether Ageing/Forgiveness-style themes are connectors or standalone
   nodes) without that decision ever being made. This brief treats that
   question as open — see §9.
3. **€50/year is now explicitly marked PROPOSED, NOT DECIDED** everywhere it
   appears. No number appears anywhere as settled fact until James approves
   it directly.
4. **A per-item consent model is added to the `reflections` table** — nothing
   persists automatically; saving is always a deliberate, visible member
   action.
5. **Encryption-at-rest is now a stated requirement**, not just deletion.
6. **The safety mechanism is redesigned to match the rest of the network's
   own proven pattern** — deterministic-first, not a bare ML classifier — see
   §7.

---

## 1. Foundational philosophy (corrected)

`contented.guide` is the member console for the whole network — a private,
opt-in space for a man to save reflections, revisit them, and notice patterns
over time.

> Contentment is not guaranteed by this product. It's a real, hard-won
> possibility — James found it later than he expected, and it took a
> genuine restructuring of his life to get there. This console doesn't
> promise an outcome. It gives a man a private place to keep track of his
> own honest observations over time.

This must read consistently with the Five Lenses' own absolute rule (Master
§3): never a score, never a completion state, never a claim that using this
product produces contentment.

The peer, not guru, framing from the original brief is correct and unchanged:
the AI voice reflects alongside a man, it doesn't instruct him.

---

## 2. Core identity and the stateless/stateful boundary

- Individual nodes (`compressed.guide`, `back.guide`, `legs.guide`,
  `recalibration.guide`, and all future nodes) remain fully stateless —
  browser-local storage only, isolated prefixes (`cg_`, `bg_`, `lg_`, `rc_`).
  They never gain server-side persistence of their own.
- `contented.guide` is the **only** application in the network authorized to
  hold persistent server-side state. It is the single nexus point for
  cross-network continuity.
- Zero third-party telemetry, ad pixels, social login, or behavioural
  tracking, network-wide — this product is not an exception.

---

## 3. The console UI — must feel like a large LLM product, on purpose

James's own reasoning, stated directly: a subscribed member has almost
certainly used at least one major AI product already. The console should
feel instantly familiar the moment a member reaches *any* touchpoint across
the network — not a bespoke, unfamiliar layout competing for attention this
product can't currently win.

### 3.1 Overall shell

- Thread-centric workspace, collapsible sidebar, high-contrast
  typography-first canvas — using the network's existing `styles.css` tokens,
  not a new visual language.
- Generous, viewport-aware composer (same principle already fixed for the
  Back build — no unlimited-height textarea, no fixed tiny cap either).
- No streak counters, gamification badges, or engagement widgets, per the
  original brief — this stays correct and unchanged.

### 3.2 The bottom-left account menu — exact specification

A persistent element, bottom-left, on every authenticated screen:

```text
[Avatar/initial]  James        [Pro]
```

Clicking opens a menu, top to bottom:

```text
Settings
Language
Get Help
Get Apps and extensions   ← see note below, needs a decision
Learn more                 (expands in place, does not navigate away)
  ├── About
  ├── Legal (Privacy, Terms, Cookies, Disclaimer — all already-produced
  │         network documents, linked, not duplicated)
  └── Keyboard shortcuts
```

**"Get Apps and extensions" needs a real decision, not a copied menu
item.** In ChatGPT/Claude/Gemini, this item exists because there's a genuine
plugin/connector ecosystem behind it. `contented.guide` doesn't have one yet.
Two honest options:
  (a) Repurpose this slot as **"Your Network"** — a list of which nodes this
      member has actually engaged with (Back, Legs, Recalibration, etc.),
      linking out to each.
  (b) Remove the item entirely until there's a real feature behind it, and
      add it back honestly later.
  Recommend (a) — it's genuinely useful and matches something the console
  actually has. **Needs James's confirmation before Perplexity names it in
  the UI.**

**Settings** must include, at minimum: the voice check-in waitlist toggle
(§6), the export/delete controls (§8), and language preference.

---

## 4. Data architecture (Cloudflare D1)

### 4.1 Required tables (revised — hardened per review)

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,          -- normalised (lowercased) before insert
  membership_status TEXT NOT NULL
    CHECK (membership_status IN ('free', 'active', 'cancelled', 'past_due'))
    DEFAULT 'free',
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

-- Status is now a real lifecycle, not a fire-and-forget log line.
CREATE TABLE stripe_events (
  id TEXT PRIMARY KEY,                 -- Stripe's event ID; enforces dedup
  type TEXT NOT NULL,
  status TEXT NOT NULL
    CHECK (status IN ('received', 'processing', 'processed', 'failed'))
    DEFAULT 'received',
  event_created_at TEXT NOT NULL,      -- Stripe's own event timestamp —
                                        -- used to reject out-of-order updates
  received_at TEXT NOT NULL,
  processed_at TEXT
);

CREATE TABLE reflections (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  node_source TEXT NOT NULL
    CHECK (node_source IN (
      'compressed.guide','back.guide','legs.guide','recalibration.guide'
      -- extend this allowlist explicitly as each node goes live; never
      -- accept an arbitrary client-supplied string here
    )),
  pillar_tag TEXT
    CHECK (pillar_tag IS NULL OR pillar_tag IN
      ('movement','presence','purpose','safety','input')),
  content_ciphertext BLOB NOT NULL,    -- see §4.3 — never plaintext
  content_iv BLOB NOT NULL,
  content_auth_tag BLOB NOT NULL,
  key_version INTEGER NOT NULL,
  saved_explicitly BOOLEAN NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE threads (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  title TEXT,
  node_id TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
-- Open question, correctly raised in review: if full chat transcripts are
-- deliberately never stored by default, confirm whether `threads` needs to
-- exist at all yet, or only once a real, deliberate "save this thread"
-- action exists — don't build storage ahead of an actual feature.

CREATE TABLE voice_preferences (
  user_id TEXT PRIMARY KEY REFERENCES users(id),
  notify_requested BOOLEAN NOT NULL DEFAULT 0,
  is_enabled BOOLEAN NOT NULL DEFAULT 0,
  frequency TEXT,
  time_preference TEXT
);

-- New: a dedicated, redaction-aware audit table — not informal logging.
CREATE TABLE audit_log (
  id TEXT PRIMARY KEY,
  user_id TEXT REFERENCES users(id),
  event_type TEXT NOT NULL,            -- e.g. 'login', 'delete_requested',
                                        -- 'membership_changed' — never raw
                                        -- reflection content
  occurred_at TEXT NOT NULL
);

CREATE INDEX idx_reflections_user_node ON reflections(user_id, node_source, created_at DESC);
CREATE INDEX idx_threads_user_node ON threads(user_id, node_id, updated_at DESC);
CREATE UNIQUE INDEX idx_stripe_event_id ON stripe_events(id);
```

### 4.2 Tenant isolation

Every query touching `reflections`, `threads`, or `audit_log` must filter by
the authenticated `user_id` from the verified session token — never from a
client-supplied value. No table is queryable without that filter present in
the prepared statement itself, not just enforced by application logic that
could be bypassed. Use parameterised queries and batched writes for any
related multi-table state change (Cloudflare D1 supports both through Worker
bindings) — never string-concatenated SQL.

### 4.3 Encryption at rest — now fully specified, not deferred

**Algorithm:** AES-256-GCM (authenticated encryption), via WebCrypto inside
the Worker.

**Key hierarchy:** a single root key held only in Worker environment
secrets, never in D1. Each encrypted record stores its own `key_version`
column so a future key rotation doesn't silently break old records — old
ciphertext stays decryptable under the version it was written with, while
new writes use the current version.

**Stored alongside ciphertext, per record:** the IV/nonce and the GCM
authentication tag — both required for decryption and for detecting
tampering. Never store the encryption key itself anywhere near the
ciphertext.

**Search/filtering limitation, stated honestly:** encrypted content cannot
be searched or filtered server-side in the ordinary way. `pillar_tag` and
`node_source` remain plaintext specifically so filtering/browsing still
works without ever touching the encrypted content itself.

**Crypto-erasure as part of deletion (§8):** destroying a user's specific
data-encryption key material (if a per-user key layer is added later) is a
stronger deletion guarantee than relying on row deletion alone — worth
building toward, not required for the first version.

**Rotation:** re-encrypting existing records under a new key version happens
as a deliberate, tracked background operation, never silently or
automatically — record which `key_version` is current in a small config
table or Worker binding.

### 4.4 Consent model, explicit

Nothing a member types is saved automatically. A reflection is only written
to the `reflections` table when the member takes a deliberate, visible
action to save it (e.g., a "Save this reflection" button, never a background
auto-save). The UI must make this distinction obvious — an unsaved
in-session conversation and a saved reflection should look and feel
different, not just be indistinguishable text sitting in the same view.

---

## 5. Authentication and Stripe

### 5.1 Auth — full lifecycle, not just "ES256 verification"

This needs to be settled as its own short design decision before UI work
begins, not left implicit:

```text
Issuer:              a dedicated auth Worker (not Stripe, not a third party)
Required claims:     iss, aud, sub, exp, iat, nbf, jti, membership_status
Session transport:   short-lived access token (bearer) + rotating refresh
                      token — not a single long-lived session cookie
CSRF defence:         required if any cookie authenticates a state-changing
                      request; not required for pure bearer-token calls
Revocation:          on logout, email/password change, membership
                      cancellation, and account deletion — the refresh
                      token must be invalidated server-side, not just
                      discarded client-side
Account lifecycle:   explicit, documented flows for creation, sign-in,
                      email verification, password/account recovery, and
                      removal
```

**This is a required deliverable in its own right** — Perplexity should
return this as a short, explicit design note before building the UI against
it, not decide it implicitly inside the implementation.

### 5.2 Stripe webhook pipeline — corrected to an atomic, status-based
process

The original linear sequence risked marking an event "processed" before its
entitlement update had actually succeeded, with no handling for duplicate or
out-of-order delivery. Corrected process:

```text
1. Retain the raw request body untouched until signature verification
   succeeds (Stripe signs against the exact raw body).
2. Verify the Stripe-Signature header against that raw body and the
   webhook secret.
3. Validate the event schema.
4. Insert the event into stripe_events with status = 'received' if its ID
   doesn't already exist (idempotency — a retried delivery is a no-op here).
5. Compare the event's own timestamp against the user's current
   membership state — reject applying an older event over a newer one
   (out-of-order protection).
6. Update status to 'processing', apply the entitlement change, then
   update status to 'processed' — or 'failed' if the update didn't
   succeed, so a retry/reconciliation pass can find it later.
7. Return 200 OK to Stripe only once the event is durably recorded,
   regardless of whether entitlement processing has fully finished —
   Stripe needs the acknowledgement promptly; the status field is what
   tracks real completion.
```

A separate, simple reconciliation check (even a manual one initially) for
any event stuck in `processing` or `failed` beyond a reasonable window is
part of this deliverable, not an afterthought.

### 5.3 Pricing — PROPOSED, NOT DECIDED

The original brief stated €50/year as settled fact. **It is not decided.**
The Stripe integration must read the price ID from a Worker environment
variable, never hardcoded, so the actual figure can be set and changed
without a code change, until James explicitly approves a number.

---

## 6. Voice check-ins — waitlist only, built now; the real engine deferred

James's decision, given directly: build the placeholder now, defer the real
voice engine.

- `voice_preferences.notify_requested` captures interest — a simple opt-in
  toggle in Settings: "Notify me when voice check-ins are available."
- **No speech-to-text, text-to-speech, or real-time audio infrastructure is
  built in this phase.** `is_enabled`, `frequency`, and `time_preference`
  exist in the schema for forward compatibility but are not wired to any
  real feature yet.
- When the real engine is eventually built, the guardrails from the original
  brief remain sound and should carry forward unchanged: one-question rule,
  30-50 word spoken turns, single signature voice, no multi-accent support,
  a prominent one-click kill switch, no automated guilt-trip follow-ups.

---

## 7. Safety mechanism — redesigned to match the network's own proven pattern

The original brief's "automated structured-output safety classifier" is the
weakest, least-auditable part of the whole design — a real concern
regardless of voice's timeline, since text-based reflection needs this too.

**Every other safety mechanism in this network (Master §7, §10a) is
deterministic-first**: known patterns, fixed, pre-approved responses. This
is simple, auditable, and its failure modes are fully understood in advance.
`contented.guide` should follow the same proven pattern, not introduce a
different, harder-to-audit one just because it's a newer part of the
system:

1. **Deterministic pattern match first** (same approach as the Back Worker's
   immediate-safety check) — a known, reviewed set of high-signal phrases
   for self-harm, immediate danger, and similar, checked before anything
   else runs.
2. **A second-pass model check only as a supplement, never the sole
   mechanism** — if the network wants broader coverage than fixed patterns
   allow, a model-based check can flag *uncertain* cases for the same fixed,
   pre-approved crisis response, but it must never generate a bespoke safety
   reply of its own. The response text itself is always one of a small,
   human-approved set.
3. On any match, from either layer: pause the normal flow instantly, show
   the non-paywalled crisis route, never continue the conversation as if it
   were an ordinary reflection.

---

## 8. Deletion protocol — corrected wording, same real intent

**The original "no hidden backup copy" wording is withdrawn — it's an
absolute promise that can't actually be verified** without documented,
tested control over Cloudflare's own backup/recovery systems, deployment
snapshots, and Stripe's own retention requirements. Promising it anyway
would be exactly the kind of overclaim this network has already corrected
elsewhere (never "absolute confidentiality").

**Public-facing wording (use this instead):**

> "We delete your active account and saved-reflection records from our
> operational database when you request deletion. Limited records may
> remain in secure backups, or be retained where necessary for legal,
> fraud-prevention, payment, or security purposes, and are never used to
> restore ordinary account access."

**Internal requirements, defined explicitly:**

```text
- DELETE /api/user/delete — cascading hard delete across every operational
  table for the authenticated user_id, executed immediately.
- A documented backup retention window (whatever Cloudflare's actual
  defaults are — confirm and state the real number, don't guess).
- A restore procedure that re-applies deletion if a backup is ever
  restored (a deletion ledger/tombstone approach, not just hoping it's
  remembered).
- Stripe/payment records that cannot be deleted by this system at all —
  named explicitly, not glossed over.
- Audit-log entries retained for a limited, stated period for
  security/fraud purposes, separate from the user's own content.
```

---

## 12. Failure and recovery matrix — required deliverable, not optional

For each state below, define: exact user-facing wording, API status
returned, retry behaviour, data-integrity guarantee, and whether the
console fails closed, goes read-only, or shows a maintenance state.

```text
- D1 unavailable / read timeout / write failure
- Stripe webhook delayed, duplicated, out-of-order, invalid, or unavailable
- Auth key/JWK unavailable
- Expired or revoked token
- Session expiry mid-save
- AI provider unavailable
- Browser offline during save
- Cross-origin/CORS failure
- Client/server API version mismatch
- Deployment failure/rollback
- Full contented.guide outage
```

This can start as a simple table Perplexity fills in alongside the build —
it doesn't need to be exhaustive on day one, but it needs to exist before
real member data is at stake.

---

## Appendix A — Voice guardrails, preserved for Phase 1b

Not active in this build (§6), but documented now so they're ready
unchanged whenever the real voice engine is actually built:

```text
- One-question rule: never ask more than one thing per spoken turn.
- 30-50 words maximum per spoken turn.
- Single, calm, neutral British delivery — no multi-accent support.
- Zero empathy clichés or conversational padding.
- Prompt reflection, never preach or instruct.
- Prominent one-click kill switch, disabling voice permanently.
- Zero automated guilt-trip follow-ups if a check-in is missed.
```

---

## 9. Open architecture question — not resolved by this document

Whether cross-cutting themes (Ageing, and James's own earlier
Forgiveness-as-connector idea) are standalone nodes or genuine cross-network
connectors is still unresolved (see the `ageing.guide` content pack, marked
UNCONFIRMED). **This brief deliberately does not decide it.** If
`contented.guide`'s thread/tagging system needs to model this relationship
technically, that data model should wait until the architecture question
itself is actually settled — building the technical structure first risks
quietly deciding the open question by accident, exactly as the original
draft's "five nodes" claim did.

---

## 10. Discovery and machine-readability

- Root `llms.txt` and `llms-full.txt` describing the console's actual
  purpose, safety boundaries, and Five Lenses structure — honestly scoped to
  what's actually live, matching the discipline already proven on
  `compressed.guide`.
- Accurate JSON-LD reflecting the consent-based, privacy-first nature of the
  product — no clinical or medical schema, matching Master §14.
- `robots.txt` and CORS headers locking out unauthorized scrapers.

---

## 11. What needs James's explicit decision before Perplexity builds further

**Resolved by this revision, confirm with a quick yes:**

```text
1. "Get Apps and extensions" → "Your Network" (§3.2) — both reviewers
   independently converged on this; showing only nodes a member has
   actually engaged with, never inferred interest.
2. Encryption approach (§4.3) — AES-256-GCM via WebCrypto in the Worker,
   with key versioning. This is now a specific proposal, not an open
   question — confirm you're comfortable with it rather than deciding
   the primitive yourself.
```

**Still genuinely open:**

```text
3. Annual membership price — proposed at €50, not approved; the build
   reads it from environment config either way, so this doesn't block
   starting.
4. Whether the safety mechanism's second-pass model layer (§7.2) is
   built now or deferred — still worth deciding on the same
   "solo operator, keep it simple" grounds as voice itself.
5. Whether `threads` needs to exist as a table yet at all, given full
   transcripts aren't stored by default (flagged in §4.1) — a real,
   fair question raised in review, not yet answered.
```

