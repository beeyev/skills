# Pipeline structure: no god YAML

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/yaml/ (keyword reference: `include`, `default`, `stages`, `extends`, `spec:inputs`)
- https://archives.docs.gitlab.com/18.11/ci/yaml/includes/ (merge semantics)
- https://archives.docs.gitlab.com/18.11/ci/yaml/yaml_optimization/ (anchors, `extends`, `!reference`)
- https://archives.docs.gitlab.com/18.11/ci/inputs/ (typed configuration contracts)
- https://archives.docs.gitlab.com/18.11/ci/components/ (CI/CD components)
- https://www.schemastore.org/api/json/catalog.json (editor schema fileMatch for `*.gitlab-ci.yml`)

## Principles

- One `.gitlab-ci.yml` that holds everything becomes unreviewable and
  unownable. Split CI config by domain (build, test, deploy, release), one
  file per domain, assembled in the root file with `include:local`.
- The root file owns pipeline-wide concerns only: `workflow:rules`,
  `stages`, `default`, global `variables`, and the `include` list. Job
  definitions live in the included files.
- One include = one responsibility. If a file needs "and" to describe it,
  split it.
- Composition over copy-paste: shared setup goes into hidden `.base` jobs
  extended by real jobs, or into `default:`.

### When not to decompose

Decomposition adds indirection: every include is another file the reader
must open. Keep a single `.gitlab-ci.yml` when the pipeline is small
(roughly: under ~100 lines or ~5 jobs, one domain). Introduce
`.gitlab/ci/` when the file starts mixing domains or when two jobs start
copy-pasting each other's config, not before. A one-job lint pipeline in
one file is correct, not lazy.

## Recommended repository layout

```
.gitlab-ci.yml              # root: workflow, stages, default, includes
.gitlab/
  ci/
    base.gitlab-ci.yml      # shared hidden jobs (.base-*), reusable rules
    build.gitlab-ci.yml
    test.gitlab-ci.yml
    deploy.gitlab-ci.yml
scripts/
  ci/                       # shell scripts extracted from script: blocks
    deploy.sh               # only when extraction is warranted; see bash-in-ci.md
```

Conventions:

- `.gitlab/ci/` keeps CI config out of the repo root; the directory is
  already conventional for GitLab metadata (issue templates, etc.).
- `.gitlab-ci/` is a common alternative you will meet in existing repos.
  GitLab reads any `include:local` path, so it works identically. Prefer
  `.gitlab/ci/` for new setups: it reuses the existing `.gitlab/` metadata
  directory instead of adding another dotdir to the repo root, and it is
  what `gitlab-org/gitlab` itself uses. If a repo already uses
  `.gitlab-ci/` (or any other location), keep it; in-repo consistency
  beats either convention, and a rename MR that touches every include
  buys nothing.
- Name included files after their domain, suffix `.gitlab-ci.yml`.
  GitLab itself accepts any `.yml` path; the suffix exists for tooling
  and readers. SchemaStore maps the GitLab CI schema to
  `**/*.gitlab-ci.yml`, so editors (VS Code YAML extension, JetBrains,
  yamlls) validate and autocomplete these files with zero setup, while a
  plain `build.yml` gets no schema or a wrong guess. And editor tabs,
  grep hits, and MR diff headers show only the basename:
  `build.gitlab-ci.yml` is unambiguous where `build.yml` could be
  anything.
- Shell logic longer than a few lines goes to `scripts/ci/` (see
  `bash-in-ci.md`).

### When a domain file grows

The splitting rules apply recursively. Signals that a
`.gitlab/ci/*.gitlab-ci.yml` file now mixes sub-domains:

- Describing it needs "and" again ("tests and coverage and reports").
- You scroll to find a job.
- Unrelated jobs keep changing in the same MR.

Fix: split by sub-domain with the domain as prefix, flat in the same
directory: `test-unit.gitlab-ci.yml`, `test-e2e.gitlab-ci.yml`. The
shared prefix keeps siblings adjacent in file listings, same reason job
names use `domain:action`.

Flat vs nested (`.gitlab/ci/test/unit.gitlab-ci.yml`): stay flat until
the file count hurts (roughly 10+ files). Nesting invites wildcard
includes (`include: local: .gitlab/ci/test/*.gitlab-ci.yml`), and a
wildcard means any new file silently joins the pipeline; an explicit
`include` list is one more line per file but shows the reviewer exactly
what the pipeline is made of.

`base.gitlab-ci.yml` grows too. Split it by fragment kind, not domain:
`rules.gitlab-ci.yml` for `.rules-*` sets, `base.gitlab-ci.yml` for
`.base-*` job bodies. Domain-specific shared config belongs in the
domain file itself, not in base.

Include order: list shared files (`base`, `rules`) first, domain files
after. Order only matters when two files define the same job name or
when one include deliberately overrides another (last one wins); if you
depend on that, say so in a comment next to the include.

## Include strategies

Includes are always evaluated first and merged before the main file,
regardless of where the `include` keyword sits. Resolution of all files
must finish in 30 seconds. Default limit is 150 files per pipeline,
nested includes count. <!-- volatile: re-verify on version bump -->

| Type | Use for | Notes |
|---|---|---|
| `include:local` | Same-repo decomposition. The default choice. | Same branch as the root file. Wildcards: `configs/*.yml` (one level), `configs/**.yml` (recursive). |
| `include:project` | Sharing config across repos on the same instance | Pin `ref:` to a tag or full 40-char SHA; an unpinned include is a third-party dependency that changes under you with no pipeline or notification. User running the pipeline must have access to that project. |
| `include:component` | Versioned, parameterized, reusable units (CI/CD catalog) | `component: $CI_SERVER_FQDN/<project-path>/<name>@<version>`. Prefer over `include:project` for anything published for reuse. |
| `include:template` | GitLab built-in templates (`Auto-DevOps.gitlab-ci.yml`, ...) | Check the template source before relying on it; not all are meant for `include`. |
| `include:remote` | Public URL, last resort | No auth support; nested includes resolve as a public user. Add `integrity:` (SHA256, GitLab 17.9+) to pin content. |

Conditional includes: `include:rules` supports `rules:if`,
`rules:exists`, and `rules:changes` only, and only a restricted variable
set (project/group/instance variables, `CI_PROJECT_*` predefined
variables, `CI_PIPELINE_SOURCE`, `CI_PIPELINE_TRIGGERED`,
`CI_COMMIT_REF_NAME`, and variables from triggers, schedules, and manual
runs; nothing defined in the YAML itself):

```yaml
include:
  - local: .gitlab/ci/docker.gitlab-ci.yml
    rules:
      - exists:
          - Dockerfile
```

### Merge semantics you must know

- Deep merge of hash maps; the main `.gitlab-ci.yml` wins over includes;
  among includes, the last one listed wins.
- Arrays never merge, they are replaced whole. Overriding a job's `script`
  or `rules` from another file replaces the entire list. To compose
  arrays, use `!reference` or YAML anchors.
- Variables are evaluated after all files merge, so a job from an included
  file can consume a variable value set in the root file.
- A job with the same name in two files is one merged job. When including
  third-party config, treat job-name collisions as bugs unless the
  override is deliberate.

## Reuse mechanics: `extends` vs anchors vs `!reference`

| Mechanism | Works across includes | Merge behavior | Best for |
|---|---|---|---|
| `extends` | Yes | Reverse deep merge; hashes merge, arrays replaced by the last level | Job inheritance from hidden `.base` jobs. Default choice. |
| YAML anchors (`&`, `*`, `<<:`) | No, single file only | Textual: alias inserts content; `*alias` inside `script` splices an array | Composing script line lists inside one file |
| `!reference [.job, keyword]` | Yes | Inserts the selected keyword's value; multiple tags can build up an array for `rules`, `script`, and `stages` (other keywords reject it) | Composing `rules` sets and script fragments across files |

Guidance:

- Default to `extends` from hidden jobs. Keep inheritance shallow: more
  than three levels and nobody can predict the merged result (GitLab
  allows up to 11).
- `extends` replaces arrays, so a child that sets `rules` throws away the
  parent's `rules` entirely. That is the top source of "my base rules
  disappeared" bugs. Compose rules with `!reference` instead:

  ```yaml
  .rules-mr:
    rules:
      - if: $CI_PIPELINE_SOURCE == "merge_request_event"

  .rules-default-branch:
    rules:
      - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

  check:lint:
    rules:
      - !reference [.rules-mr, rules]
      - !reference [.rules-default-branch, rules]
    script:
      - ./scripts/ci/lint.sh
  ```

- To remove an inherited key in a child job, set it to `null`
  (`variables: null`).
- Anchors are fine for splicing shared script lines within one file; the
  moment reuse crosses a file boundary, switch to `!reference`.
- `!reference` nests up to 10 levels deep inside `script`/`before_script`/
  `after_script`; in practice stop at 2 or the log becomes untraceable to
  its source.

## `default:` and hidden jobs

- `default:` sets fallback values (image, retry, tags, before_script,
  cache, ...) copied into every job that does not define the keyword
  itself. It does not merge with job config: a job defining the keyword
  ignores the default for that keyword completely. Supported keyword
  list: https://archives.docs.gitlab.com/18.11/ci/yaml/#default
- Use `default:` for what is genuinely pipeline-wide (image, retry
  policy, `interruptible`). Use hidden `.base` jobs + `extends` for what
  only a family of jobs shares. Don't put a `before_script` in `default:`
  that half the jobs must then override; that inverts the readability
  win.
- Hidden jobs start with `.` and are skipped by the pipeline. They need no
  `script:`, only valid YAML. Keep shared hidden jobs in one
  `base.gitlab-ci.yml` include so there is one place to look.
- A job can opt out of defaults and global variables with
  `inherit:default: false` / `inherit:variables: false`, all-or-nothing or
  as a list of names to keep.

## Parameterized includes: `spec:inputs`

For includes meant for reuse (components, cross-project templates),
parameterize with inputs, not variables. Inputs are typed, validated at
pipeline creation, and interpolated before merge, so the consumer sees
errors immediately instead of a broken job at runtime.

Reusable file (`templates/scan.yml`):

```yaml
spec:
  inputs:
    environment:
      options: ['staging', 'production']
    scan-timeout:
      type: number
      default: 60
---
scan:$[[ inputs.environment ]]:
  script:
    - ./scan --env "$[[ inputs.environment ]]" --timeout "$[[ inputs.scan-timeout ]]"
```

Consumer:

```yaml
include:
  - local: templates/scan.yml
    inputs:
      environment: production
```

Notes:

- Inputs without `default:` are mandatory; give defaults unless you want
  every consumer to pass the value.
- `!reference` is evaluated before input interpolation, so you cannot put
  inputs inside `!reference` targets.
- For same-repo domain includes, inputs are usually overkill; plain
  variables and hidden jobs are enough.

### Advanced input contracts

Inputs configure the compiled pipeline; variables configure job runtime.
Do not use one as a vague substitute for the other.

- Use `type`, `options`, and `regex` to reject invalid configuration when
  the pipeline is created. A shell check several minutes later gives worse
  feedback and may run after other jobs consume resources.
- Pipeline-level inputs in the root `.gitlab-ci.yml` should have defaults.
  MR, branch, tag, and other automatic pipelines cannot ask for a required
  value, so a missing default can prevent pipeline creation.
- A pipeline can define at most 20 pipeline inputs in GitLab 18.11.
  <!-- volatile: re-verify on version bump --> Too many inputs usually mean
  the template owns too many responsibilities.
- `spec:inputs:rules` (GitLab 18.7+) selects allowed values and defaults
  from other inputs. Rules are first-match-wins. Include a fallback when
  preceding conditions are not exhaustive.
- Array element access with `$[[ inputs.items[0] ]]` is available in
  GitLab 18.11. Prefer named scalar inputs when positions are not a stable
  public contract.
- `posix_escape` is a formatting helper, not a security control. Do not
  interpolate untrusted strings into shell code. Constrain values with a
  boolean or number type, `options`, or `regex`, then pass them as ordinary
  command arguments.

Conditional, typed deployment contract:

```yaml
spec:
  inputs:
    environment:
      options: [staging, production]
      default: staging
    deployment-strategy:
      rules:
        - if: $[[ inputs.environment ]] == 'production'
          options: [canary, blue-green]
          default: canary
        - options: [rolling, recreate]
          default: rolling
---
deploy:$[[ inputs.environment ]]:
  script:
    - ./scripts/ci/deploy.sh "$[[ inputs.environment ]]" "$[[ inputs.deployment-strategy ]]"
```

### CI/CD components are public APIs

Treat a reusable component like a versioned library, even when internal:

- Keep it self-contained. Components cannot use `spec:include`; define
  their inputs in the component file.
- Avoid global keywords such as `default`, `stages`, and `workflow`.
  They merge into and can change the consumer's entire pipeline. Put
  configuration on the component jobs or uniquely named hidden jobs.
- Parameterize job names or a job-name prefix. Fixed names collide when a
  component is included twice or when a consumer already has that job.
- Use inputs for compile-time configuration. Keep predefined variables for
  GitLab context and variables for runtime state or secrets.
- Pin third-party components to a full commit SHA when immutability matters,
  or an exact trusted release. Partial semantic versions opt in to updates
  but are not reproducible. Avoid branches and `~latest` in production.
- Test the component by including it from
  `$CI_SERVER_FQDN/$CI_PROJECT_PATH/<name>@$CI_COMMIT_SHA` in its own
  pipeline before publishing a semantic release.

Component template `templates/scan.yml`:

```yaml
spec:
  inputs:
    job-prefix:
      regex: ^[a-z0-9-]+$
    stage:
      default: test
---
"$[[ inputs.job-prefix ]]:scan":
  stage: $[[ inputs.stage ]]
  script:
    - ./scripts/ci/scan.sh
```

Same-repository consumer while developing the template:

```yaml
include:
  - local: templates/scan.yml
    inputs:
      job-prefix: security
      stage: test
```

After publishing, replace `local` with the versioned component address,
for example
`component: $CI_SERVER_FQDN/platform/ci-components/scan@1.4.2`.

### Consuming components someone else wrote

For standard cross-cutting tasks (SAST, secret detection, dependency
and container scanning, IaC pipelines), check the CI/CD Catalog and
GitLab's built-in templates before hand-rolling a job: a maintained
component keeps up with report formats and scanner changes that a
custom job would re-implement and then neglect. Do not embed a catalog
inventory in project docs; it goes stale. Search the catalog when the
need arises.

Choosing one:

- The catalog marks verification levels: **GitLab-maintained**,
  **GitLab Partner** (provided as-is, no warranty), **Verified creator**
  (creator verified by GitLab or an instance administrator), and
  unverified.
  Prefer GitLab-maintained for security-relevant jobs; treat the rest as
  third-party code.
- Audit before adopting, whatever the badge: read the component source,
  check what its jobs do with the credentials and variables they
  receive, check for global keywords (`default`, `stages`, `workflow`)
  that would rewrite your pipeline, and check its job names against
  yours (same-name jobs merge; see merge semantics above).
- Pin per the list above: full commit SHA when immutability matters,
  exact release tag otherwise; never `~latest` in production. Re-audit
  on version bumps; a component update is a dependency update.
- Security framing (what a hostile include can reach): `security.md`.

## Worked example: assembling a pipeline from includes

Root `.gitlab-ci.yml` (pipeline-wide concerns only):

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

stages:
  - build
  - test

default:
  interruptible: true

include:
  - local: .gitlab/ci/base.gitlab-ci.yml
  - local: .gitlab/ci/build.gitlab-ci.yml
  - local: .gitlab/ci/test.gitlab-ci.yml
```

`.gitlab/ci/base.gitlab-ci.yml` (shared hidden jobs and rule sets):

```yaml
.rules-mr:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

.rules-default-branch:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

.base-node:
  image: node:22-bookworm-slim
  before_script:
    - node --version
    - npm ci --cache .npm --prefer-offline --no-audit
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - .npm/
```

`.gitlab/ci/build.gitlab-ci.yml`:

```yaml
build:app:
  extends: .base-node
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  rules:
    - !reference [.rules-mr, rules]
    - !reference [.rules-default-branch, rules]
```

`.gitlab/ci/test.gitlab-ci.yml`:

```yaml
test:unit:
  extends: .base-node
  stage: test
  needs: ["build:app"]
  script:
    - npm run test:ci
  artifacts:
    when: always
    reports:
      junit: junit.xml
  rules:
    - !reference [.rules-mr, rules]
    - !reference [.rules-default-branch, rules]
```

What this demonstrates: root file holds `workflow`/`stages`/`default`/
`include` and nothing else; domain files hold jobs; shared config lives
in `.base-node` and named rule sets; rules compose via `!reference`
because `extends` would replace the array; `needs` links stages into a
DAG (see `data-flow.md`).

## Refactoring an existing god YAML

Splitting a working pipeline is safe only if each step provably changes
nothing. Separate moves from rewrites:

1. **Move verbatim first.** Create `.gitlab/ci/`, cut one domain's jobs
   into a file unchanged, add the `include:local`. One domain per MR;
   a reviewer can verify "moved, not edited" at a glance.
2. **Verify the compiled result is identical** after every step. Diff
   the merged config, not the source files: pipeline editor "Full
   configuration" tab, `gitlab-ci-local --preview` (merged YAML on
   stdout), or the project CI Lint API (`merged_yaml` field in the
   response). Save the merged YAML before starting; diff against it
   after each move.
3. **Dedupe last.** Extracting `.base-*` / `.rules-*` from repeated
   config changes merge behavior and belongs in its own MR, after the
   moves are done and verified.

One thing cannot move verbatim: YAML anchors do not cross file
boundaries. Jobs wired together with `&`/`*`/`<<:` must be converted to
`extends` or `!reference` as part of the move. Do the conversion inside
the god file first (still one file, anchors still legal, compiled diff
must stay empty), then move the jobs.
