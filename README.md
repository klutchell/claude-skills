# claude-skills

Claude Code skills marketplace for balena projects.

## Install

```
/plugin marketplace add klutchell/claude-skills
/plugin install balena-tools@klutchell-claude-skills
```

## Plugins

| Plugin | Description |
|--------|-------------|
| [balena-tools](plugins/balena-tools) | Skills for building and deploying balena fleet projects |

## Manual Install

Clone the repo and symlink individual skills into `~/.claude/skills/`:

```bash
git clone https://github.com/klutchell/claude-skills.git
ln -s $(pwd)/claude-skills/plugins/balena-tools/skills/balena-dockerfile-templates ~/.claude/skills/balena-dockerfile-templates
```
