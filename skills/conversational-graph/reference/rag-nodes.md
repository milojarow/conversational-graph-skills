# RAG nodes — a knowledge-base node is only as good as its embedding model

When a graph node answers questions from a knowledge base (RAG), answer quality depends MORE on the embedding model than on the prompt. An old or English-centric embedding (e.g. OpenAI `text-embedding-ada-002`) ranks non-English content (Spanish, etc.) poorly: the exact term the user asks about can fall far below the top-K window, so the node NEVER sees the relevant chunk and ends up hedging ("I don't have that confirmed") **even though the info IS in the docs**.

## Symptom

The node gives vague/generic answers to specific questions the KB does cover, or claims it doesn't have the fact. It looks like a prompt problem — tempting to hardcode the info or bolt on a keyword patch — but the root cause is **retrieval**.

## Diagnose it (measure, don't guess)

Embed the user's query with the model in use and rank the chunks that DO contain the term, ordered by cosine distance. If the first relevant chunk lands outside top-K (observed: a Spanish term at **rank 45** with ada-002, **0 of the top-20**), the problem is the embedding, not the prompt.

## Root fix

Re-embed with a strong multilingual model. `text-embedding-3-large` (supports a `dimensions` arg to fit an existing `vector(1536)` column) or a `multilingual-e5`. With 3-large the same term jumped from rank 45 to **top-3** and the node answered specifically — **without touching the prompt and without a keyword patch**.

## Anti-pattern

A hybrid keyword+vector retrieval "patches" the symptom for terms you already know, but doesn't fix the root (the embedding). Use it as an interim net, not the solution.

## Infra notes for the re-embed

- **Prefer a HOSTED (API) embedding over running an open-source model locally** if the host is small: a ~500M-param transformer can blow a modest VPS's RAM. Hosted = zero local compute.
- If the vector table is shared with other consumers that query with the OLD model, **re-embed into a SEPARATE table** — mixing embedding spaces breaks cosine similarity. The new table is a snapshot: re-embed when the source content changes.
- The query model and the stored-vector model MUST be the same (same model + same `dimensions`).
