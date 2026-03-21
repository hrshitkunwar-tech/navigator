# Navigator Architecture

Five independent layers. Each has one responsibility. Each is testable in isolation. Each can be replaced without touching the others.

---

## System Overview

```
[User: "Create a project named Navigator in Jira"]
    │
    ▼
┌─────────────────────────────────────┐
│         CHROME EXTENSION            │
│  Manifest V3 · Preact + Shadow DOM  │
└──────────────┬──────────────────────┘
               │
    ┌──────────┼───────────┐
    ▼          ▼           ▼
┌────────┐  ┌─────────┐  ┌──────────┐
│LAYER 1 │  │LAYER 2  │  │LAYER 3   │
│Percep- │  │Know-    │  │Reasoning │
│tion    │  │ledge    │  │          │
│Screen- │  │CLaRa +  │  │Claude    │
│Sense   │  │PageIndex│  │Sonnet    │
└────────┘  └─────────┘  └──────────┘
    │             │            │
    └─────────────┼────────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼                           ▼
┌──────────┐            ┌──────────┐
│ LAYER 4  │            │ LAYER 5  │
│Execution │            │Memory    │
│          │            │Decision  │
│DOM/WebMCP│            │Traces →  │
│          │            │CLaRa     │
└──────────┘            └──────────┘
                │
    ┌───────────▼──────────────┐
    │  CONVEX (sole backend)   │
    │  DB · serverless ·       │
    │  real-time · vector      │
    └──────────────────────────┘
```

---

## Layer 1: Perception (ScreenSense)

**Package:** `packages/screensense`
**Status:** ✅ Complete (9 Claude Code sessions)

ScreenSense converts a live browser UI into a structured, machine-readable representation. It sees — it does not act.

### Input sources

- **DOM** — element types, text content, position
- **Accessibility Tree** — ARIA roles, labels, states. The semantic layer browsers expose for assistive technology. More stable than CSS selectors.
- **WebMCP** — W3C standard for machine-readable UI metadata. Increasingly adopted by SaaS vendors. Forward-compatible.

### Output: UIState

```typescript
interface UIState {
  url: string
  title: string
  elements: InteractableElement[]
  pageType: 'form' | 'table' | 'dashboard' | 'modal' | 'nav' | 'unknown'
  timestamp: number
}

interface InteractableElement {
  id: string
  type: 'button' | 'input' | 'link' | 'select' | 'checkbox' | 'textarea' | 'other'
  text: string
  ariaLabel?: string
  role?: string
  position: BoundingBox
  isVisible: boolean
  isEnabled: boolean
  value?: string        // current value for inputs/selects
}

interface BoundingBox {
  x: number
  y: number
  width: number
  height: number
}
```

### Why Preact + Shadow DOM

The extension injects a panel into every page. That panel must not:
- Add 40KB to the page's JavaScript payload
- Conflict with the page's CSS
- Interfere with the page's event handlers

Preact: 3KB vs React's 40KB, API-compatible.
Shadow DOM: complete style and script encapsulation from the host page.

---

## Layer 2: Knowledge (Vectorless)

**Packages:** `packages/flowminer`, `packages/doctree`
**Status:** 📝 Spec complete

The standard AI retrieval pipeline:
```
Document → chunk → embed → store in vector DB → query → embed query → cosine similarity → re-rank → return
```

Navigator eliminates this pipeline. Two different approaches for two different data types.

### CLaRa 7B — Workflow Trace Retrieval

**For:** Past execution traces from Navigator sessions

CLaRa retrieves by **planning utility** — not semantic similarity. When Navigator has executed "create a GitHub repository" 50 times, those traces compress into a CLaRa artifact encoding which decisions were actually useful for planning. Not just what happened — what worked.

```
Raw traces (50 executions × ~500 tokens each = 25,000 tokens)
    ↓ CLaRa compression
Planning artifact (~200–1,500 tokens) — 16x–128x compression
    ↓
Retrieval at inference time — no separate embedding step
```

**Why CLaRa beats cosine similarity:**
Standard RAG finds traces that *sound* similar to the query. CLaRa finds traces that *helped plan* similar tasks. For agentic systems, that distinction matters.

### PageIndex — Documentation Retrieval

**For:** SaaS help center docs, API references, onboarding guides

PageIndex builds a hierarchical tree index of a document corpus. Rather than chunking and embedding:

```
Help center (hierarchical structure preserved)
    ↓ PageIndex
Tree index: root → sections → subsections → content
    ↓
LLM navigates the tree — "go to Advanced Settings → Permissions"
    ↓
Retrieves exact section without semantic similarity approximation
```

**Why PageIndex beats chunk-based RAG:**
A 512-token chunk of "Jira project permissions" loses the information that it's a subsection of "Advanced Settings" which applies only to company-managed projects. The tree preserves that context. The LLM reasons about document structure, not vector proximity.

**Accuracy on FinanceBench:** 98.7% vs ~50% for standard vector RAG.

### Convex Built-in Vector Search

Used for cold-start candidate retrieval on new tools — before CLaRa has enough trace data to build useful planning artifacts. As usage grows, CLaRa takes over. Convex handles this so there's no separate vector DB to operate.

---

## Layer 3: Reasoning

**Package:** `packages/planner`
**Model:** Claude Sonnet (Anthropic API)

Takes three inputs, produces one output:

**Inputs:**
1. `UIState` from Layer 1 — what the UI looks like right now
2. Retrieved context from Layer 2 — relevant past traces + doc sections
3. User intent — natural language instruction

**Output: WorkflowPlan**

```typescript
interface WorkflowPlan {
  intent: string
  steps: ExecutionStep[]
  confidence: number           // 0.0–1.0, overall plan confidence
  recoveryPaths: RecoveryPath[]
  stoppingConditions: string[] // conditions under which the agent should stop and ask
}

interface ExecutionStep {
  description: string          // human-readable, shown in guided mode
  targetElement?: ElementSelector
  action: 'click' | 'type' | 'select' | 'navigate' | 'wait' | 'verify'
  value?: string               // text to type, option to select, etc.
  precondition?: string        // must be true before executing this step
  successCondition?: string    // must be true after executing to proceed
  confidence: number           // 0.0–1.0, per-step confidence
}

interface RecoveryPath {
  trigger: string              // what failure condition triggers this path
  steps: ExecutionStep[]
}
```

**Why Claude Sonnet:** Tool-use native (structured output without prompting tricks), multimodal when screenshots are needed, strong at following multi-constraint instructions. The model is replaceable — the interface is the `WorkflowPlan` schema.

---

## Layer 4: Execution

**Package:** `packages/executor`

Follows the plan from Layer 3 step-by-step. Verifies each step. Escalates to recovery on failure.

### Element Targeting Priority

```
1. Exact text match         "Submit" button text
2. aria-label               aria-label="Submit form"
3. ARIA role                role="button"
4. WebMCP metadata          data-mcp-action="submit"
5. Position (last resort)   bounding box coordinates
```

CSS selectors intentionally avoided as primary targeting — they break on UI updates. Text and ARIA attributes are stable across redesigns.

### Confidence Gating

| Step confidence | Mode | Behavior |
|---|---|---|
| ≥ 0.9 | Auto-execute (paid tier) | Executes without pause |
| 0.7–0.9 | Guided (free tier) | Shows step, waits for user confirmation |
| < 0.7 | Pause | Requests clarification before proceeding |

### Observe → Verify Loop

After each action:
1. Read new UIState (call Layer 1)
2. Check step's `successCondition`
3. If met: proceed to next step
4. If not met after 3 checks: escalate to Recovery

### Recovery

The Planner provides `recoveryPaths` per step. Executor invokes the relevant path based on failure type:

| Failure | Recovery action |
|---|---|
| Element not found | Refresh UIState, retry |
| Page navigated unexpectedly | Navigate back, retry from last checkpoint |
| Success condition not met in 3 checks | Invoke recovery steps from plan |
| Recovery fails | Stop, report to user with full trace |

`max_retries` guard prevents infinite loops.

---

## Layer 5: Memory

**Schema in:** `packages/shared`
**Storage:** Convex

Every execution generates a `DecisionTrace`:

```typescript
interface DecisionTrace {
  sessionId: string
  toolId: string             // which SaaS tool (e.g., "jira", "github", "notion")
  intent: string             // original natural language instruction
  steps: ExecutedStep[]
  outcome: 'success' | 'partial' | 'failed'
  failurePoint?: number      // step index where failure occurred
  durationMs: number
  timestamp: number
}

interface ExecutedStep {
  stepIndex: number
  description: string
  action: string
  targetElement?: string
  value?: string
  preconditionMet: boolean
  successConditionMet: boolean
  retryCount: number
  durationMs: number
}
```

Traces stored in Convex → periodically compressed by CLaRa into planning-utility artifacts → stored back in Convex.

**The flywheel:**
```
More executions → richer CLaRa artifacts → better planning → more successful executions → more executions
```

Cross-tool patterns emerge over time. "Export to CSV" follows similar patterns in Notion, Airtable, and Salesforce. Navigator learns this from traces without being explicitly told.

---

## Convex as Sole Backend

| Need | Traditional stack | Navigator |
|---|---|---|
| Relational data | Postgres | Convex DB |
| Caching, pub/sub | Redis | Convex reactive queries |
| Graph relationships | Neo4j | Convex document references |
| Job queue | BullMQ / SQS | Convex scheduled functions |
| Vector search | Pinecone | Convex built-in vector |
| Real-time sync | Custom WebSocket infra | Convex subscriptions |

One SDK. One deployment. One billing line. No data sync bugs between services.

The real-time sync matters: when a user executes a workflow via the extension, the trace writes to Convex and the B2B dashboard updates live — no WebSocket service, no Redis pub/sub, no second write.

---

## Full Data Flow Example

```
User says: "Create a project named Navigator in Jira"
    │
    ▼
Extension captures current UIState (Layer 1: ScreenSense)
→ { url: "jira.atlassian.net/...", elements: [...], pageType: "dashboard" }
    │
    ▼
Knowledge layer retrieves (Layer 2):
  - CLaRa: past traces for "create project" on Jira tool
  - PageIndex: "Create a new project" section from Jira help center
    │
    ▼
Claude Sonnet plans (Layer 3):
  Step 1: Click "Projects" nav item (confidence: 0.95)
  Step 2: Click "Create project" button (confidence: 0.93)
  Step 3: Select "Scrum" project type (confidence: 0.88)
  Step 4: Type "Navigator" in project name field (confidence: 0.99)
  Step 5: Click "Create" (confidence: 0.97)
    │
    ▼
Executor runs (Layer 4):
  Step 1 → target: text="Projects" → click → verify: "Create project" button visible → ✅
  Step 2 → target: text="Create project" → click → verify: template modal appeared → ✅
  Step 3 → target: text="Scrum" → click → verify: name field appeared → ✅
  Step 4 → target: input[name=projectName] → type "Navigator" → verify: field value="Navigator" → ✅
  Step 5 → target: text="Create" → click → verify: project dashboard loaded → ✅
    │
    ▼
DecisionTrace written to Convex (Layer 5)
→ Queued for CLaRa compression
→ Dashboard updates live
    │
    ▼
User sees: "Project 'Navigator' created in Jira. 5 steps, 8.3 seconds."
```

---

## Key Design Constraints

1. **Under 300 lines per file.** Split if larger.
2. **Named exports only.** No default exports anywhere in the monorepo.
3. **Zod at package boundaries.** Runtime validation where modules interface — network calls, external model outputs, Convex reads.
4. **Preact in the extension only.** Dashboard can use React.
5. **Convex for everything backend.** No secondary data stores without a strong reason.
6. **Vitest for all tests.** Every public function, every package.
7. **esbuild/tsup for bundling.** Speed over configurability.
