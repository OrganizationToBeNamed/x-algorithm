# Thunder Service - Architecture

**Navigation:** [Home](../index.md) · [Architecture](../ARCHITECTURE.md) · [Example Scenario](../EXAMPLE_SCENARIO.md) · [C4 Context](c4-context.md) · [Pipeline Flow](pipeline-component-flow.md) · [Sequence](sequence-diagram.md) · [Phoenix ML](phoenix-ml-architecture.md)

```mermaid
graph TB
    subgraph ThunderArch["THUNDER - Real-Time Post Ingestion"]
        direction TB
        K["Kafka<br/>Tweet Events Stream"]
        
        subgraph Listeners["Kafka Consumers<br/>(multi-threaded)"]
            L1["Thread 1<br/>Partitions 0-N"]
            L2["Thread 2<br/>Partitions N-M"]
            L3["Thread K<br/>Partitions ..."]
        end
        
        K --> L1
        K --> L2
        K --> L3
        
        subgraph Processing["Event Processing"]
            CE["TweetCreateEvent<br/>→ Extract LightPost<br/>→ Classify: original / reply / video"]
            DE["TweetDeleteEvent<br/>→ Mark as deleted<br/>→ Remove from indexes"]
        end
        
        L1 --> CE
        L1 --> DE
        L2 --> CE
        L2 --> DE
        L3 --> CE
        L3 --> DE
        
        subgraph PostStore["In-Memory PostStore (DashMap)"]
            PI["posts<br/>─────<br/>post_id → LightPost"]
            OP["original_posts_by_user<br/>─────<br/>author_id → Deque of TinyPost"]
            SP["secondary_posts_by_user<br/>─────<br/>author_id → Deque of TinyPost<br/>(replies + reposts)"]
            VP["video_posts_by_user<br/>─────<br/>author_id → Deque of TinyPost"]
            DP["deleted_posts<br/>─────<br/>post_id → bool"]
        end
        
        CE --> PI
        CE --> OP
        CE --> SP
        CE --> VP
        DE --> DP
        
        GRPC["gRPC Service<br/>GetInNetworkPosts<br/>───────<br/>Input: user_id + following_ids<br/>Output: ranked LightPosts"]
        
        PostStore --> GRPC
        
        TRIM["Auto-Trim Task<br/>(every 2 min)"]
        TRIM --> PostStore
    end

    style K fill:#fff3e0,stroke:#ef6c00
    style PI fill:#e3f2fd,stroke:#1565c0
    style OP fill:#e8f5e9,stroke:#2e7d32
    style SP fill:#e8f5e9,stroke:#2e7d32
    style VP fill:#e8f5e9,stroke:#2e7d32
    style GRPC fill:#f3e5f5,stroke:#6a1b9a
```
