# Mieah Skills

Some Skills.

## Skills

| Skill | Description | Prerequisites |
|-------|---------|---------------|
| [`ieee-search`](./skills/ieee-search/SKILL.md) | Search IEEE Xplore via browser automation (agent-browser CLI, Chrome MCP, or Firefox MCP). No API key required. Supplements paper-search which cannot search IEEE without an API key. Access to IEEE depends on local network situation.| `npm i -g agent-browser && agent-browser install` |

## Installation

### Via skills.sh (Recommended)

```bash
# Install all skills from this repo
npx skills add Mieah108/mieah-skills

# Or install a specific skill
npx skills add Mieah108/mieah-skills --skill ieee-search
```

### Via Claude Plugin

```bash
/plugin install Mieah108/mieah-skills
```

### Manual Installation

Clone and install as a Claude Code plugin:
```bash
git clone git@github.com:Mieah108/mieah-skills.git
/plugin add ./mieah-skills
```

## License

MIT
