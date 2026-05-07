# BuddyPro Owner API — Auto-Install for Claude Code

Hello Claude Code! The user wants to install the BuddyPro Owner API skill. Follow these steps exactly:

## Step 1: Download skill files

Run this single bash block. Do NOT modify it. Do NOT skip steps.

```bash
set -e
BASE="https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main"
DEST_SKILL="$HOME/.claude/skills/buddypro-owner-api"
DEST_CMD="$HOME/.claude/commands"

mkdir -p "$DEST_SKILL/references" "$DEST_CMD"

# Skill core
curl -fsSL "$BASE/skill/SKILL.md"                              -o "$DEST_SKILL/SKILL.md"
curl -fsSL "$BASE/VERSION"                                     -o "$DEST_SKILL/VERSION"

# Skill references
curl -fsSL "$BASE/skill/references/api-reference.md"           -o "$DEST_SKILL/references/api-reference.md"
curl -fsSL "$BASE/skill/references/use-cases.md"               -o "$DEST_SKILL/references/use-cases.md"
curl -fsSL "$BASE/skill/references/code-recipes.md"            -o "$DEST_SKILL/references/code-recipes.md"
curl -fsSL "$BASE/skill/references/multi-tenancy.md"           -o "$DEST_SKILL/references/multi-tenancy.md"
curl -fsSL "$BASE/skill/references/management-commands.md"     -o "$DEST_SKILL/references/management-commands.md"
curl -fsSL "$BASE/skill/references/troubleshooting.md"         -o "$DEST_SKILL/references/troubleshooting.md"

# Slash command
curl -fsSL "$BASE/command/buddypro-api.md"                     -o "$DEST_CMD/buddypro-api.md"

echo "Installed version: $(cat $DEST_SKILL/VERSION)"
```

## Step 2: Verify installation

```bash
ls -la ~/.claude/skills/buddypro-owner-api/
ls -la ~/.claude/commands/buddypro-api.md
```

Both must exist. If a `curl` command failed (typically: no internet, GitHub down, raw URL wrong), STOP and tell user the exact error.

## Step 3: Read the skill into context

Read this file: `~/.claude/skills/buddypro-owner-api/SKILL.md`

This loads the skill's instructions so you can apply it immediately — no Claude Code restart needed.

## Step 4: Confirm to user (Czech, since BuddyPro audience is Czech-first)

Send EXACTLY this message to the user:

```
✅ BuddyPro Owner API skill nainstalován (verze X.Y.Z).

Další krok: vygeneruj si API klíč.

1. Otevři svého BuddyPro bota v Telegramu
2. Pošli mu příkaz: /generateApiKey:my-agent
3. Bot ti pošle klíč začínající `bapi_...` (jen jednou — zkopíruj hned!)
4. Ulož si ho jako proměnnou prostředí:
   echo 'export BUDDYPRO_API_KEY="bapi_xxxxxxxxxxxx"' >> ~/.zshrc
   source ~/.zshrc

Pak zkus: /buddypro-api pošli "ahoj" mojí instanci
```

Replace `X.Y.Z` with the actual version from `~/.claude/skills/buddypro-owner-api/VERSION`.

## Notes for Claude Code

- **No Claude Code restart needed.** The skill is hot-loadable. After Step 3, you can use it immediately.
- **Idempotent.** If skill is already installed, this re-downloads and overwrites — safe to run multiple times.
- **Update flow.** When user asks to update the skill, fetch this same INSTALL.md again and re-run.
- **Dependency.** Only requires `curl` (default on macOS, Linux, Windows 10+).
- **Security.** All files come from `raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main` over HTTPS.
