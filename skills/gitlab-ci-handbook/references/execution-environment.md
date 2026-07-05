# Execution environment: runners, images, services

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/runners/runners_scope/
- https://archives.docs.gitlab.com/18.11/ci/docker/using_docker_images/
- https://archives.docs.gitlab.com/18.11/ci/services/
- https://archives.docs.gitlab.com/18.11/ci/docker/using_docker_build/
- https://archives.docs.gitlab.com/18.11/ci/environments/
- https://archives.docs.gitlab.com/18.11/ci/resource_groups/
- https://archives.docs.gitlab.com/18.11/ci/yaml/#pages
- https://archives.docs.gitlab.com/18.11/user/project/pages/
- https://docs.docker.com/engine/release-notes/29/ (Docker release used in the example)

Where a job runs and what surrounds it: runner selection, the job
image, sidecar services, Docker builds, environments, and Pages.
Examples assume Free tier and Linux runners; executor requirements are
flagged per pattern.

Recipes call `./scripts/ci/<name>.sh` wherever real logic would live.
That is a placeholder, not a mandate: short wiring commands can stay
inline in `script:`. The inline-vs-extracted criteria are in
`bash-in-ci.md`.

## Runners and tags

A job runs on whichever runner picks it up; the YAML controls only the
matching, so know the matching rules:

- Runner scopes: instance runners (available to every project), group
  runners (projects in the group), project runners. Instance runners
  schedule with a fair-usage queue across projects; group and project
  runners are first-in-first-out.
- `tags:` on a job selects runners that carry all of those tags. A job
  without tags runs only on runners configured to run untagged jobs.
  When nothing matches, the job sits pending until timeout with a
  "job is stuck" hint; that is a configuration problem, not a load
  problem (see `debugging.md`).
- A runner can be restricted to protected branches and tags. Deploy
  jobs bound to such runners will not schedule from unprotected refs;
  that is the point, not a bug.
- Executors change what your script sees. This handbook assumes the
  Docker executor on Linux. The shell executor runs jobs as the
  runner's OS user through `bash --login` (dotfiles run, state can
  persist between jobs); Kubernetes behaves like Docker for `script:`
  purposes. Shell-level differences: `bash-in-ci.md`. Isolation and
  trust implications: `security.md`.

## Job images

- Pin concrete tags. `image: node` or any `:latest` changes on the
  upstream's schedule, not yours. A specific minor tag
  (`node:22-bookworm-slim`) keeps security patches flowing; an
  `@sha256:` digest is immutable, for when reproducibility outranks
  patch flow.
- Put the shared image in `default:image:`; override per job only when
  a job needs different tools (linter image, dind, cloud CLI).
- Alpine versus Debian slim: Alpine images are several times smaller
  but use musl libc, so toolchains that fetch glibc-linked binaries
  fail with linker errors, and busybox `sh` is not bash
  (`bash-in-ci.md`). Default to slim when in doubt; move to Alpine when
  pull size is a measured cost, not a habit.
- When every job's `before_script` installs the same system packages,
  build a project CI image once, push it to the project's registry, and
  set it as `default:image:`. Installing per job pays the cost on every
  run and every retry.
- Private registries: set `DOCKER_AUTH_CONFIG` as a CI/CD variable
  holding the Docker `auths` JSON (base64-encoded `username:password`);
  the runner uses it when pulling `image:` and `services:`. Images in
  a GitLab project's own registry pull with the job's default
  credentials, but the user who triggered the job needs at least
  Developer role in the project that hosts the image. Pulling another
  project's registry image additionally requires that project to allow
  this one on its job token allowlist, which is closed by default
  (`security.md`).

## Sidecar services

**When:** the job needs a database, message broker, or other server
alongside it; integration tests are the common case.

```yaml
test:integration:
  image: node:22-bookworm-slim
  services:
    - name: postgres:16
      alias: db
  variables:
    POSTGRES_DB: app_test
    POSTGRES_USER: runner
    POSTGRES_PASSWORD: insecure-test-only
    DATABASE_URL: "postgresql://runner:insecure-test-only@db:5432/app_test"
  script:
    - ./scripts/ci/wait-for-db.sh
    - npm run test:integration
```

- A service is a separate container on a network shared with the job,
  reachable at its alias (without `alias:`, the hostname derives from
  the image name). It adds nothing to the job container's filesystem:
  `services: [docker:dind]` provides a daemon to talk to, not a
  `docker` CLI.
- Services start before `before_script`. The runner waits for exposed
  ports, but "probably didn't start properly" is only a warning; add
  your own readiness wait when the first command needs the service.
- Job-level `variables:` are exposed to service containers (that is how
  `POSTGRES_*` above configures the service). Use per-service
  `variables:` on the service entry when two services would clash over
  a name.
- `CI_DEBUG_SERVICES: "true"` streams service container logs into the
  job log, prefixed per alias. First stop when a service misbehaves.
  Like all debug logging it can leak service-side secrets; see
  `informative-logging.md`.
- Two services from the same image need distinct `alias:` values.
  Services stop when the job ends; nothing persists between jobs.

## Building Docker images

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

## Environments, review apps, and deploy serialization

**When:** any job that deploys. Declaring `environment` gives a deploy
history, the environments UI, environment-scoped variables, and
stop/cleanup hooks.

Static environment with a manual gate (see the manual-gates pattern in `pipeline-selection.md`):

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
- The stop job also fires after the source branch is deleted, when
  checkout can fail; the docs recommend `GIT_STRATEGY: none` or `empty`
  for stop jobs. That requires the cleanup command to be available in
  the job image or fetched another way. The example checks out the repo
  because its cleanup script lives in `scripts/ci/`; either accept that
  the stop job can fail on a deleted branch, or bake the cleanup logic
  into the job image and set `GIT_STRATEGY: empty`.
- `resource_group: <name>` serializes jobs across pipelines: two
  deploys to the same target never overlap. The default process mode is
  `unordered`, so execution order is not guaranteed. Configure
  `oldest_first`, `newest_first`, or `newest_ready_first` through the
  resource groups API when ordering matters.

## GitLab Pages

**When:** publishing static content from the pipeline: docs, coverage
reports, demo builds.

```yaml
pages:docs:
  stage: deploy
  pages: true
  script:
    - ./scripts/ci/build-docs.sh
  artifacts:
    paths:
      - public
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

- `pages: true` (or a `pages:` map) marks the deploying job. The magic
  job name `pages` is the deprecated legacy trigger; with the keyword,
  any job name works.
- Content deploys from `public/` by default; override with
  `pages.publish`. The publish path is appended to `artifacts:paths`
  automatically (GitLab 17.10+).
- `artifacts:expire_in` does not expire a Pages deployment; use
  `pages.expire_in` for that.
- Parallel deployments (`pages.path_prefix`, e.g. per-MR documentation
  previews) publish multiple versions side by side; when two jobs share
  a prefix, the last one to finish wins. Check tier and deployment
  limits before building a workflow on them.
  <!-- volatile: re-verify tier/limits on version bump -->
