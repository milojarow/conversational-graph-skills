# Code graph vs hosted platform vs simple router

Three ways to structure a conversational bot. Pick by the shape of the problem — don't reach for the graph by reflex.

## Simple N-way router — the lightest

A per-message classifier routes each turn into one of a few **branches** (e.g. "general chat" vs "transactional"), each branch a prompt + its tools. No persisted current-node, no transition graph — every message is re-routed from scratch.

- **Use when:** there are only ~2-3 modes and little real multi-step *state* to hold between turns.
- **Why it's enough:** stateless per-message re-routing already gives "go anywhere, any turn" for free — you don't need a graph's machinery to move around.
- **Limit:** no per-stage focus that *persists*; a long multi-step flow (collect data → verify → act → confirm) gets muddy when one broad branch has to hold all of it, and mid-flow "stay here" vs "leave" is fuzzy.

## Code graph (this skill) — focus + control + determinism

A persisted current-node; each node a focused prompt + scoped tools; transitions including deterministic `expression` gates and tool-gates.

- **Use when:** 3+ distinct stages, a multi-step flow with state, and/or a **hard security or completion gate** you need to be deterministic.
- **Why:** per-node focus makes behavior reliable; `expression` transitions and tool-gates give instant, jailbroof gates a prompt can't; it runs in your own stack at full speed.
- **Cost:** you build and tune the engine — and the transition classifier is the part that needs care (see [transitions-and-gotchas.md](transitions-and-gotchas.md)).

## Hosted agent-workflow platform — managed, voice-first

A platform gives the same node/edge model in a visual editor with managed infra, RAG, and a voice (audio) pipeline. (For the platform mechanics, see the `elevenlabs-agents` skill.)

- **Use when:** you need **voice** (the audio pipeline is the point), or you want a managed/visual build and don't want to run the engine yourself.
- **Cost:** transport/orchestration latency — a voice-oriented platform carries real per-turn overhead even in text-only mode (measure it; it can dwarf the LLM time). And its transitions are LLM-judged — no deterministic `expression` gate. For a **text** bot that needs speed and hard gates, a code graph wins.

## One-line decision

- **Voice, or want it managed** → hosted platform.
- **Text, 3+ stages, want speed + deterministic gates** → code graph (this skill).
- **Just 2-3 modes, little persisted state** → simple router (don't over-build).

## Migrating between them

The **mental model is identical** (nodes, transitions, types), so porting is mostly mechanical: the node prompts and tool sets carry over; what changes is the engine and the transition substrate. A common path is to prototype on a platform (fast to stand up, visual) and, once the flow is proven and latency/control matter, re-implement the *same graph* in code — upgrading the LLM-judged completion/security transitions to deterministic `expression` ones on the way.
