# Deep Research — Scenario Adaptations

> **Sub-skill of `deep-research-architecture.md`.** Deep research is a base method that adapts per scenario. This file documents 7 scenarios (universal how-to, person+product, comparative, audience-tiered, content draft, knowledge audit, single principle) with their pipeline configurations + Track A external research integration. Load this file when the user's request matches a scenario trigger.

🔴 **Every scenario assumes:**
1. **Anti-clarification directive on every API call** (see top of `deep-research-architecture.md`).
2. **MASTER PATTERN as the structural base** when 3+ angles exist. Each scenario specifies WHICH topologies and WHICH roles — execution shape stays the same.
3. **Synthesis happens in Claude Code, not BuddyPro.** Each fork is raw material; final document composed by orchestrator.
4. **Minimum 6 calls per stage** — no 3-5 call mini-probes (deprecated).

---

## 🎯 Scenario Adaptation — pipeline per scénář

Deep research is a **base method** that adapts to the scenario. Here are the most common scenarios with their pipeline configurations.

🔴 **All scenarios assume:**
1. **Anti-clarification directive on every API call** (see top of this file). Without it, ~30-50% of returned content is clarifying noise.
2. **MASTER PATTERN as the structural base** when 3+ angles exist (3 forks × 6-8 turns × different topology × different role). Each scenario below specifies WHICH topologies and WHICH roles to use — but the underlying execution shape stays the same.
3. **Synthesis happens in Claude Code, not BuddyPro.** Each fork is raw material; the final document is composed by the orchestrator.
4. **Minimum 6 calls per stage** — no 3-5 call mini-probes (deprecated).

### Scenario 1 — „Universal how-to" / Generic methodology

**Trigger examples:**
- *„Universal guide on running webinars"*
- *„How to write a high-converting sales page"*
- *„The complete playbook for cold outreach"*

**🟢 DEFAULT pipeline = MASTER PATTERN** (empirically validated, lowest token cost, highest divergence per call):

```
3 forks × 6-8 turns each × different topology × different role persona
+ ANTI_CLARIFICATION_DIRECTIVE on every call
+ synthesis (Claude Code, not BuddyPro)
```

**Concrete shape for „Universal how-to":**

| Fork | Topology | Role persona (REPLACE-mode add-on) | Why this combination |
|------|----------|------------------------------------|----------------------|
| Fork 1 | HLOUBKA (drill-down) | Default expert voice | Goes operational depth on the topic's core mechanism |
| Fork 2 | ŠÍŘKA (lateral) | Strategist persona | Maps the whole surface area (sub-topics) |
| Fork 3 | INVERZE (anti-pattern lens) | Critic persona | Extracts failure modes, anti-patterns, common mistakes |

Each fork = stable `user` (e.g., `research-{slug}-fork{N}-{timestamp}`), 6-8 turns, accumulating memory within fork. All forks run in parallel (each call ~3-5s with `time.sleep(3)` between calls in same fork to respect rate limit).

**Total budget:** 18-24 calls / ~$0.90-1.20 / 3-5 min wall-clock (parallelized) / ~6000-9000 words of source material.

**🟡 FALLBACK pipeline (when MASTER PATTERN doesn't fit):** if the topic is too narrow to differentiate 3 distinct angles (e.g., owner asks for one specific framework, not a whole methodology), fall back to single Type A continuous chat 12-15 turns with HLOUBKA topology + anti-clarification directive. ~12-15 calls / ~$0.75 / 3-4 min.

**🟠 EXHAUSTIVE pipeline (when owner wants the deluxe doc):** the 6-stage 40-65 call blueprint described later in this file under „Optimal blueprint (architecture-grounded, exhaustive variant)". Use only when owner explicitly says „make it comprehensive" or budget allows.

- Output strategy: **Principle Compendium** — sections per principle/framework, with each fork's contribution visible in the synthesis.
- Synthesis happens in Claude Code, not BuddyPro. Each fork produces raw material; Claude Code weaves the final document.

**External research:** Usually NOT needed. The bot's KB IS the answer.

### Scenario 2 — „Person + product specific" (most demanding)

**Trigger examples:**
- *„Prepare a launch strategy for [Maria Novak] and her course [The Course Name] — here's her site: example.com"*
- *„Write a webinar script for John Smith promoting his SaaS tool"*
- *„Draft a sales letter for [product link] that fits this expert's style"*

**Pipeline (DUAL-TRACK — both run before synthesis):**

```
TRACK A — External research (Claude Code, WebFetch/WebSearch):
  • Read person's website (about, services, pricing, voice)
  • Read product page (positioning, audience, USP, social proof)
  • Optionally: scan recent posts/content for tone, recurring themes
  • Output: PersonProfile + ProductProfile context blocks
                    ↓
TRACK B — Internal research (BuddyPro Owner API, this architecture):
  • Phase 2 (Type B broad): how to do X for [audience type matching person]
  • Phase 4 (Type A deep): top 3 principles relevant to person's domain
  • Phase 5 (Type C variance): „what makes a launch succeed in this niche?"
  • Phase 6 (Type D persona): „how would a [audience tier] react?"
                    ↓
SYNTHESIS (Phase 7):
  • Merge Track A's person/product context with Track B's expert principles
  • Tailor every recommendation to the specific person/product
  • Use voice rules from PersonProfile if available
  • Output strategy: depends on deliverable (webinar script / sales letter / launch plan)
```

**Why dual-track:** the bot doesn't know about the specific person. It knows the principles. Claude Code's web research provides the person/product specifics. Synthesis is where they meet.

**External research budget:** ~5-10 WebFetch/WebSearch calls (~free with Claude Code). Internal research: same as standard deep research (~25 calls / $1.25).

### Scenario 3 — „Comparative analysis"

**Trigger examples:**
- *„Compare 3 frameworks for sales pages"*
- *„What are the 5 different approaches to email funnels"*
- *„Show me different pricing strategies and when each works"*

**Pipeline:**
- Phase 1: Plan with discrete-options focus (each option = one branch)
- Phase 2: One Type A continuous chat per option (deep on each separately)
- Phase 3: Principle detection PER option (what makes each unique)
- Phase 5: Type C variance probe with the question „when would each option win?"
- Phase 7: Synthesis → **Comparative Analysis** strategy
- Output: side-by-side comparison, when-to-use-each table, hybrid recommendations

**External research:** Optional — could pull benchmark data if owner has external sources.

### Scenario 4 — „Audience-tiered guidance"

**Trigger examples:**
- *„Explain X for beginners vs intermediates vs pros"*
- *„How would you teach this to a 10-year-old vs an MBA?"*
- *„Write 3 versions of this advice for different sophistication levels"*

**Pipeline:**
- Phase 1: Plan with audience tiers as primary dimension
- Phase 2: Skip generic broad probe — go straight to Type D
- Phase 4: Type D parallel branches, one per tier (replace prompt with tier persona)
- Phase 5: Skip variance (variance IS the deliverable)
- Phase 7: Synthesis → **Tier-Based Playbook** strategy
- Output: same content rendered 3 ways for 3 audiences

### Scenario 5 — „Content draft with depth" (newsletter, blog, video script)

**Trigger examples:**
- *„Use my BuddyPro to write my newsletter on X"*
- *„Draft a 10-minute YouTube video script on X"*
- *„Write me a long-form blog post on X"*

**Pipeline:**
- Phase 2 broad: angles relevant to content format (hook, structure, examples, CTAs)
- Phase 4 deep: 3-5 strongest insights to anchor the piece
- Phase 6 persona: for content output, persona = target audience of the content
- Phase 7: Synthesis → output format matches deliverable (script structure / newsletter sections / blog flow)
- Output: ready-to-publish draft in expert's voice

**Voice fidelity:** since this goes out as the expert's own content, faithfulness to their voice matters. Don't over-edit — let the bot's words carry through.

### Scenario 6 — „Knowledge audit / Q&A reference build"

**Trigger examples:**
- *„Extract a Q&A reference of everything you know about X"*
- *„Build a knowledge base of FAQs from my bot's expertise"*
- *„Document all the principles you've taught me on X"*

**Pipeline:**
- Phase 2 broad: angles = topics
- Phase 3: Principle extraction (these become the Q's)
- Phase 4 deep: each principle → A turn pulls definition + example + mistake (these become the A's)
- Phase 7: Synthesis → **Q&A Reference** strategy
- Output: list of crisp Q&A pairs, each ≤200 words

### Scenario 7 — „Single principle, exhaustive"

**Trigger examples:**
- *„Tell me everything you know about [specific framework]"*
- *„Do a deep dive on [specific concept]"*
- *„Exhaustive guide to one specific topic"*

**Pipeline:**
- Skip Phase 2 broad — already know the principle
- Phase 4 Type A continuous chat with 8-15 turns deep on the one principle
- Phase 5 Type C variance probe (does the principle work the same in all contexts?)
- Phase 7: Synthesis → **Principle Compendium** but for ONE principle, sectioned by aspects

### Scenario auto-detection logic

```
Owner mentions a SPECIFIC PERSON or PRODUCT (URL, name, brand)?
  → Scenario 2 (Person + product specific) — dual-track external + internal
  
Owner asks for COMPARISON of N options?
  → Scenario 3 (Comparative analysis)

Owner mentions multiple AUDIENCE TIERS / SKILL LEVELS?
  → Scenario 4 (Audience-tiered guidance)

Owner wants CONTENT DRAFT (newsletter, blog, video, sales letter)?
  → Scenario 5 (Content draft)

Owner asks for FAQ / Q&A / KNOWLEDGE BASE format?
  → Scenario 6 (Knowledge audit)

Owner asks about ONE SPECIFIC concept/framework?
  → Scenario 7 (Single principle exhaustive)

Otherwise (generic „how to do X" or „guide on Y")?
  → Scenario 1 (Universal how-to)
```

The pipeline executes **the same 8 phases** but with adapted budgets and branch type choices.

## External Research Integration (Track A details)

When the scenario involves a specific person/product, Claude Code's external research happens BEFORE the BuddyPro phases.

### What to fetch

For a **person**:
- Their main website (homepage, about, services)
- Top 1-3 content pieces (recent blog/newsletter — for voice)
- LinkedIn/Twitter profile (for credentials, audience)
- Pricing page if available

For a **product**:
- Product page (positioning, USP, audience)
- Pricing/checkout (price points, tiers)
- Testimonials/reviews (objections, common praise)
- Competitor positioning (search „[product] vs ..." if applicable)

### Tools Claude Code uses for Track A

- `WebFetch` — fetch single page, extract focused content
- `WebSearch` — find related pages
- `mcp__exa__exa_search` — deep web search if available

### Output of Track A — context block for synthesis

Structured data Claude Code passes to synthesis:

```json
{
  "person": {
    "name": "Maria Novak",
    "expertise": "online business coaching",
    "audience": "solopreneurs and freelancers",
    "voice_notes": ["uses 'darling' as endearment", "short sentences", "personal stories"],
    "credentials": ["10 years coaching", "$2M+ in client revenue"],
    "current_offers": ["12-week mastermind ($5k)", "1:1 coaching ($500/mo)"]
  },
  "product": {
    "name": "The Launch Method",
    "positioning": "step-by-step course for first-time launchers",
    "price": "$497 one-time",
    "audience": "newer entrepreneurs about to launch first product",
    "social_proof": ["Used by 200+ students", "Featured on XYZ podcast"]
  }
}
```

This block becomes part of the synthesis prompt: „Using these insights from the bot's KB, tailored to THIS person and THIS product..."

### When external research is NOT appropriate

- Universal how-to (Scenario 1) — no specific person, KB principles ARE the deliverable
- Comparative (Scenario 3) — usually pure principles, but external could enrich
- Knowledge audit (Scenario 6) — internal KB only


---

> **Cross-references:**
> - Top-level architecture: [`deep-research-architecture.md`](./deep-research-architecture.md)
> - Topology library (which patterns to assign per fork): [`deep-research-topologies.md`](./deep-research-topologies.md)
> - Default blueprints (MASTER PATTERN + exhaustive variant): [`deep-research-blueprints.md`](./deep-research-blueprints.md)

*Last updated: 2026-05-08 (v0.9.4 — extracted as sub-skill from monolithic deep-research-architecture.md)*
