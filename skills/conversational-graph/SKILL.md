---
name: conversational-graph
description: Use when building a conversational agent as a code-driven graph / state machine — routing a chat through focused stages, each a node with its own prompt + tools, connected by forward/back transitions — instead of one mega-prompt or a hosted workflow platform; or when a stage-routing bot mis-routes, hallucinates a completed action, or a verification gate leaks.
---

# Conversational Graph Engine (in code)

Build a multi-stage conversational agent as a **graph you run in your own code**: focused nodes (each = a tight prompt + only its tools) connected by transitions (forward and back). The same mental model as a hosted agent-workflow platform — but in your own stack, so it's fast, fully controllable, and can use deterministic transitions a platform won't give you.

> **🕸️ ACTIVE-SKILL MARKER:** While `conversational-graph` is active — any turn designing or debugging a code-driven conversational graph / state-machine engine — begin every reply with 🕸️ so the operator sees the skill is engaged. Omit it on turns that don't touch the domain.

## Overview

A monolithic chatbot stuffs every instruction and every tool into one prompt; the model re-derives context each turn and drifts. A **graph** splits the conversation into stages — e.g. `hub` (triage/advise), `intake`, `manage`, `close` — where each **node** carries only its own focused prompt + only the tools it needs, and **transitions** move between nodes based on the user's intent (or a deterministic flag). More reliable behavior, without losing the ability to move anywhere, any turn.

Core principle: **route on the user's intent, gate on facts.** Which stage to go to is an LLM judgment; "are we done?" and "is the user verified?" are deterministic predicates over state — never an LLM guess.

**Sibling skills:** for a *hosted* platform version of this model (voice, managed infra), `elevenlabs-agents`. For the backend/stack to run the engine on, `client-backend-skills`.

## When to use

- A chatbot with distinct stages (intake → action → confirmation) where one prompt + all tools has gotten unreliable.
- You want the focus of a workflow platform but the **speed and control of your own code** (no platform transport overhead).
- A multi-step flow needs a **hard security gate** (e.g. verify identity before touching account data) that a prompt instruction can't be trusted to hold.
- Symptoms of getting it wrong: the bot **transitions mid-flow and claims a step is done** that never happened; a stage **won't route** even on a clear request; a verification gate **leaks**.

**Not for:** a genuinely single-purpose bot (one stage → keep the monolith); a voice agent where a hosted platform's audio pipeline is the point (→ `elevenlabs-agents`); pure retrieval/RAG Q&A with no stages.

## The model (node-centric)

Each node owns its outgoing transitions. Full data model + the runtime loop: [reference/the-model.md](reference/the-model.md).

- **Node** = `{ prompt (additive, on a shared base), tools[], toolGate?, edges[] }`. Scope tools per node.
- **Edge (transition)** = `{ target, type, label, condition|predicate }`. `type` ∈ **`unconditional`** (always) / **`llm`** (a classifier judges a natural-language condition) / **`expression`** (a deterministic predicate over session state — code, no LLM). A "go back" is just an edge whose target is an earlier node.
- **Runtime loop:** the current node answers the turn (tool-loop, tools filtered by `toolGate`); then evaluate its edges in priority order — `expression` inline (free, short-circuit), `llm` via one lazy classifier call — to pick the next node.

## The map

| Concern | Reference |
|---|---|
| Data model (node / edge / 3 transition types / session state) + the runtime engine loop | [reference/the-model.md](reference/the-model.md) |
| **Transition gotchas** — the hard walls: route-on-intent, complete-by-expression, classifier balance, expression security gates | [reference/transitions-and-gotchas.md](reference/transitions-and-gotchas.md) |
| When a **code graph** vs a **hosted platform** vs a **simple 2-way router** | [reference/code-vs-platform.md](reference/code-vs-platform.md) |

## Quick reference

- **3 transition types:** `unconditional`, `llm` (intent — fuzzy), `expression` (deterministic predicate — for completion + security gates).
- **Completion (→ close) is `expression`, never `llm`.** An eager classifier fires "we're done" mid-flow → a tool-less close node hallucinates the action.
- **Security gate = `toolGate` by `expression`:** hide a node's sensitive tools until `verified === true`. Deterministic, not jailbreakable — beats prompt-enforcement.
- **Classify on the user's message + prior history — NOT the node's just-generated reply.** Including the reply makes the classifier think "the node is already handling it" → no transition.
- **Test the graph deterministically:** unit-test the `expr` / `toolGate` / flag-setters (pure functions, no LLM) + a few live route/gate checks.

## Common mistakes

- **The bot transitions mid-flow and claims a step finished that didn't** (e.g. "your appointment is cancelled" — nothing was cancelled). The completion edge was `llm` ("done?"); the classifier fired early; the target node had no tools and hallucinated. → make completion edges `expression` over a flag a tool sets. [transitions-and-gotchas.md]
- **A stage won't route even on a clear request** ("I want to book" stays in hub). The classifier was fed the node's own acknowledging reply → read "already being handled" → NONE. Classify on the user's message + history only. [transitions-and-gotchas.md]
- **The classifier flip-flops** (too eager → false transitions; too shy → no routing). Route on clear current intent; reserve "abandoned / said goodbye" edges for a real topic change. [transitions-and-gotchas.md]
- **A verification gate leaks** because the prompt was "supposed to" hold it. Make it a `toolGate` `expression` — the sensitive tools don't exist for the model until the flag flips. [transitions-and-gotchas.md]
- **Building a code graph when a 2-way router or a hosted platform was the right tool.** [code-vs-platform.md]
