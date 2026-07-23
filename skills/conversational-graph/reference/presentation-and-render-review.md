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
