# Management Commands via Owner API

🎯 **Key insight:** Almost every command you'd type into Telegram works through `/v1/chat/completions` too. Just send the slash command as the user message text. Source: confirmed in `BuddyApiCommands.ts` of `buddy-fm/buddy`.

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

| Command | Use case | Status |
|---------|----------|--------|
| `/update` | Process all new/changed knowledge in Drive (30s up to 5 hours). Run after editing SOURCES, SYSTEM PROMPT, ONBOARDING. | proven |
| `/stopUpdate` | Cancel running update | proven |
| `/updateRoles` | Generate expert subroles from knowledge (run AFTER `/update` finishes) | proven |
| `/updateNextRole` | Generate one next pending role | from-docs |
| `/updateRole:{name}` | Generate/update one specific role | from-docs |
| `/stopUpdateRoles` | Cancel role generation | from-docs |
| `/teach` | **NOT IMPLEMENTED** — returns "Command not allowed" | api-no |

**API gotcha for `/update`:** Update can take up to 5 hours. The HTTP request to the API will return quickly with a confirmation/progress message; the actual processing continues server-side. Don't poll faster than every 5–15 minutes. Bot reports progress every ~15 min.

### B) Testing & Diagnostics

| Command | Use case | Status |
|---------|----------|--------|
| `/investigateAnswer:{question}` | Generate answer + show which knowhow chunks + role were used. **Essential for debugging.** | proven, api-likely |
| `/lastRole` | Show role activated for the last message in this profile | proven, api-likely |
| `/del` | Silently delete the last message from memory + Telegram | proven, api-likely |
| `/del2` (or `/del10`) | Delete last 2 (or 10) messages | from-docs |
| `/setModelForMe:{SMART/ECONOMY}` | Switch AI tier for this profile only | proven |
| `/checkSetup` | List ALL missing config (run this first when troubleshooting) | proven |
| `/help` | Multi-message full reference (very long output) | proven |

**API note:** When calling `/investigateAnswer:{question}` via API, the bot responds with the analysis as a single text reply. Easy to parse programmatically — much faster than copying from Telegram.

### C) Configuration (instance-wide)

| Command | Format | Status |
|---------|--------|--------|
| `/shouldInitiateMessages:true/false` | Toggle proactive messages from bot | from-docs |
| `/setSupportEmail:{email}` | Customer support contact | from-docs |
| `/setVoice:{voice}` | TTS voice: `alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer` | from-docs |
| `/setTrialEndedMessage:{msg}` | End-of-trial message — must contain `<LINK>` placeholder | proven |
| `/setInviteTrialMessages:{count}:{timeLimit}` | Trial allowance for invitees | proven |
| `/setDefaultCost:{cost}:{currency}:{period}:{quantity}` | Subscription pricing. **Period must be English: `months`, `years`** (not `měsíc`) | proven, gotcha |
| `/setPersonalInviteLimit:{limit}` | Max invites per user | from-docs |
| `/setInviteReward:{inviter}:{invitee}:{trigger}` | Bonus messages, trigger: `trial`/`paid`/`both` | from-docs |
| `/setLanguage:{code}` | Admin system messages language (NOT end-user language — that's auto-detected) | from-docs |

### D) Invites & Users

| Command | Format | Status |
|---------|--------|--------|
| `/generateBuddyProInvite:{msgs}:{7CHARCODE}:{users}:{expire}` | Trial invite. **CODE must be exactly 7 chars (alphanumeric uppercase).** | proven, gotcha |
| `/generateInviteSub:{code}` | Invite that leads to subscription | from-docs |
| `/listInvites` | Active codes (note: hides codes used by <2 users) | proven, gotcha |
| `/deleteInvite:{code}` | Remove invite | proven |
| `/checkInvite:{code}` | Usage stats for a specific invite | proven, sometimes-stale |
| `/setInviteLimit:{code}:{newLimit}` | Change user limit on an existing code | from-docs |
| `/messageAllUsers:{dryRun}:text:{audience}:{type}:{message}` | Broadcast — see below | proven |

**`/messageAllUsers` parameters:**

- `dryRun`: `true` (test, no send) or `false` (real send) — **always test with `true` first!**
- `audience`: `me` | `all` | `subscribed` | `trial` | `active` | `active(N)` | `inactive(N)` | `new` | `new(N)`
- `type`:
  - `dynamic` (~$0.16/user) — personalized per user, expensive
  - `static` (~$0.001/user) — same text to all, cheap

**API gotcha:** `/messageAllUsers` from API can broadcast to your real users. Treat with same caution as in Telegram. Always dry-run first.

### E) Voice

| Command | Use case | Status |
|---------|----------|--------|
| `/createVoiceClone` | Clone voice from audio sample (then send 30s–2min audio file as the next message) | proven |
| `/useVoiceCloneForEverybody:true/false` | Enable cloned voice for all users | proven |
| `/useVoiceCloneForMe:true/false` | Enable for owner only | from-docs |
| `/enableVoiceWithMusicGeneration:true:{limit}` | Audio meditations (cost: 2–6× standard message) | proven |

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

| Command | Use case | Status |
|---------|----------|--------|
| `/setupStripe` | Connect Stripe — bot guides through OAuth | from-docs |
| `/setupFapi:{username}:{apikey}` | Connect FAPI (CZ payment provider) | from-docs |
| `/connectForm` | Connect a FAPI form to a product | from-docs |

Setup typically requires browser interaction (Stripe OAuth) — best done via Telegram for the first time, then everything else can be API.

### G) Team Management

Owner = the Telegram account that ran `/createPro`. Owner is implicitly always a team member.

| Command | Use case | Status |
|---------|----------|--------|
| `/addUserToTeam:{userId}#{bot_username}` | Grant team access (must include `#{bot_username}` separator) | proven |
| `/removeUserFromTeam:{userId}` | Revoke team access | from-docs |
| `/addSystemMessageRecipient:{userId}` | Forward payment/ops alerts (must be team member first) | from-docs |
| `/myid` | Returns caller's Telegram numeric ID | from-docs |

**Onboarding a team member procedure:**
1. The new person opens the bot in Telegram and sends any message (e.g. `/start`).
2. They send `/myid` and copy the returned numeric ID.
3. **Owner** sends `/addUserToTeam:{theirId}#{bot_username}` — must come from the owner's account/key.

Team members get: `/update`, `/updateRoles`, `/lastRole`, invite generation, user management. They cannot: change owner settings, do critical security ops.

### H) AI Credits & Stats

| Command | Use case | Status |
|---------|----------|--------|
| `/stats` | Dashboard: users, WAU/MAU, MRR, costs, credit balance | proven |
| `/setupCredits:{topUp}:{rechargeAt}` | Auto-recharge config (min $250 / $100) | from-docs |
| `/changeCreditsTopUp:{newTopUp}:{newRechargeAt}` | Change credit recharge settings | from-docs |
| `/getInfoAboutUser:{subId}` | Subscription details for a user | from-docs |
| `/disableUser:{subId}` | Terminate user access | from-docs |
| `/giveExtraMessages:{subId}:{count}` | Add (or remove with negative number) message allowance | from-docs |
| `/setTrialMessages` | Adjust trial allowance using TG ID | from-docs |
| `/addTrialMessages` | Add to existing trial | from-docs |

**Credit gotcha:** When credits hit zero and auto-recharge fails, the AI Expert stops responding. Hourly alerts go to system message recipients.

### I) Setup (mostly Telegram-only first time)

| Command | Use case | Status |
|---------|----------|--------|
| `/setFolder:{driveUrl}` | Connect Google Drive folder to instance. **Wait 15s** for response — first attempt may show no reply. | proven, gotcha |
| `/setup` | Guided initial setup walkthrough | from-docs |
| `/connectWeb` | Connect to web dashboard | from-docs |
| `/reminders` | List active reminders | from-docs |

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
