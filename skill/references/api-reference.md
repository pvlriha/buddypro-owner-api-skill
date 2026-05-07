# API Reference — `/v1/chat/completions`

Complete spec for the BuddyPro Owner API endpoint. Sources: live testing on production instances + official docs at https://docs.buddypro.ai/owner-api/ + internal source code analysis.

## Endpoint & Authentication

```
POST https://api.buddypro.ai/v1/chat/completions
Authorization: Bearer bapi_xxxxxxxxxxxx
Content-Type: application/json
```

**Streaming:** Not supported yet. Responses are synchronous — they return when generation completes.

## API Key Management

API keys are managed in Telegram (security — keys are shown only once and never logged):

| Telegram command | What |
|------------------|------|
| `/generateApiKey:{name}` | Create new key. Returns `bapi_...` shown ONCE. |
| `/invalidateApiKey:{name_or_key}` | Revoke key by name or full token. |
| `/getApiStats` | List all active keys with usage stats (key values are NOT shown). |

**4 commands are blocked over the API for security** — they must be sent in Telegram:
- `/generateApiKey` (would log the key)
- `/invalidateApiKey` (security)
- `/test` (would change profile context of the key)
- `/untest` (same reason)

## Request Body

### Required

| Field | Type | Notes |
|-------|------|-------|
| `messages` | array | **Exactly 1 user message.** Server holds conversation history — never include past turns. |

### Optional — Multimodal & Output

| Field | Type | Notes |
|-------|------|-------|
| `modalities` | array | Default `["text"]`. Add `"audio"` for TTS reply: `["text", "audio"]`. |
| `audio` | object | Required if `modalities` includes audio. `{"format": "mp3"}` or `"wav"`. |

### Optional — Profile & Persistence

| Field | Type | Notes |
|-------|------|-------|
| `user` | string | Isolated profile ID. **Without it, conversation goes to owner's main profile (= the Telegram account that generated the key).** Allowed: alphanumeric, dash, underscore, dot. Max 128 chars. NOT purely numeric, no spaces. |
| `x_buddy_saveToHistory` | boolean | `false` = stateless: nothing persists to memory/history/profile. Default `true`. |

### Optional — Per-request System Prompt

| Field | Type | Notes |
|-------|------|-------|
| `x_buddy_systemPrompt` | string | One-shot system prompt override. Max 50,000 chars. |
| `x_buddy_systemPromptMode` | string | `"add"` (default — append to base prompt) or `"replace"` (override base entirely). |

### Optional — Tracing

| Header | Notes |
|--------|-------|
| `X-Client-Request-Id` | Max 64 chars, alphanumeric + `-_`. Echoed back in response headers for client-side tracing. |

### Ignored fields (accepted but no effect)

- `model` — instance owner controls the model via `/setModel`, not per-request
- `audio.voice` — set instance-wide via `/setVoice` or voice clone
- Any other OpenAI-compat fields not listed above

## Message Content Types

### Plain text

```json
{"messages": [{"role": "user", "content": "What is the weather like today?"}]}
```

Max **50,000 characters** per text part.

### Multimodal array

```json
{
  "messages": [{
    "role": "user",
    "content": [
      {"type": "text", "text": "What is in this image?"},
      {"type": "image_url", "image_url": {"url": "https://example.com/photo.jpg"}}
    ]
  }]
}
```

#### Content parts

**`text`** — `{"type": "text", "text": "..."}` — max 50,000 chars.

**`image_url`** — `{"type": "image_url", "image_url": {"url": "https://..."}}`
- HTTPS URLs or `data:` URIs (base64)
- Max **5 images per request**
- Max 40 MB per remote download
- SSRF protection: private/internal/loopback URLs are blocked

**`input_audio`** — `{"type": "input_audio", "input_audio": {"data": "...", "format": "mp3", "type": "base64"}}`
- Formats: `mp3`, `wav`, `ogg`, `aac`, `flac`
- Same 40 MB limit

## Response

### Text reply

```json
{
  "id": "chatcmpl-req_abc123",
  "object": "chat.completion",
  "created": 1710964800,
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "Reply text here"},
    "finish_reason": "stop"
  }]
}
```

### With image output (BuddyPro extension)

When the bot generates an image, `message.image` is added:

```json
{
  "message": {
    "role": "assistant",
    "content": "Here's the image you asked for",
    "image": {
      "id": "image_req_...",
      "data": "<base64>",
      "media_type": "image/png",
      "file_name": "generated.png",
      "caption": "..."
    }
  }
}
```

### With audio output (TTS, BuddyPro extension)

When `modalities: ["text", "audio"]` is set and TTS succeeds:

```json
{
  "message": {
    "role": "assistant",
    "content": "transcript text",
    "audio": {
      "id": "audio_req_...",
      "data": "<base64>",
      "format": "mp3",
      "transcript": "...",
      "media_type": "audio/mpeg",
      "file_name": "response.mp3"
    }
  }
}
```

Only the **first image** and **first audio** per choice are surfaced. Additional media is dropped.

### Response headers

| Header | Notes |
|--------|-------|
| `x-request-id` | Server-generated unique ID (always present) — log this for support tickets |
| `x-client-request-id` | Echo of client-supplied `X-Client-Request-Id` (only if sent) |
| `Content-Type` | Always `application/json` |

## Errors

```json
{
  "error": {
    "message": "Human-readable description",
    "type": "error_category",
    "statusCode": 401,
    "code": "machine_readable_code",
    "param": "field_name_or_null"
  }
}
```

🔴 **Don't rely on HTTP status alone.** Always inspect response body for `error` field.

### HTTP status → error type

| Status | Type | When |
|--------|------|------|
| 400 | `invalid_request_error` | Malformed JSON, missing field, invalid value |
| 401 | `authentication_error` | Missing or invalid API key |
| 403 | `permission_error` | Key valid but lacks required permissions |
| 405 | `method_not_allowed` | Wrong HTTP method (must be POST) |
| 410 | `gone` | Endpoint deprecated |
| 429 | `rate_limit_error` | Over 30 req/min per key |
| 500 | `server_error` | Internal server error |

### Common error codes

`invalid_json` | `missing_required_parameter` | `missing_api_key` | `invalid_api_key` | `invalid_value` | `invalid_text_content` | `invalid_content_type` | `invalid_content` | `invalid_media_data` | `invalid_media_type` | `invalid_audio_format` | `invalid_image_count` | `invalid_parameter` | `rate_limit_exceeded` | `insufficient_permissions`

## Limits

| Limit | Value |
|-------|-------|
| Text per part | 50,000 chars |
| Images per request | 5 |
| Per-media download | 40 MB |
| Rate limit | 30 req/min per API key |
| Messages array length | 1 (server holds history) |

## Mental model — how this differs from stock OpenAI

| Behavior | OpenAI | BuddyPro |
|----------|--------|----------|
| Conversation history | Client sends all turns | **Server holds history** — send only current message |
| User profiles | One key = one stateless model | One key = stateful profile (or many via `user` field) |
| System prompt | Per-request, always | Instance has a base prompt; `x_buddy_systemPrompt` overrides per-request |
| `usage` token stats | Returned | NOT returned (BuddyPro doesn't expose this) |
| Streaming | Supported | Not yet |

## Privacy & recommended use

Per official docs:
- **Don't process real third-party people's profiles via this API** — instance owner has access to all profiles created via the key.
- **Recommended:** internal automation, service-to-service integrations where the owner is also the consumer.
- **Future:** End User API (separate keys per end-user) — not yet shipped.

## Source of truth

When in doubt, check (in order):
1. https://docs.buddypro.ai/owner-api/ — official endpoint spec
2. This file (synthesized from official docs + tested behavior)
3. `https://docs.buddypro.ai/{section}/{page}` — root-level paths only (the `/dashboard/*` paths are duplicates and should be ignored)
4. Source code if accessible: `github.com/buddy-fm/buddy` — `telegram/API/BuddyApi/` for endpoint behavior, `telegram/Invoked Functions/replyToUser.ts` for message processing pipeline

*Last updated: 2026-05-07 (v0.2.0)*
