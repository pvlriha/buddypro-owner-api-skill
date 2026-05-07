# API Features Deep Dive — How `user`, `saveToHistory`, `systemPrompt` Actually Work

This file is the source of truth for understanding the 3 API features that shape every BuddyPro Owner API call:

- `user` field — profile routing
- `x_buddy_saveToHistory` — write toggle for memory & history
- `x_buddy_systemPrompt` + `x_buddy_systemPromptMode` — per-request system prompt

The skill MUST understand these in their **actual mechanics** (verified against `buddy-fm/buddy` source code and live experiments) and apply them in combinations that match the user's use case.

## The complete prompt assembly model

When BuddyPro processes any API call, it builds the final system prompt sent to the LLM in layers. **Only Layer 2 is replaceable.** Everything else is always present.

```
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 1 — BuddyPro CORE (white-label, ALWAYS present)            │
│   Cannot be replaced via API. Always part of every prompt.       │
│   Contains:                                                       │
│   • Identity scaffolding ("you are Buddy, an AI version of...")  │
│   • Payment / subscription context (active sub, trial, billing)  │
│   • User profile (aboutUser, instanceSpecificAboutUser)          │
│   • Topics to discuss, banned techniques, current info           │
│   • Used messages count, cost limits, support email              │
│   • PINECONE knowledge retrieval (top 10 chunks, always)         │
│   • Conversation history (last 30 messages, unless stateless)    │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ <CUSTOMIZED-INSTANCE> XML tag (Layer 2 — REPLACEABLE)    │   │
│   │                                                           │   │
│   │ Default content: instance customSystemPrompt (from Drive  │   │
│   │   `SYSTEM PROMPT` doc — your bot's persona, voice rules,  │   │
│   │   frameworks, response patterns)                          │   │
│   │                                                           │   │
│   │ With x_buddy_systemPrompt:                                │   │
│   │   IF mode = "replace": content = x_buddy_systemPrompt    │   │
│   │     (instance customSystemPrompt IGNORED entirely)        │   │
│   │   IF mode = "add" (default): content = instance prompt    │   │
│   │     + "\n\n" + x_buddy_systemPrompt                       │   │
│   │                                                           │   │
│   │ Wrapped with explicit "highest priority" note before:    │   │
│   │   "...follow these instructions and give them highest    │   │
│   │    priority. If conflicting, instructions inside          │   │
│   │    CUSTOMIZED-INSTANCE tag have higher priority than     │   │
│   │    other parts of this prompt."                          │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Source of truth:**
- `telegram/Utils/BuddyApiRuntimeUtils.ts` — replace/add logic (verified)
- `telegram/Modules/BuddyPro/PROMPTS_AND_MANUALS.ts:387` — `addBuddyProSystemPrompt()` injection
- `telegram/Utils/PromptUtils.ts:174` — `prepareSystemPrompt()` Layer 1 dynamic content

## Implications for what custom prompts CAN and CANNOT do

### ✅ What a well-crafted custom prompt CAN do

| Capability | Why it works | Example |
|------------|--------------|---------|
| **Persona overlay** | Layer 2 replacement directly affects how the bot speaks | „You are EnglishBuddy, only respond in English" → bot becomes EnglishBuddy |
| **Tone change** | Voice/style instructions are Layer 2 content | „Be playful and use emoji" / „Be formal and concise" |
| **Format compliance (long prompts)** | Detailed format rules in Layer 2 outweigh weaker tendencies | „Always end with [TAG_42]" — works (R12 confirmed) |
| **Length constraint** | Simple, explicit constraint | „Answer in ONE word" |
| **Refuse off-topic with explicit instructions** | If you make the refuse policy detailed enough, it wins | R10 math tutor refusing pricing — works WITH long detailed prompt |
| **Language switch** | With detailed prompt, overrides Layer 1 language detection | „Always English, refuse Czech" — works (R12) |

### ❌ What custom prompt CANNOT do

| Limitation | Why it fails | What happens instead |
|------------|--------------|----------------------|
| **Change knowledge base content** | Pinecone retrieval is Layer 1 — always returns chunks from instance's KB | Marketing instance + math tutor prompt = bot CAN refuse non-math, but if user asks math, it has no math knowledge to draw from |
| **Erase user memory** | Layer 1 includes aboutUser/profile data | Bot still „knows" the user's facts even if persona says „you are a stranger" |
| **Disable knowledge retrieval** | Architectural — happens before prompt assembly | Always 10 chunks loaded, custom prompt can only frame how to use them |
| **Override identity scaffolding fully** | Layer 1 starts with „you are Buddy..." | Even with replace mode, some „Buddy-ness" leaks (referring to Buddy, mentioning the white-label) |
| **Strict JSON format reliably** | Tested, unstable — Layer 1 conversation patterns interfere | Sometimes works, sometimes returns plain text. Don't rely on this for structured output. |

### 🎯 Practical guidelines

1. **Don't fight the knowledge base.** Custom prompts work best when they're a variation of the instance's domain. Marketing instance → fitness coach via prompt = nonsense (KB returns marketing chunks).

2. **Long detailed prompts beat short prompts.** A 3-line prompt loses against Layer 1's mass. A 1000-character prompt with structured headers (`## YOUR ROLE`, `## ABSOLUTE BOUNDARIES`, numbered rules) has weight.

3. **Use `add` for behavioral additions, `replace` for whole persona swap** within the same domain. Both keep Layer 1 intact.

4. **For strict structured output (JSON, XML), use stock LLM APIs, not BuddyPro.** BuddyPro is optimized for personality, not deterministic format compliance.

5. **The `user` field amplifies custom prompt effectiveness.** Empirical observation: with no `user` field (= owner profile), the bot tends to defer to base instance persona more. With a fresh `user` field, the bot leans into the custom prompt more readily.

## `user` field — the profile router

| `user` value | Profile that processes the call |
|--------------|--------------------------------|
| (omitted) | Owner profile (Telegram account that generated the API key) |
| `"customer-42"` (first time seen) | NEW profile created, named `customer-42` |
| `"customer-42"` (subsequent calls) | SAME profile — memory continuity |
| `"customer-43"` | Different profile — totally isolated |

**Identity rules:**
- Allowed chars: alphanumeric, `-`, `_`, `.`
- Max 128 chars
- NOT purely numeric (`"42"` rejected)
- Case-sensitive (`Customer1` ≠ `customer1`)

**What's isolated per profile:**
- Conversation history
- Long-term memory (about user, topics, preferences, learned techniques)
- Onboarding state

**What's shared across profiles in the same instance:**
- Knowledge base (system prompt + SOURCES content)
- Subroles
- Voice clone
- Instance settings

### Decision matrix — when to use `user` field

| Scenario | `user` field strategy |
|----------|----------------------|
| Owner's own automation/script | None — let it use owner profile |
| Multiple end-customers, each with own memory | One stable ID per customer |
| Anonymous trial visitors | Generate session ID, store in client (localStorage / cookie / session) |
| Telegram group bot — each group member is a separate user | One `user` per Telegram member ID |
| Workshop / temporary testing | Use date-suffixed ID (`workshop-2026-05-07`) for easy cleanup awareness |
| Different "modes" for same person (work mode vs personal mode) | Use suffix: `customer-42-work` vs `customer-42-personal` |

### When same person should map to multiple `user` values

If the same end-customer interacts with the bot for **fundamentally different purposes** that should NOT share memory:
- `customer-42-pricing-bot` — for sales conversations
- `customer-42-support-bot` — for technical support
- `customer-42-coaching-bot` — for ongoing coaching

The bot keeps separate memory per profile. Customer asks pricing — pricing-bot has all pricing-context. Customer asks support — support-bot starts fresh, doesn't bring pricing baggage.

### When same person should map to ONE `user` value

If the bot should remember everything about the customer across interactions:
- One ID for entire customer lifecycle
- All conversations contribute to one cumulative memory

This is the default for most multi-tenant SaaS scenarios.

## `x_buddy_saveToHistory` — the write toggle

```typescript
// from source code (BuddyApiRuntimeUtils.ts):
export function isBuddyApiStorageBlocked(runtimeObject): boolean {
    if (!apiRuntime) return false;       // Non-API calls never blocked
    return apiRuntime.saveToHistory === false;
}
```

| `x_buddy_saveToHistory` | Reads | Writes |
|--------------------------|-------|--------|
| `true` (default) or omitted | History + memory + KB loaded | New turn IS saved to history, memory, profile |
| `false` | History + memory + KB STILL loaded | Nothing persisted — no DB write, no Pinecone upsert, no profile mutation |

**Subtle but important:** stateless mode is „write off, read on". The bot still SEES previous history if it exists, just doesn't write the current turn into it.

### When to use `x_buddy_saveToHistory: false`

| Scenario | Why stateless helps |
|----------|---------------------|
| Quality evaluation (testing 100 questions) | Don't pollute the test profile with eval junk |
| Format-strict prompts (JSON output) | One-off requests don't accumulate format-mode artifacts |
| Privacy-sensitive queries | Question + answer not stored anywhere |
| Custom persona tests | Try a custom persona without binding to long-term memory |
| One-off automations | Cron jobs, alerts, generated reports — no need to log them |
| A/B testing prompts | Each variant call doesn't influence next call |

### When to NOT use stateless

| Scenario | Why stateful matters |
|----------|---------------------|
| Multi-turn conversation | Bot needs to remember turn 1 for turn 2 |
| Customer support (build profile over time) | Memory of past tickets = better support |
| Coaching app (ongoing relationship) | Bot needs to know goals, progress |
| Personalization | Bot learns customer preferences |

## The 3 features in combination

The real power is in combinations. Some combinations are common, some are dangerous, some are weird.

### The combination matrix

| `user` | `saveToHistory` | `systemPrompt` | Use case |
|--------|-----------------|----------------|----------|
| ❌ none | ✅ true | ❌ none | Owner's own personal scripts (default) |
| ❌ none | ✅ true | ✅ add | Owner with a one-shot behavioral nudge |
| ❌ none | ❌ false | ❌ none | Owner one-off question (no pollution) |
| ❌ none | ❌ false | ✅ replace | Owner running an evaluation with a different persona |
| ✅ stable ID | ✅ true | ❌ none | **Multi-tenant SaaS — most common** |
| ✅ stable ID | ✅ true | ✅ add | Multi-tenant + customer-specific behavior nudge |
| ✅ stable ID | ❌ false | ❌ none | Anonymous one-off question with isolation |
| ✅ stable ID | ❌ false | ✅ replace | **Bulk evaluation per customer with custom format** |
| ✅ fresh ID per call | ❌ false | ✅ replace | **Cleanest stateless eval — true variance test** |

### Common patterns explained

#### Pattern: „Group chat where each member has memory"

If your bot is added to a Telegram group OR you build a multi-user chat product where each user has their own memory:

```python
def chat_in_group(telegram_user_id, message):
    return call_buddypro({
        "user": f"group-member-{telegram_user_id}",
        "messages": [{"role": "user", "content": message}],
    })
```

Each group member gets their own profile. Bot remembers individuals across messages, even when conversations interleave.

#### Pattern: „Same customer, different conversation contexts"

A consultant who uses BuddyPro for both sales calls AND ongoing coaching with the same customer might want isolated memory:

```python
# Sales call context
call_buddypro({"user": "client-42-sales", "messages": [...]})

# Coaching context (totally separate memory)
call_buddypro({"user": "client-42-coaching", "messages": [...]})

# Personal mood-tracking context
call_buddypro({"user": "client-42-mood", "messages": [...]})
```

Each context evolves independently. Sales conversations don't pollute coaching memory.

#### Pattern: „Eval grid for prompt tuning"

Testing 5 prompt variants × 20 test questions each = 100 calls. None should pollute anything:

```python
for prompt_variant in [v1, v2, v3, v4, v5]:
    for question in test_questions:
        eval_id = f"eval-{prompt_variant.id}-{question.id}-{int(time.time())}"
        result = call_buddypro({
            "user": eval_id,
            "x_buddy_saveToHistory": False,
            "x_buddy_systemPrompt": prompt_variant.text,
            "x_buddy_systemPromptMode": "add",
            "messages": [{"role": "user", "content": question.text}],
        })
        # No memory accumulation, no profile pollution, fresh eval each call
```

#### Pattern: „Public website chat with memory per visitor"

Visitor lands on the site, gets a session ID stored in their browser:

```javascript
// Browser
let userId = localStorage.getItem("buddypro_user");
if (!userId) {
  userId = `web-visitor-${crypto.randomUUID()}`;
  localStorage.setItem("buddypro_user", userId);
}

// Backend
await callBuddyPro({
  user: userId,
  messages: [{ role: "user", content: visitorMessage }]
});
```

The visitor returns a week later → same `userId` from localStorage → bot remembers everything.

**Privacy gotcha:** the bot's memory of the visitor is visible to the API key owner. Disclose in your privacy policy.

#### Pattern: „Hybrid persona — same KB, different personas per customer tier"

Your instance has marketing knowledge. You serve 3 customer tiers:
- Beginners — gentle, encouraging tone, more explanation
- Pros — direct, advanced terminology
- VIP — like a peer, no fluff

```python
TIER_PROMPTS = {
    "beginner": "## YOUR ROLE\nYou are a patient marketing teacher for beginners. Use simple language. Explain jargon. Be encouraging. Always check understanding.\n\n## ABSOLUTE: be warm, never condescending.",
    "pro": "## YOUR ROLE\nYou are a no-fluff marketing strategist for experienced practitioners. Skip basics. Use industry terms freely. Direct and dense.\n\n## ABSOLUTE: never explain things they already know.",
    "vip": "## YOUR ROLE\nYou are a peer-level marketing expert. Treat the user as your equal. Skip explanations of fundamentals. Discuss strategy at a senior level.\n\n## ABSOLUTE: no condescension, no over-explanation.",
}

def chat(customer_id, tier, message):
    return call_buddypro({
        "user": customer_id,
        "x_buddy_systemPrompt": TIER_PROMPTS[tier],
        "x_buddy_systemPromptMode": "add",  # extends instance prompt
        "messages": [{"role": "user", "content": message}],
    })
```

Same instance, same knowledge, three different feels. Each customer's profile evolves with their tier-appropriate experience.

## Decision flowchart

```
What does the user want to do?
│
├─ Send a question to bot from code (single use)
│   └─ Owner's own script? 
│        ├─ YES → no `user`, default saveToHistory
│        └─ NO  → use `user` field per end-customer
│
├─ Build multi-tenant SaaS (each customer has own memory)
│   └─ stable `user` field per customer, default saveToHistory
│   └─ Different memory contexts per customer? → use suffixed user IDs
│
├─ Bulk evaluation / batch testing
│   └─ saveToHistory: false (stateless)
│   └─ Optionally: fresh user per call to avoid any cross-call influence
│
├─ Custom persona / one-off behavior change
│   └─ x_buddy_systemPrompt + saveToHistory: false (avoid persistence)
│   └─ mode: "replace" for full swap, "add" for extension
│   └─ Make the prompt LONG and DETAILED if you need strong override
│
├─ Group chat / multi-user environment (Telegram group, public chat)
│   └─ Per-user `user` field (e.g., user-{telegram_id})
│   └─ default saveToHistory (each member has memory)
│
├─ Anonymous trial / no signup
│   └─ Session-scoped `user` field (random UUID, store in client)
│   └─ default saveToHistory; consider time-bounded retention story
│
└─ Privacy-sensitive question
    └─ saveToHistory: false (nothing stored)
    └─ Optionally: fresh `user` per call (no profile created)
```

## Custom prompt persistence — PER-CALL ONLY

🔴 **Critical empirical finding:** `x_buddy_systemPrompt` is **per-call only** — it does NOT persist into the user's profile or future calls.

Verified behavior (live tests on Pavel Říha AI, 2026-05-07):

| Test | Setup | Result |
|------|-------|--------|
| D1 | Turn 1 with long beginner-friendly prompt → Turn 2 same user, NO prompt | Turn 2 returned to **base instance persona**. The custom prompt did NOT carry over. |
| D2 | Same user — Turn 1 playful prompt, Turn 2 formal prompt, Turn 3 no prompt | **Each call applied ONLY its own prompt.** No prompt drift. T3 = base persona. |

**What persists vs what doesn't:**

| Persists across calls (same `user`) | Does NOT persist |
|-------------------------------------|------------------|
| Conversation content (what user said) | Custom system prompt |
| Bot's responses (what bot said) | Custom prompt mode (replace/add) |
| Long-term memory (about user, topics) | One-shot persona overrides |
| User profile data | Format constraints |

**Implication for the skill:** if you want user X to have a consistent custom persona across many calls, you MUST send the same `x_buddy_systemPrompt` on every call. There is no „set once and forget" persona binding.

If you want to give user X a permanently different persona, the right place is the **instance's customSystemPrompt** in Drive (`SYSTEM PROMPT` doc) — but that affects ALL users, not just X. For per-user personas, you must include the prompt in every call.

## Format compliance reality check

🔴 **Strict structured output (JSON, exact word counts) is unreliable** even with detailed prompts.

Tested patterns that **frequently fail compliance:**

| Constraint | Behavior |
|------------|----------|
| „Respond ONLY with JSON, no prose" | Sometimes returns plain text, sometimes JSON. ~50% reliable. |
| „Answer in EXACTLY 5 words" | Returns 8-14 words despite explicit rule. Treats as preference. |
| „Under 30 words" | Returns 25-50 words. Bot interprets loosely. |
| „Always end with [TAG]" | More reliable (~80%) but still occasionally drops the tag. |

**Why:** Layer 1 (BuddyPro core) is optimized for personality, conversation style, and contextual response — not for deterministic format compliance. The bot tends to view your format rules as „strong preferences" rather than absolute constraints.

**Workarounds:**
1. **Don't use BuddyPro for strict structured output.** Use stock OpenAI/Anthropic API for JSON, structured data extraction, parsing tasks.
2. **Post-process responses** programmatically — strip emoji, extract JSON from code-fences, truncate to N words.
3. **Combine `replace` + long detailed prompt + `saveToHistory: false`** for the highest compliance odds, but still expect ~70-80% success rate.
4. **Make the constraint extreme and repeated** — five times in the prompt: „RESPOND IN ONE WORD. JUST ONE. ONE WORD ONLY. ONE WORD." — sometimes works.

## Critical reminders

1. **Knowledge base is always Layer 1.** Custom prompts can frame, but cannot replace, the bot's domain knowledge.

2. **`replace` mode replaces ONLY Layer 2** (the `<CUSTOMIZED-INSTANCE>` tag content), not the BuddyPro core scaffolding.

3. **`add` mode (default) appends** API prompt to instance prompt with `\n\n` separator. Both end up inside `<CUSTOMIZED-INSTANCE>` tag.

4. **`saveToHistory: false` blocks WRITE only** — reads (history, KB retrieval) still happen. Stateless calls TO existing user STILL see prior memory.

5. **`user` field amplifies custom prompt effectiveness** — bot tends to comply more with custom instructions when `user` is set vs. when targeting owner profile.

6. **Every API call without `user` writes to owner's main profile.** This pollutes owner's chat history fast in batch jobs. Use `user` or stateless.

7. **Long prompts > short prompts** for `replace` mode. Layer 1 is large; a tiny custom prompt gets diluted.

8. **Custom prompts are per-call only** — no automatic persistence. For consistent persona across many calls, send the prompt every time.

9. **Strict format compliance is unreliable** — for JSON or exact-word-count outputs, use stock LLM APIs, not BuddyPro.

10. **Stateless + same user reads existing memory.** It only blocks writes. Use a fresh `user` value per call if you need TRUE memory isolation.

## Source

- Verified against `buddy-fm/buddy` source (current as of search): `telegram/Utils/BuddyApiRuntimeUtils.ts`, `telegram/Modules/BuddyPro/PROMPTS_AND_MANUALS.ts`, `telegram/API/BuddyApi/endpoints/V1OpenaiLike.ts`, `telegram/docs/BuddyApi/BuddyAPIOpenAILikeV1.md`
- Live empirical experiments on Pavel Říha AI instance (2026-05-07) — see git history for individual test results

*Last updated: 2026-05-07 (v0.4.1)*
