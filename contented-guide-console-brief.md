# contented.guide — Console Architecture & Build Brief (v1.0)

**For:** Perplexity, to build autonomously against, then return for review before
any GitHub/Cloudflare/Stripe production step.
**Status:** Complete starting instruction. Six issues found in an earlier draft
of this brief are fixed *in* the design below, not just noted — see §0 for what
changed and why.
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

### 4.1 Required tables

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  membership_status TEXT NOT NULL DEFAULT 'free',
  created_at TEXT NOT NULL
);

CREATE TABLE stripe_events (
  id TEXT PRIMARY KEY,
  type TEXT NOT NULL,
  processed_at TEXT NOT NULL
);

-- Corrected: consent is per-item and explicit, never implicit.
CREATE TABLE reflections (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  node_source TEXT NOT NULL,
  pillar_tag TEXT,
  content TEXT NOT NULL,
  saved_explicitly BOOLEAN NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL
);

CREATE TABLE threads (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  title TEXT,
  node_id TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

-- Waitlist only at this stage — see §6. No audio infrastructure implied.
CREATE TABLE voice_preferences (
  user_id TEXT PRIMARY KEY REFERENCES users(id),
  notify_requested BOOLEAN NOT NULL DEFAULT 0,
  is_enabled BOOLEAN NOT NULL DEFAULT 0,
  frequency TEXT,
  time_preference TEXT
);
```

### 4.2 Tenant isolation

Every query touching `reflections` or `threads` must filter by the
authenticated `user_id` from the verified session token — never from a
client-supplied value. No table is queryable without that filter present in
the prepared statement itself, not just enforced by application logic that
could be bypassed.

### 4.3 Encryption at rest (new requirement, was missing)

`reflections.content` must be encrypted at rest, not just deleted on
request. Cloudflare D1 does not provide field-level encryption natively —
this means application-layer encryption (encrypt before insert, decrypt
after fetch, using a key held only in Worker environment secrets, never in
D1 itself). **This needs a concrete implementation decision from Perplexity
before building** — flag the specific approach chosen back to James for
review, since this is genuinely security-critical, not a style choice.

### 4.4 Consent model, explicit

Nothing a member types is saved automatically. A reflection is only written
to the `reflections` table when the member takes a deliberate, visible
action to save it (e.g., a "Save this reflection" button, never a background
auto-save). The UI must make this distinction obvious — an unsaved
in-session conversation and a saved reflection should look and feel
different, not just be indistinguishable text sitting in the same view.

---

## 5. Authentication and Stripe

### 5.1 Auth

- ES256 token verification, mapped to `membership_status` in `users`.
- JWT secrets and API keys live only in Worker environment variables, never
  in D1, never client-visible.

### 5.2 Stripe webhook pipeline

- Edge-native signature verification via `Stripe.createSubtleCryptoProvider()`.
- Handler sequence: parse raw request → verify signature → insert event ID
  into `stripe_events` (deduplication) → queue entitlement update → return
  `200 OK` immediately.
- Entitlement checked against `membership_status` before any member route or
  API payload is served.

### 5.3 Pricing — PROPOSED, NOT DECIDED

The original brief stated €50/year as settled fact. **It is not decided.**
Perplexity should build the Stripe integration to accept a configurable
price point, not hardcode any figure, until James explicitly approves one.

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

## 8. Absolute deletion protocol

- `DELETE /api/user/delete` — cascading hard delete across every table for
  the authenticated `user_id`. No soft delete, no retention window, no
  hidden backup copy.
- This remains genuinely correct and unchanged from the original brief.

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

```text
1. "Get Apps and extensions" menu item — repurpose as "Your Network" (§3.2),
   or remove until a real feature exists.
2. Annual membership price — currently proposed at €50, not approved.
3. The specific encryption-at-rest implementation approach (§4.3) — needs a
   concrete proposal from Perplexity, then James's sign-off, before real
   reflection content touches the database.
4. Whether the safety mechanism's second-pass model layer (§7.2) is built
   now or deferred alongside voice — it adds real complexity and may be
   worth deferring on the same "solo operator, keep it simple" grounds as
   voice itself.
```
