# Data flow: cache, artifacts, and the job DAG

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/caching/
- https://archives.docs.gitlab.com/18.11/ci/yaml/ (keyword reference: `cache`, `artifacts`, `needs`)
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_artifacts/
- https://archives.docs.gitlab.com/18.11/ci/yaml/artifacts_reports/
- https://archives.docs.gitlab.com/18.11/ci/yaml/needs/
- https://archives.docs.gitlab.com/18.11/ci/runners/configure_runners/ (checkout strategy)
- https://archives.docs.gitlab.com/18.11/ci/pipelines/pipeline_efficiency/

How data moves through a pipeline: dependency caches across pipelines,
artifacts between jobs, and the `needs` DAG that orders them. Applied
recipes only; examples assume Free tier and Linux runners.

Recipes call `./scripts/ci/<name>.sh` wherever real logic would live.
That is a placeholder, not a mandate: short wiring commands can stay
inline in `script:`. The inline-vs-extracted criteria are in
`bash-in-ci.md`.

## 1. Caching dependencies

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

### Cache mechanics beyond the basics

- Cache paths must live under `$CI_PROJECT_DIR`; paths outside the
  project directory cannot be archived. Package managers that default
  to a home-directory cache need redirecting into the project dir (a
  cache-dir flag, or the tool's home variable), or the cache uploads
  nothing while looking configured.
- `cache:key:prefix` combines with `cache:key:files`: per-branch caches
  that still invalidate on lockfile change
  (`prefix: $CI_COMMIT_REF_SLUG`, `files: [<lockfile>]`). Trade-off
  against a plain `files:` key: branch isolation versus a cold cache on
  every new branch.
- A job can declare up to four caches as a list, each with its own key,
  paths, and policy. <!-- volatile: re-verify on version bump -->
- Warm-cache job: one job with `cache:policy: push` populates the cache
  (upload only, never download); consumers declare `policy: pull`. Put
  it in `stage: .pre` when everything else depends on the warm cache.
- Reading the log: the runner prints cache restore steps
  (`Checking cache for <key>`, `Successfully extracted cache`) and save
  steps (`Creating cache <key>`, `Created cache`). A cache created on
  every run means the key churns; check that `key:files` does not hash
  a file that changes each commit.
- When cache is wrong: if the directory changes more than it stays the
  same, upload plus download costs more than the cold start. Skip the
  cache and accept the fetch.

## 2. Passing build outputs downstream

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

## 3. DAG pipelines with `needs`

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

## 4. Test sharding and checkout cost

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

## Optimizing an existing pipeline: method before levers

Most speed levers live in this file (cache keys, artifact size,
`needs`), plus `orchestration.md` (matrices, auto-cancel, timeouts) and
job images (`execution-environment.md`). Before touching any of them:

1. Measure. Job, stage, and total runtimes; queue time versus run time;
   the critical path (the full pipeline graph shows `needs`
   dependencies); pipeline duration charts under CI/CD analytics.
2. Change one bottleneck at a time and re-measure. "Should be faster"
   is a hypothesis, not a result; report absolute numbers, not
   percentages.
3. Fail fast: cheap, failure-prone jobs (lint, static checks) belong
   early and start immediately with `needs: []`.

Anti-patterns seen in optimization MRs:

- Removing `needs:` to "simplify" gives up DAG parallelism.
- A keyless cache (`cache: {paths: [...]}` and no `key:`) is shared by
  every ref and easily poisoned; always set a key.
- Narrowing `rules:` "for performance": rules decide whether a job
  runs, not how fast it runs.
- Dropping `interruptible: true` "to be safe" lets superseded pipelines
  run to completion; keep it in `default:` and opt out per job with
  side effects.
