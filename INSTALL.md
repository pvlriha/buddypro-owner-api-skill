# BuddyPro Owner API — Auto-Install for Claude Code

Hello Claude Code! The user wants to install the BuddyPro Owner API skill. Follow these steps exactly.

> **Re-installing?** This script overwrites all skill files. If the user customized any of them locally, back up first: `cp -r ~/.claude/skills/buddypro-owner-api ~/buddypro-skill.backup`.

## Step 1: Download skill files

Run this single bash block. Do NOT modify it. Do NOT skip steps.

```bash
set -e
BASE="https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main"
DEST_SKILL="$HOME/.claude/skills/buddypro-owner-api"
DEST_CMD="$HOME/.claude/commands"

mkdir -p "$DEST_SKILL/references" "$DEST_CMD"

# Skill core
curl -fsSL --create-dirs "$BASE/skill/SKILL.md"                          -o "$DEST_SKILL/SKILL.md"
curl -fsSL --create-dirs "$BASE/VERSION"                                 -o "$DEST_SKILL/VERSION"

# Skill references
curl -fsSL --create-dirs "$BASE/skill/references/api-reference.md"       -o "$DEST_SKILL/references/api-reference.md"
curl -fsSL --create-dirs "$BASE/skill/references/use-cases.md"           -o "$DEST_SKILL/references/use-cases.md"
curl -fsSL --create-dirs "$BASE/skill/references/code-recipes.md"        -o "$DEST_SKILL/references/code-recipes.md"
curl -fsSL --create-dirs "$BASE/skill/references/multi-tenancy.md"       -o "$DEST_SKILL/references/multi-tenancy.md"
curl -fsSL --create-dirs "$BASE/skill/references/management-commands.md" -o "$DEST_SKILL/references/management-commands.md"
curl -fsSL --create-dirs "$BASE/skill/references/troubleshooting.md"     -o "$DEST_SKILL/references/troubleshooting.md"

# Slash command
curl -fsSL --create-dirs "$BASE/command/buddypro-api.md"                 -o "$DEST_CMD/buddypro-api.md"

# Sanity check — detect placeholder/stub references
STUB_COUNT=$(find "$DEST_SKILL/references" -name "*.md" -size -500c 2>/dev/null | wc -l | tr -d ' ')

VERSION=$(cat "$DEST_SKILL/VERSION")
echo "INSTALLED_VERSION=$VERSION"
echo "STUB_REFERENCE_FILES=$STUB_COUNT"
```

## Step 2: Verify installation

```bash
ls "$HOME/.claude/skills/buddypro-owner-api/SKILL.md" \
   "$HOME/.claude/skills/buddypro-owner-api/VERSION" \
   "$HOME/.claude/commands/buddypro-api.md"
```

If any file is missing, STOP and tell user the exact `curl` step that failed. Common causes: no internet, GitHub down, dependency `curl` missing.

## Step 3: Read the skill into context

Read this file: `$HOME/.claude/skills/buddypro-owner-api/SKILL.md` (use the literal `$HOME` expansion or the actual user's home path).

This loads the skill's instructions so you can apply it immediately — no Claude Code restart needed.

## Step 4: Send confirmation message — IN THE USER'S LANGUAGE

🔴 **Critical:** Detect the user's language from how they spoke to you in this conversation, then write the confirmation in that language. BuddyPro owners are global — Czech, English, Spanish, German, etc. Do NOT default to Czech.

**Detection rule:** Match the language of the user's most recent message. If unclear, default to English.

Read the `INSTALLED_VERSION` and `STUB_REFERENCE_FILES` from Step 1's output, then build a message with these elements:

### Required structure (any language)

1. ✅ icon + "skill installed" + version (mention `(v{VERSION} alpha)` if `STUB_REFERENCE_FILES > 0`)
2. (alpha only) one-line heads-up that some reference files are placeholders, basic usage works
3. Next-step instructions for generating an API key:
   - Open the BuddyPro bot in Telegram
   - Send `/generateApiKey:my-agent`
   - Save returned `bapi_...` key (shown once)
   - Set env var: `export BUDDYPRO_API_KEY="bapi_..."` in shell profile
4. Suggested first command: `/buddypro-api send "hello" to my instance`

### Reference templates (use as a starting point, translate from)

**English (use this if user spoke English, or as the default fallback):**
```
✅ BuddyPro Owner API skill installed (v{INSTALLED_VERSION}{ALPHA_SUFFIX}).

{ALPHA_NOTE_IF_STUBS}

Next step: generate your API key.

1. Open your BuddyPro bot in Telegram
2. Send the command: /generateApiKey:my-agent
3. The bot will send a key starting with `bapi_...` (shown only once — copy it now!)
4. Save it as an environment variable:
   echo 'export BUDDYPRO_API_KEY="bapi_xxxxxxxxxxxx"' >> ~/.zshrc
   source ~/.zshrc

Then try: /buddypro-api send "hello" to my instance
```

Where:
- `{ALPHA_SUFFIX}` = ` alpha` if `STUB_REFERENCE_FILES > 0`, else empty
- `{ALPHA_NOTE_IF_STUBS}` = `⚠️ Heads-up: this version ships the main SKILL.md and slash command, but {N} of 6 reference files are still placeholders (full content in a later release). Basic API calls work right now.` — only if `STUB_REFERENCE_FILES > 0`, else omit (and the blank line above it).

**Czech (use this if user spoke Czech):**
```
✅ BuddyPro Owner API skill nainstalován (v{INSTALLED_VERSION}{ALPHA_SUFFIX}).

{ALPHA_NOTE_IF_STUBS_CZ}

Další krok: vygeneruj si API klíč.

1. Otevři svého BuddyPro bota v Telegramu
2. Pošli mu příkaz: /generateApiKey:my-agent
3. Bot ti pošle klíč začínající `bapi_...` (jen jednou — zkopíruj hned!)
4. Ulož si ho jako proměnnou prostředí:
   echo 'export BUDDYPRO_API_KEY="bapi_xxxxxxxxxxxx"' >> ~/.zshrc
   source ~/.zshrc

Pak zkus: /buddypro-api pošli "ahoj" mojí instanci
```

Where `{ALPHA_NOTE_IF_STUBS_CZ}` = `⚠️ Heads-up: tato verze obsahuje hlavní SKILL.md a slash command, ale {N} ze 6 reference souborů jsou ještě placeholdery (plný obsah přijde v další verzi). Základní volání API funguje hned.`

**Other languages (Spanish, German, French, Slovak, etc.):** Translate the structure naturally into the user's language. Keep the technical terms (`bapi_`, `BUDDYPRO_API_KEY`, `/generateApiKey`, `/buddypro-api`) verbatim — they're identifiers, not words to translate.

## Uninstall (for reference)

If the user later asks to remove the skill:

```bash
rm -rf "$HOME/.claude/skills/buddypro-owner-api" \
       "$HOME/.claude/commands/buddypro-api.md"
```

## Notes for Claude Code

- **No Claude Code restart needed.** The skill is hot-loadable. After Step 3, you can use it immediately.
- **Idempotent.** If skill is already installed, this re-downloads and overwrites — safe to run multiple times.
- **Update flow.** When user asks to update the skill, fetch this same INSTALL.md again and re-run.
- **Dependency.** Only requires `curl` (default on macOS, Linux, Windows 10+).
- **Security.** All files come from `raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main` over HTTPS. Repo is public; user can audit at https://github.com/pvlriha/buddypro-owner-api-skill.
- **Reproducibility (advanced).** To pin to a specific commit instead of `main`, replace `main` in `BASE=` with a commit SHA or tag. Example: `BASE="https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/v0.1.1"`.
