---
title: X For You Feed Algorithm
---

# X For You Feed Algorithm

Welcome to the documentation for the X "For You" feed recommendation system.

This system combines in-network content (from accounts you follow) with out-of-network content (discovered through ML-based retrieval) and ranks everything using a Grok-based transformer model.

---

## Documentation Map

| Document | Description |
|----------|-------------|
| [Architecture Guide](ARCHITECTURE.md) | Three-level deep dive — beginner, mid-level, and advanced explanations of the full system |
| [Example Scenario](EXAMPLE_SCENARIO.md) | End-to-end walkthrough of a real feed request with concrete data, plus 11 advanced deep dives |

## Diagrams

| Diagram | Description |
|---------|-------------|
| [C4 System Context](diagrams/c4-context.md) | High-level system boundaries and external dependencies |
| [Pipeline Component Flow](diagrams/pipeline-component-flow.md) | The 8-stage Home Mixer pipeline with all components |
| [Request Sequence](diagrams/sequence-diagram.md) | Full request/response flow with all service calls |
| [Phoenix ML Architecture](diagrams/phoenix-ml-architecture.md) | Two-tower retrieval + transformer ranking |
| [Thunder Architecture](diagrams/thunder-architecture.md) | Real-time Kafka ingestion and in-memory store |

---

## The 4 Main Components

| Folder | What it does |
|--------|-------------|
| [`home-mixer/`](https://github.com/OrganizationToBeNamed/x-algorithm/tree/main/home-mixer) | The orchestration layer — assembles the ranked For You feed via an 8-stage pipeline |
| [`thunder/`](https://github.com/OrganizationToBeNamed/x-algorithm/tree/main/thunder) | In-memory real-time post store for in-network content, fed by Kafka |
| [`phoenix/`](https://github.com/OrganizationToBeNamed/x-algorithm/tree/main/phoenix) | Grok-based ML models for retrieval (two-tower) and ranking (transformer) |
| [`candidate-pipeline/`](https://github.com/OrganizationToBeNamed/x-algorithm/tree/main/x-algorithm/candidate-pipeline) | Reusable framework for building recommendation pipelines |

---

## Quick Start

New here? Start with the **[Architecture Guide](ARCHITECTURE.md)** for an overview, then explore the **[Example Scenario](EXAMPLE_SCENARIO.md)** to see the system in action with real data.

For a visual understanding, check the **[Pipeline Component Flow](diagrams/pipeline-component-flow.md)** diagram and the **[Request Sequence](diagrams/sequence-diagram.md)** diagram.
