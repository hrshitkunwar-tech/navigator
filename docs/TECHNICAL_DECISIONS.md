# Technical Decisions

Every significant architectural choice Navigator made, with the reasoning behind it. These aren't preferences — they're decisions made after testing alternatives and considering the specific constraints of an agentic browser system.

---

## No Vector Database

**Decision:** No Pinecone, ChromaDB, Weaviate, or any standalone vector store.

**The standard approach:**
```
Document → chunk → embed (OpenAI/Cohere) → upsert to vector DB
    → query → embed query → cosine similarity → re-rank → return top-k
```

**Why we rejected it:**

**1. Similarity ≠ planning utility.**
Cosine similarity finds text that *sounds like* the query. Navigator needs traces that *worked* when planning similar tasks. When deciding how to create a GitHub repository, a trace where step 3 was "click the green button" is more useful than a trace that semantically resembles the query text. CLaRa optimizes for planning utility. That's a fundamentally different objective.

**2. The pipeline adds latency at every joint.**
Embed the query → network roundtrip to vector DB → fetch candidates → re-rank with a second model. CLaRa retrieval happens at inference time — no separate embedding step, no vector store network hop.

**3. Chunking destroys document structure.**
A 512-token chunk of Jira documentation loses the information that it's under "Advanced Settings → Company-managed projects." PageIndex preserves the full hierarchy. The LLM navigates the tree like reading a table of contents — it knows that section 4.2.1 is a subsection of 4.2, which is under "Permissions." A chunk doesn't know that.

**4. One fewer service to operate.**
Convex handles the narrow case where vector similarity is genuinely useful (cold-start candidate retrieval). No Pinecone account, no embedding cost during retrieval, no vector DB downtime to care about.

**The numbers:**
- CLaRa Top-1 hit rate on product support docs: **61%** vs RAG: **51%**
- PageIndex accuracy on FinanceBench: **98.7%** vs vector RAG: **~50%**

---

## Convex Over Postgres + Redis + Neo4j + Queue

**Decision:** Convex as the sole backend, replacing what would otherwise be four services.

**The alternative stack:**
- Postgres for relational data (users, tools, executions)
- Redis for caching and pub/sub (real-time dashboard updates)
- Neo4j or similar for graph relationships (cross-tool trace connections)
- BullMQ or SQS for background job processing (CLaRa compression jobs)
- Pinecone for vector search (cold-start candidate retrieval)

**Why Convex:**

**Real-time sync without infrastructure.**
When a user executes a workflow via the Chrome extension, the trace writes to Convex and the B2B dashboard updates live. With a traditional stack: write to Postgres → publish to Redis pub/sub → WebSocket server pushes to client. With Convex: one write, reactive queries update automatically. Less infrastructure, fewer failure surfaces.

**TypeScript end-to-end.**
Convex's SDK is TypeScript-native. The schema, queries, mutations, and scheduled functions are all TypeScript. No SQL string interpolation, no ORM abstraction layer, no separate migration tooling. Types flow from the schema definition through every query and into the client.

**One platform, one billing line.**
During early-stage development, managing five separate services (Postgres, Redis, Neo4j, BullMQ, Pinecone) adds operational overhead that's disproportionate to what it buys. Convex covers the use cases at the current scale. As Navigator grows, individual services can be extracted where the tradeoff makes sense.

**The Kailash Nadh principle:** As simple as possible. Navigator has enough novel complexity in the knowledge layer. Infrastructure complexity should not add to it.

---

## Preact + Shadow DOM Over React

**Decision:** Preact (3KB) instead of React (40KB) for the Chrome extension UI.

**The problem with React in an extension:**
The Chrome extension injects a UI panel into every page a user visits. That means:
- 40KB of React JavaScript loads on every tab the user opens
- React's styles and event handling can conflict with the host page
- The host page's JavaScript can conflict with React

**Why Preact:**
Preact is API-compatible with React — components written for one work in the other with minimal changes. The extension panel components are straightforward enough that we don't need React's full feature surface. The cost is 3KB injected payload instead of 40KB.

**Why Shadow DOM:**
Shadow DOM creates a complete encapsulation boundary between the extension panel and the host page. The panel's CSS is scoped to the panel — it cannot inherit or conflict with the host page's styles. The panel's JavaScript events don't bubble into the host page. The host page cannot accidentally style or break the panel.

---

## TypeScript Everywhere

**Decision:** TypeScript for extension, backend, planning layer, and SDK. No Go, Python, or Rust in the production stack.

**What we avoid:**
Every language boundary in a monorepo costs:
- A separate linting configuration
- A separate testing framework
- A cognitive context switch when reading cross-boundary code
- Type contracts that the compiler cannot verify

**The specific benefit for Navigator:**
The types `UIState`, `WorkflowPlan`, `ExecutionStep`, and `DecisionTrace` flow from the extension (captures UI state) through the planner (produces workflow plan) through the executor (runs steps) into Convex (stores traces). TypeScript end-to-end means those types are compiler-verified at every step. A Go microservice at any point would require a serialization contract and manual sync between the Go struct and the TypeScript interface.

**The exceptions:**
CLaRa 7B and PageIndex are external models with Python inference servers. They're behind API boundaries. The boundaries are typed in TypeScript. Model internals are irrelevant to Navigator's type system.

---

## Named Exports Only

**Decision:** No default exports anywhere in the monorepo.

`import Widget from './widget'` — what is Widget? Is it a component? A class? A function? You have to open the file.

`import { WorkflowPlanner } from './planner'` — unambiguous.

In a monorepo where packages import from other packages, name clarity matters at every import site. Named exports also make refactoring straightforward: rename the export and the compiler finds every import site that needs updating.

---

## Files Under 300 Lines

**Decision:** Split any file that exceeds 300 lines.

Two reasons:

**Claude Code context efficiency.**
When a Claude Code session loads a 1,200-line file to make a 10-line change, it consumes context window capacity on code that isn't relevant to the task. A 300-line file with one clear responsibility uses context efficiently. Over a project lifetime with many Claude Code sessions, this compounds.

**Testability.**
A 300-line file has one responsibility. A 1,200-line file has four. Testing one responsibility is a test file with 10–20 focused cases. Testing four tangled responsibilities is a test file with 60+ cases full of setup code. The constraint enforces modularity.

---

## Zod at Package Boundaries

**Decision:** Runtime validation with Zod at every module interface.

TypeScript catches type errors at compile time. It cannot catch them at runtime — which matters at:

- **Network calls:** The extension sends UIState to the planning service. TypeScript says the type is correct. What if the serialization dropped a field?
- **External model outputs:** Claude returns a WorkflowPlan. TypeScript believes whatever you cast it to. Zod actually checks.
- **Convex reads:** Data was written correctly six months ago. The schema changed. Zod catches the mismatch before it becomes a runtime error in the executor.

The pattern: TypeScript interfaces define the compile-time shape. Zod schemas define the runtime shape. They live side by side in `packages/shared/`. At package boundaries, data goes through the Zod schema. If it passes, TypeScript takes over.

---

## n8n as Orchestration Trust Boundary (MVP)

**Decision (Navigator MVP):** n8n is the sole entity that executes decisions. AI agents only recommend.

This was the key architectural insight from the MVP, now formalized in the full Navigator design as the Layer 3/Layer 4 separation.

**The problem it solves:**
When an LLM is both planner and executor, failures are ambiguous. Did it fail because the plan was wrong, or because the execution was wrong? When a deterministic orchestrator (n8n in the MVP, the Executor package in production) executes based on LLM recommendations:

- "Claude recommended clicking the button labeled 'Submit'"
- "The executor clicked the element matching text='Submit'"
- "The success condition was not met"
- Root cause is attributable: wrong recommendation, wrong element match, or wrong success condition — each is a different fix

**In production Navigator:**
Layer 3 (Reasoning / Claude) produces a `WorkflowPlan` with explicit `ExecutionStep` objects. Layer 4 (Execution) runs each step deterministically and checks each `successCondition`. Claude recommends. The executor decides whether to proceed. The audit trail is in Convex.
