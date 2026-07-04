# Orchestration: downstream pipelines, matrices, failure control

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/pipelines/downstream_pipelines/
- https://archives.docs.gitlab.com/18.11/ci/yaml/matrix_expressions/
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_control/
- https://archives.docs.gitlab.com/18.11/ci/runners/configure_runners/
- https://archives.docs.gitlab.com/18.11/ci/yaml/lint/
- https://archives.docs.gitlab.com/18.11/api/lint/
- https://docs.gitlab.com/update/deprecations/ (changes after the 18.11 baseline)

Pipelines that start pipelines, jobs that fan out, and keeping failure
behavior deliberate: triggers, matrices, auto-cancel, retries, timeout
budgets, and validating what GitLab actually compiled. Examples assume
Free tier and Linux runners; executor requirements are flagged per
pattern.

Recipes call `./scripts/ci/<name>.sh` wherever real logic would live.
That is a placeholder, not a mandate: short wiring commands can stay
inline in `script:`. The inline-vs-extracted criteria are in
`bash-in-ci.md`.

## 1. Parent-child and multi-project pipelines

**When:** one pipeline graph is too large, independent domains have
different owners, or one project must trigger deployment in another.

Parent `.gitlab-ci.yml`:

```yaml
trigger:payments:
  stage: test
  trigger:
    include:
      - local: .gitlab/ci/payments.gitlab-ci.yml
        inputs:
          service: payments
    strategy: mirror
```

Child `.gitlab/ci/payments.gitlab-ci.yml`:

```yaml
spec:
  inputs:
    service:
      options: [payments, billing]
      default: payments
---
test:$[[ inputs.service ]]:
  script:
    - ./scripts/ci/test-service.sh "$[[ inputs.service ]]"
```

- Parent-child pipelines use the same project, ref, and commit SHA. Child
  pipelines can nest two levels deep. Multi-project pipelines are not
  subject to that nesting-depth limit, but one hierarchy contains at most
  1,000 downstream pipelines by default.
  <!-- volatile: re-verify on version bump -->
- Use `strategy: mirror` (introduced in GitLab 18.2); the trigger job
  then matches the downstream status. On 18.0-18.1 only
  `strategy: depend` exists; the docs no longer recommend it because its
  status can diverge from the downstream pipeline.
- Prefer typed `trigger:include:inputs` for parent-child pipelines and
  `trigger:inputs` for multi-project pipelines. Forward variables only
  deliberately with `trigger:forward`; forwarded variables have high
  precedence and can overwrite downstream values.
- In child jobs, `CI_PIPELINE_SOURCE` is `parent_pipeline`. In
  multi-project jobs it is `pipeline`. A child pipeline triggered from an
  MR still uses `parent_pipeline`; use `CI_MERGE_REQUEST_*` variables when
  the child must distinguish MR context.
- The user who triggers the upstream pipeline must be able to start
  pipelines in the downstream project, or the trigger job fails. A
  multi-project pipeline is not canceled merely because a newer pipeline
  starts for the same ref in the upstream project. The upstream UI exposes
  the downstream project name and status even when that project is private.
- In the parent graph, each downstream pipeline appears as a card. GitLab
  expands one downstream pipeline at a time. This reduces the root graph
  only when the child owns a coherent domain; splitting solely for
  appearance adds navigation and data-flow complexity without an ownership
  boundary. Selection guidance is in `pipeline-ui.md`.
- Child-pipeline reports can surface in the parent's MR widgets when the
  trigger uses `strategy: mirror` or `strategy: depend`. Version gates
  and the supported report types are in `pipeline-ui.md`.

## 2. Dynamic child pipelines for generated topology

**When:** the set of jobs depends on changed services, generated metadata,
or a target matrix that cannot be expressed statically.

```yaml
generate:child-config:
  stage: build
  script:
    - ./scripts/ci/generate-child.sh > generated-child.yml
  artifacts:
    paths:
      - generated-child.yml

trigger:generated-child:
  stage: test
  needs:
    - job: generate:child-config
      artifacts: true
  trigger:
    include:
      - artifact: generated-child.yml
        job: generate:child-config
    strategy: mirror
```

- GitLab, not the runner, parses the artifact path. Use path syntax for
  the GitLab host OS.
- Dynamic child configuration cannot use CI/CD variables in its own
  `include` section. Generate resolved include locations instead.
- One child trigger can merge at most three configuration files.
  <!-- volatile: re-verify on version bump -->
- Prefer `include:rules:changes` for a fixed set of monorepo domains.
  Generate YAML only when pipeline topology is genuinely data-dependent;
  generators add a compilation step that must itself be tested.

## 3. Matrix jobs and conditional DAGs

**When:** the same build or test must run over a bounded set of platforms,
versions, or feature combinations.

```yaml
stages: [build, test]

.build-matrix: &build-matrix
  parallel:
    matrix:
      - OS: [ubuntu, alpine]
        ARCH: [amd64, arm64]

build:
  stage: build
  <<: *build-matrix
  script:
    - ./scripts/ci/build.sh "$OS" "$ARCH"

test:
  stage: test
  <<: *build-matrix
  needs:
    - job: build
      parallel:
        matrix:
          - OS: ['$[[ matrix.OS ]]']
            ARCH: ['$[[ matrix.ARCH ]]']
  script:
    - ./scripts/ci/test.sh "$OS" "$ARCH"
```

- Matrix expressions are available in GitLab 18.6+ and create compile-time
  one-to-one dependencies. They can only copy identifiers from the current
  matrix; they cannot evaluate runtime variables or arbitrary expressions.
- `parallel:matrix` supports up to 200 permutations in GitLab 18.11.
  <!-- volatile: re-verify on version bump --> Matrix values become part
  of job names, so keep them short and unique.
- Use `needs:parallel:matrix` to wait for only the corresponding producer.
  A plain `needs: [build]` depends on every parallel build instance and can
  download artifacts with colliding names.
- `rules:needs` can replace a job's entire `needs` list per condition. It
  does not append. Include every required dependency in each matching rule,
  or use `needs: []` when the job should start immediately.

## 4. Cancel stale work and classify failures

**When:** developer pipelines waste capacity after a newer push or retry
deterministic failures that need a code change.

```yaml
workflow:
  auto_cancel:
    on_new_commit: interruptible
    on_job_failure: all

default:
  interruptible: true

test:integration:
  script:
    - ./scripts/ci/integration-test.sh
  retry:
    max: 2
    when:
      - runner_system_failure

test:browser:
  script:
    - ./scripts/ci/browser-test.sh
  retry:
    max: 1
    exit_codes:
      - 137

lint:advisory:
  script:
    - ./scripts/ci/advisory-lint.sh
  allow_failure:
    exit_codes:
      - 2
```

- `on_new_commit: interruptible` cancels only jobs marked interruptible.
  Keep deployments, migrations, and non-idempotent jobs
  `interruptible: false`.
- `on_job_failure: all` cancels other running work immediately. Use it only
  when partial pipeline results have no value.
- Retry transient infrastructure failures and explicit process exit codes,
  not every `script_failure`. GitLab permits at most two retries.
  `retry:when` and `retry:exit_codes` can be combined in one job; the job
  retries when either filter matches.
- `allow_failure:exit_codes` documents expected non-fatal outcomes while
  keeping unexpected exit codes blocking.

> **Deprecated (19.1):** `stuck_or_timeout_failure` and
> `job_execution_timeout` are aliases for newer, specific failure reasons
> and are scheduled for removal in GitLab 20.0. Consult the current
> `retry:when` list before adding timeout or stuck-job retries.

## 5. Budget timeouts and validate compiled configuration

**When:** a long script consumes the whole job timeout, preventing cleanup
or artifact upload, or included configuration behaves differently from the
source files.

```yaml
test:system:
  timeout: 45m
  variables:
    RUNNER_SCRIPT_TIMEOUT: 35m
    RUNNER_AFTER_SCRIPT_TIMEOUT: 5m
  script:
    - ./scripts/ci/system-test.sh
  after_script:
    - ./scripts/ci/collect-diagnostics.sh
  artifacts:
    when: always
    paths:
      - diagnostics/
```

- Keep `RUNNER_SCRIPT_TIMEOUT + RUNNER_AFTER_SCRIPT_TIMEOUT` within the job
  timeout. Reserve additional time for cache and artifact upload. Runner
  maximum timeout can still cap the job below its YAML `timeout`.
- Validate generic YAML syntax with `yamllint`, then GitLab semantics with
  CI Lint. CI Lint pipeline simulation catches `rules` and `needs` errors
  that static schema checks miss. `gitlab-ci-local` is useful for fast
  local feedback but is not GitLab's compiler and can disagree with it.
- For exact branch or tag behavior, call `POST /projects/:id/ci/lint` with
  `dry_run: true`, `ref`, and `include_jobs: true`. Inspect `merged_yaml`,
  warnings, and the job list to debug include order, inheritance, and array
  replacement.
- Validate generated child YAML before publishing its artifact. A valid
  parent does not prove that runtime-generated configuration is valid.

## Validation levels and what each proves

No single check proves a pipeline correct. The levels, cheapest first;
each catches what the previous cannot:

1. YAML syntax and schema: an editor with the SchemaStore mapping
   (`*.gitlab-ci.yml`, see `pipeline-structure.md`) or `yamllint`.
   Catches indentation and type errors only.
2. GitLab CI Lint (pipeline editor Validate tab, or
   `POST /projects/:id/ci/lint`): GitLab's own compiler. Resolves
   `include`, `extends`, `!reference`; catches unknown keywords,
   missing `needs` targets, undefined stages.
3. Lint simulation (`dry_run: true` with `ref` and
   `include_jobs: true`): evaluates `workflow:rules` and job `rules`
   for a real ref and returns `merged_yaml` plus the job list. Catches
   "job silently missing" and include-merge surprises.
4. `shellcheck` over `scripts/ci/*.sh`: job recipe in `bash-in-ci.md`.
5. A real pipeline on a branch: the only level that exercises runner
   selection, image pulls, services, caches, and the scripts
   themselves. `gitlab-ci-local` approximates parts of this locally but
   is not GitLab's compiler; treat agreement as convenience, not proof.

State the level reached when handing config over: "lints clean and the
simulated MR pipeline contains the expected jobs" is honest; a bare
"verified" is not.
