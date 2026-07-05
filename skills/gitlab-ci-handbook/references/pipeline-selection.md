# Pipeline selection: workflow and job rules

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/yaml/ (keyword reference)
- https://archives.docs.gitlab.com/18.11/ci/yaml/workflow/
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_rules/
- https://archives.docs.gitlab.com/18.11/ci/jobs/job_control/
- https://archives.docs.gitlab.com/18.11/ci/pipelines/merge_request_pipelines/
- https://archives.docs.gitlab.com/18.11/ci/yaml/deprecated_keywords/
- https://docs.gitlab.com/update/deprecations/ (changes after the 18.11 baseline)

Which pipelines run, and which jobs run in them. Applied recipes only;
mechanics of includes/`extends`/`!reference` live in
`pipeline-structure.md`. Examples assume Free tier and Linux runners;
executor requirements are flagged per pattern.

Recipes call `./scripts/ci/<name>.sh` wherever real logic would live.
That is a placeholder, not a mandate: short wiring commands can stay
inline in `script:`. The inline-vs-extracted criteria are in
`bash-in-ci.md`.

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
- To also create a GitLab Release entry (release notes, asset links, the
  Releases page), add the `release:` keyword to the tag job. The job
  environment must provide the release tool (`release-cli`, or `glab` on
  newer runners); `release:` creates the Release object, it does not
  build or attach artifacts by itself.

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
  `rules:changes` section. (`rules:exists` has a separate budget: after
  50,000 file checks, patterned globs are assumed to match instead of
  being evaluated precisely.)
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

## Deprecated selection keywords: `only` / `except`

`only` and `except` are deprecated; use `rules` for anything new. They
keep working for backwards compatibility but are candidates for removal
in a future major milestone, and a job cannot mix them with `rules`
(pipeline creation fails). When making a small change to a pipeline
that still uses them, match the existing style and mention the
deprecation; migrate to `rules` as its own MR, not as a drive-by.
Also deprecated: globally-defined `image`/`services`/`cache`/
`before_script`/`after_script` at the top level. Use `default:` instead.
