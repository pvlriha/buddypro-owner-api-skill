# Troubleshooting

When something goes wrong with the BuddyPro Owner API, work through this file in order: HTTP-level errors first, then content-level issues, then instance-level problems, then known gotchas.

🔴 **Always inspect the response body first** — error info is in `error.code` and `error.message`, not just HTTP status. See `api-reference.md` for the HTTP status → error type matrix.

## HTTP-level errors

### 401 `authentication_error` / `invalid_api_key`

**Symptoms:** every call returns 401.

**Causes & fixes:**
1. `BUDDYPRO_API_KEY` env var not set in current shell — run `echo $BUDDYPRO_API_KEY` to verify. If empty, source the shell profile (`source ~/.zshrc`) or open a new terminal.
2. Key was invalidated — check in Telegram with `/getApiStats`. If the key isn't listed, it was revoked. Generate a new one.
3. Key was generated for a different bot — keys are bound to one instance.
4. Typo when copy-pasting — the key starts with `bapi_` and is alphanumeric. No quotes, no whitespace.

**Quick verify:**
```bash
curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"x_buddy_saveToHistory": false, "messages": [{"role": "user", "content": "ping"}]}' \
  | jq '.error // "OK"'
```

### 403 `permission_error` / `insufficient_permissions`

**Symptoms:** key works for normal messages but `/update`, `/messageAllUsers`, etc. return 403.

**Cause:** API key was generated from a non-owner profile (e.g., inside `/test:scenario` mode). Test profiles don't get management permissions.

**Fix:** in Telegram, run `/untest` first, then `/generateApiKey:owner-key`. The new key has full owner permissions.

### Generic "Command not allowed" reply (HTTP 200 — confusing!)

**Symptoms:** you call `/checkSetup`, `/stats`, `/investigateAnswer:`, etc. — HTTP 200 OK, but `choices[0].message.content` says:
```
Command not allowed. You might not have permission for this command or you spelled it wrong.
```

This is **NOT a 403 HTTP error** — it's a normal 200 response with a specific text. The bot is telling you: this command exists but your current permission/profile context can't run it.

**Common cause:** Same as 403 above — API key was generated from a profile that doesn't have management permissions, OR the key was created in a previous session where the user was in `/test:` mode without realizing it.

**Fix procedure:**
1. In Telegram (NOT via API — `/untest` is API-blocked):
   - Send `/untest` to the bot
   - Bot replies "Command not available. You are not on a testing profile" if you were already in owner mode (good) or "Returned to your real profile" (you were in test mode)
2. Verify owner profile: send `/myid` — should return your owner ID without `test_` prefix
3. Invalidate the suspicious key: `/invalidateApiKey:{old-key-name}`
4. Generate a fresh key: `/generateApiKey:owner-key-v2`
5. Update `BUDDYPRO_API_KEY` env var with the new key
6. Re-test the failing command — should now return real data

### 429 `rate_limit_error` / `rate_limit_exceeded`

**Symptoms:** intermittent 429 during batch jobs.

**Cause:** Over 30 requests/minute per API key.

**Fixes:**
- Add `sleep 2.5` between calls (max 24 req/min, safe under the limit).
- Use exponential backoff:
  ```bash
  for attempt in 1 2 3 4 5; do
    response=$(curl -s -w "%{http_code}" ...)
    code=$(echo "$response" | tail -c 4)
    [ "$code" != "429" ] && break
    sleep $((2 ** attempt))
  done
  ```
- For real batch processing (>30 items), spread across multiple keys (each key gets its own 30/min budget) or pace over time.

### 400 `invalid_request_error`

**🔴 Common gotcha (empirically discovered 2026-05-07):**

`user` field validation REJECTS diacritics (á, é, í, ó, ú, ě, š, č, ř, ž) and other non-ASCII characters. Czech app developers using Czech topic names as `user` IDs get **400 every time**.

**Symptom:** Bad Request 400 on every call to a specific `user` value, while other ASCII users work fine.

**Fix:** Transliterate or sanitize `user` value to ASCII before sending.

```python
import unicodedata, re

def safe_user_id(raw: str) -> str:
    \"\"\"Convert any string to BuddyPro-safe user ID.
    Allowed: [A-Za-z0-9._-], max 128 chars, not purely numeric, no spaces.
    \"\"\"
    # Strip diacritics
    normalized = unicodedata.normalize('NFKD', raw)
    ascii_only = normalized.encode('ascii', 'ignore').decode('ascii')
    # Replace anything non-allowed with '-'
    safe = re.sub(r'[^A-Za-z0-9._-]', '-', ascii_only)
    # Collapse repeated dashes
    safe = re.sub(r'-+', '-', safe).strip('-')
    # Ensure not purely numeric
    if safe.isdigit():
        safe = f'u-{safe}'
    return safe[:128]

# Examples:
safe_user_id('téma_a_publikum')          # → 'tema_a_publikum'
safe_user_id('Customer #42 (Jiří)')      # → 'Customer-42-Jiri'
safe_user_id('zákazník-VIP')             # → 'zakaznik-VIP'
```

**Read `error.code` and `error.param` for other 400 errors:**

| `error.code` | Meaning | Fix |
|--------------|---------|-----|
| `invalid_json` | Request body isn't valid JSON | Use a JSON encoder (jq), don't hand-craft escapes |
| `missing_required_parameter` | `messages` field missing | Add `messages: [{...}]` |
| `invalid_text_content` | Empty or >50,000 chars | Trim to 50K |
| `invalid_image_count` | More than 5 images | Reduce to ≤5 |
| `invalid_audio_format` | Unsupported audio codec | Use mp3/wav/ogg/aac/flac |
| `invalid_media_data` | Failed to download/decode media | Verify URL is HTTPS public, base64 is clean |
| `invalid_value` | One field has wrong type/value | `error.param` names the field |

### 5xx `server_error`

Transient. Retry with exponential backoff (3 attempts, 1s/2s/4s). If all 3 fail, the instance backend is having issues — wait and retry later.

### Network / DNS / timeout

Outside HTTP layer:
- `Could not resolve host: api.buddypro.ai` → DNS issue, check internet
- `Connection refused` → backend down, check status
- `SSL handshake failed` → likely firewall/proxy issue, check corporate network rules

## Content-level issues

### "Bot is asking clarifying questions instead of answering"

**Symptoms:** Owner asks something like *„udělej mi průzkum o vysoce ziskových webinářích"* and the bot replies *„Pojďme se nejdřív zorientovat — jaký je tvůj cíl? Pro koho to píšeš?"* instead of giving frameworks. Especially common for short or generic prompts. Especially destructive in batch eval and deep research where 30-50% of returned content is clarification noise.

**Cause:** The bot's default behavior is to ask clarifying questions when context is incomplete. Without an explicit instruction to skip clarification and lead with frameworks, it does what coaches do — ask before answering.

**Fix:** Apply the canonical anti-clarification directive via `x_buddy_systemPrompt` with `x_buddy_systemPromptMode: "add"`. Full directive text + code skeleton in `deep-research-architecture.md` → „🔴 MANDATORY: Anti-Clarification Directive on EVERY API Call" section.

Quick example:
```bash
DIRECTIVE='## DIRECTIVE PRO TENTO REQUEST
Pracuj okamžitě s tím, co je v otázce. NEDOPTÁVEJ se. Nabídni 3-7 konkrétních rámců.
Délka: 250-500 slov hutného obsahu. Žádné „to záleží".'

curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg d "$DIRECTIVE" '{
    x_buddy_systemPrompt: $d,
    x_buddy_systemPromptMode: "add",
    messages: [{role: "user", content: "tvoje krátká otázka tady"}]
  }')"
```

🔴 **Always use `add` mode, never `replace`.** Replace would erase the bot's voice rules and persona — we want expertise + voice intact, just stripped of the clarifying behavior.

🔴 **Especially critical for deep research / batch eval.** If you're running 18-25+ calls in a pipeline, the directive must be on EVERY call — not just the first.

### "The bot answers nothing useful"

The HTTP call succeeds, but `choices[0].message.content` is generic, evasive, or off-topic.

1. **Check if knowledge is loaded:**
   ```bash
   curl -X POST https://api.buddypro.ai/v1/chat/completions \
     -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"messages": [{"role": "user", "content": "/checkSetup"}]}'
   ```
   If `/checkSetup` flags missing knowledge or system prompt, fix those first.

2. **Investigate the answer:**
   ```bash
   curl -X POST https://api.buddypro.ai/v1/chat/completions \
     -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"messages": [{"role": "user", "content": "/investigateAnswer:specific question"}]}'
   ```
   The reply shows the knowhow chunks retrieved + role used. If chunks are missing or irrelevant → knowledge gap (add SOURCES → `/update`). If chunks are good but answer is still bad → system prompt issue.

3. **Check if `/update` ever ran:**
   New instances need at least one successful `/update` before they have knowledge.

### "Same question gives different answers"

Expected. The bot picks dynamically (role selection from last 3 messages). For consistency:
- Use stateless mode (`x_buddy_saveToHistory: false`) for evaluation
- Or use a stable `user` field so the conversation context is consistent
- Or be more specific in the question (less ambiguity = more stable role activation)

### "The response is in the wrong language"

Bot detects language from user message. To force a language:
- Include the desired language in the message itself ("Please answer in English: ...")
- Or use `x_buddy_systemPrompt` with `mode: "add"` and a language directive
- Or set `/setLanguage:{code}` instance-wide (this controls ADMIN messages only, not user replies)

### `x_buddy_systemPrompt` with `replace` mode doesn't fully override

**Symptoms:** you set `x_buddy_systemPromptMode: "replace"` with a strict instruction (e.g., „respond ONLY with valid JSON in format X") — but the bot ignores it and responds in its normal voice.

**What we observed:** Even in `replace` mode, the bot retains some core instance behavior (its persona, response style, the tendency to add emoji or commentary). For example, asked „what is 2+2?" with a strict JSON system prompt, the bot replied „4 😄" instead of `{"answer": "4"}`.

**Why:** The instance has a strong baseline behavior baked into its trained personality + role definitions. The custom system prompt influences but doesn't fully replace the core persona. **`replace` is more of a "strong override" than a "hard reset."**

**Workarounds:**
1. **Make the instruction extremely explicit and repeated:** *„You MUST respond ONLY with valid JSON. NO prose. NO emoji. NO commentary. JUST JSON. Example: {\"answer\": \"4\"}. Your response:"*
2. **Combine `replace` mode with `x_buddy_saveToHistory: false`** — fresh context each call, less of the bot's accumulated personality leaking in
3. **Use multiple-choice format** when possible — the bot complies better with constrained outputs than with strict format dictates
4. **Post-process the reply** — strip emoji, code-fences, or extra prose programmatically after the fact (see `code-recipes.md` Pattern 4)
5. **For mission-critical structured output:** consider not using BuddyPro and using a stock LLM API instead. BuddyPro is designed for personality, not structured data extraction.

### "Voice clone reply doesn't sound right"

- Audio sample was too noisy / had multiple speakers / too short → re-do `/createVoiceClone` with cleaner sample
- Voice clone is enabled per-user via `/useVoiceCloneForMe:true` — verify it's enabled in the right profile
- Cost cap reached (`/enableVoiceWithMusicGeneration:true:LIMIT`) — bot falls back to text only when budget exhausted

## Instance-level issues

### "Bot was working, now nothing"

1. **Credits exhausted.** Check via `/stats`:
   ```bash
   curl -X POST https://api.buddypro.ai/v1/chat/completions \
     -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"messages": [{"role": "user", "content": "/stats"}]}'
   ```
   If credit balance is 0 and auto-recharge failed, the AI Expert stops responding. Top up via `/setupCredits`.

2. **`/update` is running.** Long updates (large knowledge bases) can take hours. The bot is still responsive but new content isn't searchable yet.

3. **Instance was disabled.** Rare but possible — check Telegram for any system messages from BuddyPro.

### "Knowledge changes don't take effect"

- Did you run `/update` after editing? Without it, edits are invisible.
- Wait for `/update` to fully complete. Premature testing shows pre-update behavior.
- For role-specific changes, also run `/updateRoles` after `/update` finishes.
- For transcription settings changes, you must delete existing transcriptions to force re-processing.

### "Test profile (`user` field) is polluted with weird memory"

Each `user` value is a separate profile. To reset:
- Use a different `user` value (e.g., add a date suffix: `customer-42-v2`)
- Or there's no public API to delete a profile — contact BuddyPro support

## Known gotchas

These are all from real testing and learned the hard way.

| # | Gotcha | Solution |
|---|--------|----------|
| 1 | `/test` and `/untest` blocked over API | They're Telegram-only by design — switch profiles in Telegram before generating the API key |
| 2 | API key from `/test:` profile lacks management permissions | Generate the key from the owner profile (run `/untest` first) |
| 3 | `/update` returns success quickly but processes for hours | Don't poll faster than every 5 min; bot reports progress every 15 min |
| 4 | Without `user` field, all calls go to owner profile | Pollutes owner's memory. Use `user` for any non-owner traffic |
| 5 | Response includes the user message + bot reply, not just bot reply | `choices[0].message.content` IS the bot's reply — that's correct |
| 6 | Sending past conversation in `messages` array | Server holds history. Send only current message. Otherwise = duplicate context |
| 7 | `/setDefaultCost` period must be English | Use `months`, `years` — not `měsíc`, `rok`, etc. |
| 8 | Trial invite code must be exactly 7 chars | `/generateBuddyProInvite:{msgs}:{7CHARCODE}:...` — pad short codes |
| 9 | `/listInvites` hides codes used by <2 people | Use `/checkInvite:{code}` for fresh codes |
| 10 | `/messageAllUsers` first param is dryRun | `true` = test, `false` = real send. **Always test with `true` first!** |
| 11 | `bapi_` keys shown only ONCE | Save immediately on generation. There's no recovery |
| 12 | Empty knowledge base → bot still responds (with system prompt only) | Generic answers. Run `/update` after uploading content |
| 13 | Multiple SOURCES with same content | Bot becomes LESS smart, not more. Avoid duplicates |
| 14 | System prompt with `## Header` markdown | Renders literally to user. Use XML tags + CAPS instead |
| 15 | `.txt` or `.md` files in SOURCES/TEXTS | Silently ignored. Must be Google Docs |
| 16 | `URL SOURCES` exists in BOTH `SOURCES/URLS/` and `RAW SOURCES/URLS/` | Use the one in `SOURCES/URLS/` for chunked content |
| 17 | YouTube videos longer than ~2 hours | Won't process. Split or use podcast transcript |
| 18 | Vimeo videos | Require Standard plan + access token; usually skip |
| 19 | Some websites block scraping | Use the YouTube/Vimeo version of the content if available |
| 20 | `/teach` command in docs | NOT IMPLEMENTED — returns "Command not allowed". Use TEXTS folder instead |
| 21 | Service account can't upload files to Drive | Edit existing docs or upload from owner's browser |
| 22 | Service account can't delete some transcriptions | Owner deletes manually in Drive |
| 23 | `/del` is silent — no confirmation | No response = success |
| 24 | Audio response without `modalities: ["text", "audio"]` | Won't get TTS. Add the modality |
| 25 | `audio.voice` field accepted but ignored | Set voice instance-wide via `/setVoice` or voice clone |
| 26 | `model` field in request body | Accepted but ignored. Owner controls model via `/setModel` |
| 27 | `usage` token statistics in response | Not returned (BuddyPro doesn't expose this). Don't depend on it |
| 28 | Streaming requested in body | Not supported yet. Always synchronous response |
| 29 | Image URL pointing to private/internal IP | Blocked by SSRF protection. Use public HTTPS URL or base64 |
| 30 | More than 5 images in one request | Returns `invalid_image_count`. Split across requests |

## When all else fails

1. **Capture `x-request-id`** from the response headers and include it in any support request.
2. **Save the full error response body** for context.
3. **Check official docs:** `https://docs.buddypro.ai/owner-api/` for the latest endpoint behavior.
4. **Ask the BuddyPro support channel** — provide request ID, error code, and what you were trying to do.

## Self-test recipe

When everything seems weird, run this end-to-end smoke test:
```bash
# 1. Auth
curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"x_buddy_saveToHistory": false, "messages": [{"role": "user", "content": "ping"}]}' \
  | jq '.error.code // "auth-ok"'

# 2. Knowledge present
curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "/checkSetup"}]}' \
  | jq -r '.choices[0].message.content'

# 3. Multi-tenancy works
curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user": "smoke-test-x", "x_buddy_saveToHistory": false, "messages": [{"role": "user", "content": "Hi, I am new"}]}' \
  | jq -r '.choices[0].message.content'
```

If all three pass, the integration is healthy.

*Last updated: 2026-05-07 (v0.2.x)*
