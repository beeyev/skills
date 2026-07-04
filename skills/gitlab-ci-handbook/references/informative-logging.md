# Informative job logs

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_logs/ (collapsible sections, timestamps)
- https://archives.docs.gitlab.com/18.11/ci/yaml/script/ (color codes)
- https://archives.docs.gitlab.com/18.11/ci/variables/#cicd-variable-security (masking)

Pipelines are read in the job log far more often than in the YAML. The
audience is a developer debugging a failed pipeline at 2 a.m.: log what
they need to reconstruct what happened, and nothing else.

## What to echo

- **Tool versions, at job start.** "Works on my machine" debugging starts
  with "which version ran in CI?". See the version banner below.
- **Inputs and decisions.** Before the action, state it with its
  parameters: `echo "Deploying ${APP_VERSION} to ${DEPLOY_ENV}"`. If a
  script branches (skip, fallback, retry), say which branch it took and
  why: `echo "No changes in ./migrations, skipping migration step"`.
- **Progress markers at meaningful phases** of long jobs, so a stalled
  log shows where it stalled. Meaningful = one per phase (build, test,
  upload), not one per file.
- **Context on failure.** When a script exits with an error, print what
  it was doing and with which values (never secret values) to stderr
  before exiting. Structure of a useful failure message:
  `developer-experience.md`.

## What not to echo

- **Secrets. Never.** Not even partially, not even base64-encoded
  (masking does not catch transformed values, and base64 is not
  encryption).
- **Full environment dumps.** `env` / `printenv` / `export` output leaks
  every variable that is not masked, and masked ones are only masked
  best-effort. If you need one value, print that one value.
- **Per-item noise.** Progress bars, dependency download logs, `set -x`
  left on for the whole script. If an opt-in debug mode uses `set -x`,
  scope it to commands with known non-secret arguments and run `set +x`
  before credential handling. Name the flag without the `CI_` prefix per
  `readability.md`. For full command tracing GitLab has the built-in
  `CI_DEBUG_TRACE: "true"`, but it exposes all variables and secrets
  available to the job (viewing is restricted to Developer role and
  above); last resort, not a convenience toggle.
- **Anything you would not paste into the issue tracker.** Job logs are
  visible to everyone who can see the job, and they outlive the job.

## Version banner pattern

One reusable hidden job; each job family appends its own tools.

```yaml
.version-banner:
  before_script:
    - echo "Runner ${CI_RUNNER_DESCRIPTION:-unknown} / job ${CI_JOB_NAME} / ref ${CI_COMMIT_REF_NAME}"

build:app:
  stage: build
  image: node:22-bookworm-slim
  before_script:
    - !reference [.version-banner, before_script]
    - node --version
    - npm --version
  script:
    - npm run build
```

`extends` would replace the `before_script` array wholesale; `!reference`
composes it (see `pipeline-structure.md`).

For extracted scripts, the same as a function-free preamble:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

terraform version
echo "Deploying ${APP_VERSION:?} to ${DEPLOY_ENV:?}"
```

(`${VAR:?}` doubles as the required-variable check, see `bash-in-ci.md`.)

## Collapsible log sections

Wrap noisy-but-necessary output (dependency install, compilation) so the
log opens on the interesting parts. Markers are magic escape sequences;
the section name allows letters, numbers, `_`, `.`, `-`. Add
`[collapsed=true]` to start collapsed:

```yaml
build:deps:
  script:
    - echo -e "\e[0Ksection_start:$(date +%s):install_deps[collapsed=true]\r\e[0KInstalling dependencies"
    - npm ci
    - echo -e "\e[0Ksection_end:$(date +%s):install_deps\r\e[0K"
    - npm run build
```

(`echo -e` matches the official docs example but needs bash; in
`sh`-only images like alpine, use the `printf` form below.)

In extracted bash scripts, as functions:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

section_start() { # <name> <header text>; starts collapsed
  printf '\e[0Ksection_start:%s:%s[collapsed=true]\r\e[0K%s\n' "$(date +%s)" "$1" "$2"
}
section_end() { # <name>
  printf '\e[0Ksection_end:%s:%s\r\e[0K\n' "$(date +%s)" "$1"
}

section_start install_deps "Installing dependencies"
npm ci
section_end install_deps
```

## Timestamps and color

- Since GitLab 18.9, job logs carry an ISO 8601 timestamp per line by
  default (earlier 18.x: opt-in). Use them to find the slow or stalled
  step instead of adding `date` calls. Toggle per job with the
  `FF_TIMESTAMPS` variable (controllable with Runner 18.7+).
  <!-- volatile: re-verify on version bump -->
- ANSI color codes render in the job log. A colored `ERROR:` / `WARNING:`
  prefix on the failure summary makes it findable while scrolling long
  logs. Codes and shell caveats:
  https://archives.docs.gitlab.com/18.11/ci/yaml/script/#add-color-codes-to-script-output

## Masking: what it does and does not protect

Facts (all UI-level settings on project/group/instance variables, see
`security.md` for the secrets workflow):

- A **masked** variable's value is replaced with `[MASKED]` when it
  appears in a job log. Requirements on the value: single line, no
  spaces, at least 8 characters, and not equal to the name of an
  existing CI/CD variable. <!-- volatile: re-verify on version bump -->
- Since GitLab 18.3, new variables created in the UI default to
  **Masked** visibility; variables created earlier keep their setting.
- Masking is pattern matching on output. Any transformation defeats it:
  shell escaping (`My\[value\]` vs `My[value]`), base64, URL-encoding,
  splitting across lines. Treat masking as a seatbelt, not a guarantee.
- **Masked and hidden** (GitLab 17.6+) additionally prevents the value
  from ever being revealed in the settings UI; choosable only at
  creation time.
- **Protected** variables are only injected into pipelines on protected
  branches/tags. Since GitLab 18.1, same-project merge request pipelines
  can optionally access them when both branches are protected and the
  triggering user can merge or push to the target. Fork merge request
  pipelines cannot access protected variables.
- Anyone who can push a `.gitlab-ci.yml` change can exfiltrate variables
  the pipeline receives; masking does not change that. Real secrets
  belong in an external secrets manager, or at minimum in file-type
  protected+masked variables.
