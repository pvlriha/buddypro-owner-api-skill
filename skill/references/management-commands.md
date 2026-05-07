# Management Commands via Owner API

🎯 **Key insight:** Almost every command you'd type into Telegram works through `/v1/chat/completions` too. Just send the slash command as the user message text. Source: confirmed in `BuddyApiCommands.ts` of `buddy-fm/buddy`.

## 🔴 CRITICAL — Safety Policy (read this first, every time)

These commands change real instance behavior, real user experience, real money. The skill **MUST follow the safety procedure for the matching risk level** before executing any non-green command. No exceptions.

### Risk levels

| Level | Scope | Confirmation requirement |
|-------|-------|--------------------------|
| 🟢 **GREEN — Safe** | Read-only inspection | No confirmation needed |
| 🟡 **YELLOW — Limited scope** | Per-profile or non-destructive change | Single explicit "yes/no" confirmation in user's language |
| 🟠 **ORANGE — High risk** | Changes instance behavior or accumulates cost | DOUBLE explicit confirmation — explain WHAT and WHY, get yes, then re-state and get yes again |
| 🔴 **RED — Extreme danger** | Mass impact, financial, irreversible | Risk warning + DOUBLE confirmation + (if broadcast) MANDATORY 3-step procedure below |

### 🔴 Mandatory broadcast procedure for `/messageAllUsers`

NEVER call `/messageAllUsers` with `audience: all|subscribed|trial|active|inactive|new` directly. **Always run this 2-step procedure**, even if the user pushes for shortcut. No skipping. No "the user said it's fine."

**Step 1 — Real send to OWNER ONLY (test):**

The owner sees the message exactly as customers would see it — formatting, link previews, line breaks, emoji rendering. Cheap, instant verification.

```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "/messageAllUsers:false:text:me:static:Your draft message here"}]}'
```

`audience: me` = sends ONLY to the owner (the Telegram account that generated the API key). No customer sees this message. Bot delivers a single message to the owner's Telegram chat with the bot.

After sending, tell the user (in their language):
> *„The test message just landed in your Telegram conversation with the bot. Open Telegram and read it carefully. Check formatting, links, line breaks. When you're sure it looks right, reply here with 'send to all' (or 'pošli všem'). To abort, reply 'cancel'."*

**Step 2 — Wait for explicit verbal confirmation, then real broadcast:**

ONLY after the user confirms in any language (*„send to all"*, *„pošli všem"*, *„yes broadcast"*, *„OK rozešli"*) — and ONLY then:

```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "/messageAllUsers:false:text:all:static:Your final message here"}]}'
```

> **Note about the BuddyPro `dryRun` parameter:** the syntax `/messageAllUsers:{dryRun}:...` accepts `true` (test mode — no sending) or `false` (real send). We use `false` in BOTH steps because Step 1 is `audience: me` (real send to owner), not a simulation. If a user explicitly asks for a "preview before anything sends," you can prepend a `dryRun: true` call — but that's optional. The 2-step procedure (test-to-me → broadcast) is mandatory.

🔴 **The skill must REFUSE to skip Step 1 even if user says "I trust it, just send."** This rule exists because broadcasts to real users are unrecoverable. A misspelled message reaches 1,000 customers. A wrong link reaches their inboxes. The 5-second delay of Step 1 is the cheapest insurance possible.

### Confirmation templates (any language — translate naturally)

**🟡 Yellow — single confirmation:**
> EN: *„This will {effect on user/profile/system}. Confirm? (yes/no)"*
> CZ: *„Tohle udělá {effect on user/profile/system}. Potvrzuješ? (ano/ne)"*

**🟠 Orange — double confirmation:**
> EN, first: *„⚠️ This will {WHAT}. It changes {SCOPE} and is {hard/easy} to undo. Confirm? (yes/no)"*
> EN, after first yes: *„One more check — to be absolutely sure: this will {RE-STATE}. Final confirmation? (yes/no)"*
> CZ, first: *„⚠️ Tohle udělá {WHAT}. Změní {SCOPE} a {jde/nejde} to vrátit zpět. Potvrzuješ? (ano/ne)"*
> CZ, after first yes: *„Ještě jedna kontrola — pro jistotu: tohle udělá {RE-STATE}. Finální potvrzení? (ano/ne)"*

**🔴 Red — extreme danger:**
> EN: *„🛑 STOP. This is a destructive operation: {action} → {consequence}. There is no undo. Type the words 'YES, I UNDERSTAND' exactly to proceed, or anything else to abort."*
> CZ: *„🛑 STOP. Tohle je destruktivní operace: {action} → {consequence}. Nelze vrátit zpět. Napiš přesně 'ANO, ROZUMÍM' pro pokračování, cokoliv jiného přeruší."*

For broadcasts specifically: replace the red template with the **3-step procedure above**. Never use the red template alone for `/messageAllUsers`.

### Skill self-check before sending

Before passing a slash command to the API, the skill MUST:
1. Look up the command in the risk table below.
2. If 🟢 — proceed.
3. If 🟡 — get single confirmation in user's language. Wait for explicit yes.
4. If 🟠 — get double confirmation. Wait for both yes.
5. If 🔴 — `/messageAllUsers` triggers 3-step procedure. Other 🔴 commands trigger destructive-confirmation pattern.
6. If you don't recognize a slash command — treat it as 🟠 by default and ask the user what it does before sending.

---

## How it works

Send the slash command as the `content` field of the user message, exactly as you would type it in Telegram:

```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "/checkSetup"}]}'
```

The bot's response is returned in `choices[0].message.content` just like a normal answer.

## 🔴 4 commands BLOCKED over the API (Telegram-only)

| Command | Why blocked |
|---------|-------------|
| `/generateApiKey:{name}` | Generated key would be logged in API conversation history (security) |
| `/invalidateApiKey:{name}` | Same security reason |
| `/test:{scenario}` | Would change profile context of the API key — meaningless and confusing |
| `/untest` | Same |

If you call these via API, the bot responds: *„This command is not available via the API. Please use Telegram directly."*

For all other commands, API works. Below: organized list with status from live testing.

## ⚠️ Permission caveat

API key is bound to the profile that generated it (no `user` field = owner profile, with `user` = isolated profile). Management commands generally require **owner permissions** — they only work when the API key was generated from the owner's profile (the Telegram account that ran `/createPro`), without a `user` override in the request.

If you want management commands available, generate the key WITHOUT calling `/test:` first. If you want to lock down the key (e.g. for a customer-facing agent that shouldn't run `/messageAllUsers`), generate it inside a `/test:` profile — slash commands will still be received by the bot but most management commands will be denied.

## Commands by category

Status legend: **proven** = tested live | **from-docs** = documented but not personally tested | **api-likely** = should work via API based on source code, untested | **api-no** = won't work via API regardless of profile

### A) Knowledge & Roles

These commands sync the bot with what's in the Drive folder. **They don't destroy anything** — at worst they re-process content (cost in credits) or take a long time. Generally safe to run.

| Risk | Command | Use case | Status |
|------|---------|----------|--------|
| 🟢 | `/update` | Process all new/changed knowledge in Drive (30s up to 5 hours). Just syncs Drive → vector DB. Non-destructive. | proven |
| 🟢 | `/updateRoles` | Generate expert subroles from knowledge (run AFTER `/update` finishes). Just regenerates from current knowledge. | proven |
| 🟢 | `/updateNextRole` | Generate one next pending role | from-docs |
| 🟢 | `/updateRole:{name}` | Generate/update one specific role | from-docs |
| 🟡 | `/stopUpdate` | Cancel running update — may leave half-state, you'll likely need to re-run `/update` | proven |
| 🟡 | `/stopUpdateRoles` | Cancel role generation — may leave half-state | from-docs |
| 🟢 | `/teach` | **NOT IMPLEMENTED** — returns "Command not allowed" | api-no |

**API gotcha for `/update`:** Update can take up to 5 hours. The HTTP request to the API will return quickly with a confirmation/progress message; the actual processing continues server-side. Don't poll faster than every 5–15 minutes. Bot reports progress every ~15 min.

### B) Testing & Diagnostics

| Risk | Command | Use case | Status |
|------|---------|----------|--------|
| 🟢 | `/investigateAnswer:{question}` | Generate answer + show which knowhow chunks + role were used. Read-only diagnostic. | proven |
| 🟢 | `/lastRole` | Show role activated for the last message in this profile | proven |
| 🟡 | `/del` | Silently delete the last message from memory + Telegram (data loss in this profile) | proven |
| 🟡 | `/del2` (or `/del10`) | Delete last 2 (or 10) messages | from-docs |
| 🟡 | `/setModelForMe:{SMART/ECONOMY}` | Switch AI tier for this profile only | proven |
| 🟢 | `/checkSetup` | List ALL missing config (read-only audit) | proven |
| 🟢 | `/help` | Multi-message full reference (read-only) | proven |

**API note:** When calling `/investigateAnswer:{question}` via API, the bot responds with the analysis as a single text reply. Easy to parse programmatically — much faster than copying from Telegram.

### C) Configuration (instance-wide)

These all change instance behavior visible to real customers — 🟠 by default.

| Risk | Command | Format | Status |
|------|---------|--------|--------|
| 🟠 | `/shouldInitiateMessages:true/false` | Toggle proactive messages from bot (cost impact!) | from-docs |
| 🟠 | `/setSupportEmail:{email}` | Customer support contact (visible to real users) | from-docs |
| 🟠 | `/setVoice:{voice}` | TTS voice: `alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer` | from-docs |
| 🟠 | `/setTrialEndedMessage:{msg}` | End-of-trial message — must contain `<LINK>` placeholder | proven |
| 🟠 | `/setInviteTrialMessages:{count}:{timeLimit}` | Trial allowance for invitees | proven |
| 🔴 | `/setDefaultCost:{cost}:{currency}:{period}:{quantity}` | **Subscription pricing — affects ALL future sales!** Period must be English: `months`, `years` (not `měsíc`) | proven, gotcha |
| 🟠 | `/setPersonalInviteLimit:{limit}` | Max invites per user | from-docs |
| 🟠 | `/setInviteReward:{inviter}:{invitee}:{trigger}` | Bonus messages, trigger: `trial`/`paid`/`both` | from-docs |
| 🟠 | `/setLanguage:{code}` | Admin system messages language (NOT end-user language — that's auto-detected) | from-docs |

### D) Invites & Users

| Risk | Command | Format | Status |
|------|---------|--------|--------|
| 🟡 | `/generateBuddyProInvite:{msgs}:{7CHARCODE}:{users}:{expire}` | Trial invite. CODE must be exactly 7 chars. | proven, gotcha |
| 🟡 | `/generateInviteSub:{code}` | Invite that leads to subscription | from-docs |
| 🟢 | `/listInvites` | Active codes (read-only; hides codes used by <2 users) | proven, gotcha |
| 🟠 | `/deleteInvite:{code}` | Remove invite — existing users with this code may lose context | proven |
| 🟢 | `/checkInvite:{code}` | Usage stats (read-only) | proven |
| 🟠 | `/setInviteLimit:{code}:{newLimit}` | Change user limit on an existing code | from-docs |
| 🔴 | `/messageAllUsers:{dryRun}:text:{audience}:{type}:{message}` | **MASS BROADCAST — use mandatory 2-step procedure above** | proven |

**`/messageAllUsers` parameters:**

- `dryRun`: `true` = bot returns preview only, no actual sending. `false` = real send. Optional first-line check before the 2-step procedure.
- `audience`: `me` | `all` | `subscribed` | `trial` | `active` | `active(N)` | `inactive(N)` | `new` | `new(N)`
- `type`:
  - `dynamic` (~$0.16/user) — personalized per user, **expensive**
  - `static` (~$0.001/user) — same text to all, cheap

**🔴 Skill MUST follow the 2-step broadcast procedure** for ALL audiences except `me` itself. See top of this file.

### E) Voice

| Risk | Command | Use case | Status |
|------|---------|----------|--------|
| 🟠 | `/createVoiceClone` | Clone voice from audio sample (overwrites existing clone!) | proven |
| 🟠 | `/useVoiceCloneForEverybody:true/false` | Enable cloned voice for all users (~2× cost per message) | proven |
| 🟡 | `/useVoiceCloneForMe:true/false` | Enable for owner only | from-docs |
| 🟠 | `/enableVoiceWithMusicGeneration:true:{limit}` | Audio meditations — cost 2–6× standard message | proven |

**Voice clone via API:** `/createVoiceClone` works via API, but you also need to send the audio sample. The audio goes as `input_audio` in a follow-up call:

```json
{
  "messages": [{
    "role": "user",
    "content": [{"type": "input_audio", "input_audio": {"data": "<base64>", "format": "mp3"}}]
  }]
}
```

Send `/createVoiceClone` first (text), then the audio in the next call (within the same profile).

### F) Payments

| Risk | Command | Use case | Status |
|------|---------|----------|--------|
| 🔴 | `/setupStripe` | **Connect Stripe — payments routing!** Bot guides through OAuth | from-docs |
| 🔴 | `/setupFapi:{username}:{apikey}` | **Connect FAPI (CZ payment provider) — payments routing!** | from-docs |
| 🔴 | `/connectForm` | Connect a FAPI form to a product | from-docs |

Setup typically requires browser interaction (Stripe OAuth) — best done via Telegram for the first time, then everything else can be API.

### G) Team Management

Owner = the Telegram account that ran `/createPro`. Owner is implicitly always a team member.

| Risk | Command | Use case | Status |
|------|---------|----------|--------|
| 🟠 | `/addUserToTeam:{userId}#{bot_username}` | Grant team admin access (must include `#{bot_username}` separator) | proven |
| 🟠 | `/removeUserFromTeam:{userId}` | Revoke team access | from-docs |
| 🟡 | `/addSystemMessageRecipient:{userId}` | Forward payment/ops alerts (must be team member first) | from-docs |
| 🟢 | `/myid` | Returns caller's Telegram numeric ID (read-only) | from-docs |

**Onboarding a team member procedure:**
1. The new person opens the bot in Telegram and sends any message (e.g. `/start`).
2. They send `/myid` and copy the returned numeric ID.
3. **Owner** sends `/addUserToTeam:{theirId}#{bot_username}` — must come from the owner's account/key.

Team members get: `/update`, `/updateRoles`, `/lastRole`, invite generation, user management. They cannot: change owner settings, do critical security ops.

### H) AI Credits & Stats

| Risk | Command | Use case | Status |
|------|---------|----------|--------|
| 🟢 | `/stats` | Dashboard: users, WAU/MAU, MRR, costs, credit balance (read-only) | proven |
| 🔴 | `/setupCredits:{topUp}:{rechargeAt}` | **Auto-recharge config — min $250 charge!** | from-docs |
| 🔴 | `/changeCreditsTopUp:{newTopUp}:{newRechargeAt}` | Change credit recharge settings | from-docs |
| 🟢 | `/getInfoAboutUser:{subId}` | Subscription details for a user (read-only) | from-docs |
| 🔴 | `/disableUser:{subId}` | **Terminate user access — real customer loses access!** | from-docs |
| 🟡 (positive) / 🟠 (negative) | `/giveExtraMessages:{subId}:{count}` | Add or REMOVE message allowance (negative = removal) | from-docs |
| 🟠 | `/setTrialMessages` | Adjust trial allowance using TG ID | from-docs |
| 🟡 | `/addTrialMessages` | Add to existing trial | from-docs |

**Credit gotcha:** When credits hit zero and auto-recharge fails, the AI Expert stops responding. Hourly alerts go to system message recipients.

### I) Setup (mostly Telegram-only first time)

| Risk | Command | Use case | Status |
|------|---------|----------|--------|
| 🔴 | `/setFolder:{driveUrl}` | **Re-connect Drive folder — could lose connection to current knowledge base!** Wait 15s for response. | proven, gotcha |
| 🟠 | `/setup` | Guided initial setup walkthrough | from-docs |
| 🟠 | `/connectWeb` | Connect to web dashboard | from-docs |
| 🟢 | `/reminders` | List active reminders (read-only) | from-docs |

## API workflow patterns

### Pattern 1: Refresh knowledge after content edit

```bash
# 1. (User edits Google Doc in SOURCES/ via browser — no API call needed)
# 2. Trigger update via API
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "/update"}]}'
# 3. Wait — `/update` returns confirmation but processes in background
# 4. Optionally, poll `/checkSetup` or `/stats` to verify completion
```

### Pattern 2: Programmatic answer-quality audit

```bash
# Test 10 questions covering main domains
for q in "${test_questions[@]}"; do
  curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
    -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg q "$q" '{messages: [{role: "user", content: ("/investigateAnswer:" + $q)}]}')"
  sleep 3   # rate-limit friendly
done
```

The response includes the answer + which knowhow chunks were retrieved + which role was used. Pipe to `jq` to extract `choices[0].message.content` and parse for analysis.

### Pattern 3: Daily stats check

```bash
# Cron job: 9 AM daily
curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "/stats"}]}' \
  | jq -r '.choices[0].message.content' | mail -s "BuddyPro daily stats" you@example.com
```

### Pattern 4: Pre-launch checklist

```bash
for cmd in "/checkSetup" "/listInvites" "/stats"; do
  echo "=== $cmd ==="
  curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
    -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg c "$cmd" '{messages: [{role: "user", content: $c}]}')" \
    | jq -r '.choices[0].message.content'
  echo
done
```

## When to fall back to Telegram

For these tasks, Telegram is still the easier path (or required):
- **First-time setup** — license activation, Drive folder connection (`/setup`, `/setFolder`)
- **Stripe OAuth** — browser interaction needed
- **Voice clone audio upload** — works via API but Telegram drag-and-drop is faster
- **Reading rich responses** — `/help` returns multi-part long output, easier to read in Telegram
- **The 4 explicitly blocked commands** (key management + test mode switch)

For everything else, API is faster, more consistent, and scriptable.

## Source

Live testing on Pavel Říha AI instance + `BuddyApiCommands.ts` source code analysis. When in doubt about a specific command's API behavior, check `https://docs.buddypro.ai/advanced/commands-list`.

*Last updated: 2026-05-07 (v0.2.0)*
