# State the bot cannot know — forbid it, then build the read-only tool

Every bot has questions its user will obviously ask and it has **no way to answer**: is that slot free, is the item in stock, where is my order, has it shipped. This file covers the two stages of handling them: the prohibition that stops the invention, and the read-only tool that actually closes the gap.

## Without a tool to check it, the model INVENTS the fact — in a confident tone

**Wall:** asked *"can I come Thursday at 4pm?"* the bot answered *"Sure, Thursday at 4:00 pm we do have space."* It had **no scheduling tool at all** — it only files requests that the business confirms afterwards. Nobody asked it to lie: the question had yes/no shape, there was no way to look it up, and the model filled the hole with the cooperative answer. In an appointments business that sends a customer to the door expecting a slot that may not exist.

This is a cousin of *"confirm a side-effect only on the tool's REAL result"* ([state-and-small-models.md](state-and-small-models.md)) but **harder to catch**: there, a tool exists and can fail; here there is **no tool at all**, so there is no result to contradict the model and no error to log. It passes silently until somebody reads a transcript.

**Rule: for every state question the bot cannot look up, put an explicit prohibition in BASE *and* give it the true alternative it CAN say.** Without the alternative the model chooses between sounding useless and inventing — and it invents.

> AVAILABILITY: you have no access to the calendar. NEVER claim there is space/room at a specific day or hour, nor that "that works" at that time. What you CAN say: whether the hour falls inside business hours, and that the business confirms availability when it contacts them. What you record is a REQUEST, never a confirmed reservation.

With that rule in place, three escalating questions (including an imperative *"just hold it for me"*) all distinguish *falls inside opening hours* from *is free*, and none promises the slot.

**How to hunt the rest of these gaps before a customer does:** list the tools the bot HAS, then ask which obvious user questions fall outside them — stock, availability, order status, delivery time, "has it arrived yet". Each gap is one BASE rule plus its alternative. Test each with yes/no questions **and** with an imperative ("hold it", "reserve it") — the imperative is what pushes the model hardest into asserting.

## The read-only tool that closes the gap — two pieces that are always missing

When the state the bot couldn't check DOES exist in some system (a calendar, an inventory, a tracking service), the BASE prohibition is the patch and a **read-only tool** is the fix. Two requirements show up while building it that are not obvious:

**1. The bot does not know what day it is.** An LLM with no date in context cannot resolve "tomorrow" or "Thursday", let alone compute the `YYYY-MM-DD` the tool needs — and a small model facing that gap **invents a plausible date**. Inject the CURRENT date/time in the business's timezone into every turn's prompt ("Today is Thursday, 12 March, 4:57 pm") and say explicitly what it is for. If the runtime is containerized, the clean way is giving the container `TZ=<zone>` (with tzdata present) so local time comes out right including DST, instead of doing offset arithmetic by hand.

**2. The result is FRAMED, not recited.** The tool only sees what its system records: the calendar holds what the staff entered, not the appointments taken over the phone that nobody wrote down. The BASE rule must force the frame ("according to our calendar…", "the business confirms when it contacts you") and keep the prohibition on promising ("you cannot hold a slot"). Without it, adding the tool converts an honest "I don't know" into a "yes, it's free" with false authority — worse than before. Scope that hedge carefully: see the next section.

Implementation details that avoid bugs:

- **Business hours as DATA, not prose** (`{0:null, 1:null, 2:[10,17], …}` indexed by weekday): the model stops reasoning about "short Saturdays"; the tool answers open/closed and with what range.
- **Do date arithmetic in a faked UTC frame** (`Date.UTC(y, m-1, d)` + `getUTCDay()`) even when the process runs in local time — this stops the weekday from shifting by timezone.
- **Format the hours in the tool's OUTPUT** (12-hour with am/pm if that's the business convention) instead of letting the model convert — one less conversion that can fail.
- **Degrade honestly:** if the state system doesn't answer, return `checked:false` with an explicit instruction NOT to invent hours and to fall back to the previous behavior (state the opening hours, say the business confirms). **A timeout must never produce a promise.**

Minimum test set: a closed day, an open day with real occupied slots seeded, an hour outside business hours, and an imperative request ("just hold it"). All four must keep *falls inside opening hours* / *is free* / *is reserved* apart — three different things that a bot without the tool collapses into one.

## Every hedge has a SCOPE — bound it to the datum the tool returns

**Wall:** the framing rule from the previous section was written as *"ALWAYS frame it as 'according to our calendar'"*. What came out in production:

> *"According to our calendar, tomorrow Friday **we're open 10:00 am to 5:00 pm** and I have free…"*

Opening hours are not dictated by the calendar. The reply fused two data of different natures:

| Datum | Nature | How it should sound |
|---|---|---|
| Business hours | a **FACT** of the business, in the knowledge base | Confidently, from memory: *"on Fridays we're open 10 to 5"* |
| Free/booked slots | a **LOOKUP** against a system | Here the hedge belongs: *"I have 10 and 12 free"*, *"that hour is taken"* |

Attributing the fact to the lookup damages perceived competence: a real employee does not say "according to the calendar, we open at 10" — they just know. The cost is not accuracy, it is **credibility**, which is exactly what a customer-facing bot sells.

**Rule: when you introduce a tool, write the hedge bound to the datum the tool returns, and say explicitly what is NOT hedged.** Wording that worked:

> HOURS vs CALENDAR — these are DIFFERENT things, don't mix them:
> - Business HOURS are a FACT you know by heart; say it with confidence, like someone who works there. NEVER attribute it to the calendar ("according to our calendar we're open from…" is WRONG and sounds like a new hire).
> - The CALENDAR only reveals which hours are free or taken. That — and only that — is the datum that comes from the tool.
> - Don't open your replies with "according to our calendar". Talk naturally: "tomorrow Friday we're open 10:00 am to 5:00 pm; I have 10:00, 12:00 and 3:00 free".

**Generalization: every new tool brings a hedge, and every hedge has a scope.** Before writing the rule, split the reply's data into **FACTS** (from the knowledge base, asserted) and **LOOKUPS** (from a system, hedged) — and explicitly forbid the hedge from crossing that line. Without the boundary the hedge spills over everything the business *does* know for certain.

**Detectable with a regex** during the render review: search for the hedge followed by a fact (`according to .{0,20}calendar.{0,40}(open|hours|we're)`), and assert that no reply OPENS with the hedge.

## When a capability lands, sweep the vocabulary around it

Adding the tool is one commit. The **vocabulary that assumed its absence** lives in the prompt, in state names, in emails and in the UI — and all of it silently starts lying.

A bot that evolved in a single day shows the shape: (1) it filed requests that staff approved; (2) it got an availability tool; (3) it was allowed to book directly. The status was called `confirmed` from stage (1), where it meant *"staff approved it"*. After (3) the name survived but now described something else: *"it has a date and time"*. The mismatch surfaced the first time somebody from the business read the dashboard and asked the obvious question: in a normal business, *confirming* means asking the CUSTOMER whether they will show up. An internal step had taken over the name of the only event the business actually cares about.

Two separate lessons:

**1. Sweep the language after every capability change.** When the booking tool landed, prompt sentences that were now false stayed alive ("the office will confirm your time", "I can't hold slots") along with the mis-named status. Checklist after enabling a new capability:
- Which prompt sentences were true only *because* the capability was missing?
- Does any status/field name a step that no longer exists?
- Do the emails and the UI still describe the old flow?

**2. A status name IS the definition of the business process.** While "confirmed" meant two things, the business could not measure the one thing that matters: how many people say they're coming, and how many don't show. The right split — `scheduled` (has a time) vs `confirmed` (the customer replied) vs `no_show` — is not cosmetic: without the third state, the metric that justifies building reminders does not exist. **Model states by the real EVENT they represent, not by who touched the record last.**

Operational corollary: renaming statuses is a change coordinated with the frontend (filters tend to survive; badges and labels don't). Migrate existing records in the same move and document the old→new mapping, or the dashboard renders a status it doesn't know how to paint.
