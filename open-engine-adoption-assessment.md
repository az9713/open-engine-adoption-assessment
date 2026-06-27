# Open Engine — Adoption Assessment for a Solo Developer

A decision record from this session: should a solo developer who does not use Linear adopt
and deploy the full Open Engine system? Short answer — **no, not now** — plus what to do
instead, and the signal that would change the answer.

---

## 1. The decision: skip full Open Engine adoption (for now)

**Verdict: don't install it.** For a solo developer in a simple environment who doesn't use
Linear, the full Open Engine system solves a problem that isn't present yet.

### Why — the system targets a pain a solo dev doesn't have
Open Engine exists to fix exactly one thing: **you being the human glue that moves work
between multiple agents and multiple people** (copy-pasting state between Claude, Codex,
ChatGPT, a browser agent, etc.). Almost every component only earns its keep under
multi-agent or multi-person conditions:

| Component | What it's for | Relevance to a solo dev |
|---|---|---|
| Shared queue + locking + claim semantics | Stop two agents colliding on one task | None — there's one worker |
| Routing map, liveness gate, "assign to the owner" | Team coordination / service discovery | None — you are the whole team |
| Receipt log / audit trail | Let *someone else* trust off-the-record work | Thin — the "someone else" is just future-you |
| Linear as substrate | Reuse an existing issue tracker as the message bus | Negative — you'd stand up a whole PM tool *as infrastructure* for a protocol you don't need |

Strip away multi-agent and multi-person, and little is left. The system's own acceptance
test — *"can work get out of your chat and come back without you being the messenger?"* —
assumes there's a trip to make. Solo, in one chat, there usually isn't.

### Why — the install cost is real and the author warns against it
The install is lightweight in absolute terms (connect a runtime to Linear's MCP server,
create a couple of issues, define ~6 statuses + a label, paste a private context packet),
but it's still **infrastructure for coordination that isn't happening.** This is the exact
failure mode Open Engine's own guide cautions against: *"Don't start with the full
automation unless you enjoy building systems more than using them."* Building a coordination
protocol for a fleet of one is the "architecture fantasy" trap.

### Net
The valuable part of Open Engine for a solo dev was never the machinery — it was the
**ideas**. Those were extracted and kept (see §3) without any install.

---

## 2. When the solo developer *should* adopt full Open Engine

Adopt it when the pain it's built for actually appears. Concrete triggers:

- **You're routing work between two or more agents by hand** — e.g. Claude Code edits a
  file, a separate browser agent verifies the render, and you're the one carrying state
  between them.
- **A teammate enters the loop** — work needs to be claimed, reviewed, and handed off by
  someone other than you, off the record.
- **You're running scheduled / always-on agents** whose work you need to trust without
  watching each run — at which point an inspectable queue and receipts stop being overhead
  and start being the thing that lets you trust delegation.
- **You already live in an issue tracker** (Linear or similar), so the substrate cost is
  near zero.

Until at least one of those is true, the three habits in §3 deliver the payoff with none of
the setup. The graduation rule, stated plainly: **the day you catch yourself being the
hallway between agents, build the queue — not before.**

---

## 3. The distillation: a 3-habit work-mode prompt template

Rather than adopt the system, this session distilled Open Engine down to the three habits
that transfer to a solo, no-infrastructure setup. They are captured in full — with
rationale, worked examples, an anti-pattern, and a real critique-and-tighten pass — in:

> **`work-mode-template.md`** (same directory as this file)

### The three habits
1. **Work mode, not prompt mode** — explain the *job* (result, sources, boundaries, stop
   condition), not just ask for *output*. This is Open Engine's task schema collapsed to
   what one person can hold in their head.
2. **One scoped task, then stop** — one unit of work = one unit you can check. Keeps
   failures small and diagnosable.
3. **Leave a receipt — even to yourself** — "Did / Didn't / Proof / Next" makes the work
   inspectable. Future-you is the "someone else" who has to trust it.

### The prompt (paste-and-go, verbatim from `work-mode-template.md`)

```
Work mode, not prompt mode. Before you act:

JOB:        <what should EXIST when you're done — the result, not the instruction>
SOURCES:    <what to read; do not invent beyond these>
DONE WHEN:  <how we both know it's good enough to stop>
ASK FIRST:  <anything irreversible or outside scope — publish, send, delete, spend, touch files outside X>

Do ONE scoped task, then STOP. Don't chain into the next thing.

End with a RECEIPT:
- Did:      what you actually did
- Didn't:   what you skipped or left for me to decide
- Proof:    the file / output / line that shows it's real
- Next:     the one thing I should look at or decide
```

### One refinement worth carrying (learned this session)
When writing `DONE WHEN`, avoid "all", "every", "fully", "complete" — **completeness can't
be verified** without redoing the work. Substitute things you *can* check from a sample:
**traceability** (outputs cite their source), **method** (a stated systematic pass), and
**declared gaps** (what was deliberately left out). See Example C in `work-mode-template.md`.

---

## Summary

- **Now:** skip full Open Engine; use the 3-habit work-mode template (`work-mode-template.md`).
- **Later:** adopt the full system the moment you're coordinating 2+ agents or a teammate by
  hand, run trusted scheduled agents, or already use an issue tracker.
- **Always:** the durable lesson — when intelligence is cheap, the moat moves to the *work
  layer* (context + handoffs). The 3 habits are that lesson, sized for one person.

---

## Appendix — Provenance: the work-mode prompt that produced this file

This document was itself created with the 3-habit template it describes. The exact prompt:

```
Work mode, not prompt mode. Before you act:

JOB: a markdown file titled open-engine-adoption-assessment.md that includes (1) the
assessment from this session of installing/deploying Open-Engine for a solo developer who
does not use Linear, (2) the subsequent distillation of Open-Engine into a 3-habit work-mode
prompt template. Point to the work-mode-template.md file.

SOURCES: the transcript for current Claude Code session, the file work-mode-template.md

DONE WHEN: (1) the choice of skipping a full adoption of Open-Engine is explained / justified
in the output file, (2) when the developer should consider adopting full Open-Engine, (3) the
work-mode prompt is included in the output.
-  nothing invented
- written to open-engine-adoption-assessment.md

ASK FIRST: anything irreversible or any doubt about when the task is considered DONE.

Do ONE scoped task, then STOP.

End with a RECEIPT (Did / Didn't / Proof / Next).
```
