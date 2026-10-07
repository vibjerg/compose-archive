---
about:
  - 01M4BD2FDA49BRZ244HCR24FBQ
at: 2026-10-07T14:42:54Z
by: human:me
cost: null
id: 01M4BD2FXXBH3F0J9T4V1JJWBX
inputs: []
kind: note
links: []
origin:
  externalId: /Users/svenven/www/compose-research/02-kernel-b-ledger.md
  fetchedAt: 2026-10-07T14:42:54Z
  sha256: f05d11e7ede84a788c2fc89d45a9c2a02d1effd1257108a9d2ea4cce53045f6c
  system: file
  url: file:///Users/svenven/www/compose-research/02-kernel-b-ledger.md
people: []
title: "02 — Kernel B: LEDGER"
type: note.research
---
# 02 — Kernel B: LEDGER

Iteration 2, candidate B of three (see `STATE.md`). A is "patch bay" (the graph is primary) and C
is "namespace" (a tree of records, with modules as file servers). Here **the record log is
primary**. Modules are reactors. They wake on records that match a query and they append
records. Composition emerges from what each module reads and writes, the way Kubernetes
controllers coordinate through the API server and never call each other (`01-prior-art.md` §10).

The brief is to push B to its strongest form and be honest about where it breaks. The short
verdict:

- **B is the strongest of the shapes on provenance, outliving the kernel, probing and M+N.**
- **B is weakest on visible composition, pre-run budgets and sequencing.**
- **B needs a small amount of explicit wiring.** It is not a cable from A to B. It is a
  *subscription*, a query owned by the consumer. The graph is derived from the subscriptions
  (§2.3, §7).

Sources: `01-data-model.md` (DM), `01-prior-art.md` (PA), `01-failure-modes.md` (FM),
`01-ai-landscape.md` (AI), `01-inventory.md` (INV), `sonic/docs/product/patch-bay.md` (PB).

---

## 1. The core idea, as a one-page spec

> **A user owns one append-only log of immutable, content-addressed records. Every record has a
> small fixed envelope: kind, type, subject, links, status, labels and body. The kernel stamps
> provenance on every record: who wrote it, which run, which records that run actually read,
> and at what cost. A module is a reactor. It declares the record types it reads and writes,
> and its instances hold *subscriptions*: queries over the log, owned by the consumer, that
> never name a producer. The kernel is the only dispatcher. When the head records matching a
> subscription change, the kernel starts one bounded run of that instance, but only if four
> things hold: the instance's outputs can reach a sink (a human inbox, a publish step, a
> merge), its loop bound and spend bound for this lineage are not exhausted, it has the
> capability, and the same inputs have not already been processed. Every decision to dispatch,
> and every refusal, is itself a record. Authority lives in write rules, not in wiring. Agents
> may append `proposed`. Only a human principal, whose key no sandbox holds, may append
> `accepted`, `admitted` or `approve`. Current state ("the draft", "the decision register") is
> always a fold over the log, never the source of truth. The composition graph is likewise a
> view, derived by joining each subscription's query against every module's declared writes.**

The whole spec, on one page:

```
Record    = { id, kind, type, subject, prev?: id|id[], status?, space, labels, links[], body | blobs[], prov }
            id     = sha256(canonical JSON of everything except id)        ; RFC 8785 JCS
            kind   ∈ { doc, claim, decision, task, act, event }            ; §4.2
            prov   = stamped by the kernel, never by the module             ; §4.1
Ledger    = append(records[]) atomically | get(id) | query(q) | tail(q, from)
Principal = human(key) | instance(id) | kernel                              ; signs prov
Manifest  = { module, version, reads[types], writes[{kind,type,status}], caps[], cost, run, sink? }
Instance  = { id, module@version, config, grants }                          ; a record
Subscription = { instance, on: query | query[] (first = primary), reads?: query,
                 mode: each|batch, from: now|all, bounds: {rounds, usd, depth},
                 debounce?, self? }                                          ; a record
Query     = envelope fields only; each a value, list or glob; null = absent; head?;
            `$trigger.<envelope field>` allowed in `reads` only
Dispatch  : head(matches(on)) changed ∧ key ∉ done ∧ reachesSink ∧ within(bounds) ∧ caps
            → run; else append event kernel.refused{reason}
Run       = process + scoped ledger handle; appends checked: writes ⊆ manifest.writes,
            links.derivedFrom ⊆ reads-set, ACL(kind,type,status,principal)
Laws      : append-only · labels inherit from the trigger unless set ·
            bounds keyed per (sub, lineage root[, subject in each mode]) ·
            additive schema evolution (new type id for breaking changes) ·
            accepted/admitted/approve ⇒ human principal · every refusal is a record
```

---

## 2. Kernel primitives

### 2.1 The complete minimal list

| # | Primitive | What it is | Why it cannot be a module |
|---|---|---|---|
| P1 | **Record + envelope** | The fixed outer shape (§4.1). The body is opaque to the kernel. | Every generic tool depends on it: probe, fold, sync, GC. It is the n8n/k8s "envelope is what composes" lesson (PA §7, §10). |
| P2 | **Ledger** | An append-only, content-addressed store. `append` takes a batch atomically (a Datomic transaction). It also offers `get`, `query` and `tail`. Heads are derived: the latest revision per subject. | It is the substrate. |
| P3 | **Principals and write rules** | Three kinds of principal: `human` (an ed25519 key held outside every sandbox), `instance` (a key minted per instance and lent per run) and `kernel`. An ACL on `(kind, type, status) → principal class`. | Authority must be enforced by the store, not by a convention among modules (FM pattern 6, INV implication 4, swarm Settled #1). |
| P4 | **Provenance stamping** | On every append the kernel writes `prov`: principal, instance, module@version, subscription, run, trigger, **reads-set**, cost and time. It also checks that `derivedFrom ⊆ reads-set`. | "The store stamps it, not the author" (DM §3.3). Modules would forget, lie or drift. |
| P5 | **Manifest** | The module's declaration (§3.1): its reads, writes, capabilities, cost in words, run mode and whether it is a sink. Stored as a record. | It is what the derived graph, parking and the write check are computed from. |
| P6 | **Instance and subscription records** | An instance is module@version plus config plus grants. A subscription is a query owned by an instance, plus a mode, a replay start and bounds. | This is the *minimal wiring* (§2.3). Without it reactors would tail the log themselves, and parking, bounds and capabilities would become unenforceable. |
| P7 | **Dispatcher** | Level-triggered dispatch over subscriptions (§2.4). It checks reachability to a sink, bounds, idempotency and capability. It appends `kernel.run.*` and `kernel.refused` records. | This is the scheduler. It is Kubernetes reconcile plus Sonic parking plus OTP's bounded restarts. |
| P8 | **Run** | One bounded execution of an instance, in its own process and sandbox, with a scoped ledger handle, a budget and a deadline. It ends in a checked batch of appends, or in a `kernel.run.failed` record. | This is swarm's "round", generalised (INV implication 2). Isolation and budgets have to be the kernel's job (FM 8). |

That is eight primitives. The query language is part of P2 and is deliberately small. It
matches on envelope fields only: `kind`, `type` (with a `prefix.*` glob), `subject`, `status`,
`space`, `labels`, `links` (`about`, `derivedFrom` and so on) and `prov.principal` class. It
adds `head: true` (the latest revision per subject) and `where` comparisons on the envelope. It
never looks inside bodies. This is the line that keeps the spec to one page (§7, weakness 4).

### 2.2 Explicitly not in the kernel

- **Body schemas.** Modules publish them as `kernel.schema` records, atproto-lexicon style (DM §2.7).
  The kernel validates a body only if the manifest names a schema.
- **Cables, flows and workflows.** There are no `from → to` edges anywhere. Sequencing is
  expressed as records with status (§5).
- **LLM calls and model access.** A *host service* (a model broker) lends models under the user's
  budget, as AI §5.5 argues. The kernel only meters the cost each run reports, and it caps it.
  The broker is outside the kernel so the kernel stays pure and testable without a model (PA
  principle 7).
- **Rendering, UI, the inbox, the graph view.** All porcelain. The human inbox is a sink
  *module*.
- **Lenses and coercions** (`doc → claim` extraction, `TypeV1 → TypeV2`). These are ordinary
  reactor modules with cost and provenance, never implicit casts (DM §3.2, §3.4).
- **Indexes beyond the envelope.** Full-text search, embeddings and Datalog are derived-view
  modules, and every one of them can be rebuilt from the log.
- **Clocks and timers.** A `clock` module appends `event clock.tick`. The kernel never runs
  anything on a timer.
- **Sync transport.** Records are immutable and content-addressed, so sync is set union (§4.5).
  Git, a dumb relay or a file copy all work. The kernel only defines the merge rule.
- **External effects.** Merging a branch, publishing, sending mail. These are done by *effector*
  modules, each triggered by a human `act` record.

### 2.3 How much explicit wiring? The answer

Pure blackboard (every reactor tails the log and decides for itself) fails two of the sixteen
tests outright. Any reactor reads anything, so the capability test fails (test 9). Nothing
written down says who reacts to what, so the plain-data test fails (test 11). It also breaks two
principles the tests assume. Parking cannot be enforced (PA principle 5, PB D-014), and loops
cannot be bounded (PA principle 4, DM §4 "each pass costs money"). So B keeps exactly four
pieces of declared wiring, all as records:

1. **The subscription.** A query that the *consumer* owns. It names types, labels and status,
   never a producer. This is a Kubernetes label selector, or an Excel reference by name.
2. **Scope labels.** `space` plus a free label map. Without scope, C1's writer and C3's critic
   would fire on every draft in the user's life. Scope is required. A subscription with no
   `space` is rejected.
3. **Bounds per subscription**: rounds per lineage, dollars and depth.
4. **Sink declarations** in manifests: the human inbox, publish, merge.

What is *not* declared is any edge. The **graph is derived**. There is an edge `A → B` labelled
`t` if an active instance A's manifest declares it writes `t`, and an active subscription of B
matches `t` within a shared scope. Because the kernel rejects any append outside
`manifest.writes`, the derived graph is **sound**: no record can flow along an edge the graph
does not show. This is what makes B more than "A with the cables renamed". Adding a producer
never touches a consumer's configuration, and a consumer never names a producer (PA principle 3).

### 2.4 Dispatch, precisely

For every active subscription `S` of instance `I`, on every append:

1. Compute `M` = the head records in scope matching `S.on`. In `each` mode there is one
   dispatch candidate per matching subject. In `batch` mode there is one candidate over the
   whole set.
2. **Idempotency key** = `sha256(S.id, sorted head ids of the candidate)`. If a `kernel.run.*`
   record with this key exists, do nothing. This is level-triggered: duplicate or missed events
   are harmless, and replay never re-runs work (swarm D-011, "restart never duplicates a
   round"; PA §10).
3. **Self-trigger guard.** A candidate whose only changed records were written by `I` itself is
   skipped, unless `S` sets `self: true`.
4. **Reachability.** If `I`'s writes cannot reach any active sink in the derived graph, the
   candidate is **parked**. The kernel appends one `event kernel.refused {reason: no-sink}` per
   key: no run, no tokens, state kept (PB "parked", D-014).
5. **Bounds**, counted per **lineage root**. The lineage root of a record is its nearest ancestor
   written by a human principal, found by walking `prov.trigger` back. The bound key is
   `(S, root, subject)` in `each` mode, and `(S, root)` in `batch` mode. So nine sources
   extracted under one brief get one round each (C1), while one draft cycling through a critic
   shares one counter (C3). A batch candidate's root is the root of the newest record matching
   its *primary* (first) `on` clause. Records matched by other clauses or read through `reads`
   do not change the root. Within a key the kernel counts prior runs (`rounds`), sums their
   cost (`usd`) and takes the longest trigger chain (`depth`). Over the bound, it appends
   `event kernel.refused {reason: bound, needs: human}`.
6. **Capability and budget.** The instance grants must cover the manifest caps. The task budget,
   where a task is in the trigger chain, must not be exhausted. Otherwise a refusal is appended.
7. Otherwise, append `event kernel.run.started {key, sub, trigger: [ids], root, depth}` and spawn
   the run.

On finish the kernel checks the run's batch atomically:

- writes ⊆ manifest `writes`;
- the ACL holds;
- `links.derivedFrom ⊆ reads-set` (the ids the run actually fetched through its handle);
- `claim` records have at least one `derivedFrom`;
- **oscillation**: a new revision whose body hash equals an earlier revision of the same subject
  inside the same lineage is refused (`reason: oscillation`).

If the batch passes, the kernel appends it with `prov`, then `kernel.run.finished {cost, appended}`.
If it fails, nothing from the batch is appended, and `kernel.run.failed {reason, transcript}`
goes on the instance's card (FM 8).

Why a human resets the bound: "N rounds, then wait for the human" (C3) falls out of it. Agents
cannot extend their own budget, because only a human-written record starts a new lineage.

---

## 3. Module contract

### 3.1 Declaration (manifest)

```yaml
# modules/critic/manifest.yaml  (appended as a record: kind doc, type kernel.manifest)
module: writing.critic
version: 1.2.0
blurb: Reads a draft and lists concrete problems with it, or passes it.
reads:  [writing.draft, writing.brief, kb.claim]
writes:
  - { kind: claim, type: critique.finding }
  - { kind: doc,   type: critique.review, status: [pass, revise] }
caps: [model:claude, ledger:read-scope]     # no net, no fs
cost: "one model call per draft, roughly $0.05–0.30 at 3k words"
latency: "20–90 s per draft"
run: { exec: process, cmd: [node, critic.js], timeout: 300s, sandbox: nono:readonly }
failure: { retry: 1, then: escalate }       # OTP-style, bounded (PA §4)
sink: false
```

`sink: true` marks a module whose effect leaves the ledger or reaches a person. The stock sinks
are `inbox` (the human), `publish` (export) and effectors such as `swarm-adapter.approve`.

### 3.2 How a module runs

- **Per-run process.** This is the default, and it follows Unix and the VS Code extension host
  (FM 5, 8). The kernel spawns `run.cmd` in the sandbox named in the manifest (nono profiles,
  as swarm does today, INV §1). It passes a **run handle** on stdin: `{run, key, trigger ids,
  scope, budget, deadline}`.
- **The ledger handle is an MCP server the kernel lends to the run.** It speaks the 2026-07-28
  stateless spec (AI §1.1), with the run handle as a server-minted handle argument. It has
  four tools:
  - `ledger.get(id)`
  - `ledger.query(q)`, limited to the instance's read scope
  - `ledger.stage(records[])`
  - `ledger.finish(outcome)`

  So an agent module needs nothing but an MCP client, which every agent runtime already has.
  Every `get` and `query` result is added to the reads-set.
- **Daemon mode** (`run.exec: daemon`) is for adapters around long-lived tools such as
  `swarm serve`. The process stays up, but it still writes only through a run handle that the
  kernel dispatched. An *ingest* run is opened by a dispatch and stays open while the external
  job lives. So swarm's rounds, which happen on swarm's own clock, are stamped under the run
  that started the feature (C2).
- **A module may also *be* an MCP server** for humans and hosts (Claude, an IDE). That is its own
  product surface, orthogonal to the kernel (FM test 1).
- **Contract test** (FM test 7, the Sonic `registry.test.ts` analogue). `ledger conform <module>`
  replays a fixture ledger into the module and asserts that:
  - every append is a valid envelope inside `writes`;
  - derivedFrom ⊆ reads;
  - a duplicate dispatch with the same key appends nothing new;
  - the module exits within its timeout under oversized and malformed inputs;
  - it handles `head` forks.

  A module that fails cannot be instantiated.

### 3.3 Adapters: how an existing tool becomes a module

An adapter is a module like any other, written once per tool and owned in one place. That is
Zapier's "adapters concentrated centrally", never left to the user (FM 7).

**Generic CLI adapter.** Most `--json` CLIs (printer, pandoc, `gh`) need only a manifest and a
mapping. No code.

```yaml
module: cli.printer-check
version: 0.1.0
reads:  [printer.doc-ref]
writes: [{ kind: claim, type: printer.finding }]
caps:   [fs:read:~/www/printer/projects]
run:
  exec: cli
  argv: [printer, check, --project, "{{body.project}}", --json]
  stdout: jsonl                 # each line becomes one record
  map: { kind: claim, type: printer.finding, body: ".", links: [{rel: derivedFrom, to: "{{trigger.id}}"}] }
```

**Swarm adapter** (`swarm-adapter`, daemon mode). It is ~300 lines of host code, outside the
sandbox. It never changes swarm's own trust rule (INV §1 "agents read, only a person writes").

| Ledger side | Swarm side |
|---|---|
| reads `task swarm.feature` (status `admitted`) | `swarm add "<idea>" --budget <capUsd> [--depends-on]` |
| reads `decision kb.decision` (status `accepted`, in scope) | pins the **basis**. It needs one small swarm change: `add --basis <file>`, since `basis` is always `[]` today (INV §1, "Not built yet") |
| reads `act swarm.reply` (human) | `swarm reply <id> "<text>"` |
| reads `act swarm.approve` (human) | `swarm approve <id>`. The adapter holds the human token, but it uses it only to carry out a human-signed act whose `snapshotSha` matches the current round (swarm D-014: approve merges exactly that sha) |
| reads `act swarm.drop` (human) | `swarm drop <id>` |
| tails `/api/events` (SSE) and feature JSON | appends `event swarm.round`, `doc swarm.handback`, `decision kb.decision` (status `proposed`) per `handback.decisions[]`, and `event swarm.question` with `needs: human` |

The adapter is the *only* place swarm's vocabulary meets the ledger's. Swarm keeps working with
no ledger at all, which is FM test 1.

---

## 4. Record model

### 4.1 Envelope

```json
{
  "id":      "sha256:9c1e…",
  "kind":    "claim",
  "type":    "kb.claim",
  "subject": "kb/claim/eurorack-spec-stable",
  "prev":    null,
  "status":  null,
  "space":   "compose",
  "labels":  { "piece": "article-1" },
  "links":   [ { "rel": "derivedFrom", "to": "sha256:51aa…" },
               { "rel": "about",       "to": "sha256:07be…" } ],
  "body":    { "text": "Eurorack's power spec has barely changed in decades.",
               "quote": "the spec has barely changed…", "locator": "§4" },
  "blobs":   [],
  "prov": {
    "principal": "instance:kb-extract#c1",   "sig": "ed25519:…",
    "module": "kb.extract@0.3.1",            "sub": "sha256:a2f0…",
    "run": "run:01J9…", "trigger": ["sha256:51aa…"], "root": "sha256:3b7d…", "depth": 2,
    "reads": ["sha256:51aa…", "sha256:07be…"],
    "cost": { "usd": 0.012, "model": "claude-sonnet-x", "ms": 8400 },
    "at": "2026-10-07T13:02:11Z", "hlc": "2026-10-07T13:02:11Z/0003/devA"
  }
}
```

- **`subject`** is the stable logical identity: a draft, a decision `D-018`, a feature
  `f-6b5490`. Revisions of a subject chain by `prev`. The head is the revision that no other
  revision names as `prev`. Two heads means a **fork**, which is surfaced and never silently
  resolved.
- **`status`** is in the envelope, not the body, so queries and ACLs can use it without reading
  bodies.
- **`blobs`** are `{sha256, mime, size, path?}`. Large or binary content (a PDF, a git tree,
  screenshots) lives in the blob store, or stays where it is as a hash plus path (DM open risk
  2). A commit sha is a blob reference of mime `application/x-git-commit`.
- **`prov`** is written only by the kernel. `reads` is the reads-set. `root` and `depth` drive
  bounds.

### 4.2 Kinds: a position on the six candidates

The test applied to each of DM's six candidates: *does any kernel rule (dispatch, ACL, provenance
check, render) actually discriminate on it?* If none does, it is a type, not a kind.

| DM candidate | Verdict | The kernel rule that discriminates |
|---|---|---|
| `doc` | **Keep.** | The default kind. Any record renders deterministically to a `doc` for a text reader (DM's asymmetric rule). No authority. |
| `claim` | **Keep, thin.** | It is the only kind with a hard provenance invariant: at least one `derivedFrom`, all inside the reads-set. That makes "every claim traces to a source" a store property (C1), not a convention. It stays thin (text, quote, locator, optional `scope`). It is not an RDF triple, which answers DM's open risk 1 and its §2.4 lesson. |
| `decision` | **Keep.** | Status lattice `proposed → accepted → superseded / rejected`. `accepted`, `superseded` and `rejected` are writable only by a human principal. |
| `task` | **Keep.** | `admitted` is human-only. `body.budget` is read by the dispatcher as a spend cap for every run whose lineage passes through the task. It is the kind that makes money move. |
| `artifact` | **Drop as a kind.** | Nothing in the kernel discriminates on "opaque bytes" except storage, and storage is the envelope's `blobs` field, available to every kind. A swarm snapshot is a `doc swarm.snapshot` with a blob. |
| `signal` | **Split in two.** | In a ledger every append is already an event, so "signal as event" is redundant. What remains are two different things with two different rules. |
| → `act` | **New kind (the authority half).** | A principal's act on a subject: `approve`, `reply`, `drop`, `publish`, `hold`. Act types listed in the ACL are human-only. Effectors trigger only on acts. |
| → `event` | **New kind (the happening half).** | Kernel and module happenings: `kernel.run.*`, `kernel.refused`, `swarm.round`, `swarm.question`, `clock.tick`. Rule: events never match a wildcard type query. A subscription must name the event type exactly, so the kernel's own audit records cannot start reactor storms. Events never count as sink input. |

Result: **six kinds, but not DM's six**: `doc`, `claim`, `decision`, `task`, `act`, `event`.
Every one has a rule. `canConnect` disappears as a cable check and reappears in two places. The
first is **activation-time checks** on a subscription. Its query must be satisfiable by some
active writer. A query on `decision status=accepted` is satisfiable only by a human, so the
graph labels that edge "human gate". The second is the **append-time ACL**.

### 4.3 Schema evolution

atproto rules are kernel law (DM §3.4):

- type ids are namespaced (`swarm.handback`, `kb.claim`);
- changes are additive only, with new fields optional;
- open unions, and `knownValues` instead of enums;
- a breaking change gets a new type id (`kb.claimV2`).

A lens is a reactor that reads `V1` and appends `V2` with `derivedFrom`. The kernel's own
envelope is versioned as a whole, the Segment rule (FM 4). It gets a `v` field only if it ever
changes, which it should not. Old records are never migrated. They are immutable. Folds
tolerate both versions, or a lens backfills (`from: all`, budgeted).

### 4.4 Storage

```
~/ledger/
  log/devA/2026-10/000017.jsonl   # canonical JSON, one record per line; per-device segments
  blobs/sha256/9c/1e…             # content-addressed bodies
  keys/                           # instance keys; the human key is NOT here (OS keychain)
  index.sqlite                    # derived from the envelope only; `ledger reindex` rebuilds it
  views/                          # derived projections, e.g. markdown exports per space
```

There is one ledger per user, with `space` as the scope, ACL and sync boundary. That settles
DM open risk 4 (per user or per project): per user, because cross-project decisions and claims
are the point. Projections into project repos, such as `docs/product/decisions.md` between
swarm's markers (D-023, D-025), are written by a projection effector, and they are never a
second source of truth (INV implication 8).

**Human editing of an "immutable" doc** (DM open risk 3). The writing tool owns a mutable
*working copy*: a Markdown file, or an Automerge doc as Patchwork does it. An explicit save, or
30 s idle, appends a new revision with `prev`. The kernel only ever sees revisions. Reactors on
drafts use `debounce` so that typing does not dispatch.

**Deletion.** Append `act kernel.redact {target}`. The kernel drops the body and blobs of the
target, keeps its id and envelope as a tombstone, and folds respect it. Provenance chains stay
intact but show "redacted". This is the only non-append operation, and it is human-only.

### 4.5 Sync

The log is a grow-only set of immutable, signed records, so the union of two replicas is always
valid (a G-Set). Order inside the log is a hybrid logical clock. Order inside a subject is `prev`.
Concurrent revisions of one subject become a **fork**. A fork is visible in the inbox, and
`head: true` subscriptions refuse to dispatch on it (`reason: fork`) until a human, or a declared
merge reactor, appends a revision whose `prev` covers both heads (`prev` may be a list). Per-device
segment files mean git can carry a ledger with no merge conflicts. Multi-party sharing (Keyhive-
style capabilities, DM §2.1) comes later and needs no change to the record model.

---

## 5. Three worked compositions

A composition is a bundle of `kernel.instance` and `kernel.subscription` records sharing a
label `composition: <name>`. The user's file is the bundle as YAML. `ledger apply c1.yaml`
appends it, and `ledger compose show c1` renders the derived graph. Presets ship as such files
(FM 2).

### C1 — research → knowledge base → writing, every claim traced to a source

```yaml
# compositions/c1-sourced-article.yaml
composition: c1
space: compose
instances:
  - { id: research#c1,   module: research.web@0.4,   grants: [net:https, model:claude], config: { maxSources: 12 } }
  - { id: kb-extract#c1, module: kb.extract@0.3,     grants: [model:claude] }
  - { id: writer#c1,     module: writing.drafter@0.2, grants: [model:claude] }
  - { id: citecheck#c1,  module: writing.citecheck@0.1, grants: [] }       # deterministic, no model
  - { id: inbox,         module: kernel.inbox@1 }                          # sink: the human
subscriptions:
  - instance: research#c1
    on:   { kind: task, type: research.brief, status: admitted, labels: { piece: "*" } }
    mode: each
    bounds: { rounds: 1, usd: 3.00 }
  - instance: kb-extract#c1
    on:   { kind: doc, type: research.source, head: true }
    mode: each                                   # one extraction per source
    bounds: { rounds: 1, usd: 0.50 }
  - instance: writer#c1
    on:                                          # primary: the piece; then the claim set
      - { kind: task,  type: writing.piece, status: admitted, labels: { piece: article-1 } }
      - { kind: claim, type: kb.claim, labels: { piece: article-1 } }  # re-run when claims change
    mode: batch
    debounce: 60s
    bounds: { rounds: 3, usd: 2.00 }
  - instance: citecheck#c1
    on:   { kind: doc, type: writing.draft, head: true }
    mode: each
  - instance: inbox
    on:   { kind: doc, type: [writing.draft, citecheck.report], head: true }
```

Derived graph, shown at `apply`:

```
human ─task research.brief─▶ research ─doc research.source─▶ kb-extract ─claim kb.claim─▶ writer
human ─task writing.piece──────────────────────────────────────────────────────────────▶ writer
writer ─doc writing.draft─▶ citecheck ─doc citecheck.report─▶ inbox
writer ─doc writing.draft─────────────────────────────────────────────────────────────▶ inbox
```

**Walk-through.**

1. The human appends two tasks with `labels.piece: article-1`. `r1 = task research.brief
   {question, budget $3}` and `r2 = task writing.piece {angle, length}`, both
   `status: admitted` and human-signed. Each is a lineage root.
2. The dispatcher sees `r1` match the research subscription. Reachability holds (research →
   kb-extract → writer → inbox), so it appends `kernel.run.started` and spawns `research`. The
   run fetches pages, and **the egress goes through the kernel's fetch proxy** (cap `net:https`),
   which records `{url, sha256 of response}` into the reads-set. It stages 9 records
   `doc research.source {url, title, retrievedAt, text}`, each with a blob of the raw page and
   `derivedFrom: r1`.
3. Nine heads now match kb-extract (`each`), so there are nine dispatches. Each reads one
   source and stages 2–6 `claim kb.claim {text, quote, locator}` records with
   `derivedFrom: <source id>`. Labels (`piece: article-1`) are inherited from the trigger.
   The bound key is per source subject, so each of the nine gets its own `rounds: 1` and
   `usd: 0.50`. The kernel checks every claim. A claim that cites a source the
   run never fetched is rejected, and so is a claim with no source.
4. The writer subscription is `batch` over `r2` (primary, so the lineage root is `r2`) plus
   the claim set, debounced 60 s. Each change to the claim set gives a new idempotency key, so
   the writer may re-run, at most 3 times under root `r2`. It
   dispatches once the extraction runs settle. The writer queries the claims for `article-1`.
   All of them go into the reads-set. It stages `doc writing.draft` (subject
   `writing/article-1`) whose Markdown cites claims inline as `[^c:sha256:9c1e…]`. Its
   `links` hold `cites → claim ids`. The kernel checks `cites ⊆ reads`.
5. `citecheck` is deterministic. It parses the draft. For every footnote it resolves claim →
   `derivedFrom` → `research.source` → `{url, response hash}`. It flags unresolved footnotes,
   and it flags sentences with numbers or proper nouns that carry no footnote. That second check
   is a heuristic and is labelled one. The run appends `doc citecheck.report {status: pass|fail}`.
6. The inbox shows the draft and the report. `ledger why <draft> --para 3` prints the chain:
   paragraph → claim (quote, locator) → source (url, fetched-at, hash) → research run → brief
   → human. **Traceability is a property of the store, not of the writer's good behaviour.**
7. If the human appends a new source by hand (a `doc research.source` written by the human),
   kb-extract fires on it, and the writer re-runs within its `rounds: 3`.

**What the kernel guarantees, and what it cannot.** It guarantees that every claim links to a
source that the producing run *actually read*, and every draft footnote to a claim it actually
read. It cannot guarantee that the source *supports* the claim. That is semantic, and it needs a
verifier module (an LLM entailment check that emits `claim verify.finding`), which is optional
and visible in the graph.

### C2 — swarm ⇄ decision register, with a human gate

```yaml
# compositions/c2-swarm-decisions.yaml
composition: c2
space: sonic
instances:
  - { id: swarm#sonic, module: swarm-adapter@0.1, config: { repo: ~/www/sonic, target: main },
      grants: [exec:swarm, swarm:human-token:effector-only] }
  - { id: register,    module: kb.decisions@0.1 }        # fold + markdown projection
  - { id: project-md,  module: effect.project-markdown@0.1,
      config: { file: ~/www/sonic/docs/product/decisions.md,
                markers: ["<!-- swarm:decisions:begin -->", "<!-- swarm:decisions:end -->"] },
      grants: [fs:write:~/www/sonic/docs/product/decisions.md] }
  - { id: inbox,       module: kernel.inbox@1 }
subscriptions:
  - instance: swarm#sonic                          # start a feature, pinning the accepted basis
    on:   { kind: task, type: swarm.feature, status: admitted }
    reads: { kind: decision, type: kb.decision, status: accepted, head: true }  # read scope, not a trigger
    mode: each
    # no usd bound here: the dispatcher enforces the trigger task's body.budget (§2.4 step 6)
  - instance: swarm#sonic                          # human acts drive swarm
    on:   { kind: act, type: [swarm.reply, swarm.approve, swarm.drop] }
    mode: each
  - instance: register
    on:   { kind: decision, head: true }
    mode: batch
  - instance: project-md
    on:   { kind: doc, type: kb.decisions.projection, head: true }
  - instance: inbox
    on:   { kind: [decision, event, doc], type: [kb.decision, swarm.question, swarm.handback, kernel.refused],
            status: [proposed, null] }
```

Derived graph. The edge into swarm's basis is satisfiable only by a human:

```
human ─task swarm.feature─▶ swarm#sonic ─doc swarm.handback / decision(proposed) / event swarm.question─▶ inbox
inbox(human) ─act swarm.approve + decision(accepted)─▶ swarm#sonic   [HUMAN GATE: ACL]
decision(any) ─▶ register ─doc kb.decisions.projection─▶ project-md ─▶ docs/product/decisions.md
decision(accepted) ··read-scope··▶ swarm#sonic (next feature's basis)
```

**Walk-through.**

1. **Migration first.** `ledger import swarm-decisions` is run by the human and signed with the
   human key. It is not a reactor, because an instance principal may not write `accepted`. It
   reads swarm's two formats today: `.swarm/decisions/<id>.json` (classic, with D-ids and
   revisions) and `.swarm/simple/decisions.jsonl` (simple, with no D-ids). It appends one
   `kb.decision` per subject. Classic entries keep their D-id as subject and their status, so
   sonic's D-001 and D-017 stay `proposed`. Simple-model lines get minted subjects
   (`kb/decision/<feature>-<n>`) and status `accepted`, because approve wrote them. The whole
   import is one human-signed batch.
2. The human appends `task swarm.feature {idea: "Latency per route", capUsd: 8, dependsOn: []}`,
   `status: admitted`.
3. The dispatcher spawns an ingest run on the adapter. The adapter queries
   `kb.decision status=accepted head` in `space: sonic`, so its reads-set holds the accepted
   heads. It
   writes a basis file `[{decisionId: "D-018", revision: "sha256:…"}…]` and calls
   `swarm add "<idea>" --budget 8 --basis <file>`. **The basis is pinned by hash, and provenance
   proves which decisions the feature was built against.** That closes the "basis is always
   `[]`" gap (INV §1).
4. Swarm runs its own rounds, in its own sandbox, on its own clock. The adapter tails SSE. On a
   `result` hand-back it stages, under the ingest run:
   - `event swarm.round {n, costUsd, snapshotSha}`;
   - `doc swarm.handback {built, tryIt, checks, risks}`, with the snapshot as a blob ref;
   - one `decision kb.decision` per `handback.decisions[]` with `status: proposed`, `body {title, choice,
     rejected}`, `links: [{rel: about, to: feature}, {rel: relation, kind: follows|conflicts|supersedes, to: D-xxx}]`.

   A `question` hand-back becomes `event swarm.question {question, options, recommended,
   needs: human}`.
5. The inbox shows the hand-back, the preview link and the proposed decisions, with
   `conflicts` flagged against the accepted ones. **Agents read; only the human approves.**
   - Sending feedback appends `act swarm.reply {feature, text}`, which wakes the adapter, which
     runs `swarm reply`. Swarm's round loop stays swarm's.
   - Approving is **one atomic, human-signed batch**: `act swarm.approve {feature,
     snapshotSha}`, plus, for each proposed decision the human accepts, a new revision
     `status: accepted, prev: <proposed id>`. The human may reject or edit some in the same
     batch.
6. The adapter, acting as an effector, sees the approve act. It checks that
   `act.prov.principal` is human, and that `snapshotSha` equals swarm's current round. Then it
   runs `swarm approve`. Swarm merges exactly that sha (D-014). The adapter stages
   `event swarm.merged {mergeSha}`. **The token is used only to carry out an act whose author
   the kernel has verified.** If the sha has moved on (a newer round), the adapter refuses and
   appends `kernel.refused {reason: stale-approval, needs: human}`.
7. The register folds all decision heads into `doc kb.decisions.projection`. `project-md`
   rewrites sonic's `decisions.md` between swarm's markers. Swarm stops writing into other
   repos (INV implication 7). The ledger does, through an effector with a single-file grant.
8. **Feedback.** The next `task swarm.feature` reads the basis again, and the newly accepted
   decisions are in it. The writing tool (C1) and any KB module can read the same `decision`
   records. This is the "first cross-module composition" DM ends on.

**Why the loop is bounded.** The cycle swarm → decision(proposed) → … → swarm(basis) only
closes through `decision status=accepted` and `task status=admitted`, and the ACL makes both
human-only. The kernel finds this statically: every cycle in the derived graph must contain at
least one edge that only a human can satisfy, *or* every subscription on the cycle must carry
`rounds`. Otherwise `apply` is refused. That is the knowledge-work version of "every loop has a
one-quantum delay" (PB, DM §4).

### C3 — a draft cycles through a critic for at most N rounds, then waits for the human

```yaml
# compositions/c3-critic-loop.yaml
composition: c3
space: compose
instances:
  - { id: critic#c3,  module: writing.critic@1.2,   grants: [model:claude] }
  - { id: reviser#c3, module: writing.drafter@0.2,  grants: [model:claude], config: { mode: revise } }
  - { id: inbox,      module: kernel.inbox@1 }
subscriptions:
  - instance: critic#c3
    on:   { kind: doc, type: writing.draft, head: true, labels: { piece: article-1 } }
    mode: each
    debounce: 30s                    # a human typing does not dispatch
    bounds: { rounds: 3, usd: 1.50 }        # N = 3
  - instance: reviser#c3
    on:   { kind: doc, type: critique.review, status: revise, head: true }
    reads: { kind: claim, type: critique.finding, links: { about: $trigger.links.about } }
    mode: each
    bounds: { rounds: 3, usd: 3.00 }
  - instance: inbox
    on:   [ { kind: doc, type: critique.review, status: pass },
            { kind: event, type: kernel.refused, labels: { composition: c3 } },
            { kind: doc, type: writing.draft, head: true } ]
```

Derived graph. The cycle carries a bound on both edges, so `apply` accepts it:

```
human/writer ─doc writing.draft─▶ critic ─doc critique.review(revise)+claim critique.finding─▶ reviser
reviser ─doc writing.draft (prev=…)─▶ critic                    [CYCLE, rounds ≤ N=3 per lineage]
critic ─doc critique.review(pass)─▶ inbox ;  kernel ─event kernel.refused(bound)─▶ inbox
```

**Walk-through, N = 3.** The critic writes every review as a revision of one subject per draft
(`critique/writing/article-1`). So in `each` mode the critic's bound key is
`(critic, root, writing/article-1)` and the reviser's is `(reviser, root,
critique/writing/article-1)`. Each loop has one counter per lineage.

| Step | Append | By | Lineage root / round count |
|---|---|---|---|
| 0 | `d0 doc writing.draft` (subject `writing/article-1`) | human (save) | root = d0 |
| 1 | `k1 kernel.run.started` → critic reads d0 → `f1a, f1b claim critique.finding (derivedFrom d0)`, `v1 critique.review status=revise about d0` | critic | critic round 1 |
| 2 | reviser reads v1, f1a, f1b, d0 → `d1 writing.draft prev=d0` | reviser | reviser round 1 |
| 3 | critic on d1 → `v2 revise` + findings | critic | critic round 2 |
| 4 | reviser → `d2 prev=d1` | reviser | reviser round 2 |
| 5 | critic on d2 → `v3 revise` | critic | critic round 3 |
| 6 | reviser → `d3 prev=d2` | reviser | reviser round 3 |
| 7 | critic would fire on d3: rounds(critic, root d0) = 3 = N → **no run**. `event kernel.refused {reason: bound, sub: critic, root: d0, needs: human}` | kernel | stop |
| 8 | inbox shows d3, the chain d0→d3, v1–v3, all findings, and the cost per round | — | waiting |

The loop ends in one of three ways:

- **It converges first.** If at step 3 the critic returns `v2 status=pass`, the reviser's query
  (`status: revise`) does not match. The loop ends, and the inbox gets the pass.
- **It oscillates.** If the reviser produced a body equal to d1 again, the kernel refuses the
  append with `reason: oscillation`. That is Kubernetes "converges or visibly oscillates" (PA
  §10).
- **The human acts.** The human edits d3 in the writing tool and saves `d4 prev=d3`. d4 is
  human-written, so it is a **new lineage root**, and the critic gets a fresh N. Or the human
  appends `act writing.hold {subject}`, which both subscriptions exclude by query. Or they
  simply publish (`act writing.publish`, an effector).

**Concurrency.** If the human saves d3' (`prev=d2`) while the reviser is writing d3
(`prev=d2`), the subject forks. `head: true` subscriptions refuse with `reason: fork`, and the
inbox asks the human to pick or merge. Agents never auto-merge a human's text.

---

## 6. Score against the 16 kernel tests (FM, "Implications for the kernel")

| # | Test | Score | One line |
|---|---|---|---|
| 1 | Single-module | **Pass** | The kernel imposes nothing on standalone use. Swarm runs unchanged, and the adapter is optional. The writer, KB and research tool do not exist yet, so this is by design, not proven. |
| 2 | Monolith-first | **Partial** | The envelope and kinds are drawn from what swarm actually emits (rounds, hand-back, decisions, snapshot). The KB, writer and research tool are not built, so their boundaries are guesses. |
| 3 | One-page spec | **Pass** | §1 fits on a page. It holds only while the query language stays envelope-only (weakness 4). |
| 4 | Native payload | **Pass** | Bodies are Markdown/JSON and blobs are files. The envelope is thin, and modules never convert to an object model. |
| 5 | Ceiling | **Pass** | Branch: several subscriptions on one type. Filter: the query. Merge: `batch` (union). Loop: bounded per lineage. Joins are clumsy (weakness 3). |
| 6 | M+N | **Pass, strongest** | A new module is a manifest plus instance and subscription records. Nobody names producers, so existing modules and subscriptions are never touched. |
| 7 | Conformance | **Pass (designed)** | `ledger conform` checks envelope validity, writes ⊆ manifest, derivedFrom ⊆ reads, idempotent re-dispatch, timeout, oversize and forks. A module that fails cannot be instantiated. |
| 8 | Isolation | **Pass** | A process per run, nono sandbox, deadline and kill. Failure is a `kernel.run.failed` record on the instance card. No shared address space. |
| 9 | Capability | **Pass** | Read scope (space, labels, `reads`), the write ACL on kind/type/status, caps shown at `apply`, and a human key no sandbox holds. Grants are per instance and query, which is finer than MCP's per-server grants (AI gap 5). |
| 10 | Skew | **Pass** | Compositions name types, not module internals. atproto evolution rules apply. An upgrade that drops a declared write shows up as a graph diff (a dangling subscription) before activation. |
| 11 | Plain data | **Partial** | Instances and subscriptions are records exported as YAML. The *topology* is derived, not written, so the file alone does not show edges without `compose show`. |
| 12 | Probe | **Pass, strongest** | Every value on every edge is a record. `ledger tail`, `ledger why <id>` walks provenance, and `ledger why-not <sub> <id>` finds the refusal record. Cost per lineage is a sum over `prov.cost`. |
| 13 | Escape hatch | **Pass** | Raw JSONL and blobs. The run transcript, including the actual prompt, is a blob on `kernel.run.finished`. A module may read raw records. |
| 14 | Front door | **Partial** | Presets are record bundles (C1–C3 as files). But a blackboard preset is harder to *show* than a drawn patch, and none of the modules exist yet. |
| 15 | Outlives the kernel | **Pass, strongest** | JSONL with canonical JSON, sha256 blobs and Markdown projections. Readable with `jq` and `grep` in twenty years. |
| 16 | Whole-composition budget | **Partial** | Runtime caps per subscription, lineage and task are exact. A *pre-run* sum is only an upper bound (Σ over reachable subscriptions of rounds × declared cost), and fan-out such as "how many sources?" makes even that loose. |

Tally: 12 pass, 4 partial (2, 11, 14, 16), 0 fail. The partials are the honest centre of the
critique.

---

## 7. Weaknesses, and what this design makes hard

1. **The composition is not where you look for it.** In A the cable is drawn. In B, "why did the
   critic run?" is answered by a query join plus provenance. The derived graph is sound
   (writes ⊆ manifest), but it is *computed*, so a user can be surprised. Installing a second
   module that writes `writing.draft` silently feeds the critic in that scope. The mitigation is
   that `apply` and activation show a **graph diff** ("this adds edges X → critic, X → inbox")
   and require confirmation. That is a strong mitigation, but it is porcelain the design cannot
   live without. Kubernetes' "selector wiring is powerful but invisible" (PA §10) is the
   standing warning.

2. **Debugging is causal archaeology.** Every dispatch and refusal is a record, so nothing is
   lost. But the reader has to follow `trigger`, `root` and `reads` across asynchronous runs.
   Without a good `why` and timeline view, B degrades to "read many controllers' logs". The raw
   material is the best of any design here. The default experience depends on tools not yet
   built.

3. **Sequencing and joins are awkward.** "Publish only after citecheck passed *and* the human
   approved" is a join across records. It needs either an `all-of` subscription (more query
   language) or a small `gate` reactor that folds the conditions and appends
   `doc gate.open`. B pushes control flow into status fields and gate records, the
   state-machine-in-data style that Pd needed `[trigger]` for (PA §7). Linear pipelines that
   are trivial in A or in Unix pipes take two or three records here.

4. **Query-language creep is the main threat to the one-page spec.** Every composition above
   wanted a bit more: list-valued `on`, `reads`, `$trigger` references, `status: [x, null]`. The pull is
   towards Datalog over bodies. The firewall is "envelope only, no body predicates". When a
   module wants a body predicate, it must promote the field to `status` or `labels` on write.
   Holding that line is a governance problem, not a technical one.

5. **Push by default, so fan-out is surprising.** One research brief became nine extraction
   runs. Bounds cap each subscription, but the *count* of dispatches is data-dependent, which is
   why test 16 is only partial. `each` versus `batch` is a cost decision users will get wrong.

6. **Two reactors can fight over a subject.** If two instances both revise
   `writing/article-1`, they ping-pong. Oscillation detection catches exact repeats, not
   near-repeats. The rule needed is single-writer per subject per role: a subject declares its
   `writers`, like Kubernetes field managers or ownerRefs. That is one more envelope concept,
   and it is not yet in §1.

7. **Late consumers and replay.** The log's great feature (a new module can replay history) is
   also a token bomb. That is why `from: now` is the default and `from: all` needs an explicit
   budget. Backfilling a lens over two years of records is a real cost decision, not free.

8. **Some interactions are request/response, not records.** "Writer asks the KB: what do we know
   about X?" B answers it with the KB's claims queried as records. Semantic search over
   embeddings, though, is a service, not a record set. The honest answer is a second channel:
   *query modules*, read-only MCP tools whose call and response the kernel memoises as a
   `kernel.read` record, so the reads-set still covers them. It works, but it is a seam the pure
   ledger story does not have.

9. **External effects are not transactional.** `swarm approve` merges a branch. If the adapter
   crashes between the merge and appending `swarm.merged`, the ledger lags reality. Effectors
   need idempotency keys and a reconcile-on-start (read swarm's state, append what is missing),
   which is Kubernetes level-triggering again. It is solvable, but every effector author has to
   get it right. The contract test must probe it.

10. **External provenance is only as good as the fetch proxy.** "derivedFrom ⊆ reads-set" is
    airtight for ledger reads. For the web it relies on egress going through the kernel's proxy,
    which is a host service with real complexity (TLS, rate limits, PDFs). Without the proxy, a
    source's `url` is the module's word.

11. **Append-only versus privacy and size.** Redaction tombstones exist, but a body that leaked
    into a downstream claim's quote stays there until those records are redacted too. A log that
    grows with every agent run needs compaction of derived views (cheap) and blob GC (needs
    reachability from heads plus a retention policy).

12. **It is the least "Unix pipe"-like of the three.** Users who think `research | kb | write`
    will find B indirect. B's mental model is the spreadsheet (cells reference by name and
    recompute) and Kubernetes (controllers converge), not the pipe. That is a teaching cost.

**What B makes hard**, in summary:

- seeing the composition at a glance, without tooling;
- ordered multi-step control flow and joins;
- pre-run cost estimates under data-dependent fan-out;
- low-latency streaming (token streams to a UI belong outside the ledger);
- exactly-once external effects;
- deletion.

**What B makes easy**, the reason to pick it anyway:

- provenance by construction ("every claim traces to a source" is a kernel invariant);
- authority as an ACL instead of a wiring rule;
- M+N with zero edits to existing modules;
- probes for free;
- replay and "re-run from here";
- data that outlives every module and the kernel itself.

**Recommendation to the synthesis.** If B is chosen, keep the four declared wiring pieces
(subscription, scope, bounds, sinks), and treat `compose show`, the graph diff and `why` as part
of the minimum product, not as polish. If A is chosen, steal B's ledger as A's substrate:
cables carrying record refs over this envelope, provenance stamping with reads-sets, and
refusals as records. DM and PA §implications already point there. The real choice is whether
the user-facing composition is *drawn edges* (A) or *declared interests* (B). The store
underneath should probably be this one either way.
