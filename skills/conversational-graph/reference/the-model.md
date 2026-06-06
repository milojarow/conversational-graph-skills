# The model — node, edge, transition types, and the runtime loop

A code-driven conversational graph has three pieces: a set of **nodes**, the **transitions** between them, and a small **runtime engine** that drives one turn. It is **node-centric**: each node owns its outgoing transitions. (Examples are TypeScript-flavored pseudocode; the pattern is stack-agnostic.)

## Node

A node is one focused stage of the conversation.

```ts
interface Node {
  id: string;
  prompt: string;                      // ADDITIVE: prepended with a shared BASE prompt, not a full replacement
  tools: string[];                     // ONLY the tools this stage needs
  toolGate?: (tool: string, ctx: Ctx) => boolean;  // optional: hide a tool until a condition holds
  edges: Edge[];                       // outgoing transitions, in priority order
}
```

- **The per-node prompt is additive.** Keep a shared `BASE` prompt (persona, tone, global rules, "only talk about X") and build each turn's system prompt as `BASE + node.prompt`. Every node speaks consistently; the node prompt only adds its stage-specific job.
- **Scope tools per node.** The action stage sees only its action tools; the model isn't choosing among many unrelated tools every turn. Fewer tools offered → fewer wrong calls.

## Edge (transition)

A transition is a directed move from one node to another.

```ts
type EdgeType = "unconditional" | "llm" | "expression";
interface Edge {
  target: string;
  type: EdgeType;
  label: string;                       // short human label ("book", "changed mind")
  condition?: string;                  // type 'llm': natural-language condition the classifier judges
  predicate?: (ctx: Ctx) => boolean;   // type 'expression': deterministic check over session state
}
```

| type | evaluated by | use it for |
|---|---|---|
| `unconditional` | nothing (always passes) | the entry edge (start → hub) |
| `llm` | one cheap classifier call | **intent routing** — fuzzy ("the user wants to book") |
| `expression` | your code (predicate over `ctx`) | **completion + security gates** — deterministic (`ctx.booking_created === true`) |

**Node-centric, so "back" is not a special case.** Each node owns its outgoing edges, each pointing to ANY node — including one earlier in the flow. "Return to the hub" is just an edge whose `target` is the hub. You do NOT need a separate forward/backward concept. (A hosted platform may force forward+backward onto a single edge object because it forbids two edges between the same node pair; your own engine has no such constraint — model it the natural way.)

## Session state (`ctx`) — what `expression` edges read

Persist a small per-session object that deterministic transitions and tool-gates evaluate:

```ts
interface Ctx {
  currentNode: string;
  verified: boolean;       // a verify tool flipped this
  action_done: boolean;    // an action tool flipped this (e.g. booking_created / cancelled)
  profile?: object;        // who the user is, if already authenticated
}
```

Tools update `ctx` as a **side-effect of their result**: after each tool call the engine inspects `(toolName, result)` and flips the relevant flag (e.g. `verify` returns `{verified:true}` → `ctx.verified = true`; the create/cancel tool succeeds → `ctx.action_done = true`). Keep this mapping in a **pure side-effects module** — no I/O — so it is trivially unit-testable.

## The runtime loop (one turn)

1. **Load** `ctx` + `currentNode` (default to the hub; normalize a terminal/`end` node back to the hub) + recent conversation history.
2. **Run the current node.** Build `messages = [BASE + node.prompt, ...history, userMessage]`; `tools = node.tools` filtered by `node.toolGate(tool, ctx)`. Run the normal tool-calling loop. After each tool result, apply its side-effect to `ctx`. **Re-filter the tools each round** — a gate may open mid-turn (e.g. the look-up tool appears the instant `verify` succeeds, in the same turn).
3. **Evaluate transitions** of the current node, **in priority order**:
   - `expression` → run the predicate against `ctx` inline (free) → on the first true, transition (short-circuit, skip the classifier).
   - `llm` → make ONE classifier call for all the node's llm edges (lazy: only if no higher-priority expression won). Map its answer to the matching edge.
   - First match wins; no match → stay in the current node.
4. **Persist** `ctx` (+ the possibly-new `currentNode`). Reset transient flags (`verified`, `action_done`) when you return to the hub — it's a fresh stage.

**Latency:** `expression` edges cost nothing; the lazy classifier is ~one cheap-model call per turn (comparable to a 2-way intent detector). Use a small fast model for the classifier and your main model for the node response.

## Classifier shape

Batch a node's `llm` edges into ONE call. Give the cheap model the conditions numbered, and the **user's message + prior history** (see transitions-and-gotchas: do NOT include the node's freshly-generated reply). Ask it to answer with the matching number or NONE; parse the first integer.
