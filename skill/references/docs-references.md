# Official Docs — Lookup Map

When the user's question goes beyond what's in this skill, fetch the relevant official docs page from `https://docs.buddypro.ai/`. The official docs are the canonical source of truth — this skill mirrors the behavior but doesn't duplicate the prose.

🔴 **Primary URLs are root-level only.** The `/dashboard/*` paths in the sitemap are duplicates and should be ignored. Always use the root paths listed below.

## When to read which page

### A. Welcome — overview & onboarding
| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/welcome/introduction | User asks „what is BuddyPro" or general overview |
| https://docs.buddypro.ai/welcome/how-it-works | User wants to understand the architecture (knowledge core, memory, relationship layer) |
| https://docs.buddypro.ai/welcome/install-telegram | User hasn't installed Telegram (rare, but happens) |
| https://docs.buddypro.ai/welcome/what-you-need | Pre-launch checklist content |

### B. Setup — initial instance creation
| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/setup/telegram-bot | Creating a new BuddyPro instance from scratch (BotFather flow) |
| https://docs.buddypro.ai/setup/activate-license | License key activation via @BuddyFM_bot |
| https://docs.buddypro.ai/setup/google-drive | Connecting Drive folder (`/setFolder`) |

### C. Training — knowledge management
| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/training/upload-know-how | What content to upload, where (SOURCES vs RAW SOURCES) |
| https://docs.buddypro.ai/training/how-ai-expert-learns | Mental model — pipeline, vector search, memory |
| https://docs.buddypro.ai/training/update-command | `/update` deep dive — when to run, what it does |
| https://docs.buddypro.ai/training/system-prompt | System prompt writing — XML tags, voice rules, sections |
| https://docs.buddypro.ai/training/expert-subroles | How roles auto-generate, when to edit manually |
| https://docs.buddypro.ai/training/onboarding-messages | ONBOARDING doc format, JSON, V_K_ variable |
| https://docs.buddypro.ai/training/optimize-transcription | Transcription settings — language, keywords, quality |

### D. Testing — diagnostics & optimization
| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/testing/diagnostic-tools | `/investigateAnswer:`, `/lastRole`, `/checkSetup` |
| https://docs.buddypro.ai/testing/testing-strategies | Pre-launch QA, multi-domain audit |
| https://docs.buddypro.ai/testing/optimization | Iterative tuning — system prompt, knowledge, roles, A/B |

### E. Sales — pricing, payments, trials
| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/sales/pricing | `/setDefaultCost` format and currency rules |
| https://docs.buddypro.ai/sales/strategic-positioning | Marketing positioning content |
| https://docs.buddypro.ai/sales/customer-onboarding | Email sequences, welcome video, drip |
| https://docs.buddypro.ai/sales/payment-integration | Stripe / FAPI setup |
| https://docs.buddypro.ai/sales/ai-credits | Credit balance, top-up, recharge |
| https://docs.buddypro.ai/sales/trial-settings | Trial messages, end-of-trial CTAs |

### F. Management — ongoing operations
| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/management/settings-check | `/checkSetup` deep dive — what each warning means |
| https://docs.buddypro.ai/management/maintenance | Regular maintenance cadence |
| https://docs.buddypro.ai/management/team-management | Adding team members (`/addUserToTeam`) |
| https://docs.buddypro.ai/management/user-management | `/disableUser`, `/giveExtraMessages`, `/getInfoAboutUser` |
| https://docs.buddypro.ai/management/customer-support | Handling user issues, support email setup |

### G. Advanced — deep dives
| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/advanced/commands-list | Comprehensive command reference (use this if our `management-commands.md` is missing something) |
| https://docs.buddypro.ai/advanced/advanced-tuning | Deep optimization techniques |
| https://docs.buddypro.ai/advanced/language-options | Multi-language support, `/setLanguage` |
| https://docs.buddypro.ai/advanced/vision | Image analysis features |

### H. Owner API — programmatic access
| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/owner-api/ | **Canonical Owner API reference.** When in doubt about an endpoint, payload, or error code — check here first. Our `api-reference.md` mirrors this; if there's ever a discrepancy, the official page wins. |

### I. Support cheatsheet — for end users
These pages are for **end-users of the bot** (the customers of the instance owner), not for the owner. Show them to a customer who's confused about the bot — they're not for the agent.

| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/support-cheatsheet/ | End-user index |
| https://docs.buddypro.ai/support-cheatsheet/commands-reference | End-user commands |
| https://docs.buddypro.ai/support-cheatsheet/messages-and-limits | Trial/subscription message limits |
| https://docs.buddypro.ai/support-cheatsheet/proactive-messages | Why bot sometimes initiates |
| https://docs.buddypro.ai/support-cheatsheet/voice-messages | Voice/audio interaction |
| https://docs.buddypro.ai/support-cheatsheet/subscriptions | Managing own subscription |
| https://docs.buddypro.ai/support-cheatsheet/upgrades | Upgrading subscription |
| https://docs.buddypro.ai/support-cheatsheet/stripe | Stripe-specific user help |
| https://docs.buddypro.ai/support-cheatsheet/troubleshooting | End-user troubleshooting |

### J. Recordings — workshops & calls
Long-form video content. Reference only if the user explicitly asks for tutorials or context.
| Page | Read when |
|------|-----------|
| https://docs.buddypro.ai/recordings/5-day-challenge | 5-day challenge walkthrough |
| https://docs.buddypro.ai/recordings/marketing-calls | Sales/marketing recordings |
| https://docs.buddypro.ai/recordings/zoom-recordings | Zoom session library |

## How to fetch a docs page

```bash
curl -fsSL "https://docs.buddypro.ai/{section}/{page}" | \
  pandoc -f html -t markdown 2>/dev/null || \
  echo "Falling back to raw HTML (install pandoc for cleaner markdown)"
```

Or use Claude Code's `WebFetch` tool — pass the URL and a focused question, and the WebFetch tool returns markdown excerpts.

## When this map is stale

Docs evolve. If a URL above 404s, check the sitemap:
```bash
curl -fsSL https://docs.buddypro.ai/sitemap.xml
```

The skill itself can be updated — reinstall via `https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/INSTALL.md`.

## Anti-pattern — don't copy docs into the skill

The reason this file is just URLs (not full content) is intentional: docs.buddypro.ai changes. If we duplicated the prose, this skill would lie within weeks. The skill owns *how to do things via API* and *workflows*; the docs own *prose explanations*. Stay in your lane.

*Last updated: 2026-05-07 (v0.2.0)*
