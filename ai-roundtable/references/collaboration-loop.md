# The collaboration loop — protocol and worked examples

This file expands on the loop described in SKILL.md. The goal is a *grounded,
converging* collaboration, not a polite exchange of monologues. The single idea
that makes it work: **you can act, ChatGPT can only talk — so use your ability to
run, read, and check to keep the discussion honest.**

## Why grounding is the whole game

Two language models left to debate each other will happily agree on a
plausible-sounding wrong answer, or argue past each other indefinitely, because
neither is checking against reality. You break that by turning talk into tests.
Every time a turn yields a claim you can verify, verify it — then the next round
starts from a fact, not an opinion. This is what makes the loop *close*: reality
is in the loop, not just two chatbots.

Good candidates for grounding:

- "The bug is in X" → actually reproduce it / add a log / read that code path.
- "This function returns Y" → run it, or read the source.
- "Library A is faster than B" → a quick benchmark, or find an authoritative source.
- "The API requires field Z" → check the docs or make a probe call.
- "This math checks out" → compute it.

If a claim genuinely can't be grounded right now (needs the user's private data,
a credential, a side-effectful action), note that in the ledger and either ask
the user or flag it as an assumption both sides are proceeding on.

## The round, in detail

1. **Compose** a self-contained turn. Include: the current question, your
   position with reasoning, and any *new evidence* since last turn. Ask something
   specific. Avoid re-pasting the whole history — the ChatGPT thread remembers
   its own side.
2. **Send and wait** for generation to fully complete (see chatgpt-web.md).
3. **Extract** ChatGPT's latest reply only.
4. **Interrogate it**: What's the strongest point? What's checkable? Where does it
   disagree with you, and is that disagreement real or just framing?
5. **Ground** the checkable parts by acting in the real environment. Record the
   result.
6. **Relay** to the user: ChatGPT's take (verbatim where it matters), what you
   tested and found, your judgment.
7. **Update the ledger** and run the convergence check.

## Worked example — a debugging roundtable

User: "My Flask endpoint intermittently returns 500s under load. Let's get
ChatGPT in on this."

**Round 1.** You open ChatGPT, lay out the symptom, the framework, and your
initial hypothesis ("I suspect a DB connection pool exhaustion"), and ask where
your reasoning is weak. ChatGPT replies: agrees pool exhaustion is plausible but
points out the intermittency-under-load pattern also fits a race condition on a
shared module-level object, and asks whether the 500s correlate with a specific
route.

→ *Ground it.* Rather than debating which is more likely, you actually read the
handler and check the logs. You find the 500s are all on one route that mutates a
global `dict`. Evidence line updated: "500s isolated to /aggregate, which writes
to a shared global without a lock." That single check just collapsed the
speculation.

**Round 2.** You feed the finding back: "You were onto something — it's not the
pool, it's a shared global mutated without locking on /aggregate. Here's the
code." ChatGPT proposes two fixes: a lock, or making the state request-local, and
argues request-local is better because the lock would serialize every request and
kill throughput. You agree the perf point is real. Ledger: Agreed = "root cause
is unsynchronized shared state"; Open = "lock vs. request-local".

→ *Ground it.* You make the state request-local in a scratch copy and run the
load test. 500s disappear, throughput holds. Evidence: "request-local fix
verified under load, no 500s, throughput unchanged."

**Round 3 / converge.** Both approaches would work; the evidence favors
request-local on throughput. There's no real disagreement left. You summarize for
the user: root cause found and confirmed, fix verified under load, and ask
whether to apply it to the real file. That's the loop closing — a confirmed
diagnosis and a tested fix, not two models agreeing in the abstract.

Notice the pattern: every round, a claim got turned into a check, and the check
moved the ledger forward. That's the difference between this and "glancing at
ChatGPT's answer."

## Worked example — a design review (no code to run)

Not every roundtable has something executable to test. Grounding then means
pulling in authoritative evidence rather than running code.

User: "Help me and ChatGPT settle on a data model for multi-tenant billing."

Here you ground by checking references (how established systems handle tenant
isolation, what the database docs say about the constraint you're proposing),
and by working through concrete edge cases on paper ("what happens to this schema
when a tenant downgrades mid-cycle?"). The ledger still tracks agreed vs.
contested, and you still converge on a recommendation with the trade-offs named,
handing the final judgment to the user where it's a matter of priorities.

## Keeping the user oriented

The user is watching, not laboring. Make each relay easy to follow:

- Lead with the movement: "New: ChatGPT caught that it's not the pool — and I
  confirmed it's a shared-global race on /aggregate."
- Give ChatGPT's actual words when the phrasing matters; summarize when it
  doesn't.
- Show the ledger at convergence so they can approve from a clear picture.
- Surface disagreements honestly, including when ChatGPT changed your mind.

## Anti-patterns to avoid

- **Relaying unverified claims as conclusions.** If you didn't check it, say it's
  unverified.
- **Round-tripping trivia.** If you can answer or check something in one step
  yourself, don't spend a ChatGPT round on it.
- **Losing the thread.** Without the ledger, round 4 forgets what round 2
  settled. Keep it current.
- **Winning instead of solving.** The aim isn't for you or ChatGPT to be right;
  it's to get the user a grounded answer. Concede fast when the evidence says so.
- **Infinite politeness loops.** Two models can agree forever. Push to a decision
  or to the user.
