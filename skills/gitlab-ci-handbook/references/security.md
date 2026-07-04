# CI security model

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/variables/#cicd-variable-security
- https://archives.docs.gitlab.com/18.11/ci/pipelines/merge_request_pipelines/ (fork pipelines)
- https://archives.docs.gitlab.com/18.11/ci/jobs/ci_job_token/
- https://archives.docs.gitlab.com/18.11/ci/components/ (component security guidance)
- https://docs.gitlab.com/runner/security/

## Pipeline code is executable code

`.gitlab-ci.yml` and everything it includes run with the credentials
and runners the pipeline receives. Review CI changes with the same
seriousness as application code: an MR that touches CI config (or any
script CI invokes) can exfiltrate variables, poison caches, or publish
artifacts. The controls below narrow what a hostile or compromised
pipeline can reach; none of them make running one safe.

## Secrets and variable hygiene

**When:** always.

- Never commit secrets in YAML `variables:`. Anything in the repo is
  readable by everyone who can read the repo, forever (git history).
- Store credentials as project/group variables, **masked** (hides value
  in job logs, best effort) and usually **protected** (only injected on
  protected branches/tags). Masking prerequisites and MR-pipeline access
  conditions for protected variables: `informative-logging.md`.
- Prefer **file** type variables for multi-line credentials (kubeconfig,
  service account JSON): the variable holds a path to a temp file, and
  tools that expect a file path work without `echo "$VAR" > file` (which
  puts the value one `set -x` away from the log).
- Never pass secrets on command lines when avoidable
  (`--password "$PW"` shows up in `ps` and error messages); use stdin:
  `printf '%s' "$PW" | tool login --password-stdin`.
- Anyone who can change `.gitlab-ci.yml` on a ref that receives a
  variable can exfiltrate it. Masking does not prevent that; protected
  variables + protected branches with mandatory review are the actual
  control.

## Fork merge request pipelines

- By default a fork MR pipeline runs in the fork project, with the
  fork's runners, settings, and CI/CD variables. It cannot read the
  parent's protected variables: protected-variable access in MR
  pipelines requires both branches protected and in the same project,
  which a fork never satisfies.
- "Run pipelines in parent project" changes that: the fork branch's CI
  configuration executes with the parent project's settings, resources,
  and variables. Fork MRs can carry malicious code that steals parent
  secrets when the pipeline runs, before any merge. Read the full diff,
  including CI files and every script they invoke, before triggering a
  fork pipeline in the parent; the confirmation warning GitLab shows
  exists for exactly this.

## Job tokens over personal tokens

- `CI_JOB_TOKEN` is valid only while its job runs and is revoked when
  the job ends. Prefer it wherever it is accepted (cloning, project
  registries, a growing set of REST endpoints) over a personal or group
  access token baked into a variable: a leaked PAT outlives the job
  that leaked it and carries the owner's full scopes.
- Private cross-project access is closed by default: another project's
  job token works here only if that project or its group is on this
  project's allowlist (Settings > CI/CD > Job token permissions) and the
  triggering user already has the required permissions. Public and
  internal projects expose some resources to non-allowlisted job tokens
  unless those project features are restricted to project members. Keep
  allowlists specific and restrict feature visibility when access must be
  allowlist-only; opening access wide reinstates the exposure the
  allowlist was introduced to close.
- The job token acts with the access level of the user who triggered
  the pipeline (within the job token's reduced resource surface). A
  pipeline triggered by a Maintainer can reach more than one triggered
  by a Developer; protected branches gating who can run deploy
  pipelines is part of the credential story, not just process.

## Runner isolation

- Privileged Docker (required by classic dind, see
  `execution-environment.md`) can escalate to the runner host. Do not
  schedule privileged jobs on runners shared with untrusted projects;
  give them dedicated, preferably ephemeral, runners.
- The shell executor runs job code as the runner's OS user with no
  container boundary, and state persists between jobs. Never point a
  shell-executor runner at a repo that accepts untrusted contributions.
- Runners restricted to protected refs keep deploy-capable
  infrastructure away from unreviewed branches. Combine with protected
  variables; they close different holes.

## Third-party CI code

Includes, templates, and components execute inside your pipeline with
your variables. Audit the source before adopting, verify credentials
are used only for what you expect, and re-audit on version bumps. Pin
exact versions; pinning discipline and the component audit checklist
are in `pipeline-structure.md`.
