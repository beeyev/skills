# Agent Skills

A collection of [Agent Skills](https://agentskills.io) for Claude Code and other
SKILL.md-compatible agents.

## Skills

| Skill | Description |
|---|---|
| [gitlab-ci-handbook](skills/gitlab-ci-handbook/) | GitLab CI/CD knowledge base: `.gitlab-ci.yml` configuration, pipeline optimization, runners, caching, artifacts, and DevOps best practices |
| [xquik-x-data](skills/xquik-x-data/) | Xquik REST API, remote MCP, and webhook guidance for X data workflows |

## Installation

### skills CLI

Works with Claude Code, Cursor, Codex, and other agents supported by
[skills.sh](https://skills.sh):

```bash
npx skills add beeyev/skills
```

Single skill:

```bash
npx skills add beeyev/skills --skill gitlab-ci-handbook
```

```bash
npx skills add beeyev/skills --skill xquik-x-data
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
cp -r skills/gitlab-ci-handbook ~/.claude/skills/
```

Codex (also the shared cross-agent location):

```bash
cp -r skills/gitlab-ci-handbook ~/.agents/skills/
```

## License

[MIT](LICENSE.md)
