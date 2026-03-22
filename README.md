# Navigator

AI execution layer for software interfaces.

> Navigator is the thesis repo for **Navigator Lab**: a portfolio of products exploring UI perception, reasoning-state telemetry, autonomous execution, and culture-aware matching.

Navigator is a system for understanding live SaaS interfaces, retrieving the right procedural context, planning multi-step actions, and executing them with verification. The core belief is simple: AI should not stop at suggestions when the interface itself is observable and actionable.

## The Problem

Most software is still operated manually, even when the workflow is repetitive and structurally obvious.

Existing approaches fall short:

- digital adoption platforms require manual authoring and constant maintenance
- copilots explain what to do, but still leave the work to the user
- generic RAG systems retrieve semantically similar text, not execution-useful workflow context

Navigator is designed as a zero-config execution layer that works from the live UI outward.

## System Shape

Navigator is organized into five layers:

1. **Perception**: turn the live browser state into structured UI data
2. **Knowledge**: retrieve workflow traces and documentation context
3. **Reasoning**: convert intent + context into an executable plan
4. **Execution**: act against the interface and verify outcomes
5. **Memory**: compress successful traces into reusable planning artifacts

The goal is not “chat with software.” The goal is “help software get used correctly.”

## What Exists Today

The current public work proves the earliest part of the thesis:

- ScreenSense-style perception work exists and has been tested in isolation
- the MVP validated browser-native guidance, pattern matching, and orchestration
- architecture, benchmarks, and technical decisions are documented in detail

This repository is the dedicated home for the flagship architecture and docs layer.

For the current public demo path, see:

- [job](https://github.com/hrshitkunwar-tech/job) for CareerAgent, a manifesto-aware matcher with autonomous execution
- internal Navigator Lab demo surfaces for visual reasoning playback and operator control-plane views

For earlier implementation-heavy public work, see:

- [mvp](https://github.com/hrshitkunwar-tech/mvp) for the earlier browser-extension proof of concept
- internal Navigator Lab research repos for early ScreenSense and module work

## Backend Experiments

Three parallel approaches to the Navigator thesis. Same problem, different implementation surfaces.

| Experiment | Approach | Stack |
|---|---|---|
| **BestWay** (private) | Chrome MV3 · TypeScript monorepo · ScreenSense perception | TypeScript · Vitest · Zod · Chrome APIs |
| **navigator-backend** (private) | Convex serverless · CLaRa bench · AppIdeasLab | Convex · TypeScript |
| [**anti-gravity**](https://github.com/hrshitkunwar-tech/anti-gravity) | Zero tooling — raw browser APIs | vanilla JS · HTML · CSS · Chrome MV3 |

The question each experiment answers is slightly different. BestWay answers: what does the full production implementation look like? navigator-backend answers: what does the cloud-native backend look like? anti-gravity answers: how far can you get with nothing?

## Technical Direction

Navigator is intentionally vector-light, not vector-first.

- **CLaRa 7B** for workflow trace retrieval by planning utility
- **PageIndex** for documentation retrieval over hierarchical structure
- **Convex** for the narrow cold-start cases where vector search is still useful

That avoids a default pipeline of `embed -> vector DB -> cosine similarity -> rerank` for every retrieval problem.

## Repository Structure

```text
navigator/
├── docs/
│   ├── ARCHITECTURE.md
│   ├── TECHNICAL_DECISIONS.md
│   └── BENCHMARKS.md
├── implementation/     ← git submodule → BestWay (private)
└── README.md
```

`implementation/` points to the private BestWay monorepo — the running Navigator system (ScreenSense, Chrome MV3 extension, CLaRa retrieval layer, Convex backend). Open thesis, private code.

## Reading Order

1. [Architecture](./docs/ARCHITECTURE.md)
2. [Technical Decisions](./docs/TECHNICAL_DECISIONS.md)
3. [Benchmarks](./docs/BENCHMARKS.md)
4. Then the live demos in [job](https://github.com/hrshitkunwar-tech/job) → [outreach](https://github.com/hrshitkunwar-tech/outreach)

## Why This Repo Matters

This repo is the clearest expression of the long-term thesis behind the wider project ecosystem: software interaction infrastructure, not just another AI wrapper.
