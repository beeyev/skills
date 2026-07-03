# Readability: names and comments

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/jobs/ (job names, grouping, hidden jobs)
- https://archives.docs.gitlab.com/18.11/ci/variables/ (variable naming constraints)

A pipeline is read far more often than it is written: in MR review, in
the pipeline graph, and in a failed-job page at 2 a.m. Optimize names for
the reader who has never seen the config.

## Job names

Convention: `domain:action` (or `domain:action:variant`), lowercase.
The pipeline graph sorts alphabetically within a stage, so a shared
domain prefix clusters related jobs.

| Bad | Good | Why |
|---|---|---|
| `job2` | `build:docker-image` | States what it produces |
| `tests` | `test:unit` | Distinguishes from `test:e2e`, `test:lint` |
| `deploy` | `deploy:production` | The target is the information |
| `do_stuff_final_v2` | `release:changelog` | Intent, not history |

Rules and conventions:

- Every job name must be unique. Duplicate names in one file: only one
  job survives, silently. Same name across included files: configs merge,
  which is either a deliberate override or a bug.
- Hard constraints: max 255 characters; reserved words (`image`,
  `services`, `stages`, `variables`, `cache`, `include`,
  `before_script`, `after_script`, `pages:deploy`) cannot be job names;
  `true`, `false`, and `nil` work only when quoted.
- Hidden (template) jobs: `.base-<what>` for extendable job bodies,
  `.rules-<when>` for reusable rule sets. The prefix tells the reader
  what kind of fragment they are looking at before they read a line of
  it.
- Parallel/split jobs named `name 1/3`, `name 2/3`, `name 3/3` (also
  `1:3` or `1 3`) are auto-grouped into one collapsed entry in the
  pipeline graph. Use this for fanned-out test suites.
- Manual gate jobs read best as an imperative with the consequence in the
  name: `deploy:production` behind a manual `rules` entry, not `button`
  or `approve`.

## Stage names

- Short lowercase nouns or verbs; the `stages:` list should read as the
  pipeline's story: `build` -> `test` -> `deploy`. Custom
  stages are fine (`lint`, `package`, `release`) when they mark a real
  phase boundary.
- Don't invent a stage per job. Stages exist to express ordering; jobs
  that can run together belong in the same stage (or use `needs:` and
  fewer stages, see `common-patterns.md`).
- If `stages:` is omitted, GitLab provides `.pre`, `build`, `test`,
  `deploy`, `.post`, and a job without `stage:` lands in `test`. Declare
  `stages:` explicitly in anything beyond a toy pipeline; implicit
  placement surprises readers.

## Variable names

- `UPPER_SNAKE_CASE`, like the predefined ones.
- Never define your own variables starting with `CI_`: that namespace
  reads as "provided by GitLab" and lying about that costs debugging
  time. Same for `GITLAB_`.
- Prefix by domain so origin is obvious at the point of use:
  `DOCKER_BUILD_ARGS`, `DEPLOY_TIMEOUT_SECONDS`, `APP_IMAGE_TAG`.
- Encode units in the name when the value is a number:
  `CACHE_TTL_MINUTES`, not `CACHE_TTL`.
- Represent booleans as `"true"`/`"false"`, named as predicates:
  `SKIP_E2E`, `ENABLE_DEBUG_LOGS`. The job shell receives CI/CD variable
  values as strings; keep rule comparisons string-typed
  (`$SKIP_E2E == "true"`).

| Bad | Good |
|---|---|
| `VAR1`, `TMP` | `SENTRY_RELEASE_NAME` |
| `timeout` | `DEPLOY_TIMEOUT_SECONDS` |
| `CI_MY_TOKEN` | `REGISTRY_PUSH_TOKEN` |

## File names

Owned by `pipeline-structure.md` (one domain per file under
`.gitlab/ci/`, suffix `.gitlab-ci.yml`). Short version: the file name is
the domain: `build.gitlab-ci.yml`, `deploy.gitlab-ci.yml`, and shared
fragments in `base.gitlab-ci.yml`.

## Comments

YAML in CI carries decisions that the syntax cannot express. Comment the
*why*; never narrate the *what*.

Comment when:

- A `rules:` clause encodes a policy: why does this job skip tags? Why is
  this allowed to fail?
- A pinned version or image digest has a reason
  (`# 3.12: 3.13 breaks aiohttp, see #1234`).
- A workaround exists for an upstream bug; link the issue so the next
  person can delete the hack when it is fixed.
- Ordering matters invisibly (e.g. an include that must come last because
  it overrides).

Do not comment:

```yaml
# Build the image        <- narrates the obvious
build:docker-image:
  script:
    - docker build .
```

Good example:

```yaml
test:e2e:
  # Flaky upstream driver, retry twice before humans get paged.
  # Remove when https://gitlab.com/example/example/-/issues/42 is fixed.
  retry: 2
  rules:
    # E2E costs ~15 min; run only where the result gates a merge.
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  script:
    - ./scripts/ci/e2e.sh
```

One more rule: when a name needs a comment to be understood, fix the
name, not the comment.
