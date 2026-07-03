---
name: gitlab-ci-handbook
description: >-
  GitLab CI/CD knowledge base (GitLab 18+). Use whenever the user writes,
  reviews, or debugs GitLab pipelines or .gitlab-ci.yml files: stages,
  jobs, rules and workflows, includes and templates, caching and artifacts,
  environments and deployments, or bash scripts running inside CI jobs,
  even if they only mention "pipeline" or "CI" in a GitLab context.
license: MIT
metadata:
  author: Alexander Tebiev
  source: https://github.com/beeyev/skills
---

# GitLab CI Handbook

Curated GitLab CI/CD guidance, verified against GitLab 18.x. This file is
a router: find the topic, read that reference file, then act. Read more
than one when the task spans topics (most pipeline work needs
`pipeline-structure.md` plus one other).

## Baseline

- Guidance targets GitLab 18+, Free tier, Linux runners, bash. Paid-tier
  or executor-specific requirements are flagged inline where they apply.
- For syntax not covered here, or when in doubt, check the official docs
  instead of guessing: keyword reference https://docs.gitlab.com/ci/yaml/,
  versioned archive https://archives.docs.gitlab.com/.

## Routing

| Read | When the task involves |
|---|---|
| `references/pipeline-structure.md` | Creating or restructuring `.gitlab-ci.yml`; refactoring an oversized pipeline file safely; splitting config with `include`; CI/CD components (`include:component`); `extends` vs anchors vs `!reference`; `default:` and hidden `.base` jobs; `spec:inputs`; repo layout for CI files; config merge/override questions |
| `references/bash-in-ci.md` | Writing or fixing `script:` / `before_script:` / `after_script:`; inline YAML vs `scripts/*.sh` decisions; `set -Eeuo pipefail`; required-env-var checks; quoting; why a multiline block didn't fail; how the runner executes commands |
| `references/readability.md` | Naming jobs, stages, variables, or CI files; commenting style; making a pipeline navigable for newcomers |
| `references/informative-logging.md` | What jobs should echo (versions, decisions) and must never echo (secrets); collapsible log sections; variable masking behavior |
| `references/developer-experience.md` | Debugging a failed pipeline end-to-end; `workflow:name` pipeline titles; job execution headers; actionable failure messages; surfacing test reports and artifact links in MRs; diagnostic artifacts; reproducing CI failures locally |
| `references/common-patterns.md` | Concrete recipes: pipeline selection, caching and artifacts, DAGs, Docker builds, environments, downstream and dynamic child pipelines, matrices, auto-cancel and retries, test sharding, checkout strategy, timeout budgeting, validation, and secrets hygiene |

## Rules of engagement

When writing pipeline YAML or CI bash for a user:

1. Follow the structure rules (no god YAML, but don't over-decompose
   tiny pipelines): `pipeline-structure.md`.
2. Default extracted bash scripts to `set -Eeuo pipefail`; deviate when
   the script deliberately handles those conditions. Use intent-revealing
   names: `bash-in-ci.md`, `readability.md`.
3. For a complete pipeline, define `workflow:rules` deliberately to
   prevent duplicate or unintended pipeline types. Do not replace an
   existing workflow when the user only asked for a job or config
   fragment.
4. Prefer a verified pattern from `common-patterns.md` over inventing
   one; adapt names and rules to the project.
5. If available, validate generated config before handing it over. Use
   GitLab CI Lint for GitLab semantics and `shellcheck` for shell scripts.
   `yamllint` and `gitlab-ci-local --list` are useful local checks, but
   they do not prove GitLab will compile the same pipeline.
