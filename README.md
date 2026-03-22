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
- architecture, benchmarks, and technical decisions are documented in detail
- three public experiments demonstrate different layers of the system

This repository is the dedicated home for the flagship architecture and docs layer.

## Running Today

Three public experiments prove different layers of the thesis:

| Experiment | What it proves | Repo |
|---|---|---|
| [VisionGuide](https://github.com/hrshitkunwar-tech/VisionGuide) | Screenshot → Gemini → real-time guidance overlay | Perception layer |
| [ZoneGuide](https://github.com/hrshitkunwar-tech/zoneguide) | DOM recording → replay, zero tooling | Recording + replay |
| [job](https://github.com/hrshitkunwar-tech/job) | Scoring → tailoring → auto-apply | Applied execution |

## Backend Experiments

Three parallel approaches to the Navigator thesis. Same problem, different implementation surfaces.

| Experiment | Approach | Stack |
|---|---|---|
| **BestWay** (private) | Chrome MV3 · TypeScript monorepo · ScreenSense perception | TypeScript · Vitest · Zod · Chrome APIs |
| **navigator-backend** (private) | Convex serverless · CLaRa bench · AppIdeasLab | Convex · TypeScript |
| [**ZoneGuide**](https://github.com/hrshitkunwar-tech/zoneguide) | Zero tooling — raw browser APIs | vanilla JS · HTML · CSS · Chrome MV3 |

The question each experiment answers is slightly different. BestWay answers: what does the full production implementation look like? navigator-backend answers: what does the cloud-native backend look like? ZoneGuide answers: how far can you get with nothing?

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
4. Then the live experiments: [VisionGuide](https://github.com/hrshitkunwar-tech/VisionGuide) → [ZoneGuide](https://github.com/hrshitkunwar-tech/zoneguide) → [job](https://github.com/hrshitkunwar-tech/job)

## Why This Repo Matters

This repo is the clearest expression of the long-term thesis behind the wider project ecosystem: software interaction infrastructure, not just another AI wrapper.

## Navigator Lab

| Repo | Layer | What it does |
|---|---|---|
| [navigator](https://github.com/hrshitkunwar-tech/navigator) | Thesis | 5-layer architecture for AI execution on software interfaces |
| [VisionGuide](https://github.com/hrshitkunwar-tech/VisionGuide) | Perception | Screenshot → Gemini vision → real-time UI guidance |
| [zoneguide](https://github.com/hrshitkunwar-tech/zoneguide) | Recording | DOM interaction capture and replay, zero dependencies |
| [job](https://github.com/hrshitkunwar-tech/job) | Applied | CareerAgent: score → tailor → apply, local-first |
| [saas-atlas](https://github.com/hrshitkunwar-tech/saas-atlas) | Data | Searchable directory of 200+ SaaS tools |
