# Sequence Diagram - For You Feed Request Flow

**Navigation:** [Home](../index.md) · [Architecture](../ARCHITECTURE.md) · [Example Scenario](../EXAMPLE_SCENARIO.md) · [C4 Context](c4-context.md) · [Pipeline Flow](pipeline-component-flow.md) · [Phoenix ML](phoenix-ml-architecture.md) · [Thunder](thunder-architecture.md)

```mermaid
sequenceDiagram
    autonumber
    participant Client as X Mobile App
    participant HM as Home Mixer
    participant UAS as UAS Fetcher
    participant Strato
    participant Thunder
    participant PhxR as Phoenix Retrieval
    participant TES as Tweet Entity Service
    participant Gizmo as Gizmoduck
    participant PhxP as Phoenix Prediction
    participant VF as Visibility Filter

    Client->>+HM: GetScoredPosts(viewer_id, seen_ids, ...)
    
    Note over HM: Stage 1: Query Hydration (parallel)
    par Hydrate user context
        HM->>+UAS: Fetch user action sequence
        UAS-->>-HM: engagement history
    and
        HM->>+Strato: Fetch user features
        Strato-->>-HM: following, muted, blocked lists
    end

    Note over HM: Stage 2: Candidate Sources (parallel)
    par Fetch in-network
        HM->>+Thunder: GetInNetworkPosts(following_ids)
        Thunder-->>-HM: ~500 recent posts from followed users
    and Fetch out-of-network
        HM->>+PhxR: Retrieve(user_embedding, corpus)
        PhxR-->>-HM: ~500 ML-discovered posts
    end

    Note over HM: Stage 3: Hydration (parallel)
    par Enrich candidates
        HM->>+TES: GetTweetData(tweet_ids)
        TES-->>-HM: text, media, reply info
    and
        HM->>+Gizmo: GetUsers(author_ids)
        Gizmo-->>-HM: screen names, follower counts
    end

    Note over HM: Stage 4: Pre-scoring Filters
    HM->>HM: Remove duplicates, old, self, blocked, muted, seen

    Note over HM: Stage 5: Scoring
    HM->>+PhxP: Predict(user_history, candidates)
    PhxP-->>-HM: P(like), P(reply), P(repost), ...per candidate
    HM->>HM: WeightedScorer: score = Σ(wi × Pi)
    HM->>HM: AuthorDiversityScorer: penalize repeated authors
    HM->>HM: OONScorer: adjust out-of-network weight

    Note over HM: Stage 6: Selection
    HM->>HM: TopKScoreSelector: sort & pick top K

    Note over HM: Stage 7: Post-selection
    HM->>+VF: Check visibility(selected_posts)
    VF-->>-HM: safety verdicts
    HM->>HM: VFFilter + DedupConversationFilter

    Note over HM: Stage 8: Side Effects (async)
    HM-->>Strato: Cache request info (non-blocking)

    HM-->>-Client: ScoredPostsResponse(ranked posts)
```
