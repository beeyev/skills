# Common patterns

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/yaml/ (keyword reference)
- https://archives.docs.gitlab.com/18.11/ci/yaml/workflow/
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_rules/
- https://archives.docs.gitlab.com/18.11/ci/caching/
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_artifacts/
- https://archives.docs.gitlab.com/18.11/ci/yaml/artifacts_reports/
- https://archives.docs.gitlab.com/18.11/ci/yaml/needs/
- https://archives.docs.gitlab.com/18.11/ci/docker/using_docker_build/
- https://archives.docs.gitlab.com/18.11/ci/environments/
- https://archives.docs.gitlab.com/18.11/ci/resource_groups/
- https://archives.docs.gitlab.com/18.11/ci/variables/
- https://archives.docs.gitlab.com/18.11/ci/pipelines/merge_request_pipelines/
- https://archives.docs.gitlab.com/18.11/ci/pipelines/downstream_pipelines/
- https://archives.docs.gitlab.com/18.11/ci/yaml/matrix_expressions/
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_control/
- https://archives.docs.gitlab.com/18.11/ci/runners/configure_runners/
- https://archives.docs.gitlab.com/18.11/ci/yaml/lint/
- https://archives.docs.gitlab.com/18.11/api/lint/
- https://docs.gitlab.com/update/deprecations/ (changes after the 18.11 baseline)
- https://docs.docker.com/engine/release-notes/29/ (Docker release used in the example)

Applied recipes only; mechanics of includes/`extends`/`!reference` live in
`pipeline-structure.md`. Examples assume Free tier and Linux runners;
executor requirements are flagged per pattern.

## 1. Branch + MR pipelines without duplicates

**When:** a project enables both branch and merge request pipelines. A
job with broad `rules` can make one push to an open merge request start
both pipeline types for the same commit. Use `workflow:rules` to choose
at pipeline level.

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS && $CI_PIPELINE_SOURCE == "push"
      when: never
    - if: $CI_COMMIT_BRANCH
    - if: $CI_COMMIT_TAG
```

Reads as: run MR pipelines; suppress the push pipeline when its branch
has an open MR; otherwise run branch pipelines; run tag pipelines. The
`push` check keeps scheduled, triggered, and API-created branch pipelines
from being suppressed by an unrelated open MR.

- `workflow:rules` is evaluated before any job rules; a pipeline type
  blocked here never runs regardless of job rules.
- Stricter variant for protected long-lived branches only:
  `- if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH` plus
  `- if: $CI_COMMIT_REF_PROTECTED == "true"` instead of the bare
  `$CI_COMMIT_BRANCH` rule.
- Gotcha: with "Pipelines must succeed" enabled, a workflow that filters
  out all MR pipelines leaves MRs stuck on "Checking pipeline status".
- Pair with `interruptible: true` (usually in `default:`) so a follow-up
  commit or force-push auto-cancels the superseded pipeline instead of
  letting a stale run finish.

## 2. Reusable rule sets

**When:** the same `rules:` list starts appearing in more than one job.
Job `rules` are evaluated first-match-wins; `extends` would replace the
whole array, so compose with `!reference`:

```yaml
.rules-mr:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

.rules-release-tag:
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/

lint:
  rules:
    - !reference [.rules-mr, rules]
  script: ./scripts/ci/lint.sh

publish:
  rules:
    - !reference [.rules-release-tag, rules]
  script: ./scripts/ci/publish.sh
```

Variable expressions cheat sheet: `==`/`!=` string compare, `=~`/`!~` RE2
regex (`/pattern/`, `i` flag for case-insensitive), `&&`/`||` and
parentheses, `$VAR` alone = defined and non-empty, `$VAR == null` =
undefined. `!` negation exists only in GitLab 18.11+.

## 3. Tag release pipelines

**When:** publishing versioned artifacts/images on `git tag`.

```yaml
release:publish:
  stage: deploy
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
  script:
    - echo "Publishing release ${CI_COMMIT_TAG}"
    - ./scripts/ci/publish.sh
```

- Anchor the regex (`^...$`); `/v\d/` would also match `not-v1-really`.
- In tag pipelines `CI_COMMIT_BRANCH` is unset and `CI_COMMIT_TAG` holds
  the tag; a job gated on `$CI_COMMIT_BRANCH` never runs for tags.
- Protect release tags (`v*`) in project settings so only allowed roles
  can trigger release pipelines and so protected variables are available
  to them.
- The `workflow:rules` block must allow tag pipelines (`if: $CI_COMMIT_TAG`).

## 4. Scheduled jobs

**When:** nightly full test runs, dependency audits, cleanup. Create the
schedule in **Build > Pipeline schedules**; in YAML, select on the
source:

```yaml
audit:dependencies:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script: ./scripts/ci/audit.sh
```

Exclude schedule-only work from other jobs explicitly, or a nightly
schedule can run work intended for normal pipelines:

```yaml
build:app:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
      when: never
    - when: on_success
  script: ./scripts/ci/build.sh
```

For multiple schedules in one project, distinguish them with a schedule
variable (e.g. `SCHEDULE_JOB: nightly-audit`) and match on it in rules.

## 5. Run a job only when files changed

**When:** monorepos, or expensive jobs tied to a subtree (docs, docker/,
migrations).

```yaml
build:docker-image:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - Dockerfile
        - docker/**/*
  script: ./scripts/ci/build-image.sh
```

- Without `compare_to`, use `rules:changes` for branch or merge request
  pipelines. It evaluates to **true** for new branch pipelines and for
  tag, scheduled, and manual pipelines, which lack a usable default
  comparison. Pair it with an `if:` clause pinning the pipeline type, as
  above, or set `changes:compare_to` explicitly.
- In MR pipelines it compares against the MR target branch; in branch
  pipelines against the previous commit.
- The same mechanism works on `include:rules` to pull whole config files
  in conditionally (see `pipeline-structure.md`).
- GitLab 18.11 permits at most 50 paths or patterns in one
  `rules:changes` section. After 50,000 file-pattern checks, patterned
  globs match instead of continuing precise evaluation.
  <!-- volatile: re-verify on version bump --> Keep monorepo patterns
  coarse, and use conditional includes or child pipelines when domains
  can be isolated.

## 6. Manual gates

**When:** a human must approve before deploy. A manual job inside
`rules` defaults to `allow_failure: false`, which makes it **blocking**:
later stages wait until someone runs it.

```yaml
deploy:production:
  stage: deploy
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
  manual_confirmation: "Deploy the default branch to production?"
  script: ./scripts/ci/deploy.sh
  environment:
    name: production
```

- Top-level `when: manual` (outside `rules`) defaults to
  `allow_failure: true`: an optional button, not a gate. Inside
  `rules`, the default flips to `allow_failure: false`. Set it
  explicitly if the difference matters; reviewers rarely remember this
  table.
- An optional manual job (allowed to fail) does not block the pipeline;
  a blocking one holds subsequent stages.
- `manual_confirmation` adds context before a destructive or expensive
  action. It reduces mistakes but is not an authorization boundary.

## 7. Caching dependencies

**When:** package-manager downloads (`node_modules`, pip, Maven, Go
modules). Cache is for **dependencies that can be re-derived**;
artifacts are for **results passed onward** (next pattern). Rule of
thumb:

| | cache | artifacts |
|---|---|---|
| Purpose | speed up repeat installs | hand files to later jobs / humans |
| Stored | on runner / object storage | in GitLab, downloadable in UI |
| Normal scope | same project, across pipelines | downstream jobs, UI, API |
| Availability | best effort; may miss | available only after successful upload and before expiry |
| Correctness may depend on it? | never | yes |

```yaml
test:unit:
  image: node:22-bookworm-slim
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - .npm/
  before_script:
    - npm ci --cache .npm --prefer-offline
  script:
    - npm test
```

- `cache:key:files` keys the cache to the lockfile hash: new lockfile,
  new cache. For branch-scoped caches use `key: $CI_COMMIT_REF_SLUG`
  with `fallback_keys: [cache-$CI_DEFAULT_BRANCH]`.
- Cache the package manager's *cache directory* (`.npm`, `.cache/pip`),
  not `node_modules` itself; installers handle partial caches, module
  dirs go stale.
- Jobs that only read the cache should declare `cache:policy: pull`
  (skip the upload step); one designated job per pipeline does
  `pull-push`.
- Cache keys get a `-protected` or `-non_protected` suffix by default, so
  protected and non-protected pipelines use separate cache pools (security
  feature). Since GitLab 18.4.5 the suffix also depends on who runs the
  pipeline: pipelines started by Maintainers or Owners use the
  `-protected` cache even on unprotected branches.
  <!-- volatile: re-verify on version bump --> A cold cache after
  switching pools is expected, not broken.
- A job never fails because the cache was missing. If it does, the job
  was depending on cache for correctness; move that file flow to
  artifacts.

## 8. Passing build outputs downstream

**When:** compile once, test/deploy that exact output.

```yaml
build:app:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

test:unit:
  stage: test
  needs: ["build:app"]
  script:
    - npm run test:ci
  artifacts:
    reports:
      junit: junit.xml
```

- With `needs`, a job downloads artifacts **only from the jobs it
  needs** (`needs:artifacts: true` is the default). Without `needs`, it
  downloads from all earlier-stage jobs; trim with `dependencies:` or
  `dependencies: []`. Do not combine `dependencies` with `needs`.
- Set `expire_in`; omitting it uses the instance's default artifact
  expiration. By default, GitLab keeps artifacts from the most recent
  successful pipeline on each ref despite `expire_in`; disable the
  keep-latest setting when strict expiry matters.
- `artifacts:reports:*` (junit, coverage, dotenv, ...) power MR widgets
  and are uploaded on success and failure regardless of `artifacts:when`.
  Add the same file to `artifacts:paths` (with `when: always`) only when
  humans must browse or download the raw report file after a failed job.

## 9. DAG pipelines with `needs`

**When:** independent chains are serialized by stage barriers and the
pipeline is slower than its critical path.

```yaml
stages: [build, test]

lint:
  stage: test
  needs: []
  script: ./scripts/ci/lint.sh

build:frontend:
  stage: build
  script: npm run build

build:backend:
  stage: build
  script: go build ./...

test:frontend:
  stage: test
  needs: ["build:frontend"]
  script: npm test

test:backend:
  stage: test
  needs: ["build:backend"]
  script: go test ./...
```

- `needs: []` starts a job immediately at pipeline start; right for
  lint/static checks that need only the source.
- A `needs` target that may be excluded by `rules` needs
  `needs: [{job: name, optional: true}]`, or pipeline creation fails
  with `'job' does not exist in the pipeline`.
- Keep stages as coarse readability grouping even when `needs` does the
  real ordering; a stageless everything-needs-everything config is hard
  to read in the graph view.

## 10. Building Docker images

**When:** the job itself must run `docker build`/`docker push`. The job
needs access to a Docker daemon. The recipe below uses Docker-in-Docker
and therefore requires the Docker or Kubernetes executor. A shell
executor can build images when Docker Engine is installed and the runner
user can access it.

Docker-in-Docker works on GitLab.com shared runners; on self-managed the
runner needs `privileged = true` plus `volumes = ["/certs/client"]` in
`config.toml` (the job container reads the daemon's TLS client certs
from there). Privileged mode is a security trade-off: privileged
containers can escalate to the host.

```yaml
build:docker-image:
  stage: build
  image: docker:29.6.1-cli
  services:
    - docker:29.6.1-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker --version
    - printf '%s' "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    - docker build --pull -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

- Pin the `docker` image and the `dind` service to the same version;
  `docker:latest` breaks on its own schedule.
  <!-- volatile: re-verify current docker image major on version bump -->
- `CI_REGISTRY*` variables are predefined and give push access to the
  project's own container registry; no manual token needed.
- Tag with an immutable ID (`$CI_COMMIT_SHORT_SHA` or `$CI_COMMIT_TAG`);
  add a floating tag (`latest`) only from the default branch.
- Each dind job starts a fresh daemon: no layer cache between jobs by
  default. Use `--cache-from` with a pushed image or BuildKit registry
  cache for reuse.
- Alternatives without privileged mode: socket binding (mount the host's
  `/var/run/docker.sock` in runner config; jobs then share and can
  affect the host daemon), or daemonless builders (Buildah, BuildKit
  rootless), which trade Docker CLI compatibility for a better security
  posture.

## 11. Environments, review apps, and deploy serialization

**When:** any job that deploys. Declaring `environment` gives a deploy
history, the environments UI, environment-scoped variables, and
stop/cleanup hooks.

Static environment with a manual gate (see pattern 6):

```yaml
deploy:staging:
  stage: deploy
  variables:
    DEPLOY_ENV: staging
  script: ./scripts/ci/deploy.sh
  environment:
    name: staging
    url: https://staging.example.com
  resource_group: staging
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
```

Dynamic review app per MR, with cleanup:

```yaml
deploy:review:
  stage: deploy
  script: ./scripts/ci/deploy-review.sh
  resource_group: review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_ENVIRONMENT_SLUG.review.example.com
    on_stop: stop:review
    auto_stop_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

stop:review:
  stage: deploy
  script: ./scripts/ci/destroy-review.sh
  resource_group: review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
      allow_failure: true
```

- `on_stop` + `action: stop` wires the stop job to the environment; it
  runs on manual stop, on `auto_stop_in` expiry, and when the MR merges
  or the branch is deleted.
- The matching `resource_group` values make the Environments UI run the
  `on_stop` job when a user stops the environment. `allow_failure: true`
  keeps this manual cleanup action from blocking the MR pipeline.
- Set `GIT_STRATEGY: none` only when the cleanup command is available in
  the job image or fetched another way. The example checks out the repo
  because its cleanup script lives in `scripts/ci/`.
- `resource_group: <name>` serializes jobs across pipelines: two
  deploys to the same target never overlap. The default process mode is
  `unordered`, so execution order is not guaranteed. Configure
  `oldest_first`, `newest_first`, or `newest_ready_first` through the
  resource groups API when ordering matters.

## 12. Linting CI bash with shellcheck

**When:** any repo with `scripts/ci/*.sh` (see `bash-in-ci.md`). Catches
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

## 13. Secrets and variable hygiene

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

## 14. Parent-child and multi-project pipelines

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

## 15. Dynamic child pipelines for generated topology

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

## 16. Matrix jobs and conditional DAGs

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

## 17. Cancel stale work and classify failures

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

## 18. Test sharding and checkout cost

**When:** one test job is the critical path, or a job does not need the
repository at all.

```yaml
test:unit:
  parallel: 4
  script:
    - ./scripts/ci/test-shard.sh "$CI_NODE_INDEX" "$CI_NODE_TOTAL"

notify:release:
  image: curlimages/curl:8.16.0
  variables:
    GIT_STRATEGY: empty
  script:
    - curl --fail-with-body --request POST "$RELEASE_WEBHOOK_URL"
```

- `parallel: N` creates N jobs and exposes `CI_NODE_INDEX` and
  `CI_NODE_TOTAL`. The test runner must partition tests deterministically;
  duplicating the complete suite N times only wastes runners.
- `GIT_STRATEGY: empty` deletes and recreates the build directory while
  skipping repository checkout. Prefer it for image-contained or
  artifact-only jobs. `none` can reuse stale repository, cache, and
  artifact files from an earlier job.
- A shallow `GIT_DEPTH` speeds checkout, but depth 1 can break queued or
  retried jobs whose commit is no longer reachable. `git describe` also
  needs sufficient history. Tune depth from evidence, not by habit.

## 19. Budget timeouts and validate compiled configuration

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
