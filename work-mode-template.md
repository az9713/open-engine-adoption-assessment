# Work-Mode Prompt Template

Paste this at the top of any agent session. It carries the three habits worth keeping
from Open Engine — Work mode, one scoped task, leave a receipt — with zero infrastructure.

---

## Quick version (paste-and-go)

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

---

## The three habits, and why each line exists

### 1. Work mode, not prompt mode
A prompt asks for *output* ("write me an email"). A job explains the *result*
("here's the transcript, the decision, the constraint — draft it, flag what needs my
judgment"). You front-load owner, sources, boundaries, and a stop condition so the agent
produces something reviewable instead of a plausible guess.

- **JOB** = the outcome that should exist, defined not instructed.
- **SOURCES** = grounding + an anti-hallucination fence ("don't invent beyond these").
- **DONE WHEN** = the stop condition, so it doesn't over- or under-shoot.
- **ASK FIRST** = default-deny on anything with blast radius you can't undo.

### 2. One scoped task, then stop
One unit of work = one unit you can check. If an agent can't do *one* thing cleanly,
it has no business chewing through a pile of them. Stopping keeps failures small and
diagnosable — you review a single clean step, not a tangled run.

### 3. Leave a receipt — even to yourself
"Did / Didn't / Proof / Next" makes the work inspectable. Future-you is the "someone else"
who has to trust it. The point: you abandon agents not because they make mistakes, but
because the mistakes become *mysterious*. A receipt kills the mystery.

---

## Worked examples

### Before / after — the same request in both modes

**Prompt mode (output, not work):**
```
Summarize this article for me.
```
You get a plausible summary with no idea what it skipped, no source discipline, and
nothing to check it against.

**Work mode (a job it can finish and you can trust):**
```
Work mode, not prompt mode.

JOB:        A 5-bullet summary of article.md that I could send to a colleague cold.
SOURCES:    article.md only. If a claim isn't in there, don't add it.
DONE WHEN:  5 bullets, each tied to a section; no outside facts.
ASK FIRST:  Don't send or post it anywhere — draft only.

Do ONE scoped task, then STOP.

End with a RECEIPT (Did / Didn't / Proof / Next).
```

### Example A — a coding task (solo)

```
Work mode, not prompt mode.

JOB:        login form rejects empty email with a visible error instead of crashing.
SOURCES:    src/auth/LoginForm.tsx and its existing validation helpers. Match their style.
DONE WHEN:  empty email shows the error; existing tests still pass; no new deps.
ASK FIRST:  before touching anything outside src/auth/ or changing the API contract.

Do ONE scoped task, then STOP.

End with a RECEIPT (Did / Didn't / Proof / Next).
```

A good receipt back from the agent might read:
```
RECEIPT
- Did:    added empty-email guard in LoginForm.tsx:42, reused validateEmail().
- Didn't: didn't touch password validation — out of scope.
- Proof:  npm test → 14 passed; manual: empty submit now shows "Email required".
- Next:   decide if whitespace-only email should count as empty too.
```

### Example B — a research / writing task (solo)

```
Work mode, not prompt mode.

JOB:        a one-page brief answering "should I switch my notes app to Obsidian?"
SOURCES:    the 3 links I pasted + my current-setup notes. Don't pull in other tools.
DONE WHEN:  a recommendation + 3 reasons + the strongest counter-argument.
ASK FIRST:  don't sign me up for anything or follow paywalled links.

Do ONE scoped task, then STOP.

End with a RECEIPT (Did / Didn't / Proof / Next).
```

### Anti-pattern — looks like work mode, isn't

This one fills in the fields but defeats the purpose. The structure is cosmetic:

```
Work mode, not prompt mode.

JOB:        make the app better.
SOURCES:    the codebase.
DONE WHEN:  it's improved.
ASK FIRST:  use your judgment.

Do a few things, then wrap up.

End with a RECEIPT.
```

Why each line fails:
- **JOB "make the app better"** — not a result that can *exist*. Nothing is checkable, so
  the agent invents its own goal. You've delegated the decision you were supposed to own.
- **SOURCES "the codebase"** — no fence at all. "Read everything" is the same as "read
  nothing in particular"; it can wander and hallucinate justification.
- **DONE WHEN "it's improved"** — restates the job instead of giving a stop condition. The
  agent can never know it's finished, so it over-reaches or stops arbitrarily.
- **ASK FIRST "use your judgment"** — the exact opposite of a boundary. It *authorizes*
  the irreversible actions the field exists to fence off (deleting, deploying, sending).
- **"Do a few things"** — abandons one-task-one-review. A tangled multi-step run is what
  you can't diagnose when it goes wrong.
- **"End with a RECEIPT"** with no shape — you'll get a vibe, not Did / Didn't / Proof / Next.

**The tell:** if you could paste the same prompt into *any* project unchanged, it's prompt
mode wearing a costume. Real work mode is specific enough that it only fits *this* task.

### Example C — critique & tighten (a real pass)

A usable first draft that still leaves the agent guessing. This shows what "good but not
tight" looks like, and the small edits that close the gaps — the realistic middle case
between the good examples and the anti-pattern.

**First draft (passes, but two soft spots):**
```
JOB:        a detailed summary in markdown format
SOURCES:    .ignore/transcript_skill_to_improve_skill_ben_holmes.txt
DONE WHEN:  all key takeaways are extracted from the transcript
ASK FIRST:  anything irreversible or any doubt about the meaning of DONE.
```
What's wrong:
- **JOB vs DONE WHEN disagree** — "detailed summary" (flowing prose) and "key takeaways"
  (a distilled list) are different artifacts. The agent has to guess which you want.
- **JOB is generic** — "a detailed summary in markdown" could paste into any project. The
  specificity all lives in SOURCES, not JOB. (That's the costume tell, mildly.)
- **DONE WHEN isn't checkable** — "all key takeaways" can't be confirmed; neither side
  knows when *all* is reached. And no output location is given.
- **ASK FIRST is the strong line** — with no blast radius, pointing it at "doubt about DONE"
  is the right adaptation. Keep that instinct.

**Tightened (first attempt — still has a hidden bug):**
```
DONE WHEN:  every distinct point in the transcript is captured as a bullet under a theme;
            nothing invented; written to .ignore/takeaways.md.
```
This *looks* tighter but "every distinct point" has the same disease as "all key takeaways":
**completeness against a source can't be verified** without re-reading the whole transcript
yourself — the exact work you were delegating. If checking the stop condition costs as much
as doing the task, it isn't a stop condition.

**Tightened (final — completeness replaced with checkable proxies):**
```
JOB:        a markdown file of the key takeaways from the transcript — grouped by theme,
            each takeaway a self-contained bullet a reader could understand without the source.
SOURCES:    .ignore/transcript_skill_to_improve_skill_ben_holmes.txt — only this. Don't add outside facts.
DONE WHEN:  - you've gone through the transcript top to bottom once, section by section;
            - each bullet cites a line/timestamp I can spot-check against the source;
            - any stretch you judged to have no takeaway is listed explicitly, so a gap is
              a decision I can see, not a silent omission;
            - nothing invented; written to .ignore/takeaways.md.
ASK FIRST:  anything irreversible, or if "detailed summary" vs "key takeaways" changes what you'd produce.
```
What the final edits bought: JOB names *one* artifact with a checkable shape; SOURCES adds
the invention fence; DONE WHEN drops the unverifiable "all/every" and substitutes three
things you *can* confirm by sampling — **traceability** (bullets cite locations),
**method** (a stated top-to-bottom pass), and **declared gaps** (silence becomes a visible
choice); ASK FIRST keeps the smart instinct but aims it at the specific ambiguity.

**Lesson:** the draft wasn't *wrong* — it would run. But notice it took *two* passes: the
obvious tighten still smuggled in an unverifiable "every." **You can't verify completeness;
you can verify traceability, method, and declared gaps.** Any time DONE WHEN reaches for
"all", "every", "fully", or "complete", swap it for something you could check from a sample.

---

## When to graduate to the real thing
The day you're routing work between **two or more agents** or **a teammate** by hand —
copy-pasting state between Claude, Codex, a browser agent, etc. — that's the signal to
revisit the full Open Engine (shared queue, status ledger, locking, receipts). Until then,
these three habits are the whole payoff with none of the setup.
