# Multi-Tenancy — `user` Field Deep Dive

> 🔴 **Read this first — the `user` field is NOT a SaaS multi-tenancy primitive.**
>
> Per official Owner API docs (`https://docs.buddypro.ai/owner-api/`): *„When you generate an API key, it is tied to the user that generated it, and the conversation happening with that API key is actually being saved into message history (even though you don't see that conversation in Telegram) and contributes to the user profile and memory."*
>
> **What this means concretely:** When you pass `"user": "customer-acme-42"`, you do NOT create an isolated tenant with privacy from you. You create a **sub-profile inside YOUR account**. The owner of the API key (= the Telegram profile that ran `/generateApiKey`) has **full read access to ALL sub-profiles** created via that key. There is currently no public API to delete a sub-profile.
>
> **Therefore:**
> - ✅ The `user` field IS appropriate for: separating contexts in the OWNER's own work (research-1 vs research-2, sales-Q1 vs coaching-Q1, agent-task-N), isolating internal team members (each colleague has separate memory in your shared brain), distinguishing batch eval runs.
> - ⚠️ The `user` field is NOT a substitute for end-customer privacy. Production SaaS where customers expect their conversations to be private from the owner should wait for the upcoming **End User API** (each customer generates their OWN `bapi_` key — true privacy boundary).
> - ⚠️ If you must run an external-customer product on Owner API today (prototype, MVP, friends-and-family beta): disclose the data-ownership model in your ToS/privacy policy, generate the key on a `/test` profile (not your real one), and treat all customer conversations as visible-to-you logs.

The remainder of this file documents how the `user` field works mechanically — when to use it, how to design IDs, isolation guarantees, and patterns. **All patterns assume you've read and understood the warning above.**

When the BuddyPro instance serves multiple sub-profiles (each with their own memory, history, and context), the `user` field is the single most important parameter to understand.

## The core mechanic

Every API call optionally accepts a `user` field:

```json
{ "user": "research-launch-q2", "messages": [{"role": "user", "content": "..."}] }
```

| Request | Where the conversation goes |
|---------|------------------------------|
| **No `user` field** | Generating Telegram profile (should be your `/test` profile, not your real account) |
| `"user": "abc"` (first time) | New sub-profile `abc` created INSIDE your account |
| `"user": "abc"` (later) | Same sub-profile `abc` — full memory continuity |
| `"user": "xyz"` | Separate sub-profile from `abc` — independent memory, but still your data |

That's it. No registration, no setup, no quotas — sub-profiles are created on first use of a new `user` value. **All sub-profiles remain visible to the owner** of the API key.

## ID format requirements

- Allowed characters: alphanumeric, `-`, `_`, `.`
- Max 128 characters
- **NOT purely numeric** (e.g., `"42"` is rejected — must include at least one non-digit)
- **No spaces, no other special chars**
- Case-sensitive: `Customer-1` and `customer-1` are different profiles

**Good IDs:**
- `customer-acme-42` (descriptive)
- `usr_d8f3a1b2c4` (random-suffixed, hard to guess)
- `tenant.42.session.7` (hierarchical)
- `email-hash-7d3f8a` (privacy-preserving)

**Bad IDs:**
- `42` (purely numeric — rejected)
- `john doe` (space — rejected)
- `<email@example.com>` (special chars — rejected)
- `customer 42` (space — rejected)
- `John` then `john` (case difference — different profiles, confusing)

## ID design strategies

Pick the strategy that fits your product:

### A) Internal customer ID (recommended for B2B SaaS)

Use your own customer/user ID from your database:
```python
user_field = f"customer-{customer.id}"  # e.g., "customer-7392"
```

**Pros:** stable, easy to map back to your own systems, audit-friendly.
**Cons:** if you change customer IDs, profiles disconnect from history.

### B) Hashed email (privacy-friendly)

For products without internal IDs, hash the email:
```python
import hashlib
user_field = f"u-{hashlib.sha256(email.lower().encode()).hexdigest()[:16]}"
# e.g., "u-7d3f8a1b2c9e4f6a"
```

**Pros:** stable across sessions, doesn't expose the email, GDPR-friendlier.
**Cons:** can't reverse-lookup the email from the user field.

### C) Session-scoped (for stateless SaaS)

If memory continuity within a single session is enough:
```javascript
const sessionId = crypto.randomUUID();
// use as user field for the duration of the session
// next session = new ID, fresh memory
```

**Pros:** clean privacy story (memory dies with session), no PII in profile names.
**Cons:** customer can't resume previous conversation.

### D) Hierarchical (for multi-instance products)

Combine tenant + user:
```python
user_field = f"tenant-{tenant_id}-user-{user_id}"
# e.g., "tenant-acme-corp-user-42"
```

**Pros:** clear scoping, easy to debug "which tenant's user is this?"
**Cons:** longer IDs, max 128 chars limit applies.

### Anti-patterns (don't do this)

| Don't | Why |
|-------|-----|
| Use the same ID for different customers | They share memory — privacy violation |
| Generate a new ID per API call | Each call = new profile = no memory at all |
| Use mutable IDs (e.g., usernames) | Customer renames → orphaned profile, fresh start |
| Use IP addresses | Changes constantly, multiple users behind one NAT |

## Isolation guarantees

Per profile (each unique `user` value):

| Isolated | Shared |
|----------|--------|
| Conversation history | Knowledge base (system prompt + SOURCES) |
| Long-term memory (about user, topics, techniques) | Subroles |
| Onboarding state (first-time vs returning) | Voice clone |
| Per-user system prompt overrides | Instance settings (pricing, etc.) |
| User preferences inferred over time | Owner credentials |

**What this means in practice:**
- Customer A asking "what did we discuss yesterday?" gets only Customer A's history.
- Customer A and Customer B get answers from the same knowledge base, in the same expert voice — they're talking to the same expert, just having separate conversations.
- The bot's "long-term impression" of Customer A doesn't leak to Customer B.

## Privacy & legal implications

🔴 **Read this section carefully — it affects your customer agreements.**

Per official docs:
- **The instance owner has full access to ALL profiles created via the API key.** This includes conversation history, stored memories, and all derived user data.
- **The Owner API is NOT designed for true end-user privacy.** A future End User API will allow each customer to generate their own keys with privacy from the owner.

### What this means

- **You can read your customers' conversations** (programmatically or in your dashboard if BuddyPro provides one).
- **Your customers should know this** — disclose in your privacy policy and ToS.
- **GDPR-style "right to be forgotten"** — there's currently no public API to delete a profile. Workaround: switch your customer to a new `user` value (effectively orphaning the old profile). Contact BuddyPro support for actual deletion.

### Recommended customer-facing language

> *„Your conversations with [Product Name] are processed by BuddyPro AI. The product owner has access to conversation logs for support and quality assurance. Do not share information you wouldn't want the owner to see."*

### Privacy-preserving design tips

1. **Hash the user ID** (Strategy B) — owner sees `u-7d3f8a...` not `john@example.com`.
2. **Use stateless mode for sensitive queries:** `x_buddy_saveToHistory: false` — nothing persists.
3. **Don't pipe customer PII into messages** unless you have to — the message content gets stored in conversation history.
4. **Wait for End User API** if your product has true privacy needs (HIPAA, financial advice, legal).

## Cost & quota considerations

The 30 req/min rate limit is **per API key**, not per user. So:

- 1 customer asking 30 questions in a minute → fine
- 30 customers asking 1 question each in the same minute → also fine
- 60 customers asking 1 question each in the same minute → 30 will succeed, 30 will get 429

**Scaling strategies:**
- **Multiple API keys** (round-robin) — multiplies your budget. Generate from the same profile to avoid leakage. See `code-recipes.md` → "Token rotation".
- **Edge rate-limiting** — throttle per-customer at your API gateway so no single customer eats the whole budget.
- **Async queue** — for non-realtime use cases (batch processing), queue requests and process at 25/min.

## Common patterns

### Pattern A: Customer support chatbot

```python
@app.post("/support")
def support(payload: SupportRequest):
    return ask_buddypro(
        user_field=f"customer-{payload.customer_id}",
        message=payload.message,
    )
```

Each customer has full memory of past tickets. Bot personalizes over time.

### Pattern B: Coaching app

```python
@app.post("/coach")
def coach(payload: CoachRequest):
    return ask_buddypro(
        user_field=f"client-{payload.client_id}",
        message=payload.message,
    )
```

Each client gets their own ongoing relationship with the AI coach. The bot remembers their goals, preferences, progress.

### Pattern C: Multi-channel (web + mobile + voice)

If the same end-customer uses multiple channels, give them the SAME `user` value across all channels:

```python
# Web
user_field = f"customer-{customer.id}"

# Mobile
user_field = f"customer-{customer.id}"   # SAME

# Voice (phone)
user_field = f"customer-{customer.id}"   # SAME
```

The bot has unified memory: "You asked about pricing on the web yesterday — shall we continue?"

### Pattern D: Anonymous trials

For unauthenticated visitors trying out the bot:

```javascript
// Browser
let trialId = localStorage.getItem("trial_user_id");
if (!trialId) {
  trialId = `trial-${crypto.randomUUID()}`;
  localStorage.setItem("trial_user_id", trialId);
}
```

The trial ID persists in localStorage. If the visitor signs up later, you can stitch the trial profile to their authenticated profile by passing both IDs to BuddyPro and asking the bot to merge context (or just continue with the trial ID after auth — same effect).

## Migration: changing `user` value strategy

If you started with one ID strategy and want to switch (e.g., from numeric customer IDs to hashed emails), profiles **don't auto-migrate**.

**Workaround — explicit context bridge:**
```python
# Old key: customer-42
# New key: u-7d3f8a1b2c9e4f6a

# Send a context-bridging system prompt for the FIRST call under the new ID
ask_buddypro(
    user_field="u-7d3f8a1b2c9e4f6a",
    message="<your actual message>",
    extra={
        "x_buddy_systemPrompt": "Note: this user is the same person as previous profile customer-42. Their key context: <summarize key facts from old profile>",
        "x_buddy_systemPromptMode": "add",
    },
)
```

After a few exchanges, the new profile builds its own memory. Plan the migration during a low-traffic window.

## Debugging multi-tenant issues

### "Customer X is getting Customer Y's memory"

Almost certainly: same `user` value sent for both. Check:
- Are you base64-decoding/encoding the customer ID correctly?
- Is your customer ID truly unique? (e.g., not just `1`, `2`, `3` reused across tenants)
- Did you accidentally hard-code a `user` value in a test that's running in production?

### "Customer Z says the bot has no memory of their last conversation"

Possible causes:
- Different `user` value sent this time vs last time (typo, ID changed, regenerated)
- Last conversation used `x_buddy_saveToHistory: false` (nothing was saved)
- Profile was wiped (rare — only happens via support intervention)

Diagnostic call:
```bash
curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user": "customer-z-id", "messages": [{"role": "user", "content": "What do you know about me?"}]}' \
  | jq -r '.choices[0].message.content'
```

If the bot responds "I'm not sure I've spoken with you before" — profile is empty. ID mismatch with previous calls is the likely cause.

## Future: End User API

BuddyPro has announced (but not shipped) an **End User API** where each customer generates their OWN `bapi_` key directly. With end-user keys:
- Customer's conversations are private from the owner
- Customer can revoke their own key
- True GDPR-compliant deletion

When this ships, multi-tenant architectures should consider migrating from owner-API-with-`user`-field to genuine per-user keys.

## TL;DR

- Use a stable, unique `user` value per end-customer
- Don't reuse, don't randomly regenerate
- Hash sensitive IDs if privacy matters
- All profiles are visible to the API key owner — disclose this to customers
- 30/min rate limit is per key — scale via multiple keys or async queues
- Future End User API will solve the privacy-from-owner gap

*Last updated: 2026-05-08 v1.0.0)*
