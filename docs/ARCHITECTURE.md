# X "For You" Feed Algorithm - Complete Architecture Guide

> A comprehensive explanation of the open-source X recommendation algorithm at three levels of depth.

**Navigation:** [Home](index.md) · **Architecture** · [Example Scenario](EXAMPLE_SCENARIO.md) · [Diagrams](#diagrams-index)

---

## Table of Contents

- [Beginner Level](#beginner-level-what-does-this-do)
- [Mid Level](#mid-level-how-does-the-pipeline-work)
- [Advanced Level](#advanced-level-how-does-the-ml-model-work)
- [Diagrams Index](#diagrams-index)

---

## Beginner Level: "What does this do?"

When you open X and see the "For You" tab, this code decides which posts to show you and in what order.

### The Simple Version

1. **Gather posts** — Collect posts from two places:
   - **People you follow** (called "in-network") via a fast service called **Thunder**
   - **People you don't follow** (called "out-of-network") via an AI system called **Phoenix** that finds posts similar to things you've liked before

2. **Clean them up** — Remove posts you've already seen, posts from people you've blocked/muted, your own posts, duplicates, etc.

3. **Score each post** — An AI model (based on the **Grok** transformer) predicts: *"How likely is this user to like / reply / repost / click this post?"*

4. **Rank and return** — Posts with the highest combined scores appear at the top of your feed

### The 4 Main Folders

| Folder | What it does |
|--------|-------------|
| `home-mixer/` | The "brain" — orchestrates the whole pipeline |
| `thunder/` | Fast in-memory store of recent posts from everyone |
| `phoenix/` | The AI/ML models that predict engagement |
| `candidate-pipeline/` | Reusable framework for building recommendation pipelines |

### Analogy

Think of it like a restaurant:
- **Thunder** is the kitchen's pantry (posts from people you follow, always fresh and ready)
- **Phoenix Retrieval** is the chef going to the market to find interesting new ingredients (posts from people you don't follow)
- **Home Mixer** is the head chef who decides the menu — combines ingredients, removes anything stale or allergenic, tastes everything, and plates the top dishes in order of deliciousness (score)

---

## Mid Level: "How does the pipeline work?"

The system is built in **Rust** (services) + **Python/JAX** (ML models) and communicates via **gRPC**.

### System Overview

```
X Mobile App
      │
      │ gRPC: GetScoredPosts(viewer_id, seen_ids, ...)
      ▼
┌─────────────┐     ┌──────────┐     ┌─────────────────┐
│ Home Mixer  │────▶│ Thunder  │     │ Phoenix         │
│ (Rust gRPC) │────▶│ (InMem)  │     │ (Python/JAX ML) │
│             │────▶│          │     │                 │
│ 8-stage     │     └──────────┘     │ • Retrieval     │
│ pipeline    │────────────────────▶ │ • Ranking       │
└─────────────┘                      └─────────────────┘
      │
      │ Also calls: TES, Gizmoduck, Strato, SocialGraph,
      │             Visibility Filtering, UAS Fetcher
      ▼
ScoredPostsResponse (ranked posts)
```

### The 8-Stage Pipeline

Every time you pull-to-refresh, this pipeline executes end-to-end:

#### Stage 1 — Query Hydration (parallel)

Before fetching any posts, the system enriches the incoming request with user context:

- **`UserActionSeqQueryHydrator`** — Fetches your recent engagement history (what you liked, replied to, shared, etc.) to feed into the ML model
- **`UserFeaturesQueryHydrator`** — Fetches your following list, blocked/muted users, muted keywords from the Strato key-value store

Both run in parallel for speed.

**Source files:**
- `home-mixer/query_hydrators/user_action_seq_query_hydrator.rs`
- `home-mixer/query_hydrators/user_features_query_hydrator.rs`

#### Stage 2 — Candidate Sources (parallel)

Two sources run in parallel to gather candidate posts:

- **`ThunderSource`** — Calls the Thunder gRPC service to get recent posts from people you follow. Thunder holds **all recent posts in memory** indexed by author, so this is sub-millisecond
- **`PhoenixSource`** — Calls Phoenix Retrieval to find relevant posts from the global corpus using your engagement history embedding. **Disabled when `in_network_only=true`**

**Source files:**
- `home-mixer/sources/thunder_source.rs`
- `home-mixer/sources/phoenix_source.rs`

#### Stage 3 — Candidate Hydration (parallel)

Enriches each candidate with additional metadata. Five hydrators run **in parallel**:

| Hydrator | What it fetches |
|----------|----------------|
| `InNetworkCandidateHydrator` | Marks posts as in-network if author is in following list |
| `CoreDataCandidateHydrator` | Post text, reply info, retweet info from TES |
| `VideoDurationCandidateHydrator` | Video length for video posts from TES |
| `SubscriptionHydrator` | Whether post is behind a subscription paywall |
| `GizmoduckCandidateHydrator` | Author screen name, follower count |

**Source files:** `home-mixer/candidate_hydrators/*.rs`

#### Stage 4 — Pre-Scoring Filters (sequential)

10 filters run sequentially, each removing candidates that shouldn't be shown:

| Order | Filter | Purpose |
|-------|--------|---------|
| 1 | `DropDuplicatesFilter` | Remove duplicate post IDs |
| 2 | `CoreDataHydrationFilter` | Remove posts that failed to hydrate |
| 3 | `AgeFilter` | Remove posts older than threshold |
| 4 | `SelfTweetFilter` | Remove user's own posts |
| 5 | `RetweetDeduplicationFilter` | Dedupe retweets of same content |
| 6 | `IneligibleSubscriptionFilter` | Remove paywalled content user can't access |
| 7 | `PreviouslySeenPostsFilter` | Remove posts user already saw |
| 8 | `PreviouslyServedPostsFilter` | Remove posts already served in session (**only enabled when `is_bottom_request = true`**) |
| 9 | `MutedKeywordFilter` | Remove posts containing user's muted keywords |
| 10 | `AuthorSocialgraphFilter` | Remove posts from blocked/muted authors |

**Source files:** `home-mixer/filters/*.rs`

#### Stage 5 — Scoring (sequential)

Four scorers run in order, each building on the previous:

1. **`PhoenixScorer`** — Calls Phoenix Prediction model to get engagement probabilities for 19 actions (like, reply, repost, click, share, dwell, block, mute, report, etc.)

2. **`WeightedScorer`** — Combines all 19 predicted probabilities into a single score:
   ```
   score = Σ (weight_i × P(action_i))
   ```
   Positive actions (like, share) have positive weights. Negative actions (block, mute, report) have negative weights, pushing down content the user would likely dislike.

3. **`AuthorDiversityScorer`** — Penalizes repeated authors to ensure feed diversity. If an author appears multiple times, each subsequent post gets a decayed score.

4. **`OONScorer`** — Multiplies out-of-network post scores by a configurable factor to control the in-network vs. OON balance.

**Source files:** `home-mixer/scorers/*.rs`

#### Stage 6 — Selection

`TopKScoreSelector` sorts all remaining candidates by final score (descending) and picks the top K.

**Source file:** `home-mixer/selectors/top_k_score_selector.rs`

#### Stage 7 — Post-Selection Processing

After selection, a final round of hydration and filtering:

1. **`VFCandidateHydrator`** — Calls Visibility Filtering service to check content safety. Uses different safety levels: **`TimelineHome`** for in-network posts and **`TimelineHomeRecommendations`** (stricter) for out-of-network posts. Both are evaluated in parallel.
2. **`VFFilter`** — Drops posts that are deleted, spam, violent, etc.
3. **`DedupConversationFilter`** — Deduplicates branches of same conversation thread
4. **Truncation** — Results are truncated to `RESULT_SIZE` (configurable) before being returned

#### Stage 8 — Side Effects (async, non-blocking)

`CacheRequestInfoSideEffect` asynchronously caches request info to Strato so future requests can avoid serving the same posts. **Only enabled in production** (`APP_ENV == "prod"`) and **only for mixed-network requests** (`in_network_only == false`). Side effects are spawned via `tokio::spawn` and never block the response.

### Key Data Structures

**`ScoredPostsQuery`** (the request going through the pipeline):
```
user_id, client_app_id, country_code, language_code,
seen_ids, served_ids, in_network_only, is_bottom_request,
bloom_filter_entries, user_action_sequence, user_features,
request_id                       // unique per request, used for tracing
```

**`PostCandidate`** (each post being evaluated):
```
tweet_id, author_id, tweet_text,
in_reply_to_tweet_id, retweeted_tweet_id, retweeted_user_id,
phoenix_scores (19 engagement predictions),
prediction_request_id, last_scored_at_ms,   // ML serving tracing
weighted_score, score (final),
in_network flag, served_type,
ancestors (conversation chain IDs),
video_duration_ms, author_followers_count,
author_screen_name, retweeted_screen_name,
visibility_reason, subscription_author_id
```

**`PhoenixScores`** (19 ML predictions per post):
```
favorite_score, reply_score, retweet_score (called repost_score in Python),
photo_expand_score, click_score, profile_click_score,
vqv_score, share_score, share_via_dm_score,
share_via_copy_link_score, dwell_score, quote_score,
quoted_click_score, follow_author_score,
not_interested_score, block_author_score,
mute_author_score, report_score, dwell_time
```

---

## Advanced Level: "How does the ML model work?"

### Phoenix Retrieval — Two-Tower Architecture

**File:** `phoenix/recsys_retrieval_model.py`

The retrieval model uses a **two-tower architecture** for efficient candidate retrieval from millions of posts:

#### User Tower (Transformer-based)

Encodes user features and engagement history into a single embedding vector:

1. **Input assembly:**
   - User hash embeddings: `[B, num_user_hashes, D]` → projected to `[B, 1, D]`
   - History items: post hashes + author hashes + action embeddings + product surface embeddings → combined via `block_history_reduce` into `[B, S, D]`
   - Concatenated sequence: `[user_emb | history₁ | history₂ | ... | historyS]`

2. **Hash-based embeddings:** Multiple hash functions map entity IDs to embedding table slots. For example, with `num_item_hashes=2`, a tweet ID is hashed twice into different embedding tables, then the two embeddings are concatenated and projected down. This reduces hash collisions while keeping tables bounded.

3. **Transformer encoding:** The concatenated sequence passes through the Grok transformer (causal attention). The transformer output is **mean-pooled** over valid positions.

4. **L2 normalization:** The pooled representation is L2-normalized to lie on the unit sphere.

#### Candidate Tower (MLP)

A lightweight two-layer MLP:
```
concat(post_hashes, author_hashes) → Linear → SiLU → Linear → L2-normalize
```

Much faster than the user tower since it runs over the entire corpus offline.

#### Retrieval

Dot-product similarity on normalized vectors:
```python
scores = user_representation @ corpus_embeddings.T   # [B, N]
top_k_indices, top_k_scores = jax.lax.top_k(scores, k)
```

This is equivalent to cosine similarity since both vectors are unit-normalized.

### Phoenix Ranking — Transformer with Candidate Isolation

**File:** `phoenix/recsys_model.py`

The ranking model uses the full Grok transformer to predict engagement probabilities.

#### Input Sequence

```
[User] [History₁] [History₂] ... [HistoryS] [Cand₁] [Cand₂] ... [CandC]
                                              ↑
                                   candidate_start_offset
```

Three types of embeddings are combined:
- **User embedding:** Hash-based, projected from `num_user_hashes × D` to `D`
- **History embeddings:** post hashes + author hashes + action embeddings + product surface → projected to `D`
- **Candidate embeddings:** post hashes + author hashes + product surface → projected to `D`

#### The Candidate Isolation Attention Mask

**File:** `phoenix/grok.py`, function `make_recsys_attn_mask`

This is the core architectural innovation. The attention mask ensures:

- **User + History tokens (positions 0 to candidate_start_offset-1):** Standard causal attention — each position can attend to all earlier positions
- **Candidate tokens (positions candidate_start_offset onwards):** Can attend to **user + history** and **themselves only** — **cannot attend to other candidates**

```python
# Start with causal mask
causal_mask = jnp.tril(jnp.ones((1, 1, seq_len, seq_len)))

# Zero out candidate-to-candidate block
attn_mask = causal_mask.at[:, :, candidate_start_offset:, candidate_start_offset:].set(0)

# Re-enable self-attention for each candidate
candidate_indices = jnp.arange(candidate_start_offset, seq_len)
attn_mask = attn_mask.at[:, :, candidate_indices, candidate_indices].set(1)
```

**Why this matters:** P(like | post_i) is independent of which other posts are in the batch. Scores are consistent and cacheable. You can score post A in batch [A, B, C] or batch [A, X, Y] and get the same result.

#### Transformer Architecture (Grok-derived)

- **Grouped Query Attention (GQA):** `num_q_heads > num_kv_heads` — multiple query heads share the same key/value heads for memory efficiency
- **Rotary Position Embeddings (RoPE):** Encodes relative position information via rotation matrices applied to Q and K
- **RMSNorm** (not LayerNorm): More efficient, no mean centering
- **GELU-gated FFN:** `output = GELU(x @ W_gate) * (x @ W_up) @ W_down` — note: the Candidate Tower MLP uses SiLU, but the main transformer FFN uses GELU
- **Attention logit capping:** Before softmax, logits are capped via `max_attn_val × tanh(logits / max_attn_val)` with `max_attn_val = 30.0`, preventing extreme attention concentrations
- **`attn_output_multiplier`:** Scales attention logits before softmax

#### Output

```python
# Get candidate embeddings from transformer output
candidate_embeddings = out_embeddings[:, candidate_start_offset:, :]

# Project to action logits
logits = candidate_embeddings @ unembedding_matrix  # [B, C, num_actions]
```

Each logit is exp'd to get a probability. The 19 action probabilities are returned to the WeightedScorer.

### Scoring Math

#### Weighted Scorer

```
weighted_score = Σ (weight_i × P(action_i))
```

For actions like favorite, reply, retweet, share → positive weights
For actions like block, mute, report, not_interested → **negative weights**

Special handling:
- VQV (video quality view) weight only applies if `video_duration_ms > MIN_VIDEO_DURATION_MS`
- Scores are offset: negative combined scores are rescaled relative to `NEGATIVE_WEIGHTS_SUM / WEIGHTS_SUM`
- A `normalize_score` function is applied post-combination

#### Author Diversity Scorer

For each author, tracks occurrence count `n`. The multiplier decays exponentially:

```
multiplier(n) = (1 - floor) × decay^n + floor
```

So the 1st post from an author gets multiplier ≈ 1.0, the 2nd gets `decay`, the 3rd gets `decay²`, etc. The `floor` parameter prevents the multiplier from reaching zero.

Candidates are processed in **descending score order** so the highest-scored post from each author gets the least penalty.

#### OON Scorer

Simple: if a post is out-of-network (`in_network == false`), multiply its score by `OON_WEIGHT_FACTOR`.

### Thunder — In-Memory Real-Time Ingestion

**Files:** `thunder/`

#### Architecture

Thunder is a Rust service that:
1. Consumes tweet create/delete events from **Kafka** via a dual-pipeline:
   - **v1 (Thrift feeder mode):** Deserializes Thrift `TweetEvent` messages, can optionally forward to a Kafka producer
   - **v2 (Protobuf serving mode):** Deserializes Protobuf `InNetworkEvent` messages, writes directly to PostStore with a semaphore limiting concurrent partition updates to 3
   - In serving mode, only v2 runs. In non-serving (feeder) mode, v1 runs.
2. Stores posts in a **DashMap** (lock-free concurrent hashmap)
3. Serves "in-network" queries via gRPC

#### Data Model

```
PostStore:
  ├── posts: DashMap<post_id, LightPost>              // Full post data
  ├── original_posts_by_user: DashMap<author_id, VecDeque<TinyPost>>   // Non-reply, non-retweet
  ├── secondary_posts_by_user: DashMap<author_id, VecDeque<TinyPost>>  // Replies + retweets  
  ├── video_posts_by_user: DashMap<author_id, VecDeque<TinyPost>>      // Video posts
  └── deleted_posts: DashMap<post_id, bool>            // Deletion tracking

TinyPost = { post_id: i64, created_at: i64 }   // Minimal reference for timeline indexes
```

The separation of `TinyPost` references from full `LightPost` data minimizes memory for timeline indexes.

#### Kafka Processing

- Partitions distributed across N threads (configurable `kafka_num_threads`)
- Each thread owns a subset of partitions
- Batch processing with deserialization, filtering (retention window), and insert
- Partition lag monitoring for observability

#### Concurrency Control

- `Semaphore` limits concurrent gRPC requests
- If at capacity, returns `RESOURCE_EXHAUSTED` immediately (no queuing)
- `InFlightGuard` tracks active requests via RAII pattern

#### Auto-Trim

- Runs every 2 minutes
- Removes posts older than `post_retention_seconds`
- Ensures bounded memory usage

### The Candidate Pipeline Framework

**Files:** `candidate-pipeline/`

A generic, reusable framework parameterized by `Q` (query type) and `C` (candidate type):

```rust
trait CandidatePipeline<Q, C> {
    fn query_hydrators(&self) -> &[Box<dyn QueryHydrator<Q>>];
    fn sources(&self) -> &[Box<dyn Source<Q, C>>];
    fn hydrators(&self) -> &[Box<dyn Hydrator<Q, C>>];
    fn filters(&self) -> &[Box<dyn Filter<Q, C>>];
    fn scorers(&self) -> &[Box<dyn Scorer<Q, C>>];
    fn selector(&self) -> &dyn Selector<Q, C>;
    fn post_selection_hydrators(&self) -> &[Box<dyn Hydrator<Q, C>>];
    fn post_selection_filters(&self) -> &[Box<dyn Filter<Q, C>>];
    fn side_effects(&self) -> Arc<Vec<Box<dyn SideEffect<Q, C>>>>;
    
    async fn execute(&self, query: Q) -> PipelineResult<Q, C>;
}
```

Key design patterns:
- **Sources** and **hydrators** run in **parallel** via `futures::join_all`
- **Filters** and **scorers** run **sequentially** (order matters for both)
- Every component has `enable(query) -> bool` for conditional execution
- Hydrators **must** return same-length vectors — they cannot drop candidates (use a filter for that)
- `update()` merges only the fields a hydrator/scorer is responsible for
- Side effects fire asynchronously after the pipeline result is ready

### Key Design Decisions

1. **Zero hand-engineered features:** The transformer learns everything from raw engagement sequences. No TF-IDF, no topic modeling, no manual feature engineering. This dramatically simplifies the data pipeline.

2. **Candidate isolation in attention:** Scores are batch-independent. This allows caching scored posts and mixing them with freshly-scored ones, reducing ML serving cost.

3. **Hash-based embeddings:** Multiple hash functions per entity (user, post, author) with learned projection matrices. Bounded table size with reduced collision rate.

4. **Multi-action prediction:** 19 separate probabilities instead of one "relevance" score. Operators can tune the feed character by adjusting weights (e.g., increase reply weight → more conversational feed).

5. **Author diversity as a scoring penalty** (not a post-hoc shuffle): Exponential decay ensures organic interleaving rather than artificial alternation, and the penalty integrates naturally with the scoring framework.

6. **Composable pipeline architecture:** The `candidate-pipeline` crate separates pipeline execution/monitoring from business logic. Adding a new filter is as simple as implementing the `Filter<Q, C>` trait and adding it to the pipeline config.

---

## Diagrams Index

All architectural diagrams are in the `docs/diagrams/` folder:

| Diagram | File | Description |
|---------|------|-------------|
| C4 System Context | [c4-context.md](diagrams/c4-context.md) | High-level system boundaries and external dependencies |
| Pipeline Component Flow | [pipeline-component-flow.md](diagrams/pipeline-component-flow.md) | The 8-stage Home Mixer pipeline with all components |
| Request Sequence | [sequence-diagram.md](diagrams/sequence-diagram.md) | Full request/response flow with all service calls |
| Phoenix ML Architecture | [phoenix-ml-architecture.md](diagrams/phoenix-ml-architecture.md) | Two-tower retrieval + transformer ranking |
| Thunder Architecture | [thunder-architecture.md](diagrams/thunder-architecture.md) | Real-time Kafka ingestion and in-memory store |

All diagrams use Mermaid syntax and can be rendered in GitHub, VS Code (with Mermaid extension), or any Mermaid-compatible viewer.

---

**Navigation:** [Home](index.md) · **Architecture** · [Example Scenario](EXAMPLE_SCENARIO.md)
