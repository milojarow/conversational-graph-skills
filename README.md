# conversational-graph-skills

**Build a multi-stage conversational agent as a graph you run in your own code — focused nodes, forward/back transitions, deterministic gates.**

## What is this?

A skill that captures the architecture for building a conversational agent as a **code-driven graph / state machine**: the chat is split into focused stages (`nodes`, each with its own tight prompt + only its tools), connected by `transitions` (forward and back). It is the same mental model as a hosted agent-workflow platform — implemented in your own stack, so it is fast, fully controllable, and can use deterministic transitions a platform won't give you.

### Why this skill exists

The non-obvious walls it encodes (each one cost real debugging):

- **Route on the user's *intent*, not the node's own freshly-generated reply** — feeding the reply to the router makes it think "this node is already handling it" and it never transitions.
- **Make "we're done" transitions deterministic (`expression`), never an LLM guess** — an eager classifier fires mid-flow and a tool-less close node then *hallucinates* a completed action.
- **The intent classifier needs balance** — too eager invents transitions; too conservative refuses to route at all.
- **Security gates belong in code (`expression` tool-gates), not in a prompt instruction** — hide a node's sensitive tools until a fact flips; a prompt "supposed to" hold the line leaks.
- **Knowing when a code graph is the wrong tool** — sometimes a 2-way router, or a hosted platform, is the right call.

## The skill

| Skill | Description |
|-------|-------------|
| **conversational-graph** | The data model (node / edge / transition types), the runtime engine loop, the hard-won transition gotchas, and when to use a code graph vs a hosted platform vs a simple router. |

## Installation

Add this marketplace in Claude Code:

```
/plugin → Marketplaces → Add Marketplace → milojarow/conversational-graph-skills
```

Then install:

```
/plugin → Discover → conversational-graph-skills → Install
```

## Requirements

- A backend you control (any language/stack) calling an LLM with tool/function-calling. Examples are in TypeScript-flavored pseudocode but the pattern is stack-agnostic.

## License

MIT
