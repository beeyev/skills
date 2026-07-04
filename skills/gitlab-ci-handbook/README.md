# GitLab CI Handbook

Agent skill with curated GitLab CI/CD guidance
Helps agents write, review, optimize, and debug GitLab pipelines and
`.gitlab-ci.yml` files.

## What it covers

- Pipeline structure: stages, jobs, `include`, `extends`, CI/CD components
  (authoring and consuming), refactoring oversized YAML
- Pipeline selection: `workflow:rules`, job `rules`, schedules, tag releases,
  manual gates
- Data flow: caching and artifacts, DAGs with `needs`, test sharding,
  optimization method
- Execution environment: runners and tags, job images, sidecar services,
  Docker builds, environments, GitLab Pages
- Orchestration: child and downstream pipelines, matrices, auto-cancel,
  retries, timeouts, validating compiled config
- Security: secrets hygiene, fork MR pipelines, job tokens, runner isolation
- Debugging: failure-signature catalog, symptom-to-cause tables, variable
  traps
- Bash in CI: `script:` blocks, `set -Eeuo pipefail`, quoting, script
  extraction, shellcheck
- Readability: naming jobs, stages, variables, and CI files
- Logging: collapsible sections, variable masking
- Developer experience: designing debuggable pipelines, MR reports,
  reproducing CI failures locally

## Layout

- [SKILL.md](SKILL.md) - entry point and topic router
- [references/](references/) - one focused file per topic, loaded on demand

## Install

```bash
npx skills add beeyev/skills --skill gitlab-ci-handbook
```

See the [repository README](../../README.md) for more options.
