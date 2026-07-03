# Agent Skills

A collection of [Agent Skills](https://agentskills.io) for Claude Code and other
SKILL.md-compatible agents.

## Skills

| Skill | Description |
|---|---|
| [gitlab-handbook](skills/gitlab-handbook/) | GitLab CI/CD knowledge base for pipelines and `.gitlab-ci.yml` |

## Installation

### skills CLI

Works with Claude Code, Cursor, Codex, and other agents supported by
[skills.sh](https://skills.sh):

```bash
npx skills add beeyev/skills
```

Single skill:

```bash
npx skills add beeyev/skills --skill gitlab-handbook
```

### Claude Code plugin

```
/plugin marketplace add beeyev/skills
/plugin install beeyev-skills
```

### Manual

Copy a skill directory into your agent's skills folder.

Claude Code:

```bash
cp -r skills/gitlab-handbook ~/.claude/skills/
```

Codex (also the shared cross-agent location):

```bash
cp -r skills/gitlab-handbook ~/.agents/skills/
```

## License

[MIT](LICENSE.md)
