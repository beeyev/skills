# Debugging failed pipelines

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/debugging/
- https://archives.docs.gitlab.com/18.11/ci/variables/predefined_variables/
- https://archives.docs.gitlab.com/18.11/api/lint/
- https://archives.docs.gitlab.com/18.11/ci/services/ (service debugging)

From symptom to cause. The other direction, designing pipelines that
are debuggable (execution headers, logs, reports), is
`developer-experience.md`.

## Debug in runner order

| Symptom | Inspect first |
|---|---|
| Pipeline absent or wrong type | `workflow:rules`, pipeline source, CI Lint simulation |
| Job absent | Conditional includes, job `rules`, compiled job list |
| Failure before `script` | Runner selection, image pull, checkout, cache, artifact download |
| Failure during `script` | Execution header, failing command, diagnostics, local entry point |
| Failure after `script` | `after_script`, cache/artifact upload, timeout budget |
| Downstream failure | Trigger job, linked pipeline, passed inputs and forwarded variables |

This avoids debugging application code when the runner never reached it,
or debugging runner infrastructure after the script emitted a specific
application failure.

For "pipeline absent" and "job absent" rows, inspect the configuration
GitLab actually compiled, not the source files: the pipeline editor's
**Full configuration** tab shows the merged YAML with includes and
`extends` resolved, `gitlab-ci-local --list` simulates it locally, and the
CI Lint API returns `merged_yaml` and the job list per ref (the validation levels in `orchestration.md`).

## Failure signatures

Concrete log or UI text mapped to cause and fix, grouped by the phase
that produced it. Quote wording varies slightly between versions; match
on the shape.

### Pipeline or job missing (configuration; CI Lint or simulation catches these)

| Signature | Cause and fix |
|---|---|
| `Unable to parse YAML` / `yaml invalid` badge | Broken YAML at the named line. Remember YAML eats syntax before GitLab sees it (colons, leading `[`/`&`/`*`); quoting rules in `bash-in-ci.md`. |
| `'job' ... needs 'other' ... does not exist` (or pipeline fails at creation) | `needs` target missing, renamed, or excluded by `rules`. For conditionally present targets use `needs: [{job: name, optional: true}]` (`data-flow.md`). |
| `chosen stage X does not exist` | Add the stage to `stages:` or fix the job's `stage:`. |
| Circular dependency reported for `needs` | Break the cycle; usually appears after a stage reshuffle. |
| `Local file 'X' does not exist` on `include:local` | Include paths resolve from the repo root, not from the including file. |
| `component 'X@V' ... could not be found` | Wrong component path or version; check the component project's releases and the `$CI_SERVER_FQDN` prefix (`pipeline-structure.md`). |
| Job defined in YAML but absent from the pipeline | Its `rules` matched nothing, a conditional include skipped its file, or `workflow:rules` changed the pipeline type. Inspect the compiled config (below). |
| No pipeline at all for a push | `workflow:rules` filtered it, or the push matched no rule. Pattern 1 in `pipeline-selection.md`; also check "Pipelines must succeed" deadlock noted there. |
| Two pipelines per push to an MR branch | Duplicate branch + MR pipelines; `workflow:rules` pattern 1 in `pipeline-selection.md`. |

### Before the script runs (environment)

| Signature | Cause and fix |
|---|---|
| `manifest for image:tag not found` | The tag does not exist (typo, or upstream removed it). Pin an existing tag (`execution-environment.md`). |
| `pull access denied` / `no basic auth credentials` | Private image without credentials; `DOCKER_AUTH_CONFIG`, or missing role on the hosting project (`execution-environment.md`). |
| `Cannot connect to the Docker daemon` in a dind job | dind service missing or not ready, or `DOCKER_TLS_CERTDIR`/host mismatch; Docker build pattern in `execution-environment.md`. |
| `WARNING: Service X probably didn't start properly` | Service crashed or was slow to open its port. `CI_DEBUG_SERVICES: "true"` shows its logs (`execution-environment.md`). |
| Job pending forever, "This job is stuck" | No eligible runner: tags or untagged-job settings do not match, the runner is offline or paused, a protected runner rejects the ref, or the relevant runner scope is disabled. If an eligible runner exists but is busy, this is queue capacity instead. Runners and tags in `execution-environment.md`. |
| Cache restored on no run, or created on every run | Key mismatch or key churn; cache forensics in `data-flow.md`. |

### During and after the script (runtime)

| Signature | Cause and fix |
|---|---|
| `command not found` | Binary not in the job image: wrong image, missing install step, or a bashism under busybox `sh` (`bash-in-ci.md`). |
| `VAR: unbound variable` or a silently empty `$VAR` | Variable not in scope: protected variable on an unprotected ref, wrong pipeline type (MR vs branch), precedence override, or typo surfaced by `set -u`. See the confusions below. |
| Exit code 137, or `Killed` with no other output | The process received `SIGKILL`. OOM is common, but an external kill, cancellation, or timeout can produce the same status. Check runner and container OOM evidence before reducing workers or increasing memory. Retry exit 137 only after confirming a transient infrastructure cause (`orchestration.md`). |
| Artifact upload fails with a size error | Over the instance's max artifact size. Narrow `artifacts:paths` to what downstream consumes (`data-flow.md`). |
| Cleanup or diagnostics missing after failure | `after_script` runs in a new shell with its own timeout and cannot change job status; budget for it (`bash-in-ci.md`, timeout pattern in `orchestration.md`). |
| Job canceled mid-run without a human canceling | `workflow:auto_cancel` plus `interruptible: true`: a newer commit superseded the pipeline (`orchestration.md`). |

## Confusions that cost hours

- **Lint passes, but nothing runs.** CI Lint validates syntax and
  structure; `workflow:rules` and job `rules` are evaluated per ref and
  trigger. Use lint simulation with `dry_run` and `ref` (validation
  levels in `orchestration.md`).
- **The job runs but `$MY_VAR` is empty.** In order of likelihood:
  protected variable on an unprotected ref; MR pipeline versus branch
  pipeline changing which variables exist (`CI_COMMIT_BRANCH` is unset
  in MR and tag pipelines); an environment-scoped variable outside its
  environment; a higher-precedence definition overriding yours
  (precedence order below).
- **`rules:` cannot see a variable a script exported.** Rules evaluate
  at pipeline creation, before any job runs. They can use variables
  available while GitLab creates the pipeline, including applicable
  predefined, instance/group/project, pipeline, manual, schedule,
  trigger, top-level YAML, and `workflow:rules:variables` values. They
  cannot use job-only variables, script exports, or dotenv values
  produced by earlier jobs. Pass runtime values between job scripts with
  `artifacts:reports:dotenv`; they cannot change the already-compiled
  rules result.
- **`needs:` broke after a refactor.** A job was renamed or its rules
  changed, so the target vanished from some pipeline types. Diff the
  compiled config before and after (Full configuration tab or CI Lint
  `merged_yaml`).
- **A retry "fixed" it.** Transient infrastructure. Codify with
  `retry:when: [runner_system_failure]` or `retry:exit_codes`
  (`orchestration.md`); blanket-retrying `script_failure` hides real
  bugs.
- **`$?` is always 0 between `script:` items.** The runner's trace echo
  overwrites it; check exit status within the same item
  (`bash-in-ci.md`).

## Variable precedence

When two definitions share a name, the higher one wins silently. From
highest to lowest: <!-- volatile: re-verify on version bump -->

1. Pipeline execution policy and scan execution policy variables
2. Pipeline variables (manual run, schedule, trigger, API,
   `trigger:forward` from upstream)
3. Project variables
4. Group variables (closest subgroup wins)
5. Instance variables
6. Variables from `artifacts:reports:dotenv`
7. Job-level `variables:` in YAML
8. Top-level `variables:` in YAML
9. Deployment variables
10. Predefined variables

Most "my YAML variable is ignored" tickets are a project/group/pipeline
variable (2-5) overriding a YAML definition (7-8).

## Predefined variables worth memorizing

The traps behind most "empty variable" tickets. Full list:
https://archives.docs.gitlab.com/18.11/ci/variables/predefined_variables/

| Variable | Trap |
|---|---|
| `CI_COMMIT_BRANCH` | Unset in MR pipelines and tag pipelines, not just "the branch". |
| `CI_COMMIT_TAG` | Set only in tag pipelines. |
| `CI_MERGE_REQUEST_*` | Set only in merge request pipelines; not in the branch pipeline of the same commit. |
| `CI_COMMIT_REF_NAME` / `CI_COMMIT_REF_SLUG` | Branch or tag name; the slug is DNS/cache-key safe, the name is not. |
| `CI_PIPELINE_SOURCE` | `push`, `merge_request_event`, `schedule`, `trigger`, `pipeline`, `parent_pipeline`, `web`, `api`. Child pipelines report `parent_pipeline`, not the parent's source. |
| `CI_OPEN_MERGE_REQUESTS` | Present in branch pipelines too when an MR is open; the duplicate-pipeline guard relies on it. |
| `CI_ENVIRONMENT_*` | Only in jobs that declare `environment:`. Environment actions and scoped-variable behavior are in `environments.md`. |
| `CI_REGISTRY_*` | Only when the container registry is enabled for the project. |
| `CI_JOB_TOKEN` | Dies with the job; scope and allowlist in `security.md`. |
