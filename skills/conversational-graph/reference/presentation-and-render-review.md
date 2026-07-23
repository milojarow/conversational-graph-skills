# Presentation — the render review before handoff

A graph engine is validated with asserts: transitions, deterministic gates, tool results, guardrails. **Presentation is not.** The defects that make a technically-correct bot look unprofessional are all *look-and-feel*, they are **predictable**, and no unit test sees a single one of them — they are visible at a glance in the first screenshot of a real chat.

Observed on an otherwise-finished bot (engine correct, gates deterministic, tools working): four rounds of operator corrections before it looked shippable, none of them a logic bug:

1. **It introduced itself twice** — the channel already greeted, and the node prompt also said "greet".
2. **It crammed five priced services into one running paragraph** — the "no markdown" rule (so the widget doesn't paint literal asterisks) killed lists, and nothing replaced them.
3. **It alternated formal and informal register** between consecutive replies — nobody declared one, so the model picked by topic.
4. **It offered a contact phone as loose digits** — dead text in a chat window, exactly at peak intent.

The pattern: **the engine is tested with asserts; the presentation is only tested by looking at it.** Until that step exists, presentation QA is done by the operator, one screenshot at a time — expensive, slow, and demoralizing.

Everything in this file is a **cross-cutting look-and-feel concern → it lives in BASE**, never in a node prompt (see [the-model.md](the-model.md), "What goes in BASE vs the node prompt").

## The render review — 5 canonical queries, read as an end user

Before handing the bot over, send these and **read the raw output** (not an assert):

| Query | What you are checking |
|---|---|
| A greeting, or any first message | Does it introduce itself when the channel already greeted? Does it open with filler? |
| One that enumerates a lot ("prices", "what do you offer") | One item per paragraph — or a wall, or over-fragmented? |
| One single-fact query ("how much is X") | Does the enumeration rule over-fragment a one-line answer? |
| One asking for contact ("how do I reach you") | Does the channel come out actionable (URL with a scheme) or as digits? |
| Two consecutive queries on different topics | Does the register hold? Does it repeat the introduction? |

One query is not enough: an enumeration rule tuned on a list breaks the single-fact answer, and a greeting rule only shows its bug on the *first* turn. Change → look at 2–3 different renders → adjust. **A formatting rule judged without seeing the output is a bet.**

## BASE checklist that prevents all four

- **Opening** — decide who owns the greeting, the channel or the bot; if it's the channel, a NEGATIVE rule is required. Removing the positive instruction is not enough.
- **Enumeration** — pin the unit, cap the number of items, and add an explicit exception to any global word limit.
- **Register** — formal or informal, declared and absolute ("not even when being formal"). Verifiable with a regex over the forbidden forms.
- **Actionable contacts** — phones/emails always as a scheme URL, and **guaranteed in the engine's post-processing**, not in the prompt.
- **Format vs channel** — if you forbid markdown because the channel won't render it, you must supply the replacement (paragraphs). Forbidding without substituting produces caked prose.

Wherever a presentation rule can be stated as an invariant, **assert it with a regex over several real replies** instead of eyeballing it — that turns a subjective impression into a check you can re-run.

## Opening: the greeting belongs to the CHANNEL, not to a node

**Wall:** the user reads two introductions in a row, the second one redundant.

**Cause:** the channel (a web widget, typically) paints a static local greeting on open — "Hi! I'm <bot> 👋 how can I help?" — without calling the API, a good pattern that saves a network round-trip. Meanwhile the hub node's prompt says *"Greet (only the first time), then answer questions about…"*. The backend has **no way to know the channel already greeted**: it sees an empty history → satisfies "first time" → introduces itself again.

**Rule — a node prompt describes only its JOB, never the act of greeting or introducing itself.** Identity is declared as **state in BASE** ("You are the assistant for X"), never as an **action** to perform ("greet", "introduce yourself"). A mature graph has zero greeting instructions in its nodes; the only greetings live in the close node, and they are goodbyes.

**If the channel greets, add an explicit opening rule to BASE** — the negative form, not just the absence of the positive one:

> The channel has already shown the opening greeting. NEVER introduce yourself or state your name/role as an opening; go straight into the answer. If the user greets you, return the greeting in a few words and continue.

Deleting the positive instruction is not enough: with an empty history a model tends to self-introduce anyway. The negative rule in BASE closes it deterministically.

**Diagnosing "backend or frontend?"** — read the session's persisted history. If the FIRST stored message is the user's and the assistant's reply already carries the introduction, the duplicate came from the model (backend), not from a duplicated client-side bubble.

**Verify with three scenarios, not one:** (1) first message is a direct question; (2) first message is a greeting from the user — must return a brief greeting WITHOUT introducing itself; (3) a normal second turn. A "self-introduction" regex over the three replies turns the check into an assertion instead of an impression.

## Enumerations: pin the unit, and exempt them from the word cap

**Wall:** the bot lists five services with prices **crammed into one running paragraph** — unreadable in a narrow chat window. Root cause is usually self-inflicted: BASE forbids markdown (so the widget doesn't paint literal asterisks), which removes lists and pushes everything into prose, and nothing was put in their place.

**The wording that works first try** — it pins the *unit* and states the anti-pattern explicitly (see the over-fragmentation gotcha in [the-model.md](the-model.md)):

> Each item goes in its OWN short paragraph, with its name and its value joined in ONE single sentence — "Service: $800 per session.". The blank line goes BETWEEN items, never inside one; do not split an item's name and its description into two paragraphs. One lead-in sentence before the list, one closing sentence after.

Naming the anti-pattern ("do not split the name from its description") is what prevents the over-fragmentation that the bare "one paragraph per thing" rule causes.

**A global word cap fights every enumeration.** If BASE says "max ~120 words", an honest list of five items does not fit, and the model either truncates or crams. Add both:
- an explicit exception — *"in enumerations you may exceed the cap as long as each item stays in one short sentence"*;
- an item cap — *"max 5; if there are more, give the main ones and offer the rest"* — so you avoid the wall AND the endless list.

## Register (formal vs informal) drifts unless it is declared

Collateral finding from the same review: across consecutive replies the same bot alternated formal and informal address. Nobody asked for it — the model picked its formality **by topic**, so a serious question got the formal register and a casual one got the informal. To a customer that reads like two different people.

Register is look-and-feel → **it goes in BASE**, next to the tone, declared and absolute:

> Always address the customer informally; never use the formal forms, not even when being formal in content.

(or the inverse, depending on the brand). **Verify it with a regex** over the forbidden forms across several replies, not by eye — a drift that shows up in one reply out of five is exactly what eyeballing misses.
