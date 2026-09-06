# contented-master-docs

Build documents for **contented.guide**, the member console for the Guide
network.

## The canonical brief

```text
contented-guide-console-brief.md      v1.1  ← build against this
archive/...-v1.0-superseded.md        v1.0     historical only
```

**v1.1 is the authority.** It was revised after independent review by
Perplexity and Gemini, and adds six substantive corrections over v1.0:

1. A full auth and session lifecycle (§5.1)
2. A specified encryption approach rather than an assumed one
3. An atomic Stripe webhook process
4. Corrected deletion wording that does not overpromise what can be erased
5. A hardened D1 schema with real constraints and an audit table
6. A required failure and recovery matrix

## Why this README exists

Until 6 September 2026 this repository held two files:
`contented-guide-console-brief.md` (v1.0) and
`contented-guide-console-brief (1).md` (v1.1).

The `(1)` suffix is what a browser adds when you download a file twice. So the
**newer, reviewed, authoritative** document was the one that looked like a
stray duplicate, and the clean, obvious filename was the superseded draft.
Anyone — a person or an AI — picking the sensible-looking file would have built
the console against a brief missing all six of the fixes above, including the
encryption approach and the atomic webhook.

That is not a filing inconvenience. It is a live risk to a build that will
handle memberships, payments and private member notes.

**Rule, per `male-network-docs/README.md`:** when a newer version arrives as a
browser download, rename it properly and archive the old one in the same
commit. Never let a `(1)` become the authority.

## Before building against this brief

Read, in this order:

1. `male-network-docs/male-guide-master-v6.md` — what the network is, and why.
   §10c covers paid membership and the 20-turn model that belongs to this node.
2. `male-network-docs/canonical-node-specification-v1.md` — how a node is
   built: files, URLs, storage prefixes, worker rules, verification gates.
   **contented.guide's reserved storage prefix is `ct_`.**
3. This brief.

The console is the one build that must read and trust every other node's
conventions. It should not be built on top of nodes that disagree with each
other — see `male-network-docs/node-conformance-audit-2026-09-06.md` for what
currently disagrees.
