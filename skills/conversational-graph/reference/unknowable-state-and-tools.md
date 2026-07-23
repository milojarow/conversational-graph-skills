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
