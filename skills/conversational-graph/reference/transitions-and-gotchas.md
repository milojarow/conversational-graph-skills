# Transition gotchas — the walls that cost real debugging

The node/tool side is easy. The **transition layer** is where a graph bot breaks. Four walls, each with the fix.

## 1. Route on the user's INTENT — not the node's own just-generated reply

**Wall:** a clear request ("I want to book") does NOT transition — the bot stays in the hub and flounders.

**Cause:** the classifier was fed the conversation *including the node's fresh acknowledging reply* ("Sure, I can help you book — what's your name?"). The model reads that and concludes *the current stage is already handling this* → returns NONE → no transition.

**Fix:** build the classifier input from the **user's message + prior history ONLY** — exclude the reply the node just produced this turn. A transition is about *what the user wants*, not *how the node answered*.

**How to find it:** add a temporary log of the classifier's exact input + output. You'll see it received the assistant's "I'll help you book…" line and answered NONE. (Tested in isolation with just the user message, the same classifier answers correctly — proving the input, not the prompt, was the bug.)

## 2. Completion transitions are `expression`, NEVER `llm`

**Wall — the dangerous one:** mid-flow, the bot **jumps to the close/done stage and claims an action finished that never happened** — e.g. *"your appointment is cancelled"* when nothing was cancelled.

**Cause:** the "→ done" edge was an `llm` condition ("the user finished"). An eager classifier fired it early (e.g. right after the node sent a verification code). The conversation lands in the close node, which has **no tools**, so the model **hallucinates the completion** to sound complete and helpful.

**Fix:** make completion edges **`expression`** over a flag that a *real action* sets — `ctx.cancelled === true`, `ctx.booking_created === true`. The flag only flips when the actual tool succeeded. No flag → no transition → no false "done."

**The rule:** anything that means *"the task is complete"* must be a **deterministic fact**, not an LLM judgment. Leave `llm` for *intent* ("which stage does the user want") only.

## 3. The classifier needs BALANCE — eager and conservative both fail

You will likely over-correct in both directions before landing it:

- **Too eager** (the naive "which condition matches?") → false transitions like #2, and "abandoned / goodbye" edges fire on normal in-flow messages.
- **Too conservative** ("default to NONE; only on the clearest signal; never if the assistant just asked something") → it stops routing *at all* — even "I want to book" returns NONE.

**Fix — the middle:** route on the user's **current intent** plainly; add exactly ONE guard — an "left this stage / said goodbye" edge applies **only on a real topic change**, NOT because the user is still giving data inside the current flow (typing a code, giving an email, answering a question = the flow continues). Do **not** pile on "when in the slightest doubt → NONE" — that is what kills routing.

## 4. Security gates live in CODE (`expression` tool-gates), not in the prompt

**Wall:** a prompt says "verify the user before showing their data," but the model can be talked past it — the gate leaks.

**Fix:** a **`toolGate`**. The node's sensitive tools (look-up / cancel / read-account) are **filtered out of the offered toolset** until `ctx.verified === true` (or the user is already authenticated). The model literally cannot call them before verification, because they aren't there. The send-code / verify-code tools stay available so the flow can proceed; the instant `verify` flips `ctx.verified`, the gated tools reappear (re-filter each tool-loop round, so it works *within the same turn*).

Deterministic, not jailbreakable — strictly better than a prompt instruction. This is a concrete place a **code graph beats a hosted platform**, which typically pays an LLM judge on every transition and can't gate tool *availability* on a hard predicate.

## 5. A close/terminal node needs an escape edge back to the hub

**Wall:** after the bot completes an action (book / cancel), the user brings a NEW request — but the conversation is parked in the "close" node, whose only edge is → end. The node can't route, so the LLM improvises the new request in-place, **skipping the flow's gates** (e.g. it offers to cancel an account action WITHOUT re-running identity verification).

**Cause:** a terminal node modeled with a single escape (→ end) is a dead end. Anything new after the action has nowhere to go but the model's improvisation.

**Fix:** every close/terminal node also carries a **"new request → hub"** edge, evaluated BEFORE its → end edge. Re-entering the hub re-routes the request through the proper node and re-applies its gates (verification, etc.).

**Symptom to watch:** the bot handles something new "by hand" in the close stage, bypassing a gate that would apply from the hub.

## Bonus: lazy + short-circuit evaluation (speed + correctness)

Evaluate a node's edges in priority order. `expression` edges are free — check them inline and return on the first match. Only when you reach an `llm` edge do you make the classifier call (once, for all the node's llm edges). After an action completes, the `expression` completion edge wins first and you never pay the classifier that turn — and, importantly, the deterministic edge can't be overridden by an eager classifier.

## Testing these

The whole transition/gate layer is **deterministic and unit-testable without an LLM**: assert the `expression` predicates (flag true → returns its target; false → doesn't), the `toolGate` (sensitive tool hidden when `!verified`, shown when `verified` or authenticated), and the side-effect flag-setters (tool result → flag). Then a handful of live checks for the `llm` routing. Don't run real side-effecting actions (bookings, payments) against a third party just to test a transition — the deterministic test covers the new logic; the action integration is whatever you already trust.
