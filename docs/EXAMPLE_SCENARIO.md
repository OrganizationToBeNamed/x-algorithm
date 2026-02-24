# Real Example Scenario: A "For You" Feed Request

> This walks through exactly what happens when a real user opens their X app and sees the "For You" tab. Every stage is traced with concrete data.

**Navigation:** [Home](index.md) · [Architecture](ARCHITECTURE.md) · **Example Scenario** · [Pipeline Diagram](diagrams/pipeline-component-flow.md) · [Sequence Diagram](diagrams/sequence-diagram.md)

---

## Meet Our User: Priya (@priya_codes)

```
User ID:        900100200
Follows:        312 accounts (mix of tech, sports, friends)
Blocked:        2 accounts (IDs: 555000111, 555000222)
Muted:          1 account  (ID: 555000333)
Muted keywords: ["crypto giveaway", "drop your wallet"]
Country:        US
Language:       en
```

Priya opens the X app on her phone at **10:15am on a Monday**. The app makes a gRPC call:

```protobuf
GetScoredPosts {
    viewer_id:       900100200,
    client_app_id:   12345,
    country_code:    "US",
    language_code:   "en",
    seen_ids:        [1893000001, 1893000002, ...47 more...],   // 49 posts she already saw
    served_ids:      [1893000001, 1893000002, ...24 more...],   // 26 posts served in last session
    in_network_only: false,
    is_bottom_request: false,
    bloom_filter_entries: [<compressed bloom filter of ~500 recently seen IDs>],
}
```

---

## Stage 1: Query Hydration

Two hydrators run **in parallel**:

### UserActionSeqQueryHydrator
Fetches Priya's recent engagement history from the UAS service:

```
Priya's last 128 actions (most recent first):
─────────────────────────────────────────────────────────────
  Action          │ Tweet ID     │ Author ID   │ Surface
─────────────────────────────────────────────────────────────
  Liked (fav)     │ 1893000050   │ 300400500   │ home_timeline
  Replied         │ 1893000048   │ 200300400   │ home_timeline
  Retweeted       │ 1893000045   │ 300400500   │ home_timeline
  Clicked         │ 1893000040   │ 100200300   │ home_timeline
  Dwelled 15s     │ 1893000038   │ 400500600   │ home_timeline
  Liked (fav)     │ 1893000035   │ 100200300   │ search
  Video viewed    │ 1893000030   │ 500600700   │ home_timeline
  ... (121 more actions going back ~3 days) ...
```

This engagement history becomes the **input sequence** for the Phoenix transformer — it's how the model "knows" what Priya likes.

### UserFeaturesQueryHydrator
Fetches from Strato:

```
UserFeatures {
    followed_user_ids:    [100200300, 200300400, 300400500, ... 309 more],
    blocked_user_ids:     [555000111, 555000222],
    muted_user_ids:       [555000333],
    muted_keywords:       ["crypto giveaway", "drop your wallet"],
    subscribed_user_ids:  [100200300],
}
```

**Result:** The `ScoredPostsQuery` is now enriched with Priya's full context.

**Time elapsed: ~12ms** (both hydrators ran in parallel)

---

## Stage 2: Candidate Sources

Two sources run **in parallel**:

### ThunderSource (In-Network)

Calls Thunder gRPC with Priya's 312 followed user IDs:

```
GetInNetworkPosts {
    user_id:            900100200,
    following_user_ids: [100200300, 200300400, 300400500, ...],
    max_results:        500,
    algorithm:          "default",
}
```

Thunder looks up its in-memory PostStore — for each of the 312 followed accounts, it retrieves recent original posts and some replies/retweets. Returns **487 candidate posts**.

Example candidates from Thunder:
```
─────────────────────────────────────────────────────────────────────────
  tweet_id       │ author_id   │ @handle          │ type     │ age
─────────────────────────────────────────────────────────────────────────
  1893100001     │ 100200300   │ @techguru        │ original │ 2h
  1893100002     │ 100200300   │ @techguru        │ reply    │ 1h
  1893100003     │ 200300400   │ @sportsfan       │ original │ 4h
  1893100004     │ 200300400   │ @sportsfan       │ retweet  │ 3h
  1893100005     │ 300400500   │ @designeramy     │ original │ 30m
  1893100006     │ 300400500   │ @designeramy     │ original │ 6h
  1893100007     │ 400500600   │ @newsbot         │ original │ 1h
  1893100008     │ 555000333   │ @annoying_person │ original │ 2h   ← muted user!
  ... (479 more) ...
```

All arrive with `served_type: ForYouInNetwork`.

### PhoenixSource (Out-of-Network)

Calls Phoenix Retrieval with Priya's engagement history embedding:

```
Retrieve {
    user_id:              900100200,
    user_action_sequence: <128-action history>,
    max_results:          500,
}
```

Phoenix Retrieval internally:
1. Encodes Priya's history through the **User Tower** (Grok transformer) → gets a 128-dim L2-normalized user embedding
2. Computes dot-product similarity against **millions of candidate embeddings** (pre-computed by Candidate Tower)
3. Returns top-500 by similarity score

Example candidates from Phoenix:
```
─────────────────────────────────────────────────────────────────────────
  tweet_id       │ author_id   │ type     │ why retrieved
─────────────────────────────────────────────────────────────────────────
  1893200001     │ 600700800   │ original │ Similar to tech posts Priya likes
  1893200002     │ 700800900   │ original │ Similar to design content
  1893200003     │ 800900100   │ video    │ Sports highlight (video, 45s)
  1893200004     │ 900100200   │ original │ Priya's OWN post! (will be filtered)
  1893200005     │ 555000111   │ original │ From blocked user! (will be filtered)
  1893200006     │ 111222333   │ original │ Trending tech thread
  ... (494 more) ...
```

All arrive with `served_type: ForYouPhoenixRetrieval`.

**Combined total: 487 + 500 = 987 candidates**

**Time elapsed: ~45ms** (both sources ran in parallel; Thunder ~3ms, Phoenix ~42ms)

---

## Stage 3: Candidate Hydration

Five hydrators run **in parallel** to enrich all 987 candidates:

### InNetworkCandidateHydrator
Checks each candidate's `author_id` against Priya's followed list:
```
Thunder posts  → in_network: true    (487 posts)
Phoenix posts  → in_network: false   (500 posts, with exceptions if Phoenix 
                                       happened to retrieve someone Priya follows)
```

### CoreDataCandidateHydrator (via TES)
Batch-fetches core tweet data for all 987 IDs:
```
tweet_id: 1893100001 → text: "Just shipped v2.0 of our Rust framework! 🦀"
tweet_id: 1893100005 → text: "New design system drop — check the thread 🧵"
tweet_id: 1893200001 → text: "Here's why async Rust changed our production infra..."
tweet_id: 1893100099 → FAILED (tweet deleted during fetch) → No core data
```
3 posts fail hydration (will be caught by `CoreDataHydrationFilter`).

### VideoDurationCandidateHydrator (via TES)
```
tweet_id: 1893200003 → video_duration_ms: 45000  (45s sports highlight)
tweet_id: 1893100050 → video_duration_ms: 8000   (8s clip)
tweet_id: 1893100077 → video_duration_ms: 120000 (2min tutorial)
... (most posts have no video → None)
```

### SubscriptionHydrator (via TES)
```
tweet_id: 1893100090 → subscription_author_id: Some(444555666)
    (Paywalled post — Priya doesn't subscribe to 444555666)
tweet_id: 1893100001 → subscription_author_id: None (free post)
```

### GizmoduckCandidateHydrator (via Gizmoduck)
Fetches author profiles for all unique author IDs:
```
author_id: 100200300 → screen_name: "techguru",     followers: 45_200
author_id: 200300400 → screen_name: "sportsfan",    followers: 12_800
author_id: 300400500 → screen_name: "designeramy",  followers: 89_400
author_id: 600700800 → screen_name: "rustacean_dev",followers: 156_000
author_id: 555000111 → screen_name: "blocked_user", followers: 2_300
...
```

**Time elapsed: ~65ms** (all 5 hydrators run in parallel; TES calls dominate)

---

## Stage 4: Pre-Scoring Filters

10 filters run **sequentially**. Let's trace the candidate count through each:

```
Starting candidates: 987
```

### Filter 1: DropDuplicatesFilter
Phoenix and Thunder sometimes return the same tweet (e.g., if a followed user's post also ranks high in retrieval).
```
Removed: 12 duplicates
Remaining: 975
```

### Filter 2: CoreDataHydrationFilter
Removes posts where TES failed to return data:
```
Removed: 3 (deleted tweets, TES errors)
Remaining: 972
```

### Filter 3: AgeFilter
Removes posts older than the configured threshold (e.g., 48 hours):
```
Removed: 38 (old posts from less-active followed accounts)
Remaining: 934
```

### Filter 4: SelfTweetFilter
Checks `author_id != viewer_id (900100200)`:
```
Removed: 2 (Priya's own posts that Phoenix Retrieval found)
Remaining: 932
```

### Filter 5: RetweetDeduplicationFilter
If multiple people retweeted the same original tweet, keep only one:
```
Removed: 15 (e.g., tweet 1893050000 was retweeted by 3 people Priya follows)
Remaining: 917
```

### Filter 6: IneligibleSubscriptionFilter
Removes paywalled posts user can't access:
```
Removed: 4 (subscription content from non-subscribed authors)
Remaining: 913
```

### Filter 7: PreviouslySeenPostsFilter
Uses Priya's `seen_ids` + bloom filter to remove already-seen posts:
```
Removed: 31 (posts Priya scrolled past in earlier sessions)
Remaining: 882
```

### Filter 8: PreviouslyServedPostsFilter
Removes posts served in the current session:
```
Removed: 8 (posts from her last pull-to-refresh 20 min ago)
Remaining: 874
```

### Filter 9: MutedKeywordFilter
Tokenizes each post's text and matches against `["crypto giveaway", "drop your wallet"]`:
```
tweet_id: 1893200099 — text: "🚀 CRYPTO GIVEAWAY! Drop your wallet address..."
  → MATCH on "crypto giveaway" — REMOVED

Removed: 6 (spam/promo posts matching muted keywords)
Remaining: 868
```

### Filter 10: AuthorSocialgraphFilter
Checks `author_id` against blocked `[555000111, 555000222]` and muted `[555000333]`:
```
tweet_id: 1893200005 — author: 555000111 (blocked) — REMOVED
tweet_id: 1893100008 — author: 555000333 (muted)   — REMOVED

Removed: 5 (posts from blocked/muted authors)
Remaining: 863
```

**Summary of filtering:**
```
987 → 863 candidates (124 removed, ~12.6% filtered out)
```

**Time elapsed: ~2ms** (all filters are in-memory operations, no network calls)

---

## Stage 5: Scoring

Four scorers run **sequentially** on the 863 remaining candidates.

### Scorer 1: PhoenixScorer

Sends all 863 candidates + Priya's action history to the Phoenix Prediction model:

```
Predict {
    user_id:              900100200,
    user_action_sequence: <128-action history>,
    tweet_infos:          [863 candidates with tweet_id + author_id],
}
```

The Grok-based transformer processes the input sequence:
```
[Priya's embedding] [History₁] [History₂] ... [History₁₂₈] [Cand₁] [Cand₂] ... [Cand₈₆₃]
```

With candidate isolation masking, each candidate attends to Priya's context but NOT to other candidates.

Returns 19 probabilities per candidate. Let's follow 5 specific posts:

```
─────────────────────────────────────────────────────────────────────────────────────────────
Post                        │ P(fav)│P(reply)│P(RT) │P(click)│P(dwell)│P(share)│P(block)│P(report)
─────────────────────────────────────────────────────────────────────────────────────────────
A: @techguru "Rust v2.0"    │ 0.42  │ 0.15   │ 0.18 │ 0.55   │ 0.72   │ 0.08   │ 0.001  │ 0.0002
   (in-network, original)   │       │        │      │        │        │        │        │
                             │       │        │      │        │        │        │        │
B: @designeramy "Design 🧵" │ 0.38  │ 0.08   │ 0.22 │ 0.48   │ 0.65   │ 0.12   │ 0.001  │ 0.0001
   (in-network, original)   │       │        │      │        │        │        │        │
                             │       │        │      │        │        │        │        │
C: @rustacean_dev "Async.." │ 0.35  │ 0.12   │ 0.14 │ 0.60   │ 0.68   │ 0.06   │ 0.002  │ 0.0003
   (OUT-of-network)         │       │        │      │        │        │        │        │
                             │       │        │      │        │        │        │        │
D: @techguru "reply to..."  │ 0.18  │ 0.05   │ 0.04 │ 0.30   │ 0.40   │ 0.02   │ 0.001  │ 0.0001
   (in-network, reply)      │       │        │      │        │        │        │        │
                             │       │        │      │        │        │        │        │
E: @sportsfan "Game recap"  │ 0.12  │ 0.03   │ 0.05 │ 0.20   │ 0.35   │ 0.01   │ 0.003  │ 0.0005
   (in-network, original)   │       │        │      │        │        │        │        │
─────────────────────────────────────────────────────────────────────────────────────────────
```

**Time elapsed: ~55ms** (ML inference is the most expensive step)

### Scorer 2: WeightedScorer

Combines all 19 predicted probabilities using configured weights. Let's use example weights:

```
Positive weights (simplified):
  FAVORITE_WEIGHT        = 1.0
  REPLY_WEIGHT           = 27.0
  RETWEET_WEIGHT         = 1.0
  CLICK_WEIGHT           = 0.1
  PROFILE_CLICK_WEIGHT   = 1.0
  VQV_WEIGHT             = 0.3   (only if video > MIN_VIDEO_DURATION_MS)
  SHARE_WEIGHT           = 1.0
  SHARE_VIA_DM_WEIGHT    = 1.0
  SHARE_VIA_COPY_LINK    = 1.0
  DWELL_WEIGHT           = 0.2
  QUOTE_WEIGHT           = 1.0
  FOLLOW_AUTHOR_WEIGHT   = 10.0
  CONT_DWELL_TIME_WEIGHT = 0.05

Negative weights:
  NOT_INTERESTED_WEIGHT  = -74.0
  BLOCK_AUTHOR_WEIGHT    = -74.0
  MUTE_AUTHOR_WEIGHT     = -74.0
  REPORT_WEIGHT          = -200.0
```

**Calculation for Post A** (@techguru "Rust v2.0", no video):
```
weighted = (0.42 × 1.0)    fav
         + (0.15 × 27.0)   reply      = 4.05
         + (0.18 × 1.0)    RT
         + (0.55 × 0.1)    click      = 0.055
         + (0.72 × 0.2)    dwell      = 0.144
         + (0.08 × 1.0)    share
         + (0.001 × -74.0) block      = -0.074
         + (0.0002 × -200) report     = -0.04
         + ... other terms ...
         ≈ 5.62

After offset_score and normalize_score → weighted_score ≈ 5.85
```

**Calculation for Post E** (@sportsfan "Game recap"):
```
weighted = (0.12 × 1.0)    fav
         + (0.03 × 27.0)   reply      = 0.81
         + (0.05 × 1.0)    RT
         + (0.20 × 0.1)    click      = 0.02
         + (0.35 × 0.2)    dwell      = 0.07
         + (0.01 × 1.0)    share
         + (0.003 × -74.0) block      = -0.222
         + (0.0005 × -200) report     = -0.10
         + ... other terms ...
         ≈ 1.15

After offset_score and normalize_score → weighted_score ≈ 1.28
```

Notice how the **high reply weight (27×)** dramatically boosts posts that Priya is likely to reply to. The negative weights on block/report **push down** sketchy content.

**Results after WeightedScorer** (our 5 tracked posts):
```
Post A: @techguru "Rust v2.0"        → weighted_score: 5.85
Post B: @designeramy "Design 🧵"     → weighted_score: 5.12
Post C: @rustacean_dev "Async..."     → weighted_score: 4.88
Post D: @techguru "reply to..."       → weighted_score: 2.01
Post E: @sportsfan "Game recap"       → weighted_score: 1.28
```

### Scorer 3: AuthorDiversityScorer

Processes candidates in **descending weighted_score order**. Tracks how many times each author has appeared.

Assume `decay_factor = 0.5` and `floor = 0.1`:

```
multiplier(n) = (1 - 0.1) × 0.5^n + 0.1

  n=0 (1st post):  0.9 × 1.0  + 0.1 = 1.00   (no penalty)
  n=1 (2nd post):  0.9 × 0.5  + 0.1 = 0.55   (45% penalty)
  n=2 (3rd post):  0.9 × 0.25 + 0.1 = 0.325  (67% penalty)
```

Processing order (by weighted_score descending):
```
1. Post A: @techguru  weighted=5.85 → author_count[techguru]=0 → mult=1.00 → score = 5.85 × 1.00 = 5.85
2. Post B: @designeramy weighted=5.12 → author_count[designeramy]=0 → mult=1.00 → score = 5.12 × 1.00 = 5.12
3. Post C: @rustacean_dev weighted=4.88 → author_count[rustacean_dev]=0 → mult=1.00 → score = 4.88 × 1.00 = 4.88
4. Post D: @techguru  weighted=2.01 → author_count[techguru]=1 → mult=0.55 → score = 2.01 × 0.55 = 1.11 ← PENALIZED!
5. Post E: @sportsfan weighted=1.28 → author_count[sportsfan]=0 → mult=1.00 → score = 1.28 × 1.00 = 1.28
```

**Post D dropped below Post E** because it was @techguru's 2nd post. This ensures Priya's feed isn't dominated by one prolific author.

### Scorer 4: OONScorer

Adjusts out-of-network posts. Assume `OON_WEIGHT_FACTOR = 0.85`:

```
Post A: in_network=true  → score unchanged: 5.85
Post B: in_network=true  → score unchanged: 5.12
Post C: in_network=false → score × 0.85:   4.88 × 0.85 = 4.15 ← reduced
Post D: in_network=true  → score unchanged: 1.11
Post E: in_network=true  → score unchanged: 1.28
```

**Final scores after all 4 scorers:**
```
─────────────────────────────────────────────────────────────────────
  Rank │ Post                          │ Score │ Type
─────────────────────────────────────────────────────────────────────
  #1   │ A: @techguru "Rust v2.0"      │ 5.85  │ In-Network
  #2   │ B: @designeramy "Design 🧵"   │ 5.12  │ In-Network
  #3   │ C: @rustacean_dev "Async..."   │ 4.15  │ Out-of-Network
  ...  │ ...198 more candidates...      │ ...   │ ...
  #202 │ E: @sportsfan "Game recap"     │ 1.28  │ In-Network
  #203 │ D: @techguru "reply to..."     │ 1.11  │ In-Network (penalized)
  ...  │ ...660 more candidates...      │ ...   │ ...
─────────────────────────────────────────────────────────────────────
```

**Time elapsed for all scoring: ~58ms**

---

## Stage 6: Selection

`TopKScoreSelector` sorts all 863 candidates by final `score` descending and picks top K (e.g., K=150):

```
Selected: Top 150 candidates
Discarded: 713 lower-scored candidates
```

Posts A, B, C make the cut. Post E barely makes it at position ~130. Post D doesn't make it.

**Time elapsed: <1ms** (just a sort)

---

## Stage 7: Post-Selection Processing

### VFCandidateHydrator
Calls Visibility Filtering service on the 150 selected posts:
```
tweet_id: 1893200088 → SafetyResult { action: Drop("spam") }
tweet_id: 1893100055 → SafetyResult { action: Drop("violence") }
All others → None (safe to serve)
```

### VFFilter
Removes posts flagged by VF:
```
Removed: 2 (1 spam, 1 violent content)
Remaining: 148
```

### DedupConversationFilter  
If multiple posts are from the same conversation thread, keep only the most relevant:
```
Removed: 3 (redundant branches of same conversation)
Remaining: 145
```

**Time elapsed: ~20ms** (VF service call dominates)

---

## Stage 8: Side Effects (async)

`CacheRequestInfoSideEffect` fires in the background (doesn't block the response):
```
Cache to Strato: {
    user_id: 900100200,
    served_post_ids: [1893100001, 1893100005, 1893200001, ...142 more],
    timestamp: 1740400500000,
    request_id: "abc123-900100200"
}
```

This ensures the **next request** won't re-serve these same 145 posts.

---

## Final Response

The gRPC response goes back to Priya's X app:

```protobuf
ScoredPostsResponse {
    scored_posts: [
        {
            tweet_id:      1893100001,          // Post A
            author_id:     100200300,
            score:         5.85,
            in_network:    true,
            served_type:   ForYouInNetwork,
            screen_names:  { 100200300: "techguru" },
        },
        {
            tweet_id:      1893100005,          // Post B
            author_id:     300400500,
            score:         5.12,
            in_network:    true,
            served_type:   ForYouInNetwork,
            screen_names:  { 300400500: "designeramy" },
        },
        {
            tweet_id:      1893200001,          // Post C
            author_id:     600700800,
            score:         4.15,
            in_network:    false,
            served_type:   ForYouPhoenixRetrieval,
            screen_names:  { 600700800: "rustacean_dev" },
        },
        // ... 142 more posts in descending score order ...
    ]
}
```

**Total latency: ~130ms** (from request to response)

---

## What Priya Sees

When Priya's app renders the feed:

```
┌─────────────────────────────────────────────────────────┐
│  For You                                          ✨    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  🔵 @techguru                                    · 2h  │
│  Just shipped v2.0 of our Rust framework! 🦀            │
│  Major perf improvements and a new async runtime...     │
│  ♡ 1.2K   💬 89   🔁 234                               │
│  ─────────────────────────────────────────────────────  │
│                                                         │
│  🔵 @designeramy                                 · 30m │
│  New design system drop — check the thread 🧵           │
│  We rebuilt our entire component library from scratch... │
│  ♡ 3.4K   💬 156  🔁 892                               │
│  ─────────────────────────────────────────────────────  │
│                                                         │
│  ✨ @rustacean_dev                                · 5h  │
│  Here's why async Rust changed our production infra...  │
│  [Long thread about async patterns]                     │
│  ♡ 8.1K   💬 342  🔁 1.5K                              │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│  ↑ Priya doesn't follow @rustacean_dev                  │
│    Phoenix Retrieval found this based on her history     │
│    of engaging with Rust + async content                 │
│  ─────────────────────────────────────────────────────  │
│                                                         │
│  🔵 @newsbot                                     · 1h  │
│  Breaking: New open-source ML framework released...     │
│  ♡ 567    💬 23   🔁 189                               │
│  ─────────────────────────────────────────────────────  │
│                                                         │
│  ... (141 more posts as she scrolls) ...                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

Notice:
- **@techguru** appears once at #1, even though they had 2 posts — the diversity scorer pushed the reply down
- **@rustacean_dev** (out-of-network) ranked #3 — Phoenix found it because Priya's history is full of Rust/async engagement
- **@annoying_person** (muted) is nowhere to be seen
- **"Crypto giveaway"** spam posts are filtered out
- Posts she already saw in her last session don't reappear

---

## Latency Breakdown

```
Stage                          │ Time    │ Notes
───────────────────────────────│─────────│──────────────────────
1. Query Hydration             │  12ms   │ UAS + Strato in parallel
2. Candidate Sources           │  45ms   │ Thunder (~3ms) + Phoenix Retrieval (~42ms)
3. Hydration                   │  65ms   │ TES + Gizmoduck + others in parallel
4. Pre-scoring Filters         │   2ms   │ All in-memory, sequential
5. Scoring                     │  58ms   │ Phoenix ML (~55ms) + weighted/diversity (~3ms)
6. Selection                   │  <1ms   │ Sort + truncate
7. Post-selection              │  20ms   │ VF service call + filters
8. Side Effects                │   0ms   │ Async, doesn't block response
───────────────────────────────│─────────│──────────────────────
TOTAL (with parallelism)       │ ~130ms  │ Stages 1-3 overlap; 4-7 sequential
```

Stages 1, 2, and 3 overlap with each other (they're sequential stages but internally parallel), but stages 4 through 7 must execute sequentially since each depends on the previous stage's output.

---

## Why Post A Beat Post C

Let's compare @techguru's original post (in-network) vs @rustacean_dev's post (out-of-network):

```
                            Post A              Post C
                            @techguru            @rustacean_dev
────────────────────────────────────────────────────────────────
P(favorite)                 0.42                0.35
P(reply)                    0.15                0.12
P(retweet)                  0.18                0.14
P(click)                    0.55                0.60      ← C wins here
P(dwell)                    0.72                0.68

After WeightedScorer        5.85                4.88
  (reply×27 helps A a lot since 0.15>0.12)

After AuthorDiversity       5.85 (×1.00)        4.88 (×1.00)
  (both are 1st post from their author)

After OONScorer             5.85 (in-network)   4.88 × 0.85 = 4.15
  (C gets penalized for being OON)

FINAL RANK                  #1                  #3
```

The **reply score difference** (0.15 vs 0.12) gets amplified by the 27× reply weight, creating a 0.81 gap. Then the **OON penalty** widens it further. Even though Post C had a higher click probability, the system values reply potential much more heavily.

---

## What If Priya Engages?

If Priya likes Post C (@rustacean_dev), that action gets appended to her engagement history. On her **next** feed request:

1. Her `user_action_sequence` now includes `Liked tweet_id:1893200001 by author:600700800`
2. Phoenix Retrieval will find **more posts similar to @rustacean_dev's content**
3. Phoenix Ranking will give **higher scores** to posts from authors/topics similar to @rustacean_dev
4. Over time, more Rust infrastructure content appears in her feed

The transformer learns entirely from these engagement signals — no hand-coded rules like "if user likes Rust content, show more Rust content." The model figures out the patterns from billions of engagement sequences across all users.

---

---

# Advanced Deep Dives

> The sections below expand on the example above for engineers who want to understand the system at production depth — tensor shapes, attention mechanics, hash collision math, failure modes, and tuning levers.

---

## Deep Dive 1: Tensor Shapes Through the Phoenix Ranking Model

Let's trace the exact tensor dimensions as Priya's request flows through the Grok transformer for scoring. We'll use the production defaults from `PhoenixModelConfig`:

```
Config:
  emb_size (D)            = 128
  key_size (K)            = 64
  num_q_heads             = 2
  num_kv_heads            = 2
  num_layers              = 2
  widening_factor         = 2
  history_seq_len (S)     = 128       (Priya's last 128 actions)
  candidate_seq_len (C)   = 32        (batch size for candidates)
  num_actions             = 19
  num_user_hashes         = 2
  num_item_hashes         = 2
  num_author_hashes       = 2
  product_surface_vocab   = 16
  fprop_dtype             = bfloat16
```

### Step 1: Embedding Assembly

```
USER EMBEDDING:
  Hash user_id (900100200) with 2 hash functions:
    hash₁(900100200) → index 4827 in embedding_table_1  → [D]
    hash₂(900100200) → index 1293 in embedding_table_2  → [D]
  Concatenate → [2D] = [256]
  Project via block_user_reduce: Linear([256] → [D]) → [128]
  Reshape → [B, 1, D] = [1, 1, 128]

HISTORY EMBEDDINGS (for each of 128 past actions):
  For action i:
    post_hash_embeds:    [num_item_hashes, D] = [2, 128] → concat → [256]
    author_hash_embeds:  [num_author_hashes, D] = [2, 128] → concat → [256]
    action_embed:        lookup in action_embedding_table → [D] = [128]
    surface_embed:       lookup in product_surface_table → [D] = [128]
    Concatenate all: [256 + 256 + 128 + 128] = [768]
    block_history_reduce: Linear([768] → [D]) → [128]
  Stack all 128 → [B, S, D] = [1, 128, 128]

CANDIDATE EMBEDDINGS (for 32 candidates in this batch):
  For candidate j:
    post_hash_embeds:    [num_item_hashes, D] = [2, 128] → concat → [256]
    author_hash_embeds:  [num_author_hashes, D] = [2, 128] → concat → [256]
    surface_embed:       lookup → [D] = [128]
    ⚠ NO action embedding (candidates don't have actions yet!)
    Concatenate: [256 + 256 + 128] = [640]
    block_candidate_reduce: Linear([640] → [D]) → [128]
  Stack all 32 → [B, C, D] = [1, 32, 128]
```

### Step 2: Sequence Assembly

```
Full input sequence:
  [User(1)] [History₁..₁₂₈] [Cand₁..₃₂]
  Shape: [B, 1+S+C, D] = [1, 161, 128]

  candidate_start_offset = 1 + 128 = 129
```

### Step 3: Attention Mask Construction

```python
# make_recsys_attn_mask(seq_len=161, candidate_start_offset=129)
seq_len = 161

# 1) Start with standard causal mask [1, 1, 161, 161]
causal = jnp.tril(jnp.ones((1, 1, 161, 161)))

# 2) Zero out candidate-to-candidate block
#    Rows 129-160 cannot attend to columns 129-160
attn_mask = causal.at[:, :, 129:, 129:].set(0)

# 3) Re-enable self-attention for each candidate on the diagonal
#    Position 129 can attend to position 129, etc.
for i in range(129, 161):
    attn_mask = attn_mask.at[:, :, i, i].set(1)
```

Visualized (simplified 8-token version with offset=5):
```
              User  H1   H2   H3   H4   C1   C2   C3
  User     [  1     0    0    0    0    0    0    0  ]  ← user sees self
  H1       [  1     1    0    0    0    0    0    0  ]  ← causal
  H2       [  1     1    1    0    0    0    0    0  ]
  H3       [  1     1    1    1    0    0    0    0  ]
  H4       [  1     1    1    1    1    0    0    0  ]  ← last history
  C1       [  1     1    1    1    1   [1]   0    0  ]  ← sees user+history+self
  C2       [  1     1    1    1    1    0   [1]   0  ]  ← sees user+history+self
  C3       [  1     1    1    1    1    0    0   [1] ]  ← sees user+history+self
                                       ↑ ↑ ↑
                                   candidate-to-candidate = 0
                                   (except self-attention on diagonal)
```

**Why this matters for production:** Because C1's score doesn't depend on C2 or C3, you can:
- Score the same tweet in different batches and get identical results 
- Cache scored candidates and mix them with fresh ones
- Parallelize scoring across multiple GPU batches without cross-batch dependencies

### Step 4: Transformer Forward Pass

```
For each of 2 decoder layers:
  ┌─────────────────────────────────────────────────┐
  │ Input shape: [1, 161, 128]                      │
  │                                                 │
  │ 1. RMSNorm: [1, 161, 128] → [1, 161, 128]      │
  │                                                 │
  │ 2. Multi-Head Attention (GQA):                  │
  │    Q = input @ W_q: [1, 161, 2, 64]             │
  │    K = input @ W_k: [1, 161, 2, 64]             │
  │    V = input @ W_v: [1, 161, 2, 64]             │
  │    Apply RoPE to Q, K                           │
  │    logits = Q @ K.T / √64: [1, 2, 161, 161]    │
  │    Cap: 30.0 × tanh(logits / 30.0)             │
  │    Mask: logits × attn_mask + (-1e10)×(1-mask)  │
  │    Softmax → attn_weights: [1, 2, 161, 161]     │
  │    Output = attn_weights @ V: [1, 2, 161, 64]   │
  │    Project: [1, 161, 128]                       │
  │    RMSNorm → residual add                       │
  │                                                 │
  │ 3. GELU-gated FFN:                              │
  │    ffn_size = int(2 × 128) × 2 // 3            │
  │            = 170 → round up to 176 (multiple 8) │
  │    h_gate = GELU(input @ W_gate): [1, 161, 176] │
  │    h_up   = input @ W_up:         [1, 161, 176] │
  │    h_combined = h_gate * h_up:    [1, 161, 176] │
  │    output = h_combined @ W_down:  [1, 161, 128] │
  │    RMSNorm → residual add                       │
  └─────────────────────────────────────────────────┘
```

> **Note:** The FFN uses `GELU` (Gaussian Error Linear Unit), not SiLU. This differs from some Grok documentation but matches the actual `grok.py` implementation.

### Step 5: Output Projection

```
transformer_output: [1, 161, 128]

# Extract only candidate positions
candidate_embeddings = layer_norm(transformer_output)[:, 129:, :]
  → [1, 32, 128]

# Project to action logits
logits = candidate_embeddings @ unembedding_matrix
  → [1, 32, 19]     (unembedding_matrix: [128, 19])

# Convert to probabilities
probs = sigmoid(logits)   # in Python runner
  → [1, 32, 19]

# Each of the 32 candidates now has 19 probability scores
```

### Step 6: Batched Scoring for 863 Candidates

Since Priya has 863 candidates but `candidate_seq_len = 32`:

```
Total batches = ceil(863 / 32) = 27 batches
Last batch: 863 - 26×32 = 31 candidates (padded to 32)

Each batch shares the same [User + History] prefix.
The transformer processes [1, 161, 128] per batch.

Total forward passes: 27
Effective throughput: 863 candidates scored in ~55ms
  → ~0.064ms per candidate
  → ~2ms per batch of 32
```

---

## Deep Dive 2: Hash-Based Embedding — Collision Analysis

The system uses **multiple hash functions** per entity to reduce collision rates while keeping embedding tables bounded.

### How It Works

For tweet ID `1893100001` with `num_item_hashes = 2`:

```
Hash function 1: hash₁(1893100001) mod TABLE_SIZE → slot 84,291
Hash function 2: hash₂(1893100001) mod TABLE_SIZE → slot 12,507

embedding₁ = table_1[84291]  → [D] = [128]
embedding₂ = table_2[12507]  → [D] = [128]

concatenated = [embedding₁ | embedding₂] → [2D] = [256]
projected = Linear([256] → [128]) → [128]
```

### Collision Probability

With a single hash function into a table of size T, the probability that two distinct IDs collide:

$$P(\text{collision}) = \frac{1}{T}$$

With 2 independent hash functions, two IDs must collide in **both** tables simultaneously:

$$P(\text{both collide}) = \frac{1}{T_1} \times \frac{1}{T_2} = \frac{1}{T^2}$$

For T = 100,000:
- Single hash: 1 in 100K collision rate
- Dual hash: 1 in 10 billion collision rate

Even with partial collisions (same slot in one table, different in the other), the concatenation + learned projection allows the model to disambiguate:

```
Tweet A: [emb_a₁ | emb_shared₂]  → project → different output
Tweet B: [emb_b₁ | emb_shared₂]  → project → different output
                   ↑ same slot in table 2, but table 1 differs
```

### Memory Trade-off

```
Naive approach (unique embedding per entity):
  50M tweets × 128 floats × 4 bytes = 25.6 GB

Hash-based approach (2 tables × 100K slots):
  2 × 100,000 × 128 × 4 bytes = 102.4 MB  ← 250× smaller
  + 1 projection matrix: 256 × 128 × 4 = 131 KB

Total: ~103 MB vs 25.6 GB
```

The same approach is used for user IDs (2 hash tables) and author IDs (2 hash tables), keeping the entire embedding infrastructure under ~300 MB.

---

## Deep Dive 3: Two-Tower Retrieval — Similarity Geometry

Let's trace how Phoenix Retrieval finds Post C (@rustacean_dev's async Rust post) for Priya.

### User Tower Processing

Priya's engagement sequence goes through the full transformer → mean pool → L2 normalize:

```
Input: [User_emb | History₁ | ... | History₁₂₈]  →  [1, 129, 128]

Transformer output: [1, 129, 128]

Mean pool (over valid positions):
  user_repr = mean(output[:, :valid_len, :])  →  [1, 128]

L2 Normalize:
  user_repr = user_repr / ||user_repr||₂  →  [1, 128]

  ||user_repr||₂ = 1.0  (unit sphere)
```

### Candidate Tower Processing (Offline)

For every tweet in the corpus, the Candidate Tower runs a lightweight 2-layer MLP:

```
For tweet 1893200001 (@rustacean_dev's post):
  post_hashes:   concat(hash₁, hash₂) → [256]
  author_hashes: concat(hash₁, hash₂) → [256]
  Total input: [512]

  Layer 1: Linear(512 → 256) → SiLU → [256]
  Layer 2: Linear(256 → 128) → [128]
  L2 Normalize → [128]

  ||candidate_repr||₂ = 1.0  (unit sphere)
```

### Dot-Product Retrieval

Since both vectors are unit-normalized, dot product = cosine similarity:

$$\text{score}(u, c) = \vec{u} \cdot \vec{c} = \|\vec{u}\| \|\vec{c}\| \cos\theta = \cos\theta$$

```
Priya's user embedding:       u = [0.12, -0.08, 0.15, 0.22, ..., 0.05]  (128-dim)

Corpus (millions of candidate embeddings):
  tweet_1893200001 (Rust async): c₁ = [0.14, -0.06, 0.13, 0.20, ..., 0.07]
  tweet_1893200099 (crypto spam): c₂ = [-0.20, 0.15, -0.08, -0.12, ..., 0.18]
  tweet_1893200050 (cooking vid):  c₃ = [0.02, 0.25, -0.15, 0.04, ..., -0.22]

  score(u, c₁) = 0.87  ← high similarity (Rust/async aligns with Priya's history)
  score(u, c₂) = 0.12  ← low similarity (crypto spam is far away)
  score(u, c₃) = 0.23  ← low (cooking content doesn't match)
```

The `jax.lax.top_k(scores, k=500)` efficiently extracts the 500 highest-scoring candidates without sorting the full corpus.

### Why Separate Towers?

```
ONLINE (per request, ~42ms):
  Only the User Tower runs → 1 embedding computed
  Dot product against pre-computed corpus → ANN lookup

OFFLINE (batch job, runs continuously):
  Candidate Tower processes all new/updated tweets
  Writes embeddings to a vector index

If both towers were combined (single model):
  Would need to score every corpus tweet against every user → O(Users × Tweets) per request
  Completely infeasible at scale
```

---

## Deep Dive 4: Visibility Filtering — Dual Safety Levels

After selection, the `VFCandidateHydrator` applies different safety thresholds depending on whether a post is in-network or out-of-network:

```
In-network posts:   SafetyLevel::TimelineHome
  → Lower threshold — user explicitly chose to follow this author
  → Only blocks: deleted, suspended accounts, hard spam, CSAM

Out-of-network posts: SafetyLevel::TimelineHomeRecommendations
  → Higher threshold — system is recommending this content
  → Also blocks: borderline content, low-quality, sensational,
    unverified claims, graphic media without labels
```

Both safety levels are evaluated **in parallel** via `futures::future::join`:

```rust
let (in_network_results, oon_results) = futures::future::join(
    vf_client.check(in_network_ids, TimelineHome),
    vf_client.check(oon_ids, TimelineHomeRecommendations),
).await;
```

### In Our Example

```
Selected 150 posts → split by in_network:
  In-network (89 posts)  → VF with TimelineHome
  OON        (61 posts)  → VF with TimelineHomeRecommendations

Results:
  In-net: 0 removed (all safe for TimelineHome)
  OON:    2 removed
    - tweet_1893200088: spam (caught even at lower threshold)
    - tweet_1893100055: graphic violence without labels (caught at stricter OON threshold
      — would have passed TimelineHome threshold if from a followed account)
```

This asymmetry means Priya's feed from followed accounts is more permissive (she opted into that content), while recommended content goes through stricter quality gates.

---

## Deep Dive 5: The `PreviouslyServedPostsFilter` Enable Gate

A subtle but important detail: this filter has a conditional `enable()` override:

```rust
fn enable(&self, query: &ScoredPostsQuery) -> bool {
    query.is_bottom_request
}
```

### What This Means

```
Pull-to-refresh (top of feed):
  is_bottom_request = false
  → PreviouslyServedPostsFilter is DISABLED
  → Previously served posts CAN reappear
  → Rationale: user might want to re-read posts from their last session

Scroll-to-bottom (infinite scroll):
  is_bottom_request = true
  → PreviouslyServedPostsFilter is ENABLED
  → Previously served posts are REMOVED
  → Rationale: showing the same post twice while scrolling feels broken
```

In Priya's case, she opened the app fresh (`is_bottom_request: false`), so this filter was **skipped**. If she had been scrolling to load more, it would have removed 8 additional posts.

---

## Deep Dive 6: Weight Sensitivity Analysis

The `WeightedScorer` weights dramatically shape feed character. Let's see how different weight configurations would change Priya's feed:

### Current Configuration (Reply-Heavy)

```
REPLY_WEIGHT = 27.0  (dominant signal)
```

```
Post A: @techguru "Rust v2.0"      → weighted = 5.85  (#1)
Post C: @rustacean_dev "Async..."   → weighted = 4.88  (#3 before OON penalty)
```

The high reply weight means **conversational content** dominates — posts Priya would likely respond to rank highest.

### Hypothetical: Share-Heavy Configuration

```
SHARE_WEIGHT = 27.0 (swap with reply)
REPLY_WEIGHT = 1.0
```

```
Post A recalculated:
  reply term:  0.15 × 1.0  = 0.15  (was 4.05)
  share term:  0.08 × 27.0 = 2.16  (was 0.08)
  → The Post A advantage shrinks; @designeramy's share_score (0.12 × 27 = 3.24) leaps ahead.

New ranking would likely be:
  #1: Post B (@designeramy) — high shareability
  #2: Post A (@techguru)    — still strong but less dominant
  #3: Post C (@rustacean_dev)
```

### Hypothetical: Safety-Aggressive Configuration

```
BLOCK_AUTHOR_WEIGHT = -200.0  (up from -74.0)
REPORT_WEIGHT       = -500.0  (up from -200.0)
```

```
Posts from authors with even slightly elevated block/report probabilities
get pushed far down. The feed becomes extremely "safe" but may feel bland —
controversial but legitimate discussion gets suppressed.

Post E recalculated:
  block term:  0.003 × -200.0 = -0.60   (was -0.222)
  report term: 0.0005 × -500.0 = -0.25  (was -0.10)
  → Post E's score drops to ~0.63 (from 1.28) — it likely falls off the top 150
```

### The `offset_score` Mechanism

The `WeightedScorer` has a subtle normalization for negative combined scores:

```
If combined < 0.0:
  offset = (combined + |NEGATIVE_WEIGHTS_SUM|) / WEIGHTS_SUM × NEGATIVE_SCORES_OFFSET

This ensures strongly-negative posts don't get arbitrarily negative scores.
Instead, they're compressed into a small negative range — preventing a single
bad probability from completely dominating the ranking.
```

---

## Deep Dive 7: Author Diversity — Exponential Decay Walkthrough

Let's trace what happens when a prolific author like @techguru has 5 posts surviving the filter stage.

```
decay_factor = 0.5, floor = 0.1
multiplier(n) = (1.0 - 0.1) × 0.5^n + 0.1

n=0: 0.9 × 1.000 + 0.1 = 1.000   (1st post, no penalty)
n=1: 0.9 × 0.500 + 0.1 = 0.550   (45% cut)
n=2: 0.9 × 0.250 + 0.1 = 0.325   (67.5% cut)
n=3: 0.9 × 0.125 + 0.1 = 0.213   (78.8% cut)
n=4: 0.9 × 0.0625+ 0.1 = 0.156   (84.4% cut)

As n→∞: multiplier → floor = 0.1  (maximum 90% penalty, never fully zero)
```

### Scoring Order Matters

Candidates are processed in **descending `weighted_score` order**. This means the BEST post from each author always gets `multiplier = 1.0`:

```
Processing order across ALL candidates (863 total):
  #1: @guru_rust    weighted=8.22 → first by @guru_rust      → ×1.000 → score=8.22
  #2: @techguru     weighted=5.85 → first by @techguru (Post A) → ×1.000 → score=5.85
  #3: @designeramy  weighted=5.12 → first by @designeramy     → ×1.000 → score=5.12
  ...
  #87: @techguru    weighted=2.01 → SECOND by @techguru (Post D) → ×0.550 → score=1.11
  ...
  #201: @techguru   weighted=0.82 → THIRD by @techguru         → ×0.325 → score=0.27
  ...
```

If we sorted AFTER diversity scoring (which we do in the selector), the ranking naturally interleaves authors without any explicit interleaving algorithm.

### Why floor > 0?

Without a floor (`floor = 0.0`), the 10th post from an author would score `0.5^9 × original ≈ 0.2%` of its value — effectively zero. With `floor = 0.1`, even the 100th post retains 10% of its score. This means truly exceptional content from a prolific author can still rank well.

---

## Deep Dive 8: Thunder's Real-Time Ingestion Path

### From Tweet Creation to PostStore

```
Timeline:
  T+0ms:    User posts a tweet
  T+~5ms:   Tweet create event written to Kafka topic
  T+~15ms:  Thunder's Kafka consumer (v2) picks up the protobuf InNetworkEvent
  T+~16ms:  Deserialize → LightPost
  T+~17ms:  Insert into PostStore (DashMap lock-free write)
  T+~17ms:  Tweet is now available for in-network queries
```

### PostStore Insert Logic

```
For each new post (e.g., @techguru's "Rust v2.0"):
  1. Validate:
     - NOT in deleted_posts set?           ✓
     - created_at < now + small_buffer?     ✓ (no future posts)
     - created_at > now - retention_secs?   ✓ (not too old)

  2. Insert into main store:
     posts.insert(1893100001, LightPost { post_id: 1893100001, author_id: 100200300, ... })

  3. Classify and index:
     is_reply?    → No
     is_retweet?  → No
     → original_posts_by_user[100200300].push_front(TinyPost { post_id: 1893100001, ... })

     has_video?   → No
     → (skip video index)

  4. Enforce per-author caps:
     if original_posts_by_user[100200300].len() > MAX_ORIGINAL_POSTS_PER_AUTHOR:
         remove oldest from deque (and from main posts map)
```

### Memory Model

```
LightPost (per post, ~120 bytes):
  post_id: i64, author_id: i64, created_at: i64,
  in_reply_to_post_id: Option<i64>, in_reply_to_user_id: Option<i64>,
  is_retweet: bool, is_reply: bool,
  source_post_id: Option<i64>, source_user_id: Option<i64>,
  has_video: bool, conversation_id: Option<i64>

TinyPost (per timeline index entry, ~16 bytes):
  post_id: i64, created_at: i64

For 10M active posts:
  Main store:     10M × 120B ≈ 1.2 GB
  Timeline indexes: ~30M entries × 16B ≈ 480 MB  (3 indexes: original, secondary, video)
  Overhead:       ~500 MB (DashMap overhead, deleted tracking, etc.)
  Total:          ~2.2 GB

This fits comfortably in a single machine's memory, enabling sub-millisecond lookups.
```

### Auto-Trim Cycle

```
Every 2 minutes:
  1. Scan all posts in DashMap
  2. For each post where (now - created_at) > retention_seconds:
     - Remove from posts map
     - Remove TinyPost from the appropriate per-user deque
     - Add to deleted_posts set (prevents re-insertion from delayed Kafka events)
  3. Log: "Trimmed 42,891 expired posts, 12,431,607 remaining"
```

---

## Deep Dive 9: Failure Modes & Resilience

### What Happens When Phoenix ML Is Down?

```
PhoenixSource.get_candidates() → Error("gRPC timeout after 5000ms")
PhoenixScorer.score() → Error("gRPC timeout after 5000ms")

Pipeline behavior (from candidate_pipeline.rs):
  - Source error: logged, source returns empty vec, pipeline continues
  - Scorer error: logged, scorer is skipped, pipeline continues

Result for Priya:
  - Only Thunder (in-network) candidates survive
  - No PhoenixScores → WeightedScorer assigns weighted_score = 0.0 for all
  - AuthorDiversityScorer still works (uses weighted_score)
  - Feed falls back to roughly chronological in-network content
  - User experience: "For You" looks like "Following" tab temporarily
```

### What Happens When TES Is Slow?

```
CoreDataCandidateHydrator takes 500ms instead of 30ms

Impact:
  - Total latency jumps from ~130ms to ~550ms
  - User sees a longer loading spinner
  - But all data is correct — just slow

Mitigation:
  - PostStore.request_timeout: if hydration exceeds this, remaining candidates
    get default/empty values and are caught by CoreDataHydrationFilter
  - Partial degradation: 800 of 987 candidates hydrate successfully,
    187 get filtered out, feed is slightly less diverse but still good
```

### What Happens When a Kafka Partition Lags?

```
Thunder Kafka consumer v2 falls behind by 5 minutes on partition 7

Impact:
  - Posts from authors whose events land on partition 7 are delayed
  - Priya might not see @techguru's latest post for 5 minutes
  - Thunder's partition lag monitoring surfaces this in metrics
  - Other partitions are unaffected

Mitigation:
  - Multiple partitions ensure most authors' posts arrive on time
  - Phoenix Retrieval (not dependent on Thunder) can still surface those posts
    if they're in the cached corpus embeddings
```

### Hydrator Length Mismatch Safety

```
The pipeline framework has a critical safety check:

If hydrator returns N results but pipeline has M candidates (N ≠ M):
  → Warning logged: "Hydrator returned {N} results, expected {M}"
  → Hydrator results are DISCARDED entirely
  → Pipeline continues with un-hydrated candidates
  → Not a crash — graceful degradation

This prevents data misalignment bugs where candidate[i] gets candidate[j]'s data.
```

---

## Deep Dive 10: End-to-End Latency Optimization

### Parallelism Map

```
                    0ms    20ms   40ms   60ms   80ms  100ms  120ms  130ms
                    │      │      │      │      │      │      │      │
Query Hydration:    ├──UAS──┤
                    ├──Strato─┤
                    │      │      │
Candidate Sources:  │      ├──Thunder──┤
                    │      ├────Phoenix Retrieval────┤
                    │      │      │      │      │    │
Candidate Hydration:│      │      │      │      ├─InNet─┤
                    │      │      │      │      ├─TES───────┤
                    │      │      │      │      ├─Video─┤
                    │      │      │      │      ├─Sub───┤
                    │      │      │      │      ├─Gizmo─────┤
                    │      │      │      │      │      │    │
Pre-scoring Filters:│      │      │      │      │      │    ├─┤
Scoring:            │      │      │      │      │      │    │ ├──PhoenixML──────┤
                    │      │      │      │      │      │    │ │       ├─Weighted─┤
                    │      │      │      │      │      │    │ │       │ ├─Author──┤
                    │      │      │      │      │      │    │ │       │ │├OON┤│
Selection:          │      │      │      │      │      │    │ │       │ ││  ├┤│
Post-Selection:     │      │      │      │      │      │    │ │       │ ││  │├VF─┤
                    │      │      │      │      │      │    │ │       │ ││  ││ ├┤│
Side Effects:       │      │      │      │      │      │    │ │       │ ││  ││ │├→ (async, non-blocking)
                    │      │      │      │      │      │    │ │       │ ││  ││ │
Response sent ──────┼──────┼──────┼──────┼──────┼──────┼────┼─┼───────┼─┼┼──┼┼─┤
                    0ms    20ms   40ms   60ms   80ms  100ms  120ms  130ms
```

### Where Time Is Spent

```
Breakdown by category:

  ML Inference:     55ms  (42%)  — Phoenix Retrieval + Phoenix Ranking
  Network calls:    45ms  (35%)  — TES, Gizmoduck, Strato, VF, UAS
  In-memory ops:     5ms   (4%)  — Filters, diversity, selection, OON
  Framework:        25ms  (19%)  — gRPC overhead, serialization, async scheduling

ML dominates. To reduce latency, you'd optimize:
  1. Reduce candidate_seq_len (fewer candidates per batch → fewer batches)
  2. Use smaller transformer (fewer layers/heads)
  3. Pre-score popular content (use candidate isolation for caching)
  4. Quantize model to int8 (reduce inference time by ~2×)
```

### Side Effect Timing

```
CacheRequestInfoSideEffect:
  enable() check: APP_ENV == "prod" && !in_network_only
  If enabled: tokio::spawn(async { strato_client.store_request_info(...) })
  → Returns immediately, Strato write happens in background (~25ms)
  → If Strato write fails, it's logged but doesn't affect the user
  → Next request might re-serve some posts (acceptable degradation)
```

---

## Deep Dive 11: The Full PhoenixScores Object

Here are all 19 discrete action scores + 1 continuous action, with what they measure and how they flow through the system:

```
PhoenixScores {
  // === Positive engagement signals (positive weights in WeightedScorer) ===
  favorite_score:            Option<f64>,  // P(user likes/hearts the post)
  reply_score:               Option<f64>,  // P(user replies) — heaviest weight (27×)
  retweet_score:             Option<f64>,  // P(user reposts) — called "repost_score" in Python
  photo_expand_score:        Option<f64>,  // P(user taps to expand a photo)
  click_score:               Option<f64>,  // P(user clicks the post to read thread)
  profile_click_score:       Option<f64>,  // P(user clicks author's profile)
  vqv_score:                 Option<f64>,  // P(video quality view) — only weighted if
                                           //   video_duration_ms > MIN_VIDEO_DURATION_MS
  share_score:               Option<f64>,  // P(user shares via any channel)
  share_via_dm_score:        Option<f64>,  // P(user shares via direct message)
  share_via_copy_link_score: Option<f64>,  // P(user copies link)
  dwell_score:               Option<f64>,  // P(user dwells/pauses on post)
  quote_score:               Option<f64>,  // P(user quote-tweets)
  quoted_click_score:        Option<f64>,  // P(user clicks through to quoted content)
  follow_author_score:       Option<f64>,  // P(user follows the author) — weight 10×

  // === Negative signals (negative weights push score DOWN) ===
  not_interested_score:      Option<f64>,  // P(user marks "not interested") — weight -74×
  block_author_score:        Option<f64>,  // P(user blocks the author) — weight -74×
  mute_author_score:         Option<f64>,  // P(user mutes the author) — weight -74×
  report_score:              Option<f64>,  // P(user reports the post) — weight -200×

  // === Continuous signals ===
  dwell_time:                Option<f64>,  // Predicted dwell time in ms — weight 0.05×
}
```

The `Option<f64>` type means each score can be `None` (if Phoenix ML was unavailable or the candidate wasn't scored). The `WeightedScorer` treats `None` as `0.0`:

```rust
fn apply(score: Option<f64>, weight: f64) -> f64 {
    score.unwrap_or(0.0) * weight
}
```

---

**Navigation:** [Home](index.md) · [Architecture](ARCHITECTURE.md) · **Example Scenario** · [Pipeline Diagram](diagrams/pipeline-component-flow.md) · [Sequence Diagram](diagrams/sequence-diagram.md)
