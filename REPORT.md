# Open Engine — Synthesis & Architecture Dissection

Sources: `transcript.txt` (video), `substack.txt` (article), `Open Engine _ Unlock AI.mhtml` (the build guide, decoded to `guide_text.txt`).

---

# PART 1 — Synthesis: what Open Engine *is* and why it exists

## The one-sentence claim
> Your AI agents don't talk to each other, so **you** have become the human glue moving work between Claude, Codex, and ChatGPT. Open Engine is a shared task queue that lets agents hand off work, carry the sources, and leave a receipt — without you stuck in the middle.

## The problem, stated precisely
The popular story is "agents go autonomous and take work off your plate." Nate's correction: the model is no longer the bottleneck — **the handoff between agents is.** A real job today doesn't live in one tool; it *commutes*:

> A client call → Claude finds the argument → Codex edits the file → ChatGPT re-reads the draft → a browser agent checks the page rendered → Slack holds the conversation → Linear holds the task → the calendar decides if any of it survives the afternoon.

That's **not seven jobs. It's one job crossing seven systems.** Every crossing, a human carries the *state*: what was decided, which source mattered, what changed, what the next tool is allowed to touch. **The integration layer is you.**

Two memorable framings:
- **"Agents are loop managers, and you are the hallway."** A useful agent is a remembered workflow that reruns, notices what changed, and pulls you in for the real decision. That works for *one* agent. But when every loop lives in its own room, the human becomes the hallway between the rooms.
- **Output vs. Work.** *Output* is what the AI returns right now (a beautiful brief in a private chat). *Work* is something the next person or agent can review, accept, route, and build on. "A beautiful brief in a private chat is just a draft in a room by itself." Open Engine is the move *from output to work without making humans the copy-paste path.*

## The reframe that does all the heavy lifting: Prompt mode → Work mode
| | Prompt mode | Work mode |
|---|---|---|
| Ask | "Write me a follow-up email." | "Here's the call transcript, the decision, the promise, the calendar constraints. Draft the follow-up, flag what needs my judgment, leave notes I can review." |
| Shape | Asks for an *answer* | Asks for a *result*, with owner, sources, boundaries, and proof |
| Who can continue | Only you, in that chat | Any next agent or human, off the record |

A **prompt asks for output. A handoff explains the job.** Most agent failures aren't failures of language — they're failures of **ownership, source, boundary, status, and proof.**

## Where it sits in Nate's larger thesis
- **Open Brain** = memory ("stop making every AI start from zero").
- **Open Skills** = your way of working stays yours.
- **Open Engine** = coordination ("stop making the human carry the handoff").
Memory helps the AI *know more*; coordination helps the *work survive the trip* between tools, people, and decisions.

## Positioning vs. the field
- **OpenClaw** — makes the desire obvious: people want local agents that *act* outside a chat box. But "autonomous" ≠ solved; a capable private agent just becomes another inbox you manage.
- **Hermes** — repeated/scheduled loops that learn the pattern. Real value, but still needs somewhere for the work to live.
- **Symphony (OpenAI/Codex)** — issue trackers as control planes for always-on coding agents. Same core assumption: work needs a place to live, a way to be claimed, a way to be reviewed. Aimed at engineers comfortable turning a tracker into an always-on system.
- **Open Engine's distinct promise:** the *smallest shared handoff* an ordinary operator or team can use — one queue, one task record, visible limits, visible status, receipts — that works *across* harnesses instead of crowning one. "What they share is the task record, not the model."

## The acceptance test (how you know it works)
Can the work **(1) get out of your chat, (2) carry its sources, (3) respect its limits, and (4) come back with a receipt** — "this is what I did, this is what I didn't, here's the proof"? If yes even in a small way, the question changes from *"what can one AI do in a session?"* to *"what can my whole system carry without me being the messenger?"*

---

# PART 2 — Architecture dissection (how to think like the architect who built it)

## 0. The big idea in one line
**Open Engine is a distributed task-queue / blackboard coordination system whose message bus is a human-readable issue tracker, and whose workers are heterogeneous LLM agents.** Nearly every design decision is a classic distributed-systems pattern *re-applied to agents* — and the cleverest move is that **almost none of it is new infrastructure.** It's a *protocol* layered on tools you already have.

> `★ Insight ─────────────────────────────────`
> The whole system is "borrowed substrate + a written contract." Linear already has issues, assignees, statuses, labels, comments, and history. Those six features happen to be exactly: a work item, an addressing scheme, a state machine, a routing tag, an event log, and an audit trail. The architect's first act of genius is *recognizing that a project-management tool is already 80% of a multi-agent message bus* — so the build reduces to writing the protocol, not the plumbing.
> `─────────────────────────────────────────────`

## 1. Top-down: the system as a whole
A working engine, per the guide's Overview, is nine things working together:

> a **queue**, **private setup context**, a **status ledger**, **standing updates**, a **repeatable runner**, **resumable blockers**, **human-thread holds**, **delegated follow-up**, and **one smoke-tested task**.

Group those into five architectural layers:

```
┌──────────────────────────────────────────────────────────────┐
│  PLANE                  │  COMPONENT(S)         │  CS ANALOGUE │
├─────────────────────────┼───────────────────────┼──────────────┤
│ 1. Substrate / Bus      │  Linear queue          │ message bus  │
│                         │  (6 statuses + label)  │ + work queue │
│ 2. Contracts / Schema   │  Task record (8 parts) │ message      │
│                         │  Naming convention     │ schema +     │
│                         │  (title brackets)      │ addressing   │
│ 3. State & Observability│  Status ledger         │ service      │
│                         │  Receipt vocabulary    │ registry +   │
│                         │                        │ event log    │
│ 4. Execution            │  Queue runner (1 loop) │ worker /     │
│                         │  Private context packet│ poller +     │
│                         │                        │ config/secret│
│ 5. Coordination (team)  │  Routing map           │ service      │
│                         │  Standing updates      │ discovery +  │
│                         │  Delegation/follow-up  │ shared config│
└──────────────────────────────────────────────────────────────┘
```

The guide's own table of contents *is* the build order, and it's the dependency order too:
`Overview → Before(naming) → Connect(MCP) → Queue → Context → Ledger → Runner → Receipts → Smoke test → Team → Templates → Troubleshoot.`
You cannot build a later layer before its dependency exists — e.g. the private context references the ledger issue ID, so the context file leaves it as a **placeholder** (`ENG-000`) until the ledger is created. That single placeholder is a lesson in **bootstrapping a system with circular references**: stub the forward reference, fill it in once the dependency materializes.

## 2. Bottom-up: the primitives

### Primitive A — The task record (the "message")
The atom of the system. The two sources describe it at **two different altitudes** — keep them distinct:

**(a) The article's conceptual checklist — "what a good task answers"** (the *seven parts*, the design intent):
1. **Requester** — who owns the decision (accountability anchor)
2. **Desired outcome** — what should *exist* when done (definition, not instruction)
3. **Sources** — what to read, what *not* to invent (grounding + anti-hallucination boundary)
4. **Acceptance criteria** — what "good enough to stop" means (the stop condition)
5. **Boundaries** — what must be asked before doing (the blast-radius fence)
6. **Blocker rule** — what kind of question, asked where (failure-handling contract)
7. **Receipt** — what to leave behind on claim/block/resume/finish (proof obligation)

**(b) The guide's literal `<task_issue>` body template (§11) — the *eight fields* you actually type:**
`requester · desired_outcome · context · sources · do · acceptance_criteria · output_handoff · boundaries`

Note the layering: the literal task body does **not** carry "blocker rule" or "receipt" as fields. Those are **protocol-level**, not per-task — the blocker rule lives in the runner prompt + private packet (Primitive E / §3), and receipts are the event log (Primitive C). The task body adds two operational fields the conceptual list folds away: **`context`** (why it matters) and **`do`** (step-by-step instructions), plus **`output_handoff`** so the *next* hop can find the artifact.

> `★ Insight ─────────────────────────────────`
> Notice this is just a **well-designed API request schema** for a non-deterministic worker. "Sources" = inputs + grounding. "Acceptance criteria" = the postcondition / test. "Boundaries" = authorization scope. "Receipt" = the structured response + side-effect log. The reason it reads like bureaucracy is that it *is* an interface contract — and the author's line "Call it bureaucracy if you want. It's respect for the task" is the architect admitting that **explicit contracts are the price of decoupling.** You pay in upfront verbosity to buy the ability for any worker to pick up any task cold.
> `─────────────────────────────────────────────`

### Primitive B — The status taxonomy (the state machine)
Six Linear statuses, and they are not decoration — they encode a lifecycle:

(`Standing` is a *parallel, never-closing category* for durable context — not a predecessor state in the task flow below.)

```
Standing        ← durable context (parallel lane; NOT a task; never closes)

Agent Todo  ──claim──►  Agent Working  ──┬── done, no review ──► Agent Done (completed category)
   ▲                         │           └── done, needs human ──► Agent Review
   │                         │
   └──── resume ──── Agent Needs Input ◄─┘  (blocked OR human-hold)
```

Design decisions worth copying:
- **`Agent Working` is the visible lock.** Moving the issue there *is* the claim. (More on locking below.)
- **`Standing` is a separate category** so durable context never gets "completed" and swept away. Config and tasks must not share a lifecycle.
- **`Agent Done` is mapped to Linear's "completed" category**, but `Agent Review` is not — the system distinguishes "the agent is finished" from "the *work* is finished," which is the honest middle most demos skip.

### Primitive C — The receipt vocabulary (the event protocol / observability layer)
Eleven exact tokens agents post as comments (a couple more — `AGENT AUTOMATION READY`, `AGENT CONNECTION TEST` — appear in the setup/connect steps outside this canonical list). This is the **append-only event log** that makes the system inspectable:

| Token | Meaning | DS analogue |
|---|---|---|
| `AGENT CLAIMED` | work started; the lock is taken | acquire-lock event |
| `AGENT DONE` | scoped work finished | success result |
| `AGENT BLOCKED` | needs an answer that belongs *on the issue* | recoverable pause (data) |
| `AGENT UNBLOCKED` | the on-issue answer arrived | pause cleared |
| `AGENT HUMAN HOLD` | needs an answer in the *owner's own thread* (permission/install/auth) | escalation to privileged actor |
| `AGENT HUMAN ANSWERED` | the hold was answered | escalation cleared |
| `AGENT RESUMED` | continuing a paused issue | resume from checkpoint |
| `AGENT FAILED` | unrecoverable; records last safe step + retry count | dead-letter / crash report |
| `AGENT APPLIED` | a runtime actually installed a standing context version | config-applied ack |
| `AGENT FOLLOW-UP` | a delegated issue this agent routed changed state | callback/webhook |
| `AGENT STATUS` | the single ledger comment each agent updates in place | heartbeat / health check |

> `★ Insight ─────────────────────────────────`
> The split between **`AGENT BLOCKED`** (answer belongs *on the task*) and **`AGENT HUMAN HOLD`** (answer belongs *with the owner* — a permission, an install, account authority) is the most underrated design call in the whole system. It separates **"missing data"** from **"missing authority."** Data can be answered by anyone reading the queue; authority must route to a specific privileged human out-of-band. Most naive agent systems collapse both into "it asked a question and stalled." Splitting them is what lets data-blocks resolve in the open while auth-holds escalate privately — exactly how real incident response separates "need info" from "need approval."
> `─────────────────────────────────────────────`

### Primitive D — The naming contract (addressing & routing)
Titles follow `[agent instructions][<agent-code>][task] <outcome>`, plus the `agent-instructions` label, plus assignee. Eligibility requires **all** of: right label + `[agent instructions]` marker + agent-code in the second bracket + correct assignee + `Agent Todo` status. Agent codes are stable strings like `alex-codex`, `sam-claude`.

This is **content-addressable routing built out of a string convention.** No integration, no API contract between vendors — a Claude runtime and a Codex runtime coordinate purely because they agree on how to parse a title. That agreement *is* the interoperability layer.

## 3. The execution model: one loop, run as a heartbeat
The runner is **a single prompt the agent repeats**; one execution = one "run" = one heartbeat (triggered by hand, by the runtime's scheduler, or by cron). The ordering inside a run is the entire concurrency-control design:

```
1. Mark own ledger comment → "checking"            (announce liveness)
2. Standing preflight: compare context versions    (config refresh BEFORE work)
3. Resume HUMAN HOLDs first                         (privileged waits clear first)
4. Resume BLOCKED next                              (data waits clear next)
5. Check delegated issues you routed out            (follow up on callbacks)
6. ONLY THEN claim the oldest eligible Agent Todo   (FIFO fairness)
7. Claim → re-read → do scoped work → receipt → update ledger
8. STOP after exactly one task
```

Three rules carry disproportionate weight:

- **"Process exactly one task per run."** This is deliberate **idempotency + blast-radius control.** Keep failures small; make the receipt readable; "if the agent can't do one task cleanly, it has no business chewing through a queue." One unit of work = one unit of review.
- **"Move to `Agent Working` and leave `AGENT CLAIMED`, *then re-read the issue.*"** This is **optimistic locking with a read-after-write check.** The status move is the lock; re-reading after claiming detects a race where two runtimes grabbed the same issue. (The troubleshooting entry "Agent claims but another agent also works it" prescribes exactly this, plus scoping pickup by agent-code bracket when one operator runs two runtimes.)
- **Resume-before-claim ordering.** Paused work is always finished before new work is started, so the queue drains rather than accumulating zombie tasks. A blocked task is a **pause, never a terminal state** — the single most common way these systems rot is treating "blocked" as "dead."

> `★ Insight ─────────────────────────────────`
> "One task per run" looks like a performance limitation; it's actually the **transaction boundary.** Each run is an atomic, restartable unit: claim → work → receipt → stop. If the runtime dies mid-run, the worst case is one half-finished claimed issue with a visible `AGENT CLAIMED` and no `AGENT DONE` — instantly diagnosable, trivially recoverable. Compare a "drain the whole queue" loop that crashes at item 7 of 20: now you're forensically reconstructing state. **Small transactions are cheap to reason about; that property is worth more than throughput here.**
> `─────────────────────────────────────────────`

## 4. The trust & secrecy boundary (public method vs. private engine)
A deliberate split:
- **Public guide** teaches the *method* (the protocol, the status names, the receipt tokens).
- **Private context packet** (e.g. `~/.codex/skills/open-agent-engine/SKILL.md`, or a private `Standing` issue) holds the *engine*: local paths, allowed sources, account boundaries, org chart, brand voice, customer context, secrets, the ledger issue ID, and the agent's hard rules.

The packet's `Rules:` block is the **policy engine / authorization layer** that travels with every runtime: one-task-per-run, assignee scoping, label+title+bracket eligibility, preflight before work, claim semantics, the done/review/blocked/hold branching, and the ask-first list (publish, email, deploy, billing, credentials, destructive deletes, customer-facing changes).

> `★ Insight ─────────────────────────────────`
> This is the classic **"separate policy from mechanism, and secrets from code"** principle. The shared public protocol is the mechanism; each operator's private packet is the policy + credentials. It means two strangers (or two teammates) can interoperate on the same queue *without ever sharing their private rules or secrets* — they only share the protocol. That's the same reason OAuth scopes, not passwords, cross service boundaries.
> `─────────────────────────────────────────────`

## 5. The team layer: from one worker to a fleet
A team engine is "the same system with one extra rule: **route work to the human who owns the target agent.**" Components added:
- **Routing map** (private `Standing` issue): each human → Linear assignee → runtime → agent code(s) → ownership area. This is **service discovery** — how one agent finds where to send work.
- **Liveness gate:** "If the target agent is not online in the status ledger, say that before relying on the handoff." Never assume a consumer is alive; check the registry first.
- **The hard operational gotcha:** assign the issue to the *human who owns the target agent*, not to yourself — otherwise their automation (which filters on its own assignee) never sees it. This is a **misrouted-message** failure, and it's the team-level version of "wrong topic, no subscriber."
- **Standing updates:** one issue per context family; bump the version + changelog *in place*; agents compare target-vs-local version during preflight. This is **versioned config rollout** without a config server — pull-based, eventually consistent.

## 6. Verification is a first-class component
The smoke test is not an afterthought; it's step 6 and the guide refuses to let you trust the engine until **four behaviors** demonstrably work: **claim → done**, **blocked → resume**, and **human-hold → answer**, each as its own tiny throwaway task. "You are testing the loop, not the agent's intelligence." This is **integration-testing the protocol, not the worker** — you assert the state machine and the receipts, not whether the LLM is smart.

## 7. The unifying lens: every contract is a decoupling, and every decoupling removes you from the middle

Everything above can be read through one idea, and it's the idea that turns a pile of conventions into a *system*.

**A contract** is an agreed-upon interface two parties interact *through*, so neither has to know the other's internals. **Decoupling** is what a contract buys: it removes a direct, synchronous, knowledge-bearing dependency between two parties — they no longer have to be the *same* party, *present at the same time*, or *aware of each other's details* to cooperate.

Here is the move that makes Open Engine click: **before the contracts exist, the thing coupling every pair of tools together is _you_.** You are the universal adapter — the default coupling wherever no explicit interface exists. You carry the state Claude produced over to Codex; you translate the decision from the call into the email; you remember which source mattered; you decide who works next. Each contract in the system externalizes *one dimension* of that coupling into a shared artifact — and the instant it does, you stop being the conduit for that dimension.

### The contracts, and exactly what each one decouples

| # | The contract | Sits between… | What it decouples (so one side no longer needs the other through *you*) | How it pulls you out of the middle |
|---|---|---|---|---|
| 1 | **Task record / schema** (Primitive A — the 8-field `<task_issue>`) | the **requester** and the **worker** | *who asks* from *who executes*. The job is fully described in the artifact — sources, acceptance, boundaries — so **any** worker can pick it up cold. | You no longer have to be the one who *runs* what you requested. Asking ≠ doing. |
| 2 | **Status taxonomy / state machine** (Primitive B — 6 statuses) | the **state of work** and **anyone observing it** | *doing the work* from *knowing where it stands*. The status **is** the truth; reading it requires no conversation. It also splits **"agent done"** from **"work done"** (`Agent Done` vs `Agent Review`). | You don't have to be asked "where is this?" — the lane answers. You're pulled in only at `Agent Review`, the one honest place a human is required. |
| 3 | **Receipt vocabulary** (Primitive C — 11 tokens, the event log) | the **agent that did something** and **whoever reads it later** | *the act* from *the audience*. What happened is written once, append-only; the next reader reconstructs state from the log, not from your memory. | You stop being the **carrier of "what was decided."** The history holds it, so the next hop reads the artifact instead of asking you. |
| 4 | **Naming contract** (Primitive D — title brackets + label + assignee) | a **Claude runtime** and a **Codex runtime** (or any two) | *routing* from *vendor integration*. Heterogeneous agents coordinate purely by agreeing how to parse a title — no API between vendors. | You're no longer the **translator/router between tools.** The string convention *is* the interoperability layer; you don't hand-carry work across the vendor gap. |
| 5 | **Blocked vs. Human-Hold split** (Primitive C, the key receipt pair) | a **missing fact** and a **missing authority** | *missing data* from *missing permission*. Data-blocks resolve in the open (anyone reading the queue can answer); auth-holds escalate to a specific privileged human. | Only the questions that are *genuinely yours* (permission, install, account authority) reach you. Data questions stop landing on your desk. |
| 6 | **Execution contract** (§3 — one task per run + optimistic lock + resume-first) | **one unit of work** and **the rest of the queue** | *each task's success/failure* from *every other task's*. One run = one atomic, restartable transaction; a crash leaves one diagnosable `AGENT CLAIMED`. | You're not the **recovery mechanism.** Failures are small and self-describing, so you don't have to forensically reconstruct a half-drained queue. |
| 7 | **Policy/mechanism + secrets split** (§4 — public method vs. private packet) | **one operator** and **another** (or a runtime and its rules) | *the shared protocol* from *each party's private rules and secrets*. Two strangers run the same queue while sharing only the mechanism, never their policy or credentials. | You're not the **integration negotiator.** Interop needs no exchange of secrets or rulebooks, so onboarding another agent/teammate doesn't route through you. |
| 8 | **Versioned standing context + routing map** (§5 — config rollout + service discovery) | a **config change** and its **consumers**; a **sender** and a **target's location** | *publishing config* from *consuming it* (pull-based preflight), and *sending work* from *knowing where the target lives* (the map). | You don't **hand-deliver updates** or memorize who-runs-what. Agents self-update on preflight and look up the route themselves. |

### The concrete artifacts behind each contract

The contracts above are abstractions; here is what each one actually *is* on disk and in Linear. The striking pattern: **almost nothing is a local file.** The artifacts live in the borrowed substrate (Linear issues, statuses, comments, titles) — the operator's machine stays nearly empty. Only the private packet (contract 7) is a real file, and it exists precisely because it holds the secrets that *must not* go into the shared substrate.

1. **Task record / schema**
   - *Artifact:* one **Linear issue**, its body filled from the §11 `<task_issue>` template (8 fields).
   - *Inputs:* the requester's typed fields (requester · desired_outcome · context · sources · do · acceptance_criteria · output_handoff · boundaries).
   - *Outputs:* a created issue sitting in `Agent Todo`, plus the `output_handoff` pointer to wherever the resulting artifact will land.
   - *Local files:* **none** — it is a Linear issue.

2. **Status taxonomy / state machine**
   - *Artifact:* the **six custom workflow statuses** configured once in Linear team settings (guide step "create the six workflow statuses"), plus each issue's current `status` field.
   - *Inputs:* status transitions (a runtime *moving* the issue between lanes).
   - *Outputs:* the issue's lane — the single source of truth for "where this stands."
   - *Local files:* **none** — Linear configuration + the issue's status field.

3. **Receipt vocabulary**
   - *Artifact:* **comments on the Linear issue** — the 11 tokens posted append-only.
   - *Inputs:* a token + a short note, posted by the runtime at each lifecycle event.
   - *Outputs:* the issue's comment history (the event log) and the single in-place `AGENT STATUS` ledger comment.
   - *Local files:* **none** — Linear comments.

4. **Naming contract**
   - *Artifact:* the **issue title string** `[agent instructions][<agent-code>][task] <outcome>`, the `agent-instructions` **label**, and the **assignee** field.
   - *Inputs:* the title text + label + assignee chosen at creation.
   - *Outputs:* an *eligible, addressable* issue (the parse target every runtime agrees on).
   - *Local files:* **none** — it is metadata on the Linear issue.

5. **Blocked vs. Human-Hold split**
   - *Artifact:* two receipt comments (`AGENT BLOCKED` / `AGENT HUMAN HOLD`) + the `Agent Needs Input` status. The block answer lands **on the issue**; the hold answer lands **off-issue, in the owner's own agent thread/app**.
   - *Inputs:* the unanswered question, typed where the rule dictates (issue comment vs. private thread).
   - *Outputs:* either an `AGENT UNBLOCKED` comment (data arrived) or an `AGENT HUMAN ANSWERED` comment (authority granted).
   - *Local files:* **none in Linear terms** — but note the hold channel is an *external* surface (the human's private agent thread), the one artifact deliberately *outside* the shared queue.

6. **Execution contract (the runner)**
   - *Artifact:* the **runner prompt** — "one instruction your agent repeats" (the guide's Queue-run / runner prompt) — plus its **trigger**: a manual run, the runtime's scheduler, or a **cron job** (e.g. an Oz container / Codex automation).
   - *Inputs:* the current queue + ledger state at the moment of the run.
   - *Outputs:* exactly one issue advanced (`claim → work → receipt → ledger update`), then a stop.
   - *Local files:* the runner prompt usually lives **inside the private packet** (`SKILL.md`); the scheduler/cron definition is a small config in the runtime or cloud platform. The *contract itself* is text, not code.

7. **Policy/mechanism + secrets split**
   - *Artifact:* **the one real file** — a per-runtime **private context packet**, default `~/.codex/skills/open-agent-engine/SKILL.md` (guide line 212), optionally mirrored by a private `Standing` issue; on the public side, the **published guide** (method only, no secrets). MCP wiring also touches `~/.codex/config.toml`.
   - *Inputs:* local paths, allowed sources, account boundaries, the `Rules:` block, secrets/credentials, and the ledger issue ID.
   - *Outputs:* a loaded policy+secret context each runtime reads *before* it acts; nothing secret ever enters the shared queue.
   - *Local files:* **yes — `~/.codex/skills/open-agent-engine/SKILL.md` (+ `~/.codex/config.toml`).** This is the *only* contract whose primary artifact is a file, by design.

8. **Versioned standing context + routing map**
   - *Artifact:* **private `Standing` Linear issues**, one per context family — the `standing_skill` setup issue, the `standing_status` ledger, and the **routing map** issue (human → assignee → runtime → agent-code → ownership area). Each carries a **version + changelog** in its body.
   - *Inputs:* a version bump + changelog edit (in place — never a new ticket); the `AGENT APPLIED` receipt a runtime posts after it installs the new version locally.
   - *Outputs:* an updated Standing issue that agents compare against their local version at preflight (pull-based, eventually consistent).
   - *Local files:* the *target* is a Linear Standing issue; the **local mirror** is whatever the runtime installs into its own `SKILL.md` to match the bumped version.

> `★ Insight ─────────────────────────────────`
> Tally the "Local files" lines: **seven of eight contracts have none.** The entire coordination fabric — work items, lifecycle, event log, addressing, escalation, config rollout — is *rented* from Linear, so there is no database, no message broker, and almost no local state to back up, migrate, or corrupt. The lone exception is contract 7's `SKILL.md`, and it's local for the one reason that *forces* locality: it holds secrets and machine-specific paths that the shared substrate must never see. The architecture's "files" footprint is therefore a near-perfect readout of its trust boundary — public protocol in Linear, private secrets on disk, and nothing else.
> `─────────────────────────────────────────────`

### What the decouplings achieve, together

Stack them and a pattern appears — they are decouplings along the exact axes a human normally has to bridge by hand:

- **Decoupling in *identity*** (contracts 1, 4): the requester need not be the worker; one vendor need not integrate with another. → *You aren't the executor or the translator.*
- **Decoupling in *time*** (contracts 2, 3, 6, 8): the queue, the statuses, the receipts, and the versioned config all let work move **asynchronously** — no party has to be present at the moment of handoff, and a pause is a checkpoint, not a stall. → *You don't have to be in the room when the baton is passed.*
- **Decoupling of *knowledge*** (contracts 2, 3): state and history live in inspectable artifacts, not in your head. → *You aren't the memory.*
- **Decoupling of *authority from data*** (contract 5): only true permission/authority escalations are routed to a human. → *You're interrupted for judgment, never for lookup.*
- **Decoupling of *policy from secrets*** (contract 7): cooperation needs only the shared protocol. → *You aren't the trust broker.*

The throughline: **a human is the default coupling precisely wherever an explicit contract is missing.** "The integration layer is you" (Part 1) is not a complaint about effort — it is a *diagnosis*: every place you feel stuck in the middle is a place where two parties are cooperating with no interface between them, so the only available adapter is a person. Open Engine's whole design is to find each of those gaps and drop in the *minimum viable contract* that fills it.

### How this produces "without you stuck in the middle"

The goal was never to remove you from the system — it's to remove you from the **critical path of every handoff.** Watch what each role you used to play becomes once its contract exists:

- **Transport** (carrying state tool→tool) → the **queue + receipts** carry it.
- **Translation** (reformatting between vendors) → the **naming contract** makes them speak directly.
- **Routing** (deciding who's next) → the **routing map + assignee rule** decide it.
- **Memory** (holding what was decided) → the **status ledger + event log** hold it.
- **Recovery** (cleaning up failures) → the **one-task transaction** makes failures self-contained.

What's left for you after all five are externalized is only the **two things that are irreducibly human**: **judgment** (the `Agent Review` decisions you own) and **authority** (the `Human Hold` permissions only you can grant). You drop from being the *bus that everything rides* to being an *endpoint that gets escalated to*. The work can now traverse the entire pipeline agent-to-agent and come back to you with a receipt — and that, precisely, is the promise from the one-sentence claim: agents "hand off work, carry the sources, and leave a receipt — **without you stuck in the middle.**"

> `★ Insight ─────────────────────────────────`
> The deepest lesson is that **"without you stuck in the middle" and "explicit contracts everywhere" are the same statement read from two ends.** You are the cost of *implicit* coupling; a contract is what makes the coupling explicit; so each contract is literally one unit of you-removed-from-the-loop. This is why the author can say "Call it bureaucracy if you want. It's respect for the task" without irony — the verbosity of the task record is not overhead, it is the *price of your own freedom from the hallway*. Every field you fill in is a dependency you no longer have to satisfy in person.
> `─────────────────────────────────────────────`

---

# PART 3 — How to think about building systems like this

The guide's **Troubleshooting** table is secretly the best teaching artifact, because every failure maps backward to the primitive that prevents it. That mapping *is* the architect's mental model:

| Failure mode ("Tuesday failure") | Root cause | Primitive that fixes it | Transferable principle |
|---|---|---|---|
| "No issue exists" | wrong assignee / label / status / title | Naming contract | **Addressing must be exact and checkable.** |
| Two agents do the same task | claimed without locking | `Agent Working` move + re-read | **Locking + read-after-write.** |
| Ledger floods with heartbeats | new comment each run | One status comment, updated in place | **Idempotent state, not append spam.** |
| Blocked task never resumes | treated `BLOCKED` as terminal | Blocker rule + resume-first ordering | **Model "paused" distinctly from "failed."** |
| Agent asks permission in the wrong place | conflated data vs. authority | `BLOCKED` vs `HUMAN HOLD` | **Separate missing-data from missing-authority.** |
| Duplicate standing tickets | new ticket per update | One issue per context family + versioning | **Mutable versioned config, not new objects.** |
| Cross-agent work goes nowhere | assigned to self | Routing map + assignee rule | **Route to the owner, check liveness.** |
| Agent publishes/deploys/deletes | no ask-first list | Boundaries in private packet + task body | **Default-deny on high blast-radius actions.** |

### The architect's checklist this system embodies
When you build a coordination system for non-deterministic workers, you must answer the same questions a distributed-systems engineer answers — Open Engine answers each one with the *minimum viable* mechanism:

1. **Where does work live?** → a shared, inspectable store (the queue). *Not* a chat, *not* Slack — those can't hold state you can audit.
2. **How is a unit of work described?** → an explicit schema with sources + acceptance + boundaries (the task record).
3. **How does a worker claim work without collisions?** → a visible lock + read-after-write (status move + re-read).
4. **How do you know who's alive?** → a heartbeat registry (the ledger).
5. **How does work fail safely?** → typed pauses and a dead-letter (`BLOCKED` / `HUMAN HOLD` / `FAILED` + last safe step).
6. **How do you trust "done"?** → a receipt log, not a transcript audit.
7. **How do heterogeneous workers interoperate?** → a shared *protocol*, not a shared vendor.
8. **How do you contain damage?** → one task per run + default-deny boundaries on irreversible actions.
9. **How do you roll out config?** → versioned standing context + preflight comparison.
10. **How do you know it works?** → a smoke test of the *loop*, before you trust it.

### The five design philosophies to internalize
1. **Find the real bottleneck, then attack only that.** Nate resisted the impressive build (dashboard, background runner, agent-watching-agents) because every step toward impressive moved *away* from the thing he needed on a hard day. The bottleneck was the *boundary between agents*, so the system is almost entirely "boundary plumbing" and nothing else.
2. **Reuse substrate; write protocol.** The hard parts (persistence, comments, history, assignees, statuses) already exist in Linear. Don't build a message bus — *recognize* one you already own and define how agents speak over it.
3. **Make state explicit and inspectable.** Every important fact (who owns it, what stage, who's alive, what happened) is a human-readable artifact. "You don't abandon agent systems because they make mistakes — you abandon them because the mistakes become *mysterious*." Inspectability is the feature.
4. **Start with the smallest useful version and grow by need.** The "30-minute version" is one task, three statuses, one blocker rule, one receipt. "Don't start [with the full automation] unless you enjoy building systems more than using them." Architecture fantasies are how useful tools become weekend projects you never trust.
5. **Design from the failure modes backward.** The boundaries, the receipt split, the locking, the one-task rule — each exists because a specific, *boring*, recurring failure demanded it. "We don't have to make the agent smarter. We have to make the loop less ambiguous."

### The single most portable lesson
> When intelligence becomes cheap, **the moat moves to the work layer — the context and the handoffs that don't leave your system on their own.** The model is becoming a commodity; the coordination protocol around it is not. Build the queue that moves work between agents *before you become the bottleneck.*
