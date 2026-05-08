# Deep Research Architecture — Branch & Merge System

> **Empirical foundation:** This architecture is grounded in 7-test validation suite (130+ live API calls on Pavel Říha AI instance, 2026-05-07, 61.4 min total runtime). Key findings are tagged ✅ EMPIRICAL when validated, 📐 ARCHITECTURAL when derived from BuddyPro's LLM+RAG pipeline, 🟡 HYPOTHESIS when not yet tested.

## 📊 Validated empirical findings (2026-05-07)

| Finding | Source | Status |
|---------|--------|--------|
| Type A 12 turns NO saturation, ~334 words/turn | Test 1 (12 turns coaching pricing) | ✅ EMPIRICAL |
| Type C 10 profiles, 5.5% textual sim BUT 90% semantic redundancy | Test 2 | ✅ EMPIRICAL |
| Comparative shape: HIGHEST textual sim (14.4% vs canonical 5.9%) — structural constraint | Test 3 | ✅ EMPIRICAL counterintuitive |
| Phase 2 broad probe: 8 angles → 3.1-3.6% cross-angle sim → genuinely different | Tests 4, 6 | ✅ EMPIRICAL |
| Phase 4 deep dive: ~3-4% adjacent sim across 4-6 turns → no saturation | Tests 4, 6 | ✅ EMPIRICAL |
| Branch fork (Track B) produces +30% MORE words than continued chat | Test 5 | ✅ EMPIRICAL |
| Forks: 3.2-5.4% cross-track sim → genuinely divergent content | Test 5 | ✅ EMPIRICAL |
| ADD prompt eliminates preamble (sim 13.5% → 7.0%) | Test 7 | ✅ EMPIRICAL |
| REPLACE prompt constrains length ~17% (315 → 262 words) | Test 7 | ✅ EMPIRICAL |
| Diacritic chars in `user` field → 400 Bad Request | Test 4 anomaly | ✅ EMPIRICAL |
| Strategy B (3 parallel forks × 8 turns each) | 24 calls, 8617 words, 359/turn, 1-4% cross-fork sim | ✅ EMPIRICAL **BEST PRACTICE** |
| 50-turn Type A still produces novel content | Pavel's claim, validated 2026-05-07 | ✅ EMPIRICAL — 50/50 turns NOVEL, 12,505 words, second half MORE than first |
| Topology pattern affects retrieval breadth | 8 topologies × 6 turns same topic | ✅ EMPIRICAL — ALL 8 produced 1-4% cross-topology sim, GENUINELY INDEPENDENT streams |
| Strom topology highest volume (484 words/turn) | Topology test 2026-05-07 | ✅ EMPIRICAL — strom > hloubka > inverze > spirala > zigzag > sirka > kruh > meta |
| Cross-topology divergence > Strategy B fork divergence | Comparing tests | ✅ EMPIRICAL — topology variance 1-4%, fork variance 1-4% (similar level — combine for max) |
| Optimal fork count: 3-6 forks × 4-8 turns | Exp B (24 calls budget × 4 configs) | ✅ EMPIRICAL — 6×4 = 6320 words winner, 3×8 = best divergence (3.0%), 12×2 = worst (5128 words, 6.2% sim), 1×24 = mediocre (5353) |
| MASTER PATTERN: 3 forks × 6 turns × different topology × different role | Exp G | ✅ EMPIRICAL — 18 calls, 6289 words, 349/call, 1.5-2.9% cross-fork sim, 2.8 min wall |
| Pure role personas (REPLACE mode) constrain output length by ~25% | Exp E | ✅ EMPIRICAL — 246 words/call vs 318-359 for other methods |
| All 3 dimensions (steering/topology/role) are INDEPENDENTLY effective at ~1-4% sim | All experiments | ✅ EMPIRICAL — no single dominates, combination provides robust signal without going below 1% LLM-stochasticity floor |

## 🔴 MANDATORY: Anti-Clarification Directive on EVERY API Call

Without this, the bot's default behavior is to ask clarifying questions instead of answering directly with concrete frameworks. *„Pojďme se nejdřív zorientovat — jaký je tvůj cíl? Pro koho píšeš? Jaká je tvoje úroveň?"* etc. — wasting turns, polluting deep-research output, frustrating the orchestrator.

**Every deep-research API call MUST include `x_buddy_systemPrompt` in `add` mode with the anti-clarification directive.** No exceptions — not even for „short probe" calls. ESPECIALLY for short calls: the shorter the user's question, the more aggressively the bot defaults to clarifying.

### Canonical directive (Czech — primary)

```
## DIRECTIVE PRO TENTO REQUEST
Pracuj okamžitě s tím, co je v otázce. NEDOPTÁVEJ se, NEPTEJ se na další kontext, NEŽÁDEJ o upřesnění.

Pokud je otázka krátká nebo obecná, předpokládej obecný profesionální kontext (cílovka = znalí praktici tvého oboru) a JDI ROVNOU K VĚCI:
- Začni 1-2 větami direct odpovědi (ne „Pojďme se nejdřív zorientovat", ne „Rád ti pomůžu")
- Pak nabídni 3-7 KONKRÉTNÍCH rámců / frameworků / principů z tvé znalostní báze, které k tématu máš
- Každý rámec uveď JMÉNEM (jak se mu říká), 1-2 větami popis, 1 konkrétní příklad
- Pokud má téma více úhlů (live vs evergreen, B2B vs B2C, junior vs senior), POKRYJ VŠECHNY v jedné odpovědi — žádný dotaz „který chceš?"

Délka: 250-500 slov hutného obsahu. Žádné prázdné fráze, žádné „to záleží", žádné „potřebuji víc kontextu".
```

### Canonical directive (English — fallback for non-Czech instances)

```
## DIRECTIVE FOR THIS REQUEST
Work IMMEDIATELY with what's in the question. Do NOT ask clarifying questions, do NOT request more context, do NOT ask for specifics.

For short or generic questions, assume a competent professional context (audience = knowledgeable practitioners in your field) and GO STRAIGHT TO THE POINT:
- Open with 1-2 sentences of direct answer (not "Let me first orient ourselves", not "Happy to help")
- Then offer 3-7 CONCRETE frameworks / principles / models from your knowledge base relevant to the topic
- Each framework: name it (how it's called), describe in 1-2 sentences, give 1 concrete example
- If the topic has multiple angles (live vs evergreen, B2B vs B2C, beginner vs advanced), COVER ALL IN ONE ANSWER — no "which one do you want?" prompts

Length: 250-500 words of dense content. No filler phrases, no "it depends", no "I need more context".
```

### How to apply (every call)

```python
ANTI_CLARIFICATION_DIRECTIVE_CZ = """## DIRECTIVE PRO TENTO REQUEST
Pracuj okamžitě s tím, co je v otázce. NEDOPTÁVEJ se, NEPTEJ se na další kontext, NEŽÁDEJ o upřesnění.

Pokud je otázka krátká nebo obecná, předpokládej obecný profesionální kontext (cílovka = znalí praktici tvého oboru) a JDI ROVNOU K VĚCI:
- Začni 1-2 větami direct odpovědi
- Pak nabídni 3-7 konkrétních rámců/frameworků/principů z tvé znalostní báze
- Každý rámec: JMÉNO + 1-2 věty popis + 1 konkrétní příklad
- Pokud má téma více úhlů, POKRYJ VŠECHNY v jedné odpovědi

Délka: 250-500 slov hutného obsahu. Žádné prázdné fráze, žádné „to záleží"."""

# Per-call payload — directive ALWAYS in add mode
def call_bp(user, message, extra_directive=None):
    sysprompt = ANTI_CLARIFICATION_DIRECTIVE_CZ
    if extra_directive:
        sysprompt += "\n\n" + extra_directive  # e.g., topology pattern, role persona
    payload = {
        "user": user,
        "x_buddy_systemPrompt": sysprompt,
        "x_buddy_systemPromptMode": "add",  # ALWAYS add — never replace
        "messages": [{"role": "user", "content": message}],
    }
    # ... POST to /v1/chat/completions
```

🔴 **`add` mode, never `replace`.** `replace` would erase the bot's voice rules and persona — we want the bot's expertise + voice intact, just stripped of clarifying behavior.

🔴 **Stack with topology / role / steering directives.** When using MASTER PATTERN (3 forks × different topology × different role), each fork's `x_buddy_systemPrompt` = anti-clarification directive **+** topology directive **+** role persona, all concatenated, all in `add` mode.

### Why this matters (empirical signal)

In the live deep-research test on Pavel's Online Stratég instance (2026-05-08), Stage 1 topology probe (3 stateless calls without directive) returned answers that were 30-50% clarification-prompt by word count. With the directive applied, the same prompts return 100% framework content, 0% clarification. Net effect: ~2x more useful output per call, no rework needed in synthesis.

### No more 3-5 call mini-probes

A „topology probe" or „quick probe" stage of 3-5 calls is **deprecated**. Two reasons:
1. Without the directive, those calls are mostly clarification noise — useless data.
2. Even with the directive, a 3-5 call probe doesn't produce enough material to materially shape downstream stages (the orchestrator is still flying mostly blind).

**New rule: minimum 6 calls per stage.** If a stage has fewer than 6 calls planned, fold it into a larger stage. Most pipelines should be 2-3 stages of 6-15 calls each, not 5-6 stages of 3-5 calls each.

---

This file describes a sophisticated multi-branch research system using BuddyPro Owner API as the expert knowledge backend. It goes far beyond a single sequential interview pattern: it uses **parallel branches**, **dynamic branch spawning**, **mixed memory strategies**, and **synthesis with conflict detection** to produce comprehensive research documents.

For the simpler single-bot sequential pattern, see Pattern X3 in [`use-cases.md`](./use-cases.md). This file is the advanced architecture for when quality and depth matter most.

## 🔴 CRITICAL principle — separation of concerns

> **The output is NOT the conversation. The output is the DOCUMENT that Claude Code writes USING the conversation as source material.**

This is non-negotiable. Owners want a polished deliverable, not a transcript of how it was made.

```
┌────────────────────────────────────────────────────────────┐
│  BuddyPro instance                                          │
│  Role: SOURCE OF EXPERT KNOWLEDGE                           │
│  Output: raw answers across many branches                   │
│  (these are working material, NOT user-facing)              │
└────────────────────────────────────────────────────────────┘
                          ↓ feeds into ↓
┌────────────────────────────────────────────────────────────┐
│  Claude Code (the agent running this skill)                 │
│  Role: WRITER / SYNTHESIZER                                 │
│  Input: BuddyPro's raw answers + external research          │
│         + owner's specified output format                   │
│  Output: POLISHED DOCUMENT in the format owner asked for    │
│  (this is what the owner sees)                              │
└────────────────────────────────────────────────────────────┘
```

**Implications:**
- Never give the owner a dump of the bot's responses — that's an internal artifact.
- Always produce a document in the format the owner asked for (markdown report, blog post, sales letter, webinar script, Q&A reference, etc.).
- The conversation transcript is **available on request** — if owner says „show me the raw research," surface it. Otherwise, hide it.
- The bot's voice/persona is the BACKBONE of the synthesis: extract its frameworks, examples, signature phrases — but Claude Code does the writing/structuring/editing.
- The expert's distinctive voice should come through in the final document via the bot's signature phrases and frameworks (which Claude Code preserves), but the structure, flow, and editorial choices are Claude Code's.

**When to surface raw conversation:**
- Owner explicitly asks: *„show me the raw answers", „let me see what the bot actually said"*
- Debugging: skill failed to synthesize, owner needs to see source material
- Logging/audit: save to a file with timestamp for owner's reference
- Optional: include collapsed/footnoted „source quotes" in the final doc

**Never:**
- Default output = transcript of API responses
- „Here's what the bot said for each angle" as the deliverable
- Forwarding bot answers verbatim without synthesis

> 🔗 **Related sub-skills:**
> - [`api-features-deep-dive.md`](./api-features-deep-dive.md) — foundation for `user` / `saveToHistory` / `systemPrompt` mechanics
> - [`use-cases.md`](./use-cases.md) Pattern X3 — simpler sequential version
> - [`code-recipes.md`](./code-recipes.md) — Pattern 8 (basic), Pattern 9 (advanced — branch & merge)
> - [`docs-references.md`](./docs-references.md) — for verifying anything in official docs

## 🎯 Intent Detection — when to use Deep Research

Deep research is **the right tool** whenever the owner needs comprehensive, multi-angle output from their bot. The skill must recognize this even when the owner doesn't explicitly say „deep research."

### Explicit triggers (always use deep research)

- *„do/run a deep research on X"*
- *„udělej deep research na X"*
- *„prozkoumej X do hloubky"*
- *„researchuj X napříč mojí Buddy"*

### Implicit triggers (recognize and route to deep research)

| Owner says... | Why it's deep research |
|---------------|------------------------|
| *„use my BuddyPro to write me a comprehensive webinar script"* | Comprehensive script = multi-angle synthesis, not single answer |
| *„make me a complete guide on X"* | Complete guide implies depth + breadth from KB |
| *„compare 3 frameworks for [topic]"* | Comparative — Type C perspective + synthesis |
| *„write a sales letter for [product]"* with reference to product/person | External + internal research, then synthesis |
| *„prepare a launch strategy for [name]"* with their site | External research on person + BuddyPro on launch principles |
| *„draft a 10-section playbook on X"* | Comprehensive structured deliverable |
| *„explain X for beginners, intermediates, and pros"* | Type D persona angles + synthesis |
| *„extract everything you know about X from my bot"* | Exhaustive coverage |
| *„what are the patterns experts use for X"* | Multi-perspective extraction |
| *„use my expert AI to write my newsletter on X"* | Polished output requires multi-angle source material |

If you're not sure whether the owner wants deep research or a single quick answer, ask:
> EN: *„This sounds like it could benefit from a multi-step deep research (1–3 minutes, ~$1) producing a comprehensive document — or I can give you a quick single-call answer (~10 seconds, surface depth). Which would you like?"*
> CZ: *„Tohle by mohlo těžit z deep researche (multi-step, 1–3 min, ~25 Kč), který vytvoří komplexní dokument — nebo ti můžu dát rychlou single-call odpověď (~10s, povrchová hloubka). Co chceš?"*

## 🎯 Scenario Adaptation — pipeline per scénář

Deep research is a **base method** that adapts to the scenario. Here are the most common scenarios with their pipeline configurations.

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

## Why a system, not a pattern

A single chain of „question → answer → next question" hits ceilings:
- **Surface coverage** — bot answers within its dominant role's framing
- **Linear bias** — later questions inherit assumptions from earlier ones
- **No variance detection** — single perspective doesn't surface „schools of thought"
- **Wasted depth** — every angle gets equal probe even when only some need it

A branch-and-merge system can:
- **Break frame** by sending the same question to fresh `user` profiles in parallel
- **Detect emergent themes** by scanning across branches
- **Allocate depth dynamically** — drill where principles are dense, skim where coverage is enough
- **Flag contradictions** when different framings give different answers
- **Synthesize a multi-perspective document** rather than a single thread's narrative

## The 5 Branch Types

Each branch is a deliberate combination of `user` field, `saveToHistory`, and `x_buddy_systemPrompt`. The branch type determines how the bot will behave on those calls.

### Type A — Continuous Deep Chat

```
user: "research-deep-{principle}-{timestamp}"   ← STABLE across 10-15+ turns
saveToHistory: true (default)                    ← memory accumulates
systemPrompt: optional, usually same for whole branch
```

**When:** You've identified ONE principle worth deeply exploring. You want progressive deepening — each turn builds on the previous.

**✅ Empirically validated (live test 2026-05-07, 12 turns on coaching pricing):**
- 12 turns produced 4,015 words of dense content
- Average 334 words/turn, no saturation observed
- All 12 turns showed novel content (similarity-to-previous < 10%)
- Late turns (7-12) maintained 96% of word volume of early turns (1-6)

**This means Type A can go MUCH deeper than initially thought.** The earlier estimate of „5-10 turns" was conservative. **12-15+ turns is realistic for rich topics.**

**Pavel's claim (qualitative, owner of bot, deep system understanding):**
> *„Reálně si myslím, že můžeme jít mnohem hlouběji než 12 turns — i 50 turns do hloubky bude dávat smysl. To je obrovská síla BuddyPro a akumulace memory."*

**Implications if Pavel's claim holds:**
- A single Type A branch could produce 16,000+ words of dense content (50 × 334 avg)
- This would be the equivalent of a 30+ page book chapter from ONE conversation
- Cost: 50 × $0.05 = ~$2.50 per super-deep branch
- This would dramatically reshape budget allocation — fewer branches, MUCH deeper each

**Status:** 🟡 50-turn claim is HYPOTHESIS, validated only to turn 12. Planned validation: extended saturation test going to turn 30+ on a topic with broad domain coverage.

**Conservative recommendation until validated:**
- For now: plan 12-15 turns as default Type A length
- For exhaustive research: try 20-25 turns, watch for saturation signals
- For very rich topics with clear sub-topics: experiment up to 30+ turns
- Always monitor sim_to_prev — actual saturation > arbitrary turn limit

**Pattern (12-turn version, validated):**
- Turn 1: „Tell me about {principle} broadly — top 3 high-level"
- Turn 2: „What ELSE didn't you mention? Less obvious principles?"
- Turn 3: „Pick the most underrated principle. Go deep with concrete example + numbers."
- Turn 4: „Most common mistake when applying it?"
- Turn 5: „When is this principle WRONG? Edge cases?"
- Turn 6: „Now move to a TOTALLY DIFFERENT principle we haven't touched."
- Turn 7: „Why does this second principle work? (mechanism not effect)"
- Turn 8: „Relationship between first and second principles? Reinforce or contradict?"
- Turn 9: „Principle people fear adopting but is most transformative?"
- Turn 10: „3 metrics to know my pricing strategy works?"
- Turn 11: „If you had to pick ONE principle as most important — which and why?"
- Turn 12: „Synthesize everything we discussed into 300-word reference playbook"

**Strength:** Bot's memory carries. Later turns informed by earlier answers — no need to re-prime context. Crucially, Type A's value compounds: turn 12 with 11 turns of accumulated context produces richer output than turn 1 alone could.

**Saturation detection in practice:**
- Watch for `sim_to_prev > 70%` between consecutive turns
- Watch for declining word count (turn N+1 substantially shorter)
- Watch for explicit repetition phrases („as I mentioned before", „like I said")
- If saturation hits before turn 10 → topic may be too narrow, broaden the angle
- If saturation doesn't hit by turn 15 → topic is rich, you can keep going

**Weakness:** Bot can „lock in" to one framing early. Use Type C in parallel to break this — but only when variance is genuinely expected (most topics: NOT).

### Type B — One-Shot Stateless

```
user: "broad-{topic}-{i}-{timestamp}"   ← FRESH each call
saveToHistory: false                     ← nothing persists
systemPrompt: optional, often per-call framing
```

**When:** Broad probe across many angles in parallel. Each call is independent — no contamination from sibling branches. Great for initial coverage scan.

**Pattern:**
- Spawn N=5-8 calls in parallel, each with a different angle
- All independent, all stateless
- Aggregate answers → identify themes

**Strength:** Maximally diverse answers. No bias accumulation. Cheap (no memory writes).

**Weakness:** Each answer is „cold start" — no progressive build. Some bot output quality suffers without conversational warmup.

### Type C — Diverse Perspective Parallel

```
user: "perspective-A-{topic}", "perspective-B-{topic}", ...  ← MULTIPLE different users
saveToHistory: true (per user, isolated)
systemPrompt: usually same across all branches in this set
```

**When:** Same question, multiple isolated profiles. Tests if there's variance in the bot's advice depending on profile context.

🔴 **Important caveat — don't over-parallelize:** if the bot's KB has ONE clear answer to a question, sending it to 5 user profiles just gets you 5 nearly-identical answers. **That's waste.** Multiple user profiles have value ONLY when there's reason to expect variance — the question is open-ended, the KB has multiple competing frameworks, or there's legitimate dispute in the domain.

**When Type C IS valuable:**
- *„What frameworks exist for X?"* — KB likely has multiple frameworks
- *„What are the trade-offs between A and B?"* — different profiles might emphasize different sides
- *„Range of expert opinion on X"* — explicitly asking for variance
- *„Common mistakes vs best practices"* — open-ended

**When Type C is WASTE (use Type A instead):**
- *„How exactly does X work?"* — single canonical answer
- *„Step-by-step process for X"* — single procedural answer
- Any question where bot's role-selection consistently picks the same role

**Heuristic:**
1. Send question to 1 user profile first (cheap probe)
2. If answer feels comprehensive and decisive → skip Type C
3. If answer hints at variance („there are several approaches", „some experts say...") → Type C with 3-5 users to surface that variance
4. If first answer is generic/shallow → use Type A (continuous chat to dig deeper) instead of Type C parallel

**Strength when used right:** surfaces real variance the single-thread approach would miss.

**Weakness when overused:** identical answers × 5 calls = 5× cost for 1× insight. Worse than Type A for most use cases.

## 🧠 Architecture-derived strategy reasoning (the foundation)

Before any strategic choice, derive expectations from BuddyPro's actual pipeline. Source: `big-picture.md` + `replyToUser.ts:366` + `ChatCompletions.ts:330` + `Pinecone.ts:126` + `PromptUtils.ts:174`.

### The pipeline determines what works

```
User message → ROLE SELECTION (Gemini 2.5 Flash, last 3 msgs)
            → KNOWHOW RETRIEVAL (Pinecone hybrid search, top 10 chunks)
            → PROMPT ASSEMBLY (Layer 1 + role + chunks + aboutUser + history)
            → HISTORY (last 30 msgs, max 75K chars — TRUNCATED beyond)
            → Claude Sonnet 4.6 → response
```

### 6 architectural implications for research strategy

**1. Question phrasing > profile count**
Same Q to N profiles = ~same Pinecone vector = ~same chunks = ~same semantic content (only stochastic LLM formulation variance). N different question phrasings on 1 user produce more variance than 1 question across N profiles.

**2. Type A's depth advantage is structural, not coincidental**
Turn 5's effective query includes turns 1-4 context → embeds differently → may retrieve DIFFERENT chunks than turn 1's cold query did. This is why Test 1 saw zero saturation in 12 turns. The retrieval-context expands per turn.

**3. Memory limit ~30 turns / 75K chars — critical for 50-turn pushes**
Beyond turn 30, oldest turns drop out of LLM context. Turn 50 doesn't see turns 1-20 anymore. For 50-turn research: need **periodic summary refresh** every ~20 turns („Summarize key insights from our conversation so far") — that summary then sits in recent window and preserves foundation.

**4. Role drift through long conversation = FEATURE**
Last-3-message role selection means topic evolution → automatic role switching:
- Turn 1: pricing → `pricing_strategist`
- Turn 15: pricing psychology → `business_psychology`
- Turn 30: pricing at scale → `ai_business_implementation_specialist`

Long Type A naturally explores multiple roles' worth of knowledge. **Don't fight this, leverage it.**

**5. Question SHAPE determines retrieval breadth — but textual similarity ≠ retrieval breadth**

Initial hypothesis: open-ended retrieves broader → more variance.

**Empirically falsified (Test 3, 2026-05-07):**

| Shape | Avg textual similarity (5 profiles) |
|-------|-------------------------------------|
| Canonical („Jak X funguje?") | 5.9% |
| Open-ended („Jaké přístupy k X existují?") | 7.7% |
| Comparative („X vs Y vs Z") | **14.4%** (highest) |

Counterintuitive at first, but architecturally correct:

**LLM+RAG has 2 sources of variance:**
- RAG retrieval (deterministic per query → low variance)
- LLM formulation (stochastic → variable, depends on freedom)

**Comparative questions IMPOSE STRUCTURE** („compare A, B, C") → LLM has LESS formulation freedom → HIGHER textual similarity.

**Canonical/open questions allow free-flowing prose** → LLM has formulation freedom → LOW textual similarity.

**Meta-lesson:** Textual similarity measures **LLM formulation freedom**, NOT retrieval breadth. Don't use textual similarity as a proxy for content diversity.

For real semantic variance:
- Use semantic content analysis (keyword extraction across answers — see Test 2 method)
- OR use Strategy B (steered forks) — context-divergent retrieval is structural
- OR use Strategy C (lens rotation via Type D) — explicit reframing forces diversity

**Practical retrieval strategies:**
- Canonical → same chunks every time
- Open-ended → similar chunks (may include slightly more chunks because „kolik škol existuje" is meta)
- Comparative → chunks for each compared item, but structured response
- Specific („Konkrétní příklad") → case-study chunks (genuinely different from broad)
- Meta („Co je nejdůležitější o X") → consolidation chunks

Within ONE Type A chain, **ROTATE shapes** to expand retrieval diversity per turn — but understand that variance comes from RETRIEVAL DIFFERENCES, not from textual similarity drops.

**6. Branch fork ≠ Type C parallel**
Main chat's accumulated context biases retrieval toward sub-topic 1.
Fork's fresh context = unbiased query = different chunks retrieved.
Forks are **structurally context-divergent**, not cold-parallel. This is why forks produce genuinely different content.

### 5 research strategies derived from architecture

#### Strategy A — Continuous Deep Chat (single user, 15-25 turns)
- Memory accumulates within 30-turn window
- Role naturally drifts as topic evolves → automatic multi-role coverage
- Question-shape rotation amplifies retrieval diversity
- Output: 6000-9000 words, ~95% novel content
- Saturation hits when shapes stop rotating, not from time-in-session

#### Strategy B — Steered Branch Forks (2-3 parallel chains)
Force topical divergence at turn 2:
```
Branch A: Q1 broad → Turn 2 steer "into VALUE-BASED side"
Branch B: Q1 broad → Turn 2 steer "into PSYCHOLOGY side"
Branch C: Q1 broad → Turn 2 steer "into SCALE/ECONOMICS side"
```
Each branch's accumulated context biases retrieval differently → genuine semantic diversity (structural, not random).

**✅ Empirically validated (Test 5, 2026-05-07):**
- Setup chat (3 turns establishing context, mentioning sub-topic A and sub-topic B)
- Track A continued same chat with 5 turns on sub-topic A → 1090 words
- Track B FORKED new user, 5 turns on sub-topic B → **1414 words** (30% more!)
- Cross-track textual similarity: **5.4%** → tracks GENUINELY DIVERGED
- Final synthesis cross-similarity: **7.3%** → independent content

**Surprise finding — forks may produce MORE content than continuous chats:**
Continuous chat has setup-turn context → bot frames follow-ups more efficiently (shorter, building on prior). Forked chat has fresh user context → each turn gets comprehensive standalone treatment → longer answers per turn.

This means forks are NOT a budget compromise. They produce genuinely different + comprehensive content — strictly better than naively continuing the main chat into uncovered sub-topics.

#### Strategy C — Spiral / Lens Rotation (Pavel's „točí se v kruhu")
Same TOPIC, rotate LENS each turn:
```
Turn 1: Broad
Turn 2: BEGINNER's view (constraint)
Turn 3: EXPERT's view
Turn 4: SKEPTIC's view  
Turn 5: CONTRARIAN's view
Turn 6: PHILOSOPHICAL depth (meta)
```
Best with Type D (`replace` system prompt) for stronger lens forcing. Best for comprehensive single-topic coverage from N angles.

#### Strategy D — Spawn-on-Mention (dynamic branching)
Type A main chat continues. When bot mentions independent principle → fork captures depth in parallel without polluting main chat:
```
Main chat: turn 1-5 on topic A
   ↓ [bot mentions principle X]
   ↓ FORK: 8-turn drill on X
   ↓ [main chat unaffected, continues turn 6-10 on A]
```

#### Strategy E — Question-Shape Rotation (within single chain)
Single user, rotate shape per turn:
```
T1: canonical | T2: comparative | T3: specific | T4: edge case
T5: meta | T6: contrarian | T7: synthesis
```
Same user, different effective query each turn → different chunks → semantic breadth without forking.

### 🟢 DEFAULT blueprint — MASTER PATTERN (recommended for ~95% of jobs)

This is the empirically validated default. Use this unless the owner explicitly asks for the exhaustive variant or the topic is too narrow.

```
3 forks × 6-8 turns × different topology × different role
+ ANTI_CLARIFICATION_DIRECTIVE on every call
+ synthesis in Claude Code

TOTAL: 18-24 calls / $0.90-1.20 / 3-5 min wall-clock (parallelized)
OUTPUT: 6000-9000 words source material → 3-7 page polished document
```

**Empirical validation (2026-05-08):** 18 calls / 6289 words / 349 words per call avg / 1.5-2.9% cross-fork similarity / 2.8 min wall-clock. Best ratio of divergence-per-call across all tested configurations.

**Topology assignments per fork:**
- Fork 1: HLOUBKA (depth) — operational drill-down
- Fork 2: ŠÍŘKA (breadth) — sub-topic surface coverage
- Fork 3: INVERZE (anti-pattern) — failure modes / what NOT to do

(Other topology combinations work — see Question Topology section below. These three give the widest divergence per the empirical test.)

### 🟠 EXHAUSTIVE blueprint — only when explicitly requested

Use this when owner says *„make it comprehensive"* / *„the deluxe version"* / *„I want everything you have on X"*. Costs 2-3x more, produces 3-5x more raw material, takes 2-3x longer.

```
STAGE 1 — Skip topology probe (folded into Stage 2 fork topologies)
          Old design used 3 stateless calls here — DEPRECATED, see
          „Anti-Clarification Directive" section above for why mini-probes
          waste calls.

STAGE 2 — Primary deep chat (15-25 calls, Strategy A + E)
  Single user, question-shape rotation, 15-25 turns + directive
  Foundation: 6000-9000 words

STAGE 3 — Steered branch forks (2-3 branches × 10-15 turns, Strategy B)
  Different context biases + topology + role per fork → different retrievals
  Output: 8000-15000 words divergent angles

STAGE 4 — Lens rotation (5-7 calls, Strategy C + Type D)
  Custom role personas force semantic re-framing

STAGE 5 — Spawn-on-mention drill-downs (5-10 calls, Strategy D)
  Captured in parallel during Stages 2-3 — when bot mentions a sub-concept
  worth drilling, fork a child task

STAGE 6 — Synthesis verification (3-5 calls)
  „What am I missing in [cluster X]?" — gap-filling

TOTAL: 38-62 calls / $1.90-3.10 / 7-14 min
OUTPUT: 25,000-40,000 words → 5-15 page polished document
```

🔴 **Every call in every stage above MUST include the anti-clarification directive (and the appropriate topology + role directive stacked via `add` mode).** No call ever goes out raw — see the „Anti-Clarification Directive" section near the top of this file.

### Pavel's „depth vs breadth vs spiral" — architecture answer

| Strategy | Architectural mechanism | Best for |
|----------|------------------------|----------|
| **Depth** (A: 25 turns + E shape rotation) | Memory accumulation + role drift + retrieval-context expansion | Single rich topic, comprehensive |
| **Breadth** (B: 2-3 forks) | Context-divergent retrieval streams | Multi-faceted topic, genuine variance |
| **Spiral** (C: lens rotation) | Same chunks, different LLM framing | Single topic, maximum angle coverage |
| **Hybrid** (all 3 in blueprint above) | All mechanisms combined | Production deliverables |

The „spiral that goes in circles" Pavel mentioned isn't bad — it's a valid strategy when you want exhaustive coverage of ONE topic from MANY perspectives. The key is recognizing it's spiral by DESIGN, not by accident.

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

## 🎯 General principles for branch strategy (empirically validated)

These are the field-tested heuristics for picking and combining branch types. Built on Pavel's insights + live testing on Pavel Říha AI instance (2026-05-07).

### Principle 1 — Depth beats breadth in most cases

Live test confirmed: **4 Type A continuous turns produced ~3000 words of rich, structured content with concrete examples + edge cases + synthesis-ready output**. **4 Type C parallel profiles on same question produced ~1500 words with 75%+ redundancy** and 1 wasted call (bot asked for context instead of answering).

**Per-call value comparison:**
| Branch | 4 calls produces | Quality |
|--------|------------------|---------|
| Type A (continuous, builds on memory) | concrete example, edge cases, synthesis-ready section | ⭐⭐⭐⭐⭐ |
| Type C (parallel isolated profiles) | mostly redundant general principles | ⭐⭐ |

Default budget allocation:
- 60-70% → Type A (continuous chat with memory)
- 15-20% → Type B (broad probe one-shot, just to identify principles)
- 0-15% → Type C (perspective probe — ONLY when variance expected)
- 5-10% → Type D (persona) when deliverable needs tier framing

### Principle 2 — Open new session when, not by default

Don't pre-allocate „N parallel sessions" up front. **Sessions are opened deliberately** based on what's discovered:

**Open a NEW session when:**
- Branch split — bot mentions 2+ distinct sub-topics in one turn that each warrant own depth (start new Type A per sub-topic)
- Original session is locked in to one perspective and you need a fresh take
- Memory pollution — previous turns made the bot overly biased toward one framing
- Discovered an unexpected principle worth its own deep dive (Type E spawned)
- Need genuinely independent perspective check (Type C — but use sparingly)

**Don't open a new session when:**
- Just to „get more answers" — that's waste if KB is consistent
- For variance probing of a question with a single canonical answer
- For depth — depth comes from MORE TURNS in the same session, not more sessions

### Principle 3 — Question phrasing varies by branch type

| Branch type | How to phrase question |
|-------------|------------------------|
| **Type A** (continuous) | Each turn BUILDS on previous: „Now go deeper on X you mentioned", „What's the most common mistake when applying X?", „Give me a concrete example of X with numbers" — never re-ask, always advance |
| **Type B** (broad probe) | Different angles per call: „From angle A...", „From angle B...", „From the perspective of [audience]..." — diversity is the point |
| **Type C** (perspective) | IDENTICAL question word-for-word to N profiles. The variance comes from different memory contexts, not different questions. |
| **Type D** (persona) | Same topic, different persona framings via `replace` system prompt. Each call has its own custom persona. |
| **Type E** (drill-down spawned) | Narrow + specific: „Earlier you mentioned [principle X]. Tell me everything specific about it." — focused, not broad |

### Principle 4 — Branch splitting (rozdvojit větev)

When DURING a Type A continuous chat, the bot mentions 2+ distinct sub-topics in one turn that you both want to explore deeply, **don't try to follow both in the same chat** — fork:

```
Continuous chat (Type A, user=research-pricing-X)
├─ Turn 1: bot mentions principles A, B, C
├─ Turn 2: deep on A
├─ Turn 3: bot reveals A has two sub-aspects A1 and A2
│           ↓ FORK HERE
├─ KEEP this chat going on A1 (Turns 4-6)
└─ SPAWN new chat for A2 (user=research-pricing-X-A2)
   └─ Continue A2 deep dive there in parallel
```

**Why fork:** following A1 and A2 in same chat causes turn 4 to inherit context from A1, which biases the A2 framing. Forking keeps each pure.

**When NOT to fork:** if A1 and A2 are tightly related (same principle, different facets) — you may want to keep them together for richer cross-reference. Split only when they're genuinely independent.

### Principle 5 — Memory bias accumulates (but slowly — empirical update)

**Updated 2026-05-07 based on 12-turn live test:** Earlier wisdom said „after 5+ turns the bot locks in." Live data showed **NO saturation through 12 turns** on a rich topic — bot kept producing novel content.

**The real story:**
- Memory bias is real but accumulates slowly when each turn introduces genuine new angles
- Saturation happens when YOU stop varying the question shape, not just from time-in-session
- A turn that asks for „the same thing in different words" → bot repeats. A turn that asks „totally different angle" → bot expands.

**Reset by spawning a NEW session ONLY when:**
- Sim-to-previous-turn > 70% (real saturation, not theoretical)
- Topic genuinely shifted to unrelated domain
- Bot's role-selection is consistently wrong for new sub-topic
- You want pure perspective check unbiased by prior context

**Don't reset just because you hit turn 10.** Empirically, turn 12 was still novel. Going to turn 15-20 on rich topics is reasonable.

### Principle 6 — Type C requires SEMANTIC variance check (architecture-mandated)

🔴 **This is logically expected from the BuddyPro architecture, not a surprising finding:**

BuddyPro = **LLM (Claude Sonnet 4.6) + RAG (Pinecone vector retrieval)**.

```
Same question → same Pinecone retrieval → same top-10 chunks
                                       ↓
                              Same content fed to LLM
                                       ↓
                Different formulations (LLM stochasticity)
                BUT same underlying knowledge / same principles
```

Empirically confirmed (Test 2, 10-profile run, 2026-05-07):
- **Textual similarity: 5.5%** — each profile formulates differently
- **Semantic similarity: ~90%** — 9/10 profiles mention the SAME 3 dominant principles
- **Conclusion: Type C is wasted on canonical questions** despite low textual similarity

**Why naive textual-similarity check is misleading:** Char-level diff sees „Za prvé, prodávej výsledky" vs „První princip — hodnota nad časem" as 95% different. Semantically they're identical.

**When can Type C produce TRUE semantic variance?** Architecture-derived answer:

1. **Different `user` profiles with DIFFERENT memory** → effectively different query context → different RAG retrieval. Memory accumulates from past conversations, so profile A who has discussed niche X has different effective context than profile B fresh.
2. **Question shape forces different chunks** → asking about same topic from radically different framings can hit different KB regions
3. **System prompt explicitly forces different framing** (Type D + Type C combined) → LLM rewrites with constraints that surface different aspects

**Heuristic — semantic, not textual, variance check:**

```python
# Cheap probe (1 call) before deciding to run Type C
probe_answer = call_bp(stable_user, top_question)

# Decide based on probe content
if signals_variance_expected(probe_answer):
    # explicit signals: "several approaches", "depends on...", multiple frameworks named
    run_type_c_with_3_to_5_profiles()
else:
    skip_type_c()
```

**Where Type C IS valuable (when used right):**
- Surface SECONDARY/edge principles (40% of Test 2 profiles mentioned 'anchor pricing' that 60% didn't — that's surface area worth probing)
- Explicit variety prompts: *„Give me UNUSUAL or contrarian principles for X"*, *„From a NON-OBVIOUS angle"*, *„The principle most coaches IGNORE"*
- Comparing with explicit framing differences (Type D + Type C combined)

**Where Type C is WASTE:**
- Canonical questions (KB has ONE primary answer)
- Generic „give me top N" — bot will list same N every time
- Domain has well-defined frameworks (KB will return them)

**Better alternative when wanting variety:** Use **Type D (custom persona)** to FORCE different framings:
- Same topic, but with explicit „you are now a contrarian" persona
- „You are explaining to a complete beginner" persona
- „You are providing the most counter-intuitive advice" persona
- This produces semantic variance through explicit framing, not by hoping isolated profiles diverge.

### Principle 7 — Bot may refuse generic questions

In live test: 1/4 Type C profiles **refused to answer** the generic „pricing principles?" question — instead asked for context (typical coaching behavior). This is real domain-specific behavior of the bot's persona.

**Implications:**
- For Type C variance probes, formulate questions that ARE answerable without context (state-of-domain questions, framework taxonomies)
- For Type A continuous chats, give the bot context UPFRONT to skip its „what's your situation?" gate
- Budget for ~10-20% lost calls when probing generic questions in expert-coach instances

### 🔴 Heuristic: depth (Type A) > breadth (Type C) in most cases — empirically confirmed

Pavel's insight from empirical observation: **long continuous conversations that build on memory often produce richer material than parallel isolated probes.**

Why:
- Type A turn 5 has the bot's full self-context from turns 1-4. The bot can build, refine, give nuanced follow-ups.
- Type C 5 isolated turns each start cold. Each may give a competent but generic-feeling answer.
- For depth and quality of single-document output, prefer Type A.
- For breadth-coverage and variance-detection, Type C has its place — but selectively.

**Default budget allocation should lean depth:**
- 60-70% of calls into Type A (continuous chat with memory)
- 15-20% into Type B (broad probe one-shot)
- 10-15% into Type C (perspective probe ONLY when variance expected)
- 5-10% into Type D (persona) when the deliverable needs tier-specific framing

This is the OPPOSITE of „more parallel = better." For BuddyPro deep research, deeper > wider in most cases.

### Type D — Custom Persona

```
user: optional (or fresh per call)
saveToHistory: false (usually — persona is per-call only)
systemPrompt: replace mode with detailed persona definition
```

**When:** You want the bot to respond from a specific viewpoint — beginner, pro, skeptic, customer-perspective, competitor-lens.

**Pattern:**
- For each persona role, replace system prompt with detailed role definition
- Ask the same topic question
- Get persona-specific framing of the same knowledge

**Strength:** Reveals how the same principle should be communicated to different audiences. Generates ready-to-use content for tier-based products.

**Weakness:** `replace` mode reliability is imperfect — bot still references its base persona occasionally (see api-features-deep-dive.md). Long detailed personas work better.

### Type E — Drill-Down (Dynamically Spawned)

```
user: "drill-{parent-branch}-{principle}-{timestamp}"
saveToHistory: chosen per case
systemPrompt: optional, narrow framing
```

**When:** During Phase 3 (principle detection), the system identifies a concept that warrants its own branch. Type E branches are SPAWNED, not pre-planned.

**Triggers for spawning a Type E branch:**
1. **Named framework mention** — bot references a specific methodology by name
2. **Recurring principle** — same concept appears in 2+ existing branches
3. **Owner-flagged keyword** — owner specified topics they want extra depth on
4. **Contradiction detection** — two branches gave different advice on same point

Type E branches use either Type A (continuous) or Type B (stateless) execution depending on whether the principle needs progressive depth or a quick clarification.

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
