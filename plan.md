# badminton-style — Project Plan

## Research Question

**What types of rallies exist in elite badminton, and which players play them most?**

More specifically — do distinct rally archetypes emerge from stroke-level data without predefining them, and does each player have a characteristic distribution across those archetypes that serves as their tactical fingerprint?

---

## Motivation

Badminton coaches and analysts already describe players qualitatively — "net rusher", "defensive grinder", "aggressive attacker" — but these labels are entirely subjective. This project asks: can stroke-level telemetry make those archetypes quantitative?

The approach is two-stage: let the math discover rally structure without bias, then use interpretable features to name what it found. Discovery first, interpretability second.

---

## Dataset

- **ShuttleSet** (KDD 2023) — 36,492 strokes, 3,685 rallies, 44 matches, 27 top-ranking players, 2018–2021
- **ShuttleSet22** (IJCAI 2024) — ~33,000 strokes, ~4,000 rallies, 58 matches, 35 players, 2022
- Both are singles only, stroke-level annotations including shot type, court coordinates, player positions
- Source: [CoachAI-Projects GitHub](https://github.com/wywyWang/CoachAI-Projects)

---

## What Has Been Done

### Data Loading
- [x] Downloaded ShuttleSet (ShuttleSet22 still to add)
- [x] Loaded all 106 CSV files into a single dataframe (36,572 rows, 53 columns)
- [x] Created unique `rally_id` combining match name and rally number
- [x] Confirmed 1,696 unique rallies across 45 matches

### Feature Engineering
- [x] Aggregated stroke-level data into rally-level feature vectors
- [x] Features: rally length, prop aroundhead, prop backhand, prop above net, avg/std landing x/y, avg player x/y
- [x] Mapped 19 Chinese shot type labels to 5 English buckets: attacking, defensive, net, neutral, serve
- [x] Identified and dropped dead features (prop_aroundhead, prop_backhand, prop_above_net all constant at 1.0)
- [x] Confirmed rally length is NOT the dominant signal — removing it improved silhouette score

### Clustering (Viability Check)
- [x] UMAP dimensionality reduction and visualization
- [x] K-means clustering with silhouette scoring across k=2 to k=7
- [x] Best result: k=2 (score 0.134), k=4 most interpretable
- [x] Confirmed 4 interpretable rally archetypes (see below)

---

## Key Findings So Far

### Rally Archetypes (k=4, silhouette 0.126)

| Cluster | Label | Characteristics |
|---|---|---|
| 0 | Serve and Attack | High attacking (0.24), high serve (0.11), deep landing, moderate net |
| 1 | Net Dominant | Highest net (0.45), low attacking (0.14), wide court coverage |
| 2 | Front Court Duel | High net (0.42), shortest landing distance (420), tightest court width |
| 3 | Defensive Baseline | Highest defensive (0.35), lowest attacking (0.14), widest court coverage |

These map onto rally types coaches already recognise — which validates the approach without predefining anything.

### Silhouette Scores by k (clean features, no rally length)
| k | Score |
|---|---|
| 2 | 0.134 |
| 3 | 0.128 |
| 4 | 0.126 |
| 5 | 0.120 |
| 6 | 0.109 |
| 7 | 0.108 |

k=2 wins statistically but k=4 is more narratively interesting and still defensible.

---

## Known Limitations

- **Silhouette scores are modest (0.13 range)** — expected for overlapping real-world categories. Interpretability matters more than the number here.
- **Aggregation destroys sequence** — two rallies with identical shot composition but different ordering look identical. A rally that transitions from defensive to attacking is indistinguishable from one that goes attacking to defensive.
- **ShuttleSet22 not yet loaded** — adding it nearly doubles rally count which should improve cluster stability.
- **Serve contaminates early strokes** — every rally starts with a serve which is a fixed predictable stroke. May be worth dropping the first stroke or handling serves separately.
- **Player sample is small** — 27-35 unique players is thin for strong statistical claims in Phase 2. Player fingerprints should be described, not compared statistically.
- **Unknown shot type** — 未知球種 (1,407 occurrences) bucketed as neutral. Worth investigating what these actually are.

---

## Phases

### Phase 1 — Rally Clustering ✅ Viability Proven
*The core ML work. Let the data define what rally types exist.*

- [x] Load and clean ShuttleSet
- [x] Engineer rally-level feature vectors
- [x] Map shot types to English buckets
- [x] UMAP + k-means clustering
- [x] Silhouette sweep across k values
- [x] Interpret cluster means
- [ ] Load ShuttleSet22 and combine
- [ ] Handle serve strokes separately
- [ ] Investigate unknown shot types
- [ ] Tune final k choice and produce clean UMAP visualization
- [ ] Radar charts per cluster for interpretability

### Phase 2 — Player Fingerprinting
*The sports analytics payoff. Who plays which rally types?*

- [ ] Assign each rally to a cluster
- [ ] Parse actual player names from match folder names
- [ ] Compute per-player distribution across clusters
- [ ] Identify specialists (concentrated) vs adaptable players (flat distribution)
- [ ] Player similarity matrix — which players have similar fingerprints
- [ ] Validate against known reputations (does Axelsen look aggressive?)

### Phase 3 — Sequence Modelling (Extension)
*Motivated by the transition problem — aggregation hides momentum shifts within rallies.*

Almost every rally has a transition point: neutral opening → one player gains advantage → attacking phase. Two rallies with identical shot composition but opposite ordering tell completely different tactical stories. Aggregation cannot distinguish them.

An LSTM autoencoder in PyTorch would encode the full stroke sequence into a latent vector, preserving temporal structure. Clustering on learned embeddings rather than handcrafted features could reveal richer archetypes:
- "Quick kill" — aggressive from stroke 1
- "Grind and pounce" — long defensive phase then sudden attack
- "Back and forth" — multiple momentum shifts
- "Capitulation" — one player defensive throughout

- [ ] Implement LSTM autoencoder in PyTorch
- [ ] Train on stroke sequences (shot type + location per stroke)
- [ ] Extract latent rally embeddings
- [ ] Re-cluster on learned embeddings
- [ ] Compare to Phase 1 clusters — do new archetypes emerge?

### Phase 4 — Player-Conditioned Sequence Modelling (Extension)
*Does conditioning the rally encoder on player identity reveal player-specific sub-archetypes?*

The project naturally progresses through three levels of increasing sophistication:

| Level | Question | Method |
|---|---|---|
| 1 | What rally types exist? | Rally clustering, no player identity |
| 2 | Which players play which rally types? | Attach player labels after clustering |
| 3 | Do rally types differ by player identity? | Condition LSTM encoder on player identity |

Level 1 and 2 find archetypes universal across all players. Level 3 asks "whose version of that rally type" — two players can both be "aggressive" by cluster assignment but execute aggression completely differently. One smashes from the rear court, one rushes the net. Level 1 and 2 call them the same archetype. Level 3 distinguishes them.

This is the difference between characterizing rally types and characterizing player-specific expressions of those rally types. The second is richer and more useful to a coach.

- [ ] Requires ShuttleSet22 loaded first for sufficient per-player data
- [ ] Condition LSTM encoder on player identity (learned player embedding as additional input)
- [ ] Extract player-conditioned rally embeddings
- [ ] Cluster and compare to Phase 1/3 clusters — do player-specific sub-archetypes emerge?
- [ ] Requires enough data per player — monitor per-player rally counts before attempting

### Phase 5 — Counter-Styling (Extension)
*Does a player's rally distribution shift depending on their opponent?*

- [ ] Requires sufficient match count per player pair (currently thin)
- [ ] Compute per-player per-opponent rally distributions
- [ ] Compare to baseline distribution — does style adapt?
- [ ] Only feasible for player pairs with 3+ matches in the dataset

---

## Tech Stack

| Tool | Purpose |
|---|---|
| `pandas` / `numpy` | Data loading, feature engineering |
| `scikit-learn` | K-means, silhouette scoring, StandardScaler |
| `umap-learn` | Dimensionality reduction and visualization |
| `matplotlib` | UMAP scatter, radar charts |
| `PyTorch` | LSTM autoencoder for sequence embeddings (Phase 3) |

---

## Analogies to Other Sports and Beyond

This project is easier to understand if you're not a badminton person. The core idea — cluster sequences of events into archetypes, then build player fingerprints from those archetypes — shows up across sports and outside sports entirely.

### Direct Sport Analogies

**Hockey (most relevant for UWAGGS)**
Zone entries in hockey are classified as carry-ins, dump-ins, or passes. Each entry type leads to different shot and scoring patterns. This project is the same idea applied to badminton rallies — classify the play type, then ask which teams or players use each type most. Jack Davis's work at Sportlogiq is directly in this space.

**Tennis**
Rally clustering has been done in tennis — classifying rallies as baseline exchanges, net approaches, or short points. Player fingerprinting from rally distributions is essentially the same problem. The difference is badminton has more shot type variety and faster transitions than tennis, making the clustering problem richer.

**Basketball**
Play type classification in the NBA — pick and roll, isolation, transition, post-up — is conceptually identical. Teams are profiled by their offensive play type distribution, which determines matchup strategy. This project builds the equivalent for individual badminton players.

**Soccer**
Possession sequence clustering — short passing builds vs long ball vs counter-attack — is a well studied problem in soccer analytics. A rally in badminton is structurally similar to a possession sequence: a series of events between two parties ending in a terminal outcome.

### Outside Sports

The underlying method — clustering variable-length sequences by aggregated features, then profiling agents by their cluster distributions — generalises broadly:

**Customer behaviour analysis** — cluster sessions on an app or website into behaviour types (browsers, searchers, buyers), then build user fingerprints from session distributions. Identical pipeline.

**Medical / clinical** — cluster patient treatment episodes by type and profile patients by their episode history. Used in chronic disease management to identify patient subgroups.

**Finance** — cluster trading sessions by market regime (trending, volatile, range-bound), then profile traders or funds by which regimes they perform in. Same two-stage structure.

**NLP / dialogue systems** — cluster conversation turns into intent types, then profile users by their interaction fingerprint. Used in chatbot analytics.

The reason the method generalises is that it makes minimal assumptions — you don't need to predefine what the categories are, just that categories exist and that agents have characteristic distributions over them.

---

The CoachAI papers (ShuttleNet, DyMF, RallyNet) all focus on **forecasting** — predicting the next stroke given previous ones. This project is orthogonal — characterizing the **shape of whole rallies** as tactical archetypes and building player fingerprints from those archetypes. Nobody has done this with ShuttleSet.

## Future Vision: Scaling via Computer Vision (Proposed Extension)

### The Data Bottleneck & The Open-Source Solution
The primary limitation of Phase 1 and 2 is data scarcity: manual stroke-level annotation is a major human bottleneck. To scale this project into a global intelligence engine, we can leverage open-source computer vision (CV) blocks to ingest unannotated video directly from broadcasts (e.g., the BWF YouTube channel).

State-of-the-art architectures like the **BST (Badminton Stroke-type Transformer)**, published at the **CVPR 2026 Sports Workshops**, are entirely open-source (`Va6lue/BST-Badminton-Stroke-type-Transformer`). These pipelines use a multi-stage approach to transform raw video into structured data:
1. **TrackNetV3:** Detects and records the high-velocity shuttlecock trajectory pixels.
2. **RTMPose / MMPose:** Extracts 2D skeletal joint graphs (17 keypoints) to capture player posture.
3. **Court Keypoint Detection:** Computes a **Homography Matrix ($H$)** to warp the camera's angled perspective, projecting screen pixels back onto a flat, standard 2D coordinate system:

$$\begin{bmatrix} x_{\text{court}} \\ y_{\text{court}} \\ 1 \end{bmatrix} = H \begin{bmatrix} x_{\text{pixel}} \\ y_{\text{pixel}} \\ 1 \end{bmatrix}$$

### Bridging the Gap: CV vs. Sports Analytics
Computer vision engineers design models to solve a low-level perception problem: *“In these 30 video frames, did the player hit a smash or a drop?”* Once their model hits high accuracy, they stop. They output a raw stream of isolated text labels and numbers. 

**This project bridges the gap by building the tactical reasoning layer on top of their perception layer.** Instead of looking at isolated strokes, our pipeline ingests their raw event streams, groups them sequentially into full rallies, and processes them through our Phase 1 & 2 clustering model. This translates machine-learning metrics into actionable coaching assets.
[Raw Broadcast Video]
            │
            ▼
┌───────────────────────┐
│     CV Layer (BST)    │  <-- Turns raw pixels into a stream 
│  Labels every shot    │      of individual event coordinates.
└───────────┬───────────┘
            │  (Sequential List of Labeled Strokes)
            ▼
┌───────────────────────┐
│  Your Layer (UWAGGS)  │  <-- Synthesizes granular shots into 
│ Groups into Archetypes│      overarching "Rally Archetypes".
└───────────┬───────────┘
            │  (Calculates Cluster Distributions)
            ▼
┌───────────────────────┐
│   The Sports Payoff   │  <-- Automates player fingerprints 
│  Player Fingerprints  │      and maps dynamic counter-strategies.
└───────────────────────┘
---

### The Strategic Payoff: Tactical Counters
Scaling the dataset via CV unlocks true **counter-styling analytics**. By assigning cluster distributions to hundreds of unannotated matches, we can evaluate **Win-Rate Pivots** when different styles collide. 

Rather than viewing player style as a static percentage, a coach can use this automated tool t

## Authors

Ryan — UWAGGS | University of Waterloo
