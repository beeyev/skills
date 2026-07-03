# GitLab CI Handbook

Agent skill with curated GitLab CI/CD guidance
Helps agents write, review, optimize, and debug GitLab pipelines and
`.gitlab-ci.yml` files.

## What it covers

- Pipeline structure: stages, jobs, `include`, `extends`, CI/CD components,
  refactoring oversized YAML
- Bash in CI: `script:` blocks, `set -Eeuo pipefail`, quoting, script extraction
- Readability: naming jobs, stages, variables, and CI files
- Logging: collapsible sections, variable masking, secrets hygiene
- Developer experience: debugging failed pipelines, MR reports, reproducing CI
  failures locally
- Common patterns: caching and artifacts, DAGs, Docker builds, environments,
  child pipelines, matrices, retries, test sharding

## Layout

- [SKILL.md](SKILL.md) - entry point and topic router
- [references/](references/) - one focused file per topic, loaded on demand

## Install

```bash
npx skills add beeyev/skills --skill gitlab-ci-handbook
```

See the [repository README](../../README.md) for more options.
