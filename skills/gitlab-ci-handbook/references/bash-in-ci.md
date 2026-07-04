# Bash in CI jobs

Verified against: GitLab 18.11, GitLab Runner 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/yaml/script/ (script syntax, multiline behavior)
- https://archives.docs.gitlab.com/18.11/ci/yaml/#script (also `before_script`, `after_script`)
- https://docs.gitlab.com/runner/shells/ (shell selection and script generation)

Assumes Linux runners with bash, per this handbook's scope.

## How GitLab actually runs your `script:`

Mental model first; everything else follows from it.

- The runner turns each job into a generated shell script: clone, restore
  cache, download artifacts, then your commands, then upload cache and
  artifacts. Your YAML lines are commands injected into that script, not a
  script of their own.
- Shell selection: on Unix systems the runner uses `bash`, falling back to
  `sh` if bash is absent (busybox-based images like `alpine` give you
  `sh`; don't write bash-isms in inline scripts for such images). With the
  Docker executor the shell is non-interactive and non-login: no
  `.bashrc`/profile. The shell executor pipes through `bash --login`, so
  dotfiles do run there.
- `before_script` and `script` are concatenated and run in one shell
  session: variables exported in `before_script` are visible in `script`.
- `after_script` runs in a new shell: working directory reset, no
  variables or aliases from `script`, a separate timeout controlled by
  `RUNNER_AFTER_SCRIPT_TIMEOUT` (300 seconds by default), and its exit
  code never affects the job result.
  <!-- volatile: re-verify timeout default on version bump --> Since GitLab
  17.0 it also runs when the job is canceled while `before_script` or
  `script` was running; guard with
  `- if [ "$CI_JOB_STATUS" == "canceled" ]; then exit 0; fi` if that
  matters.
- Each array item in `script:` is echoed to the log and executed; the
  first item exiting non-zero fails the job and skips the remaining items.
- A `|` multiline block is a single item running under the runner's
  `set -o errexit` and `set -o pipefail`: a plain failing line aborts the
  block, but the usual `set -e` blind spots (`&&`/`||` chains, `if`/`while`
  conditions, `local`/`export var=$(cmd)`) swallow failures. Still start
  blocks with the preamble below; `-u` is the flag the runner does not set.
- `$?` does not carry across `script:` items (the runner's trace `echo`
  overwrites it); check exit status in the same item.
- YAML eats your syntax before bash sees it: a command containing `: `
  must be quoted whole (`- 'curl --header "Content-Type: application/json" ...'`),
  and commands starting with YAML indicator characters such as `[`, `{`,
  `&`, `*`, `#`, `!`, `|`, `>`, `%`, `@`, or a backtick must not be plain
  scalars. Quote the whole command. When quoting gets ugly, extract it to a
  file.

## Inline vs extracted scripts

Rule of thumb: **inline is for wiring, files are for logic.**

This is a balance, not a dogma. Extraction buys shellcheck, local runs,
and reuse, but costs an indirection: the reader must open another file to
see what the job does. A short inline script that meets the criteria
below is the better choice, not a shortcut. Do not extract by default.

Keep inline when all of these hold:

- Up to ~5 linear commands, no `if`/loops/functions.
- Used by one job only.
- Reads naturally in the job log line by line.

Extract to `scripts/ci/<name>.sh` when any of these hold:

- Control flow (conditionals, loops, retries) or string manipulation.
- The same logic is needed by more than one job.
- You want to run or test it locally, or lint it with `shellcheck`
  (extracted files are straightforward to lint as complete scripts).
- The YAML quoting workarounds start to obscure the command.

For extracted scripts:

```yaml
deploy:app:
  stage: deploy
  script:
    - ./scripts/ci/deploy.sh
```

- Shebang `#!/usr/bin/env bash`.
- Commit the executable bit: `git update-index --chmod=+x scripts/ci/deploy.sh`.
  Calling `bash scripts/ci/deploy.sh` also works but hides the fact that
  the bit is missing.
- Use explicit command arguments for ordinary inputs. Use environment
  variables for GitLab-provided context and values that should not appear
  in process arguments. Validate both at the script boundary.

## The fail-fast preamble

Use this default for extracted scripts unless the script deliberately
handles non-zero statuses or unset variables itself:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
```

Flag by flag:

- `-e` (errexit): exit immediately when a command fails, instead of
  barreling on with broken state. Caveat: it is suspended inside `if`
  conditions, `&&`/`||` chains, and `while`/`until` tests; do not treat it
  as a substitute for checking the results you actually care about.
- `-E` (errtrace): makes an `ERR` trap fire inside functions and
  subshells too. Only matters if you use `trap ... ERR`; harmless
  otherwise and cheap insurance.
- `-u` (nounset): expanding an unset variable is an error. Turns typos
  and missing CI variables into immediate, pointed failures instead of
  empty-string bugs (`rm -rf "$PREFIX/"` with unset `PREFIX`).
- `-o pipefail`: a pipeline fails if any element fails, not just the
  last. Without it, `cmd | tee log` hides `cmd`'s failure.

For longer scripts, an `ERR` trap is the one addition that pays for
itself: the log then names the failing command and line instead of just
stopping. This is what `-E` exists for:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
trap 'echo "ERROR: command failed at ${BASH_SOURCE[0]}:${LINENO}: ${BASH_COMMAND}" >&2' ERR
```

For what a useful failure message contains beyond the location, see
`developer-experience.md`.

Inside YAML `|` multiline blocks, set the same flags on the first line.
The runner already enables `errexit` and `pipefail` around your script,
but not `-u`, and spelling the flags out keeps the block correct when it
is copied into a file or run locally:

```yaml
build:image:
  script:
    - |
      set -euo pipefail
      docker build --tag "$IMAGE" .
      docker push "$IMAGE"
```

(`-E` is pointless without an ERR trap, so the short form is fine
inline.)

That is the whole ceremony. Do not add retry wrappers, logging
frameworks, or `main()` scaffolding to a ten-line script; the logic must
stay visible.

## Required-variable checks

Fail at the top, with a message naming the variable, not halfway through
a deploy. bash has this built in:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

: "${DEPLOY_ENV:?DEPLOY_ENV is required (staging|production)}"
: "${IMAGE_TAG:?IMAGE_TAG is required}"

echo "Deploying ${IMAGE_TAG} to ${DEPLOY_ENV}"
```

`${VAR:?message}` aborts with the message when `VAR` is unset or empty.
The leading `:` is a no-op command that hosts the expansion. This beats
an `if [ -z ... ]` block: one line per variable, and it works under
`set -u`.

For many variables, a loop keeps it flat:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

for var in DEPLOY_ENV IMAGE_TAG KUBE_CONTEXT; do
  if [ -z "${!var:-}" ]; then
    echo "ERROR: required variable ${var} is not set" >&2
    exit 1
  fi
done
```

## Quoting and other non-negotiables

- Quote expansions used as command arguments or paths: `"$VAR"`,
  `"$(cmd)"`. Leave them unquoted only when word splitting or globbing is
  intentional and documented.
- Build command arguments in bash arrays, not space-joined strings:

  ```bash
  #!/usr/bin/env bash
  set -Eeuo pipefail

  docker_args=(--pull --no-cache)
  if [ "${CI_COMMIT_REF_PROTECTED:-false}" = "true" ]; then
    docker_args+=(--label "protected=true")
  fi
  docker build "${docker_args[@]}" --tag "$IMAGE_TAG" .
  ```

- Use `printf` over `echo -e` for anything with escapes or variables.
- Temporary files: `mktemp`, and clean up with
  `trap 'rm -f "$tmpfile"' EXIT`.
- Run `shellcheck` on `scripts/ci/*.sh` in the pipeline itself; it is a
  one-job lint stage and catches most of the above mechanically (recipe
  below).

## Linting CI bash with shellcheck

**When:** any repo with `scripts/ci/*.sh` (see the extraction criteria above). Catches
quoting bugs, unset-variable typos, and most other shell mistakes
mechanically, in seconds.

```yaml
lint:shellcheck:
  stage: test
  image: koalaman/shellcheck-alpine:stable
  needs: []
  script:
    - shellcheck scripts/ci/*.sh
```

- `needs: []` starts it at pipeline creation; it only needs the source.
- Pin the image tag to the project's standard instead of `stable` if
  reproducible lint results matter.
  <!-- volatile: re-verify image name on version bump -->
