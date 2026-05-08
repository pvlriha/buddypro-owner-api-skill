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

## 🔗 Scenario Adaptation — see sub-skill

Deep research adapts to the user's scenario. The 7 scenario adaptations + their pipeline configurations + external research integration (Track A) have been moved to a dedicated sub-skill for fresh-agent friendliness:

→ [`deep-research-scenarios.md`](./deep-research-scenarios.md)

The 7 scenarios are:
1. **Universal how-to** — generic methodology guide (USE MASTER PATTERN as default; hybrid as fallback)
2. **Person + product specific** — most demanding (DUAL-TRACK: external research + internal forks)
3. **Comparative analysis** — N options side-by-side (one Type A continuous chat per option)
4. **Audience-tiered guidance** — same content, 3 audiences (Type D persona forks per tier)
5. **Content draft with depth** — newsletter / blog / video script (output-format-anchored synthesis)
6. **Knowledge audit / Q&A reference** — exhaustive principle extraction
7. **Single principle, exhaustive** — one topic to operational depth (Type A 8-15 turns + HLOUBKA topology)

Load `deep-research-scenarios.md` when the user's request matches one of these triggers, or when you need the per-scenario topology + role assignments.

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

## 🔗 Question Topology — the 3rd design dimension — see sub-skill

The 8 question topology patterns (KRUH, HLOUBKA, ŠÍŘKA, SPIRÁLA, STROM, ZIGZAG, INVERZE, META) + topology selection per scenario + combining topologies within a single Type A branch — moved to dedicated sub-skill:

→ [`deep-research-topologies.md`](./deep-research-topologies.md)

**Quick selection guide:**
- **HLOUBKA** (drill-down) — Fork 1 of MASTER PATTERN; operational depth on one mechanism
- **ŠÍŘKA** (lateral) — Fork 2 of MASTER PATTERN; sub-topic surface coverage
- **INVERZE** (anti-pattern) — Fork 3 of MASTER PATTERN; failure modes & common mistakes
- **STROM** (tree branching) — empirically highest volume per turn (484 words/turn)
- **KRUH** (concentric) — same topic, multi-perspective treatment
- **SPIRÁLA** (spiral) — exhaustive coverage of one topic from many angles
- **ZIGZAG** (compare-contrast) — alternating between two related sub-topics
- **META** (questioning the answer) — deepest insight extraction, lowest volume

Load `deep-research-topologies.md` when designing per-fork question trajectories, or when fork output feels redundant and you need to switch topology mid-research.

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

## 🔗 Advanced blueprints, pipeline & implementation — see sub-skill

The 8-phase pipeline, coordination & resource management, knowledge graph data structure for synthesis, synthesis strategies, full Python implementation skeleton, branch-type decision tree, and privacy/cost summary have been moved to a dedicated sub-skill for fresh-agent friendliness:

→ [`deep-research-blueprints.md`](./deep-research-blueprints.md)

Load that file when:
- Running the EXHAUSTIVE blueprint variant (40-65 calls / $1.90-3.10 / 7-14 min)
- Implementing the orchestration code from scratch in Python/Node
- You need the branch-type decision tree to pick between Type A/B/C/D/E
- The user is asking about cost / privacy implications of running deep research

For 95% of deep-research jobs, **the MASTER PATTERN documented in `Optimal blueprint` section above is the right tool — you do NOT need to load `deep-research-blueprints.md`.**

*Last updated: 2026-05-08 v1.0.0)*
