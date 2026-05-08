# Deep Research — Question Topology Library

> **Sub-skill of `deep-research-architecture.md`.** This file documents the 8 question topology patterns + how to pick one per scenario + how to combine multiple topologies within a single deep-research run. Load this file when designing the per-fork question trajectory in MASTER PATTERN, or when an existing fork's output is feeling redundant and you need a different angle.

🔴 **All calls in any topology MUST include the anti-clarification directive** (see `deep-research-architecture.md` near top). Without it, the bot defaults to clarification regardless of which topology you've chosen.

---

## 🎯 Question Topology — the 3rd design dimension

Pavel's insight: *„Strategie, jak se ptát a kam směřovat těma otázkama, je stejně důležitá, jako kolik otázek položím a v jakých větvích."*

Deep research has THREE design dimensions, not two. Pick all three:

| Dimension | Choices | Example |
|-----------|---------|---------|
| **Branch Type** | A continuous / B stateless / C parallel / D persona / E spawned | „Type A continuous chat" |
| **Strategy** | A continuous / B steered forks / C lens rotation / D spawn-on-mention / E shape rotation | „Strategy B with 3 forks" |
| **Question Topology** | Kruh / Hloubka / Šířka / Spirála / Strom / Zigzag / Inverze / Meta | „Spirála trajectory" |

The Branch Type tells you the API parameters. The Strategy tells you how branches connect. The Question Topology tells you **how questions trajector within a branch.**

### 8 Question Topology Patterns

#### 🔵 KRUH (Concentric questioning) — Pavel's „kruh"

Same topic, rotating frames around one center. Each turn looks at the same thing differently.

```
Q1: Tell me about X (broad)
Q2: From a beginner's view about X
Q3: From an expert's view about X  
Q4: From a critic's view about X
Q5: From a contrarian's view about X
Q6: Synthesis — what's true regardless of view?
```

**Architectural mechanism:** Pinecone returns similar chunks each turn (similar effective query), but LLM is forced to RE-FRAME by the rotating perspective. Output: comprehensive coverage of a single topic from many angles, like sculpting around a central core.

**Best for:** Single principle that needs multi-perspective treatment. Pricing, coaching philosophy, marketing fundamentals.

**Combine with:** Type D (replace mode) for stronger lens forcing per turn.

#### ⬇️ HLOUBKA (Drill-down) — Pavel's „hloubka"

Progressive narrowing. Start broad, end at the most specific concrete.

```
Q1: What is X? (broad)
Q2: What's the most important sub-aspect of X?
Q3: How does THAT sub-aspect work mechanically?
Q4: What's the most common failure mode?
Q5: Concrete numerical example?
Q6: Code-level/operational specifics?
```

**Architectural mechanism:** Each turn's query becomes more specific → effective embedding shifts toward specialized chunks → retrieval surfaces deeper material. Memory accumulation amplifies — turn 6's effective query includes all prior context.

**Best for:** Mastering one principle to operational depth. Implementing a framework.

**Combine with:** Type A continuous (memory accumulation), Strategy E (shape rotation amplifies).

#### ➡️ ŠÍŘKA (Lateral / Breadth) — Pavel's „šířka"

Cover the whole topic surface, no drilling.

```
Q1: What are 7 main sub-areas of X?
Q2: Tell me about sub-area 1 (300 words)
Q3: Tell me about sub-area 2 (300 words)
Q4: Tell me about sub-area 3 (300 words)
Q5-7: ...continue across all sub-areas
```

**Architectural mechanism:** Each turn switches sub-topic → different chunks retrieved → broad surface coverage. Memory carries the „survey" intent.

**Best for:** Knowledge audit, building Q&A reference, executive summary needing breadth.

**Combine with:** Strategy A (continuous chat carries the survey context) or Strategy B (forks per sub-area for true depth+breadth).

#### 🌀 SPIRÁLA (Spiral) — Pavel's „spirála"

Alternates broad and deep with returns to center. The most powerful general-purpose pattern.

```
Q1: Broad on X
Q2: Deep on X.A (drill)
Q3: Back to broad — what's another aspect?
Q4: Deep on X.B (drill different aspect)
Q5: Back to broad — how do A and B relate?
Q6: Deep on X.C
Q7: Back to broad — meta-pattern across A, B, C?
Q8: Deep on synthesis itself — what's the underlying mechanism?
```

**Architectural mechanism:** Combines breadth (Q1, Q3, Q5, Q7 explore aspects), depth (Q2, Q4, Q6 drill specific), AND synthesis (Q5, Q7 force connection). Memory accumulates rich substrate.

**Best for:** Comprehensive deliverable that's both broad and deep. Production research output.

**Combine with:** Type A continuous (memory critical for synthesis turns).

#### 🌳 STROM (Tree / Branching from each node)

Each significant answer spawns its own branch.

```
Q1 (main): Top 3 principles? → bot mentions A, B, C
   ↓
Branch A: 5 turns deep on A
Branch B: 5 turns deep on B (FORK)
Branch C: 5 turns deep on C (FORK)
   ↓
Synthesis: Q (back in main): How do A, B, C relate? (uses no branch context)
```

**Architectural mechanism:** Multiplies coverage by branching. Each branch independent retrieval (Strategy B forking).

**Best for:** Maximum coverage when topic decomposes cleanly into independent sub-areas.

**Combine with:** Strategy D (spawn-on-mention) — bot's own mentions trigger forks.

#### ⚡ ZIGZAG (Compare-and-contrast)

Forces dialectical exposition through contrasting pairs.

```
Q1: How do BEGINNERS approach X?
Q2: How do EXPERTS approach X?
Q3: What's the GAP between them?
Q4: How do ROOKIES fail at X?
Q5: How do MASTERS succeed at X?
Q6: What's the bridge from rookie to master?
Q7: What ONE shift makes the difference?
```

**Architectural mechanism:** Each contrasting pair forces principle-surfacing. The bot can't stay in one frame — each pair demands articulation of the difference.

**Best for:** Teaching content, before/after framing, transformation-style coaching content.

**Combine with:** Type A continuous, Strategy E shape rotation.

#### 🔄 INVERZE (Reverse / Negative space)

Start from failure, find truth.

```
Q1: What does NOT work in X? Common mistakes.
Q2: WHY exactly do those fail? Mechanism of failure?
Q3: What's the OPPOSITE — what does work?
Q4: WHY does that work? Mechanism of success?
Q5: Where's the line between failure and success?
Q6: What rules govern that line?
```

**Architectural mechanism:** Failure analysis often retrieves DIFFERENT chunks than success analysis (failure case studies vs framework chunks). Both perspectives surface principles, but inverze surfaces them more sharply by negation.

**Best for:** Edge case work, troubleshooting guides, „common mistakes" content.

**Combine with:** Type A continuous, can pair with Spirála.

#### 📍 META (Questioning the answer)

Ask about the bot's own answers.

```
Q1: Tell me about X (broad)
Q2: What in your last answer was MOST important?
Q3: What did you NOT say that I should know?
Q4: If you had to delete 80% of your answer, what survives?
Q5: What's the underlying principle behind everything you said?
Q6: What did your answer ASSUME that I should question?
```

**Architectural mechanism:** Meta-questions don't bring new chunks — they force LLM to consolidate from existing context (memory window). Surface priorities, blind spots, hidden assumptions.

**Best for:** Synthesis turns at end of any pattern. Catching what the bot left out. Teaching the bot's reasoning.

**Combine with:** ANY pattern — meta is universal as final synthesis tool.

### Topology selection per scenario

| Scenario | Recommended topology |
|----------|---------------------|
| Universal how-to guide | Spirála (broad + deep alternation) |
| Person + product specific | Strom (branching per person/product aspect) |
| Comparative analysis | Zigzag (contrasting frames) |
| Audience-tiered content | Kruh (rotate audience lens) |
| Single principle exhaustive | Hloubka (drill) + Inverze (failure modes) |
| Knowledge audit | Šířka (lateral coverage) |
| Content draft (blog/script) | Spirála or Hloubka with final Meta |
| Q&A reference build | Šířka + Meta synthesis at end |

### Within a single Type A branch — combine topologies

A 20-turn Type A chat doesn't have to use ONE topology. Combine:

```
Turns 1-3: Šířka (lay out the territory)
Turns 4-8: Hloubka (drill into top principle)
Turns 9-12: Inverze (failure modes of that principle)
Turns 13-15: Šířka return (other principles overview)
Turns 16-18: Spirála (deep on each, with returns)
Turns 19-20: Meta (consolidate, find essence)
```

**This is the master pattern.** Topology rotation within a single chain produces dramatically richer output than any single topology held throughout.


---

> **Cross-references:**
> - Top-level architecture: [`deep-research-architecture.md`](./deep-research-architecture.md)
> - Scenario-by-scenario topology assignments: [`deep-research-scenarios.md`](./deep-research-scenarios.md)
> - Default blueprint (MASTER PATTERN) topology fork assignments: [`deep-research-blueprints.md`](./deep-research-blueprints.md)

*Last updated: 2026-05-08 (v0.9.4 — extracted as sub-skill from monolithic deep-research-architecture.md)*
