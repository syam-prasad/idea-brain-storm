# ConceptIQ — Redesigned Architecture

> Revision of the original [conceptiq-idea-brainstorm.md](./conceptiq-idea-brainstorm.md).
> Scope decisions driving this redesign:
> - **MVP goal:** learning demo (exercise the interesting tech, not production hardening)
> - **Content:** curriculum-aligned (this is what justifies keeping RAG)
> - **Concept graph:** dumb version first (flat concepts + confidence, no ontology)

## Why redesign

The original brainstorm over-builds the commodity parts (a generic LLM wrapper, a
standalone Node backend, three datastores) while giving the one true differentiator —
the per-student concept graph — a single sentence. It also carries RAG without a stated
reason. This redesign inverts that: collapse the infrastructure, justify RAG through
curriculum-alignment, and keep the concept graph deliberately minimal until engagement
is proven.

## Design principles

1. **One datastore** unless a second earns its place. (RAG earns a vector store; the
   dumb concept graph does not earn a graph DB.)
2. **RAG is the star of the demo** — it's *why* the tutor is curriculum-aligned rather
   than generic — so build it properly.
3. **The tutor loop is one legible pipeline** you can trace end to end. No magic
   "Hermes" box.

## The stack

| Layer | Choice | Why (vs. original) |
|---|---|---|
| Frontend | **Next.js** | Keep. Fine. |
| Backend | **Next.js API routes** (drop separate Node server) | A standalone Node backend is a second process for no demo benefit. Co-locate it; add a real backend only when you outgrow it. |
| Relational + student data | **Postgres** | Replaces SQLite. Concurrent students; also hosts the concept graph as plain tables. |
| Vector store | **pgvector** (same Postgres) | The key move: RAG vectors live in the *same* Postgres. One datastore, not two. LlamaIndex has a native pgvector backend. |
| Embeddings + LLM | **OpenRouter** | Keep. |
| RAG framework | **LlamaIndex** | Keep — now genuinely justified by curriculum-alignment. |

**Net:** one datastore (Postgres + pgvector), one web app. Down from SQLite +
separate vector DB + separate Node server.

## Subsystem A — Curriculum RAG pipeline

What makes the tutor *aligned* rather than merely retrieval-augmented: every chunk is
tagged with the concept and standard it belongs to. That tag is also the bridge to the
concept graph.

**Ingestion (offline, once per textbook/curriculum):**

```
textbook / curriculum docs → chunk (by section/concept) → embed → pgvector
                                        ↓
                    tag each chunk with concept_id + standard_ref
```

**Query (per student question):**

```
question → embed → pgvector similarity search (top-k chunks)
                        ↓
        retrieved chunks (with concept tags) → tutor prompt
```

## Subsystem B — The tutor loop (replaces "Hermes")

One explicit, traceable pipeline — three steps, ~2 round-trips, one LLM call:

```
1. student message
2. retrieve  → top-k curriculum chunks from pgvector      (1 vector query)
3. generate  → LLM: explain using retrieved chunks,
               adapt depth to concept confidence,
               end with a check-understanding question     (1 LLM call)
4. update    → bump confidence for the concept(s) the
               retrieved chunks were tagged with           (1 Postgres write)
```

Intent detection and misconception detection are **deliberately cut** for the MVP —
noted as "later," not built.

## The "dumb" concept graph (two tables, no graph DB)

```sql
concepts(id, name, subject, standard_ref)          -- authored from curriculum during ingestion
student_concept(student_id, concept_id,
                confidence REAL, last_seen)          -- the entire "graph"
```

- **Confidence signal (simplest that works):** each time a concept's chunks are
  retrieved for a student, nudge confidence up; optionally down-weight when the
  student's answer to the check-question is wrong. No mastery propagation, no
  prerequisite edges.
- **Demo visual:** a heatmap of the student's concept confidences. Looks like a
  knowledge map; is actually two joins.

> Confidence is a flat per-concept score, not a real Bayesian knowledge model.
> Add prerequisite edges + mastery propagation when the flat version proves students
> engage.

## Explicitly cut (and when to add it back)

| Cut | Add when |
|---|---|
| Separate Node backend | You need background jobs or non-HTTP workloads |
| Intent detection | You have >1 distinct workflow to route between |
| Misconception detection | The flat concept graph proves engagement; then it gets its own doc |
| Prerequisite edges / mastery propagation | Same — after engagement is proven |
| Semantic caching of generations | Never for personalized output; cache *retrieval* only if cost bites |
| Voice / whiteboard / homework scan / gamification | Post-MVP, unchanged from original roadmap |
| COPPA/FERPA / safety hardening | The moment this stops being a demo and a real minor uses it |

## Caching — resolves a conflict in the original

The original "semantic caching" idea fights personalization: a cache that returns
Student B the explanation tuned for Student A undercuts the core differentiator.
Resolution: **cache the retrieval layer** (same question → same chunks, safe and cheap)
but **never cache the generation** (it's depth-adapted per student). Keeps the ~$30/mo
intent alive without breaking the differentiator.

## Cost note

The original's "under $30/month" holds only at demo scale. The single-LLM-call tutor
loop above (vs. the original's implied intent + retrieval + generation + verification
chain of 3–4 calls) is what keeps it there. RAG retrieval is a cheap vector query, not
an LLM call.

## One-line summary

Collapse three datastores into one Postgres + pgvector, replace the "Hermes" hand-wave
with a 3-step traceable loop, invest the saved complexity into doing curriculum-RAG
properly, and keep the concept graph as two tables until engagement is proven.
