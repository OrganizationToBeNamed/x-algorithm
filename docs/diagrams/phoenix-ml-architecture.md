# Phoenix ML Architecture - Retrieval & Ranking

**Navigation:** [Home](../index.md) · [Architecture](../ARCHITECTURE.md) · [Example Scenario](../EXAMPLE_SCENARIO.md) · [C4 Context](c4-context.md) · [Pipeline Flow](pipeline-component-flow.md) · [Sequence](sequence-diagram.md) · [Thunder](thunder-architecture.md)

```mermaid
graph LR
    subgraph Phoenix ML["PHOENIX ML ARCHITECTURE"]
        direction TB
        
        subgraph Retrieval["Two-Tower Retrieval Model"]
            direction LR
            UT["User Tower<br/>─────────<br/>Transformer encodes:<br/>• User embedding<br/>• History sequence<br/>• Action embeddings<br/>→ L2-normalized vector"]
            CT["Candidate Tower<br/>─────────<br/>MLP projects:<br/>• Post embedding<br/>• Author embedding<br/>→ L2-normalized vector"]
            SIM["Dot Product<br/>Similarity<br/>─────────<br/>top-K nearest<br/>neighbors"]
            UT --> SIM
            CT --> SIM
        end

        subgraph Ranking["Transformer Ranking Model"]
            direction TB
            INP["Input Sequence<br/>─────────<br/>[User] + [History₁...HistoryS] + [Cand₁...CandC]"]
            ATT["Grok Transformer<br/>─────────<br/>• Multi-head attention (GQA)<br/>• RoPE positional encoding<br/>• RMSNorm + GELU FFN<br/>• Candidate Isolation Mask:<br/>  candidates see user+history<br/>  but NOT each other"]
            OUT["Output Logits<br/>─────────<br/>Per candidate × Per action:<br/>P(like), P(reply), P(repost),<br/>P(click), P(share), P(dwell),<br/>P(block), P(mute), P(report)..."]
            INP --> ATT --> OUT
        end
    end

    style UT fill:#e3f2fd,stroke:#1565c0
    style CT fill:#fff3e0,stroke:#ef6c00
    style SIM fill:#e8f5e9,stroke:#2e7d32
    style INP fill:#f3e5f5,stroke:#6a1b9a
    style ATT fill:#f3e5f5,stroke:#6a1b9a
    style OUT fill:#e8f5e9,stroke:#2e7d32
```
