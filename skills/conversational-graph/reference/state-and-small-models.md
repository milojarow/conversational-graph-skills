# State-driven reliability — making a small model behave in a code graph

A small ("mini") LLM driving the nodes is cheap but unreliable in specific, repeatable ways: it invents tool arguments, claims actions that didn't happen, and repeats itself. The cure is almost never a better prompt — it is to move the critical decisions OUT of the model and into deterministic engine code over session state. Deriving/validating critical params in the backend is what lets a small model match a large one on tool-calling reliability.

## Critical params come from STATE, not the model's tool-call args

A small model invents plausible values for `required` tool params the user hasn't given (e.g. a placeholder email like `user@example.com`). Never trust `args.<field>` from the model for identity or side-effecting data.

**Pattern:** capture the datum from the user's INPUT (regex in the engine) → store it in graph state → the tool reads `ctx.<datum>` (state), NEVER `args.<datum>` → the `toolGate` does NOT expose the tool until that datum is in state. The model can't even call the tool with an invented value.

## Confirm a side-effect only on the tool's REAL result

A prompt that says "I've sent X / done X" after a tool call LIES if the tool failed (rate-limit, network, missing data). The bot asserts an effect that never happened.

**Pattern:** the tool returns a REAL `{ok, reason}` (not a hardcoded `{ok:true}`); the state flag flips ONLY if `ok`; the prompt confirms ONLY if `ok`, else it apologizes / retries. Pin the datum from the `result` (what was actually used), not from `args`.

## Don't strip a node's history to stop "contamination"

Removing a node's history (so it won't scrape a stale datum from the text) also removes its context: it forgets WHAT the user asked (look-up vs cancel vs reschedule) and re-asks data already given. The real cause of "contamination" is that the TOOLS were reading the datum from the text — fix it there. Keep the history for context/intent; derive SENSITIVE data from state.

## A changed datum invalidates the old pin (deterministic)

If the user corrects a datum (typed one email, then gives another), the previously pinned value contaminates: a state note like "already sent to <old>" makes the LLM keep using the old one (symptom: "I gave B but the bot insists on A"). **Engine fix:** when a NEW datum differs from the pinned one, invalidate the old pin and its derived flags (e.g. verification). Deterministic, not LLM-dependent.

## Deduplicate the model's response in code

Small models sometimes emit the SAME answer twice (verbatim or reworded) within a single `content`. Confirmed by instrumenting the tool-loop: it appears in a round with NO tool calls — the model repeating itself, not the loop. A prompt ("don't repeat") isn't enough.

**Fix:** post-process the final text with a conservative dedup — collapse near-identical consecutive segments; for short replies use a token-similarity threshold, for long blocks (catalog-style lists) use EXACT dedup only so distinct items aren't merged; if no duplicate, return the text untouched. Pure function → testable.

## Force the final tool-loop round to text (`tool_choice: "none"`)

When the tool-call loop hits its round cap (e.g. 3 rounds) without having produced any user-facing text, force the LAST round with `tool_choice: "none"`: the model still SEES the tools (full context intact — it knows what already happened) but cannot call them, so it is obliged to emit text for the user.

This kills two real bugs:
1. **Generic "I'm stuck" fallback** — every round was spent on tool calls and none produced text, so the engine fell through to its hardcoded fallback string.
2. **Internal-notes leak** — with nothing addressed to the user, the model emits scratch notes ("Now exit.", "Need name next.") as if they were the reply.

`tool_choice: "none"` ≠ removing the tools from the request: stripping them breaks context coherence (the model then sees tool results for tools that "no longer exist"); `"none"` leaves them visible but uncallable.

## Debug by forcing state and measuring

To reproduce a routing/state bug without the whole flow: seed graph state (`currentNode` + flags) directly in the store (e.g. Redis), send one turn, observe the resulting node + state. To catch duplication/hallucination, log one line per tool-loop round (which tool, what content) — the cause becomes visible instead of guessed.

> Completion edges and security gates also belong in code, not LLM judgment — see [transitions-and-gotchas.md](transitions-and-gotchas.md) (`expression` completion edges and `toolGate` security gates). An eager classifier fires "we're done" mid-flow and the close node hallucinates an action that never passed its gates.
