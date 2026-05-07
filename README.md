# BuddyPro Owner API — Claude Code Skill

A self-installing Claude Code skill that teaches your AI assistant how to integrate with your BuddyPro instance via the Owner API.

## For BuddyPro instance owners

You have a BuddyPro AI bot in Telegram and want to connect it to your own apps, agents, or automations. This skill makes Claude Code an expert at calling the Owner API for you.

### Install

In Claude Code, just say:

> Install the skill from `https://docs.buddypro.ai/skill`

That's it. Claude Code downloads the skill, installs it, and tells you how to generate an API key.

### Use

After install, say things like:

- *„send 'ahoj' to my BuddyPro bot"*
- *„create a Python script that asks my bot 5 questions"*
- *„embed my bot in a customer support chat — each customer gets their own memory"*
- *„run /update on my instance"*
- *„show me what knowledge chunks my bot used for the last answer"*

Or use the slash command directly: `/buddypro-api {your request}`.

## What's inside

```
buddypro-owner-api-skill/
├── INSTALL.md                       Self-install instructions for Claude Code
├── VERSION                          Current version
├── skill/
│   ├── SKILL.md                     Main skill entry point
│   └── references/
│       ├── api-reference.md         Endpoint, payload, errors
│       ├── use-cases.md             7 production patterns
│       ├── code-recipes.md          Python / Node / curl
│       ├── multi-tenancy.md         user field, isolation
│       ├── management-commands.md   /update, /investigateAnswer: via API
│       └── troubleshooting.md       Errors, rate limits, debugging
└── command/
    └── buddypro-api.md              /buddypro-api slash command
```

## Privacy

This skill never reads your API key — it only tells Claude Code to read `BUDDYPRO_API_KEY` from your environment. Keys stay on your machine.

## Contributing

Issues and PRs welcome. This skill is maintained by [@pvlriha](https://github.com/pvlriha).

## License

MIT
