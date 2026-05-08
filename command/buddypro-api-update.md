# BuddyPro Owner API — Force Update

Force an immediate update check + atomic re-install of the BuddyPro Owner API skill, bypassing the 4-hour cache TTL.

Use this when you know there's been a new release and want it NOW (instead of waiting for the next auto-trigger), or when something seems off and you want to ensure you're on the latest version.

## What this does

1. Bust the version-check cache (`rm $SKILL_DIR/.last_version_check`)
2. Run the SKILL.md Step 0 logic (compare local vs remote VERSION via GitHub raw)
3. If a newer remote version exists → fetch `https://docs.buddypro.ai/skill` (which redirects to the canonical INSTALL.md), extract the Step 1 bash block, run it
4. After successful install → fetch `CHANGELOG.md` and surface the new version's section to the user (3-5 word summary lines)
5. Confirm the user is now on the latest version

## What this preserves

- `~/.claude/skills/buddypro-owner-api/state.env` — onboarding state (API key, topic, ack flags)
- `~/.claude/skills/buddypro-owner-api/.onboarded` — onboarding marker
- `~/.claude/skills/buddypro-owner-api/.pinned_version` — if user pinned, this command HONORS the pin (refuses to update; tells user to remove the pin first)

## What gets overwritten (every time)

- `SKILL.md`, all `references/*.md`, `command/*.md`, `VERSION`, `CHANGELOG.md`

## When the user invokes this

Run the `/buddypro-api` skill's normal flow but with the cache-bust as the first step. Tell the user what version they were on and what version they're on now. If they were already current, say so without re-installing.

$ARGUMENTS
