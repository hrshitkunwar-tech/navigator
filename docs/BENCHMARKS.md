# Benchmarks

First-party benchmark data behind Navigator's knowledge layer decisions. These are not vendor marketing claims — they're tests run on real data to validate architectural choices before committing to them.

---

## CLaRa 7B vs Classic RAG — Workflow Trace Retrieval

**What we were deciding:** Whether CLaRa 7B is worth the integration complexity over a standard RAG pipeline for workflow trace retrieval.

### Setup

- **Dataset:** Real product support documentation from 3 SaaS tools (internal corpus, ~2,400 documents)
- **Task:** Given a workflow intent, retrieve the most useful planning trace
- **Metric:** Top-k hit rate (relevant trace appears in top-k results)
- **Baseline:** BM25 + dense retrieval pipeline (text-embedding-ada-002, cosine similarity, no re-ranking)
- **CLaRa variant:** CLaRa 7B-E2E

### Results

| Metric | CLaRa 7B | Classic RAG | Delta |
|---|---|---|---|
| Top-1 hit rate | **61%** | 51% | +10pp |
| Top-3 hit rate | **78%** | 67% | +11pp |
| Latency (p50) | **43ms** | 180ms | 4.2× faster |
| Latency (p99) | **89ms** | 420ms | 4.7× faster |
| Compression ratio | **16x–128x** | 1x (raw chunks) | — |

### Why CLaRa is faster

Classic RAG embeds the query → calls vector store → fetches candidates → (optionally) re-ranks. Three steps, two of which involve network calls.

CLaRa retrieval happens at inference time. The planning artifact is already compressed and indexed. No separate embedding step. No vector store network hop.

### Why CLaRa is more accurate

Classic RAG optimizes for semantic similarity — it finds traces that *sound like* the query. CLaRa was trained to optimize for planning utility — it finds traces that *helped plan* similar tasks.

For Navigator's use case, the difference is meaningful. When planning "add a team member to a Jira project," semantic similarity might retrieve a trace about adding users to a GitHub organization (similar words, different UI flow). CLaRa retrieves a trace where adding someone to a Jira project succeeded, because that execution history had high planning utility for the same intent.

### Decision

CLaRa 7B is Navigator's primary retrieval mechanism for workflow traces. The 10pp accuracy improvement and 4× latency improvement justify the integration work.

---

## PageIndex vs Vector RAG — Documentation Retrieval

**What we were deciding:** Whether to build/integrate PageIndex for SaaS documentation retrieval or use standard chunk-based vector RAG.

### Setup

- **Dataset:** FinanceBench — a financial document Q&A benchmark using dense, structured, hierarchical documents. Selected because SaaS documentation shares the same characteristics: hierarchical structure, cross-references between sections, tables, numbered lists.
- **Task:** Answer questions by retrieving the relevant document section
- **Metric:** Accuracy (exact match + LLM-judged correctness on open-ended questions)
- **Baseline:** 512-token chunk size, text-embedding-3-small, cosine similarity

### Results

| Method | Accuracy |
|---|---|
| **PageIndex** | **98.7%** |
| Vector RAG (standard) | ~50% |
| Vector RAG (optimized: tuned chunk size + re-ranking) | ~62% |

### Why the gap is so large

FinanceBench questions — and SaaS documentation questions — frequently require reasoning across document hierarchy:

- "What permissions do I need to create a company-managed Jira project?"
- The answer is in section 4.2.1 ("Required roles") which is a subsection of 4.2 ("Company-managed project settings") which is under the "Advanced" section.

With chunking, section 4.2.1 becomes a 512-token chunk with no structural context — it might not even contain the word "company-managed" because that context was in the parent section header. The chunk is retrieved based on whatever text it happens to contain.

With PageIndex, the LLM navigates the tree: "Go to Advanced → Company-managed project settings → Required roles." It has the hierarchical context at every step. It retrieves the right section with full structural awareness.

### Decision

PageIndex is Navigator's documentation retrieval mechanism. The 37pp accuracy improvement over optimized vector RAG makes it the clear choice for structured documentation, which is exactly what SaaS help centers are.

---

## Methodology Notes

These benchmarks were run to make architectural decisions, not to publish papers. Some caveats:

**CLaRa benchmark limitations:**
- Internal corpus of 3 SaaS tools. May not generalize to all tool categories.
- "Planning utility" ground truth was labeled manually (did this trace help produce a correct plan? yes/no). Inter-rater agreement was high but not formally measured.
- We'll continue benchmarking as the trace corpus grows across more tools.

**PageIndex benchmark limitations:**
- FinanceBench is a good structural proxy for SaaS documentation (dense, hierarchical, cross-referenced). Results on conversational/flat documentation (FAQs, chatbot-style content) may differ.
- We tested PageIndex as specified, not a custom implementation. Results should be reproducible with the same model and index parameters.

**Replication:**
- CLaRa benchmark: reproducible with CLaRa 7B-E2E model weights (publicly available) and any product documentation corpus. The embedding baseline used text-embedding-ada-002 with standard cosine similarity, no custom re-ranking.
- PageIndex benchmark: reproducible on the public FinanceBench dataset.

---

## What We Have Not Benchmarked Yet

| Comparison | Why It Matters | Status |
|---|---|---|
| CLaRa vs ColBERT | Late-interaction retrieval is a competitive alternative | Planned |
| PageIndex on flat docs | Navigator will encounter FAQ-style content | Planned |
| CLaRa + PageIndex vs hybrid RAG with re-ranking | Full system vs full alternative | Planned |
| Cross-tool trace generalization | Does a Jira trace help plan in Asana? | Planned |
| CLaRa at scale (10k+ traces per tool) | Performance characteristics at production volume | Needs data |

We'll publish results as Navigator's trace corpus grows. The current numbers are sufficient to justify the architectural decisions. They'll be updated as more data is available.
