# Developer experience: from pipeline list to local reproduction

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/debugging/ (debugging workflow and pipeline names)
- https://archives.docs.gitlab.com/18.11/ci/yaml/#workflowname (pipeline names)
- https://archives.docs.gitlab.com/18.11/ci/inputs/ (manual pipeline inputs)
- https://archives.docs.gitlab.com/18.11/ci/yaml/#artifactsexpose_as (MR artifact links)
- https://archives.docs.gitlab.com/18.11/ci/yaml/artifacts_reports/ (native reports and job annotations)
- https://archives.docs.gitlab.com/18.11/ci/testing/unit_test_reports/ (test details and screenshots)
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_logs/ (log sections and timestamps)

Optimize for three questions:

1. What is this pipeline doing?
2. Why did it fail?
3. How do I reproduce or investigate it?

Naming mechanics live in `readability.md`; log safety and section syntax
live in `informative-logging.md`; shell behavior lives in `bash-in-ci.md`.
This file connects them into the developer's debugging path.

## Make the pipeline identifiable

When a project runs materially different pipeline types, use
`workflow:name` so the pipeline list distinguishes MR, scheduled, release,
and deployment work. Describe the purpose, not information GitLab already
shows next to the pipeline.

```yaml
variables:
  APP_PIPELINE_NAME: "Branch pipeline"

workflow:
  name: "$APP_PIPELINE_NAME"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      variables:
        APP_PIPELINE_NAME: "MR: $CI_MERGE_REQUEST_SOURCE_BRANCH_NAME"
    # Suppress duplicate branch pipelines for open MRs (pattern 1 in common-patterns.md).
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS && $CI_PIPELINE_SOURCE == "push"
      when: never
    - if: $CI_PIPELINE_SOURCE == "schedule"
      variables:
        APP_PIPELINE_NAME: "Scheduled maintenance"
    - if: $CI_COMMIT_TAG
      variables:
        APP_PIPELINE_NAME: "Release: $CI_COMMIT_TAG"
    - if: $CI_COMMIT_BRANCH
```

Use a project-specific name variable. `workflow:rules:variables` are
forwarded to downstream pipelines by default and can overwrite a downstream
variable with the same name.

Treat manually started pipelines as forms. Pipeline-level `spec:inputs`
should have useful descriptions, safe defaults, and `options`, `type`, or
`regex` constraints where values are bounded. See `pipeline-structure.md`
for the input contract. A rejected input at pipeline creation is better
than a job failing after consuming runner time.

## Give each job an execution header

At job start, print the safe context needed to identify the exact run:

- Pipeline source, name, and URL.
- Job name and URL.
- Ref and full commit SHA.
- Deployment target or test variant.
- Runner description and relevant tool versions.

Do not dump the environment. Print only non-secret values that affect the
job's behavior. The version-banner pattern and secret rules are in
`informative-logging.md`.

Use headings for meaningful phases such as dependency installation, build,
test, and upload. Use GitLab collapsible sections for long output. Emit
the section end marker even when a command inside the section fails (for
example from an EXIT trap in an extracted script), so every section is
closed.

## Make failures actionable

A useful failure message states:

- The action that failed.
- Safe identifiers and targets involved.
- Expected and actual state when relevant.
- The source file, service, or script entry point.
- The next check or local reproduction command, when one exists.

```text
ERROR: failed to publish image
Image: registry.example.com/app:abc123
Registry: registry.example.com
Entry point: ./scripts/ci/publish-image.sh
Next step: verify registry availability, then retry this job
```

Do not replace a tool's specific error with a generic wrapper. Add only the
missing context. Send failure summaries to stderr and preserve the original
exit code. For the shell mechanics (an ERR trap that names the failing
command and line), see `bash-in-ci.md`.

## Prefer GitLab UI results over log archaeology

Use structured output whenever GitLab can render it better than a log dump:

- JUnit reports put failures in the MR Test summary and pipeline Tests tab.
  Include unique test names, source-file attributes, failure output, and
  screenshot attachments when supported. File attributes let developers
  copy failed test names for local reruns.
- Coverage and code-quality reports put results in MR widgets or diffs.
- `artifacts:expose_as` gives an important result a direct MR link.
- An `annotations` report with `external_link` adds a preview, trace, or
  external dashboard link to the job output page.

Reports do not necessarily control job status. The producing command must
still exit non-zero when its findings should fail the job. See
`common-patterns.md` for artifact and report mechanics.

## Retain bounded diagnostics

Upload evidence that is expensive or impossible to reconstruct: test
reports, screenshots, traces, generated configuration, and sanitized
service logs. Use `artifacts:when: always`, an identifying archive name,
and explicit retention. Never archive a workspace or home directory that
may contain credentials.

```yaml
test:system:
  script:
    - ./scripts/ci/system-test.sh
  artifacts:
    name: "diagnostics-$CI_JOB_NAME_SLUG-$CI_COMMIT_SHORT_SHA"
    when: always
    expire_in: 1 week
    paths:
      - diagnostics/
      - test-results/
    reports:
      junit: test-results/junit.xml
```

Use `after_script` for best-effort collection after a failed `script`.
It runs in a fresh shell and cannot change the original job result. Reserve
time for it; pattern 19 in `common-patterns.md` covers timeout budgeting.

## Keep the entry point reproducible

When runner behavior is not what the job tests:

- Run the same `scripts/ci/*.sh` entry point locally and in CI.
- Assume and document the repository root as the working directory.
- Keep runner orchestration in YAML and build/test/deploy logic in the
  extracted script. This applies to jobs whose logic warranted extraction
  in the first place (criteria in `bash-in-ci.md`); a job whose `script:`
  is a few inline commands is already reproducible by copy-paste.
- List required variable names. Never put secret values in a printed
  reproduction command.
- Pin the job image and print relevant tool versions. Document an equivalent
  container command when local toolchain drift matters.

If the failure depends on runner networking, services, mounts, or protected
credentials, state that constraint. Do not promise a misleading local
reproduction path.

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
CI Lint API returns `merged_yaml` and the job list per ref (pattern 19 in
`common-patterns.md`).
