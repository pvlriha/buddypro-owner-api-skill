# Deep Research Architecture — Branch & Merge System

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

**Pipeline:**
- Phase 1: Plan with breadth focus (8-12 angles covering all sub-areas)
- Phase 2: Broad probe (Type B parallel) — bulk angle coverage
- Phase 3: Principle detection — extract the bot's strongest frameworks
- Phase 4: Deep dive (Type A) on top 3-5 principles
- Phase 5: Skip perspective probe (no person-specific variance needed)
- Phase 6: Optional Type D persona angles if owner wants tier-specific
- Phase 7: Synthesis → **Principle Compendium** strategy
- Output: structured guide, sections per principle/framework

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
user: "research-deep-{principle}-{timestamp}"   ← STABLE across 5-10 turns
saveToHistory: true (default)                    ← memory accumulates
systemPrompt: optional, usually same for whole branch
```

**When:** You've identified ONE principle worth deeply exploring. You want progressive deepening — each turn builds on the previous.

**Pattern:**
- Turn 1: „Tell me about {principle} broadly"
- Turn 2: „Now go deeper on the most important aspect from your last answer"
- Turn 3: „Give me a concrete example of [thing from turn 2]"
- Turn 4: „What's the most common mistake when applying [thing from turn 3]?"
- Turn 5: „Summarize the principle in 3 sentences for someone new"

**Strength:** Bot's memory carries. Later turns are informed by earlier answers — no need to re-prime context.

**Weakness:** Bot can „lock in" to one framing early. Use Type C in parallel to break this.

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

### Principle 5 — Memory bias accumulates

After 5+ turns in Type A, the bot's „session role" is locked in. It will frame all subsequent answers through that role. This is usually good (consistency), but sometimes you want to break out:

**Reset by spawning a NEW session** when:
- You've reached saturation in current chat
- Want to test if the same question yields different framing under fresh context
- Topic shifted significantly mid-research (new sub-topic that doesn't fit the established frame)

### Principle 6 — Type C only with prior probe

🔴 **Empirically validated** — running Type C blindly with 4-7 user profiles on a question that has a canonical answer just costs 4-7× the budget for 1× insight. **ALWAYS probe first:**

```python
# Cheap probe (1 call) before deciding to run Type C
probe_answer = call_bp(stable_user, top_question)

# Decide based on probe content
if signals_variance_expected(probe_answer):  # mentions "several approaches", "depends on...", multiple frameworks
    run_type_c_with_3_to_5_profiles()
else:
    skip_type_c()  # save the budget for Type A depth instead
```

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
