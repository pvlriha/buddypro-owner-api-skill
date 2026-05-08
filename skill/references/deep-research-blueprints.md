# Deep Research — Blueprints, Pipeline & Implementation

> **Sub-skill of `deep-research-architecture.md`.** The advanced 8-phase pipeline (used in the EXHAUSTIVE blueprint), coordination & resource management, knowledge graph for synthesis, synthesis strategies, full implementation skeleton, branch-type decision tree, and privacy/cost summary. Load this when running the EXHAUSTIVE deep-research variant or when implementing the orchestration code from scratch.

🔴 **Default blueprint = MASTER PATTERN** (3 forks × 6-8 turns × different topology × different role). Use this file's 8-phase pipeline ONLY when the user explicitly asks for the EXHAUSTIVE / „deluxe" variant. For 95% of jobs, MASTER PATTERN is enough — see `deep-research-architecture.md` § „Optimal blueprint".

---

## The 8-Phase Pipeline

```
PHASE 1: PLAN
  ↓ Topic decomposition
  ↓ Budget allocation (max calls, max parallelism)
  ↓ Output target spec (format, length, tone)
  
PHASE 2: BROAD PROBE
  ↓ Spawn 5-8 Type B branches in parallel
  ↓ Each covers a different angle (situations, audiences, frameworks, edge cases)
  ↓ ~5-8 calls in 30-60 seconds
  
PHASE 3: PRINCIPLE DETECTION
  ↓ Scan all answers for:
  ↓   - Named frameworks (capitalized multi-word terms, "X Method", "Y System")
  ↓   - Recurring concepts (mentioned in 2+ answers)
  ↓   - Strong claims (numbers, specific advice, rules)
  ↓ Output: list of principles ranked by importance
  
PHASE 4: DEEP DIVE
  ↓ For each top-N principle: spawn Type A or Type E branch
  ↓ Run 3-5 turns per branch
  ↓ Run multiple branches in parallel where possible
  ↓ ~10-15 calls
  
PHASE 5: PERSPECTIVE PROBE (CONDITIONAL — use only when variance is expected)
  ↓ FIRST: send the candidate question to 1 user profile
  ↓ IF answer feels decisive/canonical → SKIP Phase 5 entirely
  ↓ IF answer hints at multiple approaches → Type C with 3-4 users
  ↓ NEVER blindly run Type C with 5+ users on a single-answer question
  ↓ ~0-4 calls (often 0)
  
PHASE 6: PERSONA ANGLES (optional, for content generation)
  ↓ For each output audience tier: spawn Type D
  ↓ Get tier-specific framing of the topic
  ↓ ~2-3 calls
  
PHASE 7: SYNTHESIZE (Claude Code is the writer here, not BuddyPro)
  ↓ Cluster all answers by theme
  ↓ Dedupe near-identical content
  ↓ Identify consensus (everyone says X)
  ↓ Flag contradictions (branch A says X, branch B says ¬X)
  ↓ Construct DOCUMENT (not transcript) in target output format
  ↓ The bot's frameworks/phrases/examples are preserved; structure is Claude's
  
PHASE 8: REFINE
  ↓ Show document to owner
  ↓ Owner feedback → spawn targeted Type B branches
  ↓ Weave new content in
  ↓ Repeat until owner approves
```

## Coordination & Resource Management

### Budget cap

Every research session has a hard cap on calls. **Deep research is not cheap on calls — quality requires depth.** Realistic budget tiers:

- **Quick**: 15-20 calls / ~$1 / ~2 min — surface coverage, single perspective
- **Standard**: 30-40 calls / ~$2 / ~4 min — default for most deep research, good multi-angle coverage
- **Deep**: 50-75 calls / ~$3-4 / ~6-8 min — comprehensive multi-perspective with thorough drill-downs
- **Exhaustive**: 80-150 calls / ~$4-7.50 / ~10-15 min — owner explicitly opts in for client deliverables, books, master courses

**Why higher than you'd guess:**
- Phase 2 broad needs 8-12 angles minimum for true breadth
- Phase 4 deep dive on each principle = 5-10 turns × 5 principles = 25-50 calls
- Phase 5 perspective probe with 5-7 user profiles (not 3-4) = better variance signal
- Phase 6 persona angles often need 3-5 personas, each with follow-ups
- Phase 8 refinement may need re-probing — keep 10% reserve

The orchestrator distributes the budget — **DEPTH-BIASED** (depth > breadth in most cases):

- Phase 1 (plan + decompose) — 1-2 calls
- Phase 2 (broad probe) — 15-20% of budget (5-8 angles, just enough to identify principles)
- Phase 3 (principle detection) — 0 calls (analysis on existing answers)
- Phase 4 (DEEP DIVE) — **55-65% of budget** (largest allocation — Type A continuous chat per principle, 5-10 turns each)
- Phase 5 (perspective probe) — 0-10% of budget (CONDITIONAL — only when variance expected; often skipped)
- Phase 6 (persona angles) — 5-10% of budget when relevant (skip for some scenarios)
- Phase 7 (synthesis) — 0-2 calls (mostly Claude Code, occasional clarification call)
- Phase 8 (refine reserve) — 5-10% buffer

**Concrete example for Standard tier (35 calls):**
- Plan: 1 call
- Broad probe: 7 calls (7 angles to identify principles)
- Deep dive: 20 calls (4 principles × 5 turns each — DEPTH FOCUS)
- Perspective: 0-3 calls (only if variance probe-shows it's worth doing)
- Persona: 3 calls (3 audience tiers IF deliverable needs tier framing)
- Refine reserve: 2-4 calls

**Concrete example for Deep tier (60 calls):**
- Plan: 2 calls
- Broad probe: 10 calls (10 angles)
- Deep dive: 36 calls (6 principles × 6 turns each)
- Perspective: 4 calls (variance probe for the central question)
- Persona: 4 calls (4 audience tiers)
- Refine reserve: 4 calls

### Parallelism

API rate limit is 30 req/min per key. Safe parallelism = max 5 concurrent + 3s sleep between batches. So:
- Burst of 5 parallel calls
- Wait 12 seconds (lets quota replenish)
- Next burst

Time per phase:
- Phase 2 broad (8 calls) → ~25 seconds
- Phase 4 deep (15 calls) → ~50 seconds
- Phase 5 perspective (4 calls) → ~15 seconds
- Total roughly 1.5–3 minutes per research session

### Stop conditions

Stop a branch (or the whole session) when:
- **Budget exhausted** — hit max calls
- **Saturation** — last 2 answers in a Type A branch contain >70% repeat content
- **Gap-check passed** — Phase 7 synthesis surfaces no critical missing piece
- **Time cap hit** — explicit timeout (default 5 min)
- **Error rate high** — >20% of calls fail → abort, report

### Error recovery

Per-call retry: 3 attempts with exponential backoff for 429/5xx. If a branch's calls keep failing, mark branch as „incomplete" but don't abort whole session. Synthesis includes „Coverage gaps:" section.

## Knowledge Graph

The orchestrator maintains an in-memory knowledge graph throughout the session:

```
{
  "topic": "SaaS pricing for solo founders",
  "branches": {
    "broad-pricing-tactics-1": {
      "type": "B",
      "calls": [{...}],
      "themes_extracted": ["value-based pricing", "anchor effect"],
    },
    "broad-customer-segmentation-2": {
      "type": "B",
      ...
    },
    "deep-value-based-pricing": {
      "type": "A",
      "parent_principle": "value-based pricing",
      "spawned_from": "broad-pricing-tactics-1",
      "turns": [{...}, {...}, {...}],
    },
    "perspective-A-pricing": {"type": "C", ...},
    "perspective-B-pricing": {"type": "C", ...},
  },
  "principles_detected": [
    {"name": "value-based pricing", "branches_mentioning": ["broad-1", "broad-3"]},
    ...
  ],
  "themes_clustered": [...],
  "consensus_points": [...],
  "contradictions": [
    {
      "claim": "trial period should be 14 days",
      "supporting_branches": ["broad-1"],
      "claim_alt": "trial period should be 30 days",
      "supporting_branches_alt": ["perspective-A"],
    }
  ],
}
```

This graph is the input to Phase 7 synthesis.

## Synthesis Strategies

The orchestrator picks a synthesis strategy based on what the graph reveals:

### Strategy: Consensus Document
**When:** Phase 5 perspective probe shows convergence, no major contradictions.
**Output:** Confident, single-narrative document. „The expert says X, illustrated by Y, with caveat Z."

### Strategy: Comparative Analysis  
**When:** Phase 5 reveals significant variance, OR contradictions detected in Phase 7.
**Output:** Document explicitly presents multiple perspectives. „There are two schools of thought: A (when ...) vs B (when ...)."

### Strategy: Principle Compendium
**When:** Phase 4 deep dives produced rich material on many distinct principles.
**Output:** One section per principle, each with definition + example + mistake to avoid + when to apply.

### Strategy: Tier-Based Playbook
**When:** Phase 6 persona angles produced rich tier-specific framings.
**Output:** Same content for 3 tiers (beginner / pro / VIP), reading naturally to each audience.

### Strategy: Q&A Reference
**When:** Owner specifically wants this format (e.g., for a knowledge base, FAQ, internal wiki).
**Output:** List of crisp Q&A pairs — one per principle, max 200 words per answer.

## Implementation skeleton

Full code is in `code-recipes.md` Pattern 9. Skeleton:

```python
async def deep_research_advanced(topic: str, budget: str = "standard", output_strategy: str = None) -> str:
    session = ResearchSession(topic, budget_cap=BUDGET_CAPS[budget])
    
    # Phase 1
    plan = await session.phase_plan()
    
    # Phase 2 (parallel)
    broad_branches = await session.phase_broad_probe(angles=plan.angles[:8])
    
    # Phase 3
    principles = session.phase_detect_principles(broad_branches)
    
    # Phase 4 (parallel)
    deep_branches = await session.phase_deep_dive(top_principles=principles[:5])
    
    # Phase 5 (parallel)
    perspective_branches = await session.phase_perspective_probe(top_question=plan.core_question)
    
    # Phase 6 (optional, parallel)
    if plan.requires_personas:
        persona_branches = await session.phase_persona_angles(personas=plan.personas)
    
    # Phase 7
    synthesis = session.phase_synthesize(strategy=output_strategy or auto_pick_strategy(session))
    
    # Phase 8 (loop with owner)
    while True:
        owner_feedback = present_to_owner(synthesis.document)
        if owner_feedback.approve:
            return synthesis.document
        synthesis = await session.phase_refine(feedback=owner_feedback)
```

## Branch Type Decision Tree

When the orchestrator (or owner) decides what branch type to use for a given inquiry:

```
What are you trying to learn?
│
├─ Quick broad coverage (multiple angles, no need to go deep)
│   → Type B (one-shot stateless), parallel batch
│
├─ Deep on ONE specific principle/concept
│   → Type A (continuous chat), 5-10 turns
│
├─ Want to see if expert advice varies by context/perspective
│   → Type C (multiple users in parallel), same question to N profiles
│
├─ Need persona-specific framing for content output
│   → Type D (replace system prompt), per persona
│
├─ Something interesting was just mentioned, dig in
│   → Type E (spawn drill-down), pick A or B execution per need
│
└─ Refining an existing document
    → Targeted Type B (no need to re-establish context)
```

## When to use this advanced system vs Pattern X3

| Scenario | Use |
|----------|-----|
| Owner wants a quick topic exploration, single document, ~30 seconds | **Pattern X3** (use-cases.md) — sequential, simple |
| Owner wants comprehensive multi-perspective document, willing to wait 2-3 minutes | **This architecture** — branch & merge |
| Topic is straightforward, single principle dominates | **Pattern X3** |
| Topic is complex, multiple frameworks compete, audience is mixed | **This architecture** |
| Budget is limited (<$0.50) | **Pattern X3** |
| Quality > cost | **This architecture** |
| Demo/prototype | **Pattern X3** |
| Production deliverable for client | **This architecture** |

The simpler X3 pattern is the right tool 80% of the time. This advanced architecture is the right tool when the task warrants it.

## Open design questions (for future iteration)

This architecture is the v1 design. Areas to refine through use:

1. **Principle detection quality** — currently relies on text patterns. A small-model embedding pass might surface principles more reliably.
2. **Saturation detection** — „last 2 answers >70% repeat" is heuristic. Better: semantic similarity threshold.
3. **Cross-branch contradiction detection** — currently text-pattern. Could use second LLM pass to compare claim pairs.
4. **Adaptive budget reallocation** — if Phase 4 reveals an unexpectedly rich principle, redirect Phase 5 budget to deepen it instead.
5. **Owner-in-the-loop preferences** — let owner mark branches mid-flight: „skip this", „go deeper here", „this is the most important angle".
6. **Multi-instance research** — when owner has access to multiple BuddyPro instances (Pavel's ecosystem has 7+), federate research across them. Each instance contributes its expertise; synthesis cross-references.

## Privacy & cost summary

| Budget tier | Calls | Cost | Time | Memory written |
|-------------|-------|------|------|----------------|
| Quick | 15-20 | ~$1 | ~2 min | ~3-5 user profiles |
| Standard | 30-40 | ~$2 | ~4 min | ~10-12 user profiles |
| Deep | 50-75 | ~$3-4 | ~6-8 min | ~15-20 user profiles |
| Exhaustive | 80-150 | $4-7.50 | 10-15 min | ~25+ user profiles |

Memory note: each Type A/C branch creates a `user` profile. Profiles persist (no public delete API). For sensitive research: use stateless mode for all branches (sacrifices some quality but leaves no trace).

## Reference

- **Inspiration:** `claude-buddy-connection/conversation_orchestrator/` — multi-party Telegram orchestrator (uses Telethon, not Owner API). This file adapts its concepts for Owner-API-only deployments with single-bot, multi-branch design.
- **Related:** `use-cases.md` Pattern X3 (simpler version), `api-features-deep-dive.md` (foundation), `code-recipes.md` Pattern 8 (basic deep research code), Pattern 9 (advanced — see `code-recipes.md`).

*Last updated: 2026-05-07 (v0.7.0)*

---

> **Cross-references:**
> - Top-level architecture: [`deep-research-architecture.md`](./deep-research-architecture.md)
> - Topology library: [`deep-research-topologies.md`](./deep-research-topologies.md)
> - Scenario adaptations: [`deep-research-scenarios.md`](./deep-research-scenarios.md)

*Last updated: 2026-05-08 (v0.11.0 — multi-instance support release from monolithic deep-research-architecture.md)*
