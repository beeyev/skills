# GitLab UI integration: put CI/CD results where users work

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/pipelines/ (pipeline names, New pipeline form, and manual variables)
- https://archives.docs.gitlab.com/18.11/ci/inputs/ (pipeline configuration inputs)
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_inputs/ (typed manual-job inputs)
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_control/ (manual jobs, variables, confirmation, and protection)
- https://archives.docs.gitlab.com/18.11/ci/yaml/artifacts_reports/ (native report surfaces)
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_artifacts/ (artifact browsing, access, and MR links)
- https://archives.docs.gitlab.com/18.11/ci/testing/unit_test_reports/ (JUnit rendering requirements)
- https://archives.docs.gitlab.com/18.11/ci/testing/code_coverage/ (coverage percentage and diff visualization)
- https://archives.docs.gitlab.com/18.11/ci/pipelines/downstream_pipelines/ (child reports in MR widgets)
- https://archives.docs.gitlab.com/18.11/ci/pipeline_editor/ (Visualize and Full configuration)
- https://archives.docs.gitlab.com/18.11/user/project/releases/ (release pages and assets)
- https://gitlab.com/gitlab-org/gitlab/-/blob/v18.11.0-ee/.gitlab/ci/rails.gitlab-ci.yml (GitLab's own coverage configuration)
- https://gitlab.com/gitlab-org/cli/-/tags/v1.102.0 (pinned release-job image)

Choose the UI surface before choosing the CI keyword. A useful pipeline
should not require users to infer state from job names or search raw logs for
structured results.

Graph layout lives in `pipeline-ui.md`. Artifact transfer and retention live
in `data-flow.md`. Environment and deployment pages live in
`environments.md`. This file owns the mapping from CI output and actions to
GitLab UI surfaces.

## Surface map

| User need | GitLab surface | CI/CD feature |
|---|---|---|
| Distinguish pipeline purpose | Build > Pipelines, MR pipeline summary | `workflow:name` |
| Start a pipeline with validated choices | New pipeline form | Pipeline `spec:inputs` |
| Run or retry one job with validated choices | Manual job form | Job `inputs` (GitLab 18.10+, Runner 18.9+) |
| Confirm a destructive action | Manual job confirmation dialog | `when: manual` plus `manual_confirmation` |
| Understand test failures | MR Test summary, pipeline Tests tab | `artifacts:reports:junit` |
| See overall coverage and its delta | MR widget, job list, analytics, badge | `coverage` regex |
| See uncovered changed lines | MR diff | `artifacts:reports:coverage_report` |
| Review quality, security, performance, or IaC findings | MR widgets, MR diff, pipeline tabs, dashboards | Matching `artifacts:reports:*` type |
| Open one important generated file | MR exposed-artifact section | `artifacts:expose_as` plus `artifacts:paths` |
| Open an external preview or trace | Job output page | `artifacts:reports:annotations` with `external_link` |
| Browse or download diagnostics | Job page, Build > Artifacts | `artifacts:paths`, `name`, `expire_in`, `access` |
| Publish version notes and stable asset links | Deploy > Releases | `release:` or `glab release create` |
| Show deployment state and live URL | Operate > Environments, Deployments, MR | `environment:` and deployment metadata |

Do not duplicate the same result across logs, generic artifacts, and a native
report without a specific reason. Keep a raw artifact when humans or tools need
the source file. Use the native report for review.

## Make pipeline starts into forms

For values fixed when the pipeline is created, prefer pipeline inputs over
free-form pipeline variables. Inputs provide descriptions, types, options,
regex validation, and explicit scope. Give every top-level input a default
when branch, tag, MR, schedule, or trigger pipelines can start automatically;
otherwise those pipelines can fail before creating any jobs.

GitLab 18.11 allows at most 20 pipeline inputs. They must resolve into the
main configuration header. Ordinary inputs declared inside included job files
do not become pipeline inputs; use `spec:include` when reusing input
definitions from an external file. After migrating consumers, restrict or
disable pipeline variables to prevent an unvalidated second control path.

```yaml
spec:
  inputs:
    deploy_target:
      description: "Target for this pipeline"
      default: staging
      options: [staging, production]
    run_smoke_tests:
      description: "Run smoke tests after deployment"
      type: boolean
      default: true
---

deploy:
  script:
    - ./scripts/ci/deploy.sh "$[[ inputs.deploy_target ]]"
```

Use pipeline-level variables with `description`, `value`, and `options` only
when variable semantics are required. They appear on the New pipeline page,
but are less strongly typed. Job-level variables cannot prefill that page.

Manual pipeline variables are not a secret-entry form. In GitLab 18.4+, a
project Owner can enable the Manual Variables tab. Guests can then see names
and Developers can reveal values. Manually overridden values are expanded and
not masked. Use protected variables or external secrets instead.

## Make manual jobs safe and specific

Use job inputs when an operator must change one job while running or retrying
it. They are typed, validated, scoped to that job, and require defaults.

```yaml
deploy:service:
  when: manual
  allow_failure: false
  manual_confirmation: "Deploy the selected version to production?"
  inputs:
    version:
      default: latest
      description: "Immutable image tag or digest"
      regex: '^[A-Za-z0-9._:@-]+$'
    replicas:
      type: number
      default: 3
      description: "Desired replica count"
  script:
    - ./scripts/ci/deploy.sh '${{ job.inputs.version }}' '${{ job.inputs.replicas }}'
  environment:
    name: production
```

Job inputs can be interpolated only into `script`, `before_script`,
`after_script`, `artifacts`, `cache`, `image`, and `services`. They cannot
change job names, stages, rules, includes, or other pipeline-creation fields.
Use pipeline configuration inputs for those fields.

The play button uses defaults without opening the input form. A configured
confirmation can still intercept the action. To edit job inputs or manual job
variables, open the job name and use its form. `manual_confirmation` adds a
confirmation step, not authorization. Use a protected environment to restrict
who can run a deployment job. A blocking manual job needs
`allow_failure: false`; with "Pipelines must succeed", it also blocks merging
until an authorized user runs it.

Never ask operators to enter credentials as manual job variables. Users who
can retry the job can view the originally supplied variables, and overridden
values are not masked.

## Publish reports to native review surfaces

Use the report type that matches the result. Tier-gated features must be
checked against the target GitLab instance.

| Report | Primary UI result | Important constraint |
|---|---|---|
| `junit` | MR Test summary, pipeline Tests tab | JUnit XML; report does not fail the job |
| `coverage_report` | Line annotations in changed MR files | Cobertura or JaCoCo; does not set overall percentage |
| `codequality` | MR widget, diff annotations, full report | Use GitLab code quality format |
| `accessibility` | MR accessibility widget | pa11y report |
| `browser_performance` | MR browser-performance widget | Premium/Ultimate; one report only |
| `load_performance` | MR load-testing widget | Premium/Ultimate; one report only |
| `metrics` | MR metrics widget | Premium/Ultimate |
| `terraform` | MR OpenTofu plan widget | Remove credentials before upload |
| `requirements` | Project requirements state | Ultimate; marks referenced requirements satisfied |
| Security reports | MR security widget, pipeline Security tab, vulnerability views | Most scanners and surfaces are tier-gated; use supported schemas or GitLab templates |
| `annotations` | Links on the producing job's output page | Currently useful for `external_link` annotations |

`artifacts:reports` files upload on job success or failure regardless of
`artifacts:when`. Uploading a report does not usually decide job status. The
producer must still exit non-zero when findings should fail the job. Add the
same file under `artifacts:paths` only when users must browse or download the
raw report; reports alone are not browsable from the job page.

Not every report type creates an MR widget. For example, `dotenv` passes
variables to later jobs, while CycloneDX feeds SBOM and dependency features.
SARIF ingestion in GitLab 18.11 is Ultimate and behind the disabled
`sarif_ingestion` feature flag. Confirm both the report's destination and its
tier or feature-flag state before promising a UI result.

### Coverage needs two declarations

Overall coverage and line annotations are separate features. Configure both
when users need both:

```yaml
test:coverage:
  script:
    - ./scripts/ci/test-with-coverage.sh
  coverage: '/TOTAL.*? ([0-9]+(?:\.[0-9]+)?)%/'
  artifacts:
    paths:
      - coverage/index.html
    expose_as: "Coverage report"
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura.xml
```

The `coverage` regex extracts a percentage from the job log for the MR widget,
pipeline job list, analytics, and coverage badge. `coverage_report` parses XML
for changed-line annotations in the MR diff. GitLab's own Rails CI uses both
keywords and also retains the HTML report, which confirms these features are
complementary, not alternatives.

Test every coverage regex against actual tool output. A format change can
silently remove the displayed value while the job still succeeds.

### Make JUnit data actionable

JUnit reports do not make a failing test command fail the job. Preserve the
test runner's exit status. In generated XML:

- Make test names and classes unique. GitLab uses only the first duplicate and
  silently ignores the rest.
- Populate `file` so the UI can identify and copy the test for a local rerun.
- Put useful failure output in the failure/error element.
- Use `system-out` attachment syntax when supported to expose screenshots in
  the test details dialog.
- Upload reports when tests fail. The report artifact itself already uploads
  regardless of job result; do not suppress report generation after failure.

In GitLab 18.11, each JUnit file must be under 30 MB and all JUnit files in one
job must total under 100 MB. Split the work across jobs when needed, while
keeping names unique across the resulting report set.

GitLab compares source and target branch reports to classify newly failed,
resolved, and existing failures. If the target branch has no comparable test
report, that comparison is unavailable.

## Link files and external systems deliberately

Use `artifacts:expose_as` for one review artifact that belongs in the MR:

```yaml
docs:preview:
  script:
    - ./scripts/ci/build-docs.sh
  artifacts:
    expose_as: "Rendered documentation"
    paths:
      - public/index.html
    expire_in: 1 week
```

Limits in GitLab 18.11:

- One `expose_as` per job and at most ten exposing jobs per MR.
- Glob patterns and CI/CD variables in exposed paths are unsupported.
- A directory path must end in `/`.
- One file opens directly; multiple files or a directory open the artifact
  browser.
- Preview behavior depends on file type, GitLab Pages, and Pages access
  control. Do not assume an HTML artifact renders inline on every instance.

Use `artifacts:access` to restrict UI and API download by role. It does not
control artifact forwarding to downstream pipelines. Treat artifact URLs as
subject to both expiry and authorization; do not publish them as permanent
release links.

For an external preview, trace, or dashboard, generate an annotations JSON
report:

```json
{
  "links": [
    {
      "external_link": {
        "label": "Preview deployment",
        "url": "https://preview.example.com/build/123"
      }
    }
  ]
}
```

```yaml
artifacts:
  reports:
    annotations: annotations.json
```

The link appears on the job output page, not in the MR widget. Use an
environment URL when the link represents a deployment; that integrates with
MRs, environments, and deployment history instead.

## Preserve reports across child pipelines

GitLab 18.6+ can show JUnit, code quality, Terraform, and metrics reports from
child pipelines in MR widgets. GitLab 18.9+ adds supported security reports.
The trigger must use `strategy: mirror` or `strategy: depend`; otherwise the
parent can finish before the report exists. Prefer `mirror` because the trigger
status matches the downstream pipeline.

```yaml
test:backend:
  trigger:
    include:
      - local: .gitlab/ci/backend.yml
    strategy: mirror
```

This UI aggregation does not merge child artifacts into the parent pipeline.
Coverage reports from child pipelines can annotate MR diffs, but parent jobs
still cannot consume those reports automatically. Use explicit artifact-fetch
mechanisms when a parent job needs the file.

## Publish releases, not expiring artifact URLs

Use a GitLab Release for versioned output that users must find after pipeline
artifacts expire. Build and publish binaries or packages first, then create the
release and link stable package-registry URLs.

```yaml
release:publish:
  stage: deploy
  image: registry.gitlab.com/gitlab-org/cli:v1.102.0
  rules:
    - if: $CI_COMMIT_TAG
  script:
    - echo "Creating release ${CI_COMMIT_TAG}"
  release:
    tag_name: $CI_COMMIT_TAG
    name: "Release $CI_COMMIT_TAG"
    description: CHANGELOG.md
```

The `release:` section runs after `script` and before `after_script`. It creates
the Release object only if `script` succeeds. It does not build or upload the
assets. If the release already exists, the job fails instead of updating it.
Use `glab release create` or the Releases API when asset upload, update, or
idempotent behavior needs explicit control. Protect release tags because tag
creation permission also controls release creation.

## Use the editor as a UI integration check

Pipeline Editor > Visualize shows stages, jobs, and `needs` edges. Full
configuration expands includes and `extends`, but evaluates conditional rules
as a default-branch push. Neither view proves MR, tag, schedule, or manual
behavior.

For a UI-facing change:

1. Validate compiled YAML with CI Lint for the relevant ref.
2. Run the actual pipeline source that owns the surface.
3. Open the exact surface: New pipeline form, manual job form, MR widget, MR
   diff, pipeline tab, job page, release, or environment.
4. Verify permissions with both an authorized and unauthorized role when the
   action or artifact is restricted.
5. Confirm the job status still reflects the policy represented by the report.

## Failure signatures

| Symptom | Likely cause and correction |
|---|---|
| Input is absent from New pipeline | It is declared in an ordinary included file instead of the main `spec:inputs` header, or the selected ref has different configuration. Move or import the definition with `spec:include`, then inspect the selected ref. |
| Manual job runs without showing its input form | The user selected the play button. Open the job name to edit inputs. For job inputs, also require GitLab 18.10+ and Runner 18.9+. |
| MR report widget is absent | The pipeline is not an MR pipeline, the report schema failed to parse, the feature is tier-gated, the target branch lacks required baseline data, or the child trigger does not wait. Inspect the job's artifact-upload warnings and use `strategy: mirror` for child reports. |
| Report exists but cannot be browsed from the job | It is declared only under `artifacts:reports`. Add it to `artifacts:paths` if raw download or browsing is required. |
| Exposed artifact link is absent | The path uses a glob, variable, or directory without a trailing `/`; the MR exceeds ten exposing jobs; or the artifact expired. Use a literal path and explicit retention. |
| Coverage percentage appears without diff annotations | Only `coverage` is configured. Add a Cobertura or JaCoCo `coverage_report`. |
| Diff annotations appear without a coverage percentage | Only `coverage_report` is configured. Add and test a `coverage` regex against the job log. |
| Child report appears in the MR but parent jobs cannot download it | MR report aggregation does not transfer child artifacts. Fetch the artifact explicitly with the appropriate parent-child data-flow mechanism. |
| Release job fails although publication succeeded previously | `release:` does not update an existing Release. Make the job tag-idempotent or use `glab` or the Releases API for update semantics. |

## Review checklist

1. Can users identify pipeline purpose without opening it?
2. Are operator choices typed and validated at pipeline or job creation?
3. Are secrets excluded from manual pipeline and job variable forms?
4. Do destructive buttons have confirmation and separate authorization?
5. Does every structured result use the most specific supported report type?
6. Does the producing command still set the correct job exit status?
7. Are raw artifacts retained only as long as required and access-restricted?
8. Do child reports wait for the child pipeline and appear in the parent UI?
9. Are long-lived release assets stored outside expiring job artifacts?
10. Was the result checked in the real UI surface and pipeline source?
