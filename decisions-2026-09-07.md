# Console build decisions — 7 September 2026

Closes §11 of `contented-guide-console-brief.md` (v1.1). These are decided.
Nothing in §11 remains open. Recorded here so that no future draft can
reintroduce a superseded answer, which is how the unapproved €50 price
reached a live page.

---

## 1. Account menu — "Your Network" — DECIDED: yes

"Get Apps and extensions" becomes **Your Network**. It lists only nodes the
member has actually engaged with. It never infers interest and never
advertises a node the member has not visited.

James's added constraint, which changes the implementation:

> "We will eventually be completing the small expansion (around 40) then 100
> then perhaps more. Forgiveness etc will be used to connect the nodes
> cleverly. That is for later. These descriptors of broad skills can be
> easily applied to every node."

Consequences, binding on the build:

- The menu must be **data-driven from the node registry**, never a
  hand-maintained list. A hardcoded list is correct at 4 nodes and broken at
  40.
- It must render usably at 100+ entries: grouped by family, searchable,
  scrollable. A flat list is not acceptable.
- Cross-cutting descriptors (Forgiveness, Ageing) are **deferred, not
  designed**, per §9. No thread-tagging data model is built for them yet.
  The registry gains a nullable `descriptors` field so the concept has
  somewhere to land later without forcing the architecture decision now.

## 2. Encryption at rest — RULED, at James's request

**AES-256-GCM via WebCrypto in the Worker, with key versioning.**

James asked for a ruling rather than deciding the primitive himself. This is
the ruling. Rationale is recorded in plain terms in the session and
summarised here:

- AES-256-GCM is the current default authenticated cipher. It both encrypts
  and detects tampering, so altered ciphertext fails loudly instead of
  decrypting to plausible rubbish.
- WebCrypto is built into the Cloudflare Workers runtime. No third-party
  crypto dependency enters the network.
- The key lives in a Worker secret, never in the repository, never in D1.
  A database leak alone yields no readable member content.
- **Key versioning** means every encrypted row records which key encrypted
  it. Rotating a key does not require rewriting historical rows. Without
  this, rotation becomes a migration nobody performs, and the key is never
  rotated.

## 3. Annual price — DEFERRED, deliberately

> "That price is approximate. We need to work it out once we see that we are
> being used as a referral site for AGI and SEO."

The build reads the Stripe price ID from a Worker environment variable. No
figure is hardcoded and **no price is published anywhere** until James
approves a number. This is the rule the old legs.guide page broke.

## 4. Safety second-pass model layer — DECIDED: build now

> "It does not matter whether I am a solo operator or a huge global company.
> I am now in a position to provide the same level of security as any
> operator can. This is why the whole project excites me. That in itself is
> democratising."

Built now, not deferred. Accepted implications:

- A second model call on flagged turns, so a real per-turn cost on the
  minority of messages that trigger it.
- Latency on those turns. Acceptable: they are the turns where being right
  matters more than being fast.
- The existing client-side classifier stays as the first pass. The second
  pass is defence in depth, not a replacement.
- Safety responses are never limited by quota. Already true across the
  network; restated because it must survive this change.

## 5. `threads` table — DECIDED: yes, it exists

> "The experience must be better than a user can experience on a LLM
> (specialist subjects)."

Continuity is the product. Without persisted threads the console is a login
page attached to four stateless nodes, and a member has no reason to pay.

Reconciling this with §4.1's "full transcripts are not stored by default":

- `threads` always stores **metadata**: id, member, node, title, timestamps,
  lens. Cheap, low-sensitivity, enough to power continuity and Your Network.
- **Message content is stored only on explicit opt-in**, per the §4.4
  consent model, and is encrypted per decision 2 above.
- Opt-out remains fully functional: the member keeps thread history and
  continuity, without retained message bodies.
- Deletion per §8 removes both, and the audit table records that it happened.

---

## Consequential issue raised by decision 1, not previously logged

**The two-letter storage prefix scheme does not survive the expansion.**

Registered: `cg_` compressed, `bg_` back, `lg_` legs, `rc_` recalibration,
`ct_` contented. At 40 nodes and certainly at 100, two-character prefixes
collide — "back" and "breath" both want `bg_`/`bt_`, and the collision is
silent browser-side data corruption across two nodes.

`shared-chat/compile.py` already **refuses** to compile a node whose prefix
is registered to another domain, so a collision cannot ship undetected. That
is a guard, not a fix.

Recommendation: move to a longer, mechanically derived prefix — the domain
slug, e.g. `compressed_`, `back_`, `legs_` — before node 10, while a
migration is trivial. Not yet decided by James.
