# BuddyPro Owner API — Install Guide

This document is read by the **Claude Code agent** that the user has asked to install this skill. It describes a 4-step install procedure: atomic download, verification, skill load, user confirmation. The bash block in Step 1 is the install script.

> **Re-installing?** Step 1 overwrites all skill files atomically. If the user customized any of them locally, back up first: `cp -r ~/.claude/skills/buddypro-owner-api ~/buddypro-skill.backup`.

## Step 1: Download skill files (atomic — all-or-nothing)

The agent runs this bash block as-is. Files are downloaded to a temp directory first, then moved into place only if every download succeeded. A partial install is impossible.

```bash
set -euo pipefail

BASE="https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main"
DEST_SKILL="${HOME:?HOME must be set}/.claude/skills/buddypro-owner-api"
DEST_CMD="${HOME}/.claude/commands"
TMP=$(mktemp -d -t bpoasinstall.XXXXXX)
trap 'rm -rf "$TMP"' EXIT

mkdir -p "$TMP/skill/references" "$TMP/command"

# Download into temp staging area
files=(
  "VERSION:VERSION"
  "skill/SKILL.md:skill/SKILL.md"
  "skill/references/getting-started.md:skill/references/getting-started.md"
  "skill/references/api-reference.md:skill/references/api-reference.md"
  "skill/references/use-cases.md:skill/references/use-cases.md"
  "skill/references/code-recipes.md:skill/references/code-recipes.md"
  "skill/references/multi-tenancy.md:skill/references/multi-tenancy.md"
  "skill/references/management-commands.md:skill/references/management-commands.md"
  "skill/references/instance-management.md:skill/references/instance-management.md"
  "skill/references/troubleshooting.md:skill/references/troubleshooting.md"
  "skill/references/docs-references.md:skill/references/docs-references.md"
  "command/buddypro-api.md:command/buddypro-api.md"
)

for entry in "${files[@]}"; do
  remote="${entry%%:*}"
  local_path="${entry##*:}"
  curl -fsSL --create-dirs --retry 2 --connect-timeout 10 \
    "$BASE/$remote" -o "$TMP/$local_path"
done

# Atomic move into final location (only reached if every curl above succeeded)
mkdir -p "$DEST_SKILL/references" "$DEST_CMD"
cp "$TMP/VERSION" "$DEST_SKILL/VERSION"
cp "$TMP/skill/SKILL.md" "$DEST_SKILL/SKILL.md"
cp "$TMP/skill/references/"*.md "$DEST_SKILL/references/"
cp "$TMP/command/buddypro-api.md" "$DEST_CMD/buddypro-api.md"

# Sanity check — detect placeholder/stub references (under 500 bytes)
STUB_COUNT=$(find "$DEST_SKILL/references" -name "*.md" -size -500c 2>/dev/null | wc -l | tr -d ' ')
INSTALLED_VERSION=$(cat "$DEST_SKILL/VERSION")

echo "INSTALLED_VERSION=$INSTALLED_VERSION"
echo "STUB_REFERENCE_FILES=$STUB_COUNT"
```

If any `curl` fails, the script exits non-zero and **nothing is installed** (the user's existing install, if any, is untouched). The agent should then tell the user the specific failure.

## Step 2: Verify installation

```bash
ls "$HOME/.claude/skills/buddypro-owner-api/SKILL.md" \
   "$HOME/.claude/skills/buddypro-owner-api/VERSION" \
   "$HOME/.claude/commands/buddypro-api.md"
```

All three paths must list successfully. If not, install failed — tell user.

## Step 3: Load the skill into the current conversation

Read this file: `$HOME/.claude/skills/buddypro-owner-api/SKILL.md`

Reading it brings the skill's instructions into the agent's context, so the user can use the skill immediately in this same conversation.

> **Note on `Skill(...)` invocation:** the slash command `/buddypro-api` and auto-trigger via the Skill tool become available to Claude Code's skills index from the **next session onward** (the index is built at session start). For the **current** session, the agent uses the skill by reading SKILL.md and following its instructions — no slash command needed.

## Step 4: Send confirmation message — IN THE USER'S LANGUAGE

🔴 **Critical:** Detect the user's language from how they spoke to the agent in this conversation, then write the confirmation in that language. BuddyPro owners are global — Czech, English, Spanish, German, etc. Do NOT default to Czech.

**Detection rule:** Match the language of the user's most recent message. If unclear, default to English.

Read the `INSTALLED_VERSION` and `STUB_REFERENCE_FILES` from Step 1's output, then build a message with these elements:

### Required structure (any language)

1. ✅ icon + "skill installed" + version (mention `(v{VERSION} alpha)` if `STUB_REFERENCE_FILES > 0`)
2. (alpha only) one-line heads-up that some reference files are placeholders, basic usage works
3. Next-step instructions for generating an API key:
   - Open the BuddyPro bot in Telegram
   - Send `/generateApiKey:my-agent`
   - Save returned `bapi_...` key (shown once)
   - Set env var: `export BUDDYPRO_API_KEY="<paste-your-bapi-key-here>"` in shell profile
4. Suggested first command with concrete example:
   - `/buddypro-api send "hello, who are you?" to my instance`
   - Note: in current session, agent will reply directly; from next session, the slash command works as auto-trigger

### Reference templates

**English (use this if user spoke English, or as the default fallback):**
```
✅ BuddyPro Owner API skill installed (v{INSTALLED_VERSION}{ALPHA_SUFFIX}).

{ALPHA_NOTE_IF_STUBS_EN}

Next step: generate your API key.

1. Open your BuddyPro bot in Telegram
2. Send the command: /generateApiKey:my-agent
3. The bot will reply with a key starting with `bapi_...` (shown only once — copy it now!)
4. Save it in your shell profile (replace the placeholder with your real key):
   echo 'export BUDDYPRO_API_KEY="<paste-your-bapi-key-here>"' >> ~/.zshrc
   source ~/.zshrc

Try it out — ask me right now: 'send "hello, who are you?" to my BuddyPro instance'
You should get a personalized reply from your bot within a few seconds.
```

Where:
- `{ALPHA_SUFFIX}` = ` alpha` if `STUB_REFERENCE_FILES > 0`, else empty
- `{ALPHA_NOTE_IF_STUBS_EN}` = `⚠️ Heads-up: this version ships the main SKILL.md and slash command, but {N} of 6 reference files are still placeholders (full content in a later release). Basic API calls work right now.` — only if `STUB_REFERENCE_FILES > 0`, else omit (and the blank line above it).

**Czech (use this if user spoke Czech):**
```
✅ BuddyPro Owner API skill nainstalován (v{INSTALLED_VERSION}{ALPHA_SUFFIX}).

{ALPHA_NOTE_IF_STUBS_CZ}

Další krok: vygeneruj si API klíč.

1. Otevři svého BuddyPro bota v Telegramu
2. Pošli mu příkaz: /generateApiKey:my-agent
3. Bot ti pošle klíč začínající `bapi_...` (jen jednou — zkopíruj hned!)
4. Ulož si ho do shell profilu (placeholder nahraď reálným klíčem):
   echo 'export BUDDYPRO_API_KEY="<sem-vloz-svuj-bapi-klic>"' >> ~/.zshrc
   source ~/.zshrc

Vyzkoušej to — řekni mi rovnou: 'pošli "ahoj, kdo jsi?" mojí BuddyPro instanci'
Během pár sekund dostaneš personalizovanou odpověď od svého bota.
```

Where `{ALPHA_NOTE_IF_STUBS_CZ}` = `⚠️ Heads-up: tato verze obsahuje hlavní SKILL.md a slash command, ale {N} ze 6 reference souborů jsou ještě placeholdery (plný obsah přijde v další verzi). Základní volání API funguje hned.`

**Other languages (Spanish, German, French, Slovak, Polish, etc.):** Translate the structure naturally into the user's language. Keep the technical identifiers verbatim across all languages: `bapi_`, `BUDDYPRO_API_KEY`, `/generateApiKey`, `/buddypro-api`.

## Uninstall (for reference)

If the user later asks to remove the skill:

```bash
rm -rf "$HOME/.claude/skills/buddypro-owner-api" \
       "$HOME/.claude/commands/buddypro-api.md"
```

## Notes for the installing agent

- **Hot-load works for THIS session via Step 3** (Read SKILL.md). Slash command + auto-trigger become available from the next Claude Code session onward.
- **Atomic install.** Step 1 stages downloads in a temp dir and only moves them into place if every file downloaded successfully. Existing install is preserved if anything fails.
- **Dependency.** Only requires `curl` (default on macOS, Linux, Windows 10+).
- **Defensive shell.** `set -euo pipefail` + `${HOME:?HOME must be set}` catch unset variables and missing env. Failed downloads abort cleanly.
- **Security.** All files come from `raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main` over HTTPS. Repo is public; user can audit at https://github.com/pvlriha/buddypro-owner-api-skill. Optional: verify against `MANIFEST.sha256` in repo root.
- **Reproducibility (advanced).** To pin to a specific commit instead of `main`, replace `main` in `BASE=` with a commit SHA or tag. Example: `BASE="https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/v0.1.3"`.
