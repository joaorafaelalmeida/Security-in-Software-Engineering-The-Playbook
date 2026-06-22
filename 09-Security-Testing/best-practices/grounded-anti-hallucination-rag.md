# Grounded, Anti-Hallucination RAG for a Security Assistant

> A local retrieval-augmented pipeline that makes an AI security assistant checkable, gating retrieval on distance so it refuses out-of-scope questions instead of inventing answers.

## The Problem

An AI assistant that helps triage security findings is only useful if you can trust it. A model that sounds authoritative but is wrong is worse than no assistant at all: it produces confident, plausible prose about a CWE, a severity, or a fix that a reviewer then propagates without checking. Every claim it makes should trace back to a source the reviewer can open.

The naive defense is a system prompt that says "only answer from the provided context, otherwise say you do not know." That is necessary but it is not a control. A small local model will still answer out-of-scope questions from its own parametric memory and sound confident doing it. In one measured case, an 8B model happily answered a Kubernetes RBAC question that had nothing to do with the corpus, despite being told not to. A do-not-hallucinate instruction is a wish, not a guarantee, because it depends on the model choosing to behave.

What can go wrong if you ship the prompt-only approach:

- the assistant answers questions outside its knowledge base with fabricated detail;
- citations get attached to answers that were actually out of scope;
- reviewers over-rely on output that reads as authoritative but is ungrounded.

## The Solution

### Overview

Retrieval-augmented generation (RAG) grounds a model by retrieving relevant source text at query time and forcing the model to write from that text. The key move for a security assistant is to treat anti-hallucination as a deterministic engineering control that runs before the model is ever called, not as something you ask the model to do.

A minimal pipeline that runs fully locally through [Ollama](https://ollama.com) (no API key, no cost, no data leaving the machine) has five steps:

1. **Chunk the corpus by heading.** Split your reference material (lab notes, standards like OWASP or CWE, runbooks) into chunks at section boundaries. Heading-aligned chunks keep each piece self-contained and give you a natural citation unit (`file # heading`).
2. **Embed each chunk with a local model.** Run an embedding model (for example `nomic-embed-text`) over every chunk to get a vector.
3. **Store the vectors in a local vector DB.** [Chroma](https://www.trychroma.com) persists the vectors plus each chunk's text and source path on disk. Configure it for cosine distance.
4. **Retrieve top-k by similarity.** Embed the incoming question, query the store for the nearest k chunks, and keep their source paths for citations.
5. **Generate grounded prose only.** Pass the retrieved chunks to the model with instructions to answer from that context and cite the sources.

Keep deterministic facts out of the model. In a security triage tool, the severity, the `file:line` location, and the OWASP or CWE mapping come straight from the scanner output; only the explanatory prose is generated, and the citations are the chunks that were actually retrieved. As an example, the `ai-appsec-copilot` project in this portfolio follows exactly this split: the scanner gives the facts, the model writes the grounded note, and the retrieved chunks are the references.

### Implementation

#### Lesson 1: Embedding prefixes are part of the model's contract

Several embedding models are trained with task prefixes that tell the model whether it is encoding a stored passage or a search query. `nomic-embed-text` expects `search_document: ` in front of passages at ingest time and `search_query: ` in front of questions at query time. Skipping the prefixes silently degrades retrieval, and the damage is worst on short queries, which are exactly what a question-answering assistant receives.

```python
prefix = "search_query: " if as_query else "search_document: "
inputs = [prefix + t for t in texts]
embeddings = client.embed(model=EMBED_MODEL, input=inputs)
```

Adding the prefixes turns vague top-k results into sharply on-topic ones, which is what makes the citations trustworthy. Always read your embedding model's card for its expected input format, and gate the prefix on the model name so swapping models does not silently apply the wrong convention.

#### Lesson 2: Anti-hallucination is a deterministic gate, not a prompt wish

This is the headline control. Instead of trusting the model to refuse, measure the cosine distance of the nearest retrieved chunk and refuse before generation if it exceeds a threshold:

```python
hits = retrieve(question, k=...)
if not hits or hits[0].distance > RELEVANCE_MAX_DISTANCE:
    return "I don't have that in my knowledge base."
# only now call the model, with the retrieved context
```

Pick the threshold empirically by measuring distances on known in-corpus and out-of-corpus questions. In the reference project, in-corpus questions landed at a distance of roughly 0.23 to 0.36 and out-of-corpus questions at roughly 0.41 and up, so a threshold near 0.38 to 0.40 separates the two populations with margin. These numbers are corpus-specific: re-measure for your own data and embedding model rather than copying the value. Expose the threshold as config so it is tunable without code changes.

This gate is the single most valuable control here, because it converts "the model should refuse" (a wish) into "the system refuses" (a guarantee that does not depend on the model behaving).

#### Lesson 3: Agentic tool-use with a deterministic fallback

You can let the model call a `search_knowledge_base` tool itself and decide when to retrieve. This agentic style is appealing because the model can issue several queries and reason over the results. The problem is that small local models are unreliable at tool use: they sometimes skip the tool, malform the call, or loop.

So always pair the agentic path with a deterministic retrieve-then-generate fallback. Try the tool-using loop, but if the model never calls the tool, errors, or returns nothing usable, fall back to a plain "retrieve top-k, then generate from that context" pass. The output is grounded either way.

```python
# Retrieve canonical sources once up front; these ground the fallback
# and are the citations shown, regardless of which path writes the prose.
baseline = retrieve(query, k=3)
try:
    prose, tool_used = agentic_analyze(item)   # model drives the search tool
    if not (tool_used and prose.strip()):
        prose = grounded_prose(item, baseline)  # deterministic fallback
except Exception:
    prose = grounded_prose(item, baseline)
```

A useful pattern is to retrieve a canonical set of sources for the item once, up front, and use that set both as the fallback context and as the citations you display, no matter which path produced the prose. That keeps citations stable and on-topic even when the agentic path wandered. Offer a flag (for example `--no-tools`) to force the deterministic path for reproducibility and debugging.

#### Lesson 4: Offline by construction

For security data, the whole stack should run on the machine and nothing should phone home:

- **Local generation** through Ollama, so prompts and findings never go to a third party.
- **Local embeddings** with a local embedding model, so your corpus is never uploaded to an embedding API.
- **On-disk vector store** with Chroma's persistent client, so the index lives next to the project.
- **Telemetry off.** Disable the vector DB's analytics so it does not emit usage data:

```python
chromadb.PersistentClient(
    path=str(CHROMA_DIR),
    settings=Settings(anonymized_telemetry=False),
)
```

Because you always pass your own vectors at query time, you do not need the vector DB's built-in embedding function, which also means it never downloads a model on its own.

### Bad Practice vs Good Practice

The two approaches differ in one decision: who is responsible for refusing an out-of-scope question.

**Bad Practice (prompt-only self-policing):**

```python
SYSTEM = "Answer ONLY from the provided context. If it is not there, say you don't know."
context = retrieve(question, k=5)          # always returns something
return model.generate(SYSTEM, context, question)   # model decides whether to refuse
```

Why this is problematic:

- top-k retrieval always returns the nearest chunks, even for a question with no real match;
- the model is free to ignore the instruction and answer from parametric memory;
- the answer reads as authoritative and the off-topic chunks get cited as if they supported it.

**Good Practice (deterministic distance gate):**

```python
hits = retrieve(question, k=5)
if not hits or hits[0].distance > RELEVANCE_MAX_DISTANCE:
    return "I don't have that in my knowledge base."   # system refuses, not the model
return grounded_prose(question, hits)                  # grounded, with on-topic citations
```

Why this works:

- the refusal is enforced by code before the model runs, so it does not depend on model behavior;
- the threshold is measured against real in-corpus and out-of-corpus distances, with margin;
- citations only ever attach to retrievals that passed the gate.

## Common Pitfalls

1. **Trusting the model to self-police.** A prompt instruction to refuse out-of-scope questions is not a control. Add the distance gate (Lesson 2) so refusal does not depend on the model.
2. **Missing embedding prefixes.** Forgetting `search_document:` or `search_query:` (or whatever your model expects) quietly wrecks retrieval, especially on short queries, and you will not get an error, just worse answers.
3. **Citing whatever was retrieved, even if off-topic.** Top-k always returns something. Anchor your citations on a canonical query for the item and respect the distance gate, so you never attach confident-looking sources to an answer that was actually out of scope.
4. **Over-tight distance thresholds.** Setting the threshold too low makes the assistant refuse valid in-corpus questions. Tune against real examples from both populations and leave margin; the goal is to separate in-corpus from out-of-corpus, not to maximize refusals.

## When to Apply

- **Always:** When an AI assistant produces security claims (CWE mappings, fixes, triage notes) that a human will act on, and those claims must be checkable against a source.
- **Recommended:** When you process security data that must not leave the machine, so local generation, local embeddings, and an on-disk store are required.
- **Consider:** For any focused-corpus question-answering assistant where a confidently wrong answer would do more harm than a refusal.

## Standards & Compliance

- **OWASP Top 10 for LLM Applications:** Directly addresses LLM09 (Misinformation / over-reliance on fabricated output) and supports the over-reliance and prompt-injection (LLM01) context by grounding answers in retrieved sources and refusing when none are relevant.
- **RAG note:** Retrieval-augmented generation is the structural pattern here; the deterministic relevance gate is what turns RAG from a quality improvement into an anti-hallucination control.

## Further Reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks (Lewis et al., 2020)](https://arxiv.org/abs/2005.11401)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Ollama (local model serving)](https://ollama.com)
- [Chroma (local vector database)](https://www.trychroma.com)
- [nomic-embed-text (embedding model and its prefix convention)](https://ollama.com/library/nomic-embed-text)

## Tags

`ai` `rag` `llm` `anti-hallucination` `devsecops` `local-llm`

---
**Contributed by:** Henrique Teixeira
**Last Updated:** 2026-06-16
**Difficulty Level:** Advanced
**Impact:** Medium
