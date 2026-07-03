# Development brief for the `gitlab-ci-handbook` skill

> **This file is not skill content.** It is a standing brief for AI agents
> (and humans) working on this skill. When asked to "work on the skill" or
> "improve the skill", read this file first, then act on it. Keep this file
> up to date: record progress in the Status log, amend scope when it
> changes, and note resolved ambiguities so the next session doesn't
> re-derive them.

## Mission

Build not just a skill, but a proper documentation base for GitLab: a
curated, opinionated knowledge base that lets an AI agent write
production-quality GitLab CI/CD pipelines the way a senior platform
engineer would. `SKILL.md` stays a short router; the knowledge lives in
`references/` files loaded on demand.

## Guiding principles

These override everything else in this brief when they conflict:

1. **No content beats wrong content.** A gap is recoverable. The agent
   falls back to fetching docs. Misleading information gets confidently
   applied and breaks pipelines. If a claim can't be verified per the
   rules below, leave it out and log a TODO in the Status log instead of
   writing a plausible-sounding guess. Never pad a section to look
   complete.
2. **Write for longevity; minimize drift surface.** Every volatile fact
   written down is a future maintenance liability. Prefer guidance that
   stays true across GitLab versions (principles, structure, trade-offs)
   over duplicating volatile specifics the official docs already maintain
   (exhaustive keyword options, default values, tier feature matrices).
   Link to those instead. When a volatile fact is genuinely needed inline,
   mark it so maintenance sessions know what to re-check:
   `<!-- volatile: re-verify on version bump -->`. Keep examples minimal:
   fewer moving parts, less to rot.

## Scope

**In scope now:** the five content pillars below plus the cross-cutting
developer-experience reference, all CI/CD-focused.

**Backlog (do not build until pillars are done; expand the `SKILL.md`
description only when the content actually exists):** GitLab API,
permissions model, releases, container registry.

**Out of scope:** runner administration, self-hosting GitLab, server
configuration and operations.

**Default assumptions** (state exceptions explicitly in the content):
guidance targets GitLab Free tier, Linux runners, and bash as the job
shell. Windows/PowerShell runners are out of scope. Beyond that, stay
executor-agnostic; when a pattern requires a paid tier or a specific
executor (e.g. `docker:dind`), flag it inline.

## Baseline and sources

- **GitLab 18 is the minimum supported version.** Everything documented
  here must be valid for GitLab 18+. Do not document syntax or behavior
  removed before 18.
- Flag deprecations with a consistent callout: `> **Deprecated (18.x):**
  ...`; use the same format in every file.
- **Source precedence:** for baseline (18.x) claims, the versioned 18.x
  archive is authoritative: https://archives.docs.gitlab.com/ (pick the
  latest 18.x, e.g. `18.11`, path like
  `/18.11/topics/build_your_application/`); where current docs
  (https://docs.gitlab.com/ci/, keyword reference
  https://docs.gitlab.com/ci/yaml/) diverge from the archive, the archive
  wins. Features added after 18 may be included only with an explicit
  "introduced in X.Y" note; for those, the current docs are the
  authority.
- When sources conflict (archive vs. current, official vs. community),
  record the conflict and the chosen resolution inline as a short callout,
  so future sessions don't re-litigate it.
- Community material such as https://github.com/ondrejsika/gitlab-ci-training
  is **inspiration for pattern ideas only**. Check its recency first, and
  independently re-verify every adopted pattern against official 18.x docs
  before writing it down. Prefer primary documentation over blog posts.
- Training-data memory is not a source. Verify against the docs above; CI
  syntax and defaults change between versions.
- **If official docs cannot be fetched, do not write that section from
  memory.** Record the blocker in the Status log and move to work that can
  be sourced.

### What "verified against GitLab 18.x" means

A claim or example counts as verified only when:

- YAML examples pass validation with credential-free tools by default:
  `gitlab-ci-local` plus `yamllint`; the GitLab CI Lint API/UI is a
  bonus when credentials happen to be available. Verification must never
  block on GitLab auth.
- Bash examples pass `shellcheck` with no errors.
- Keywords and behavior were checked against the 18.x archive keyword
  reference, not just remembered.

Each `references/` file states at the top: `Verified against: GitLab 18.x`
(actual minor) and lists its sources.

## Content pillars

### 1. Pipeline structure: no god YAML

How to structure CI configuration with single-responsibility thinking,
applied pragmatically. The "balanced SOLID" goals matter; do not
force-map OOP vocabulary onto YAML:

- Never one giant `.gitlab-ci.yml`. Extract domain logic into small,
  focused pieces (`include:local`, component/template files, `extends`,
  YAML anchors, `!reference`) and assemble them in the root file.
- Single responsibility per include/job; sensible defaults via `default:`
  and hidden `.base` jobs; composition over copy-paste.
- When decomposition is overkill (tiny repos, one-job pipelines), say so.
  Balance is part of the guidance.
- Recommended file/directory layout for CI config in a repository.

**Done when:** covers include strategies, `extends`/anchors/`!reference`
trade-offs, defaults and hidden jobs, a recommended repo layout, the
when-not-to-decompose guidance, and at least one complete worked example
assembling a pipeline from includes.

### 2. Bash scripts for CI jobs

Basic but solid guidance for shell scripts executed inside GitLab jobs:

- When to keep script lines inline in YAML vs. extract to `scripts/*.sh`
  files in the repo.
- Defensive, fail-fast style: `set -Eeuo pipefail`, explicit error
  handling, quoting variables, and checking required env vars up front,
  without bloating scripts with ceremony that hides the actual logic.
- How GitLab executes `script:` blocks (shell selection, exit-code
  semantics, multiline behavior) so authors understand what they are
  really writing.

**Done when:** inline-vs-extracted rule of thumb, the fail-fast preamble
explained flag by flag, a required-env-var check pattern, and the
script-execution mechanics are covered; all examples pass `shellcheck`.

### 3. Human readability first

Namings, titles, and code must be easy for humans to navigate and read:

- Job, stage, variable, and file names that state intent
  (`build:docker-image`, not `job2`).
- Consistent naming conventions across the pipeline; predictable
  structure so a newcomer can find things.
- Comments where the YAML cannot speak for itself: the *why*, not the
  *what*.

**Done when:** naming conventions for jobs, stages, variables, and files
are documented with good/bad example pairs, plus commenting guidance.

### 4. Informative verbosity

Pipelines are read in the job log far more often than in the YAML.
Document the practice of being verbose where it informs:

- Echo tool versions in use (`node --version`, `terraform version`, …) at
  the start of jobs.
- Echo key variable values and decisions ("Deploying $APP_VERSION to
  $ENVIRONMENT") so developers see what happens under the hood.
- Informative progress messages at meaningful steps, but no noise: log
  what a developer debugging a failed pipeline at 2 a.m. would want to
  know, nothing more. Never echo secrets.

**Done when:** what-to-echo / what-not-to-echo guidance exists with a
reusable snippet (e.g. a version-banner pattern) and the never-echo-secrets
rule with masking notes.

### 5. Common patterns

A catalog of proven, GitLab-18-verified patterns, each with a short
explanation of when to use it and a copy-paste-adaptable example, e.g.:

- `rules:` / `workflow:rules:` recipes (MR pipelines, tag releases,
  scheduled jobs, avoiding duplicate pipelines).
- Caching vs. artifacts: what each is for and how to configure them well.
- Docker image building (including `docker:dind` vs. alternatives).
- Environments and deployments; manual gates; `needs:` for DAG pipelines.
- Reusable templates / CI components; `include` strategies.
- Secrets and variable handling.

**Done when:** at least 8 individual patterns exist (each named recipe
counts as one; e.g. "MR pipelines" and "tag releases" are two), each with
when-to-use guidance and a runnable, verified example. Extend the list as
research uncovers more; each pattern earns its place by being commonly
needed, not by being exotic.

### Cross-cutting: developer experience

Connect the existing structure, readability, logging, and debugging
guidance from the developer's point of view: identifying a pipeline,
understanding a job, finding the failure, retaining useful evidence, and
reproducing the problem locally when possible. Prefer GitLab-native report
and link surfaces over forcing developers to search raw logs.

**Done when:** covers pipeline names, manual-run ergonomics, execution
headers and phase titles, actionable failures, native reports and direct
links, bounded diagnostic artifacts, local reproduction, and a
symptom-to-first-check troubleshooting map without duplicating the
canonical mechanics in other references.

## File layout

Fixed pillar-to-file mapping. Do not rename or reshuffle between
sessions; extend by splitting only when a file outgrows its budget
(~300–400 lines) or covers a genuinely separate topic:

| Pillar | File |
|---|---|
| 1. Pipeline structure | `references/pipeline-structure.md` |
| 2. Bash in CI jobs | `references/bash-in-ci.md` |
| 3. Readability | `references/readability.md` |
| 4. Informative verbosity | `references/informative-logging.md` |
| 5. Common patterns | `references/common-patterns.md` (split by theme when over budget) |
| Cross-cutting developer experience | `references/developer-experience.md` |

Each topic has exactly one canonical file; other files link to it instead
of duplicating. In particular: `pipeline-structure.md` owns the mechanics
of includes, templates/components, and `extends`; `common-patterns.md`
shows applied recipes and links back for the mechanics. `readability.md`
owns naming; `informative-logging.md` owns job-log output mechanics and
safety; `developer-experience.md` owns the developer's debugging path and
which results to surface where, linking to the canonical files for
mechanics.

`SKILL.md` is a router: per topic, a pointer to the right reference file
and when to read it. Target **~50–100 lines**. The 500-line repo cap is a
ceiling, not a goal; a fat router defeats on-demand loading.

## Quality bar

- The same readability rules apply to this documentation: clear titles,
  navigable structure, examples over abstraction.
- Each `references/` file covers one domain and states at the top which
  GitLab version it was verified against plus a list of its sources.
  Inline links go only on load-bearing or volatile claims; both levels
  have a job; don't clutter every sentence with citations.
- Examples must be complete enough to run, not fragments with `...` where
  the hard part goes, and actually validated per the verification section.
- Defensive and fail-fast by default in every example; no bloated scripts.
- Per the guiding principles: no unverified claims (gap + TODO instead),
  volatile facts marked with the `volatile` comment, and links to official
  docs in place of duplicated reference material that would drift.

## Build workflow

1. Read this file and the current `SKILL.md` + `references/` state.
2. Research before writing: fetch the relevant official docs (18.x archive
   first, current docs second) and community sources per the source rules
   above.
3. Write or improve one pillar at a time; keep changes cohesive.
4. Update the `SKILL.md` router only when routing changes: files added,
   renamed, or re-scoped; content-only fixes don't require router edits.
5. Update repo-level artifacts: the README skill table (drop "(WIP)" per
   the readiness rule in the Status log section).
6. Record a Status log entry before ending the session.

## Maintenance workflow

Once pillars are built, sessions asked to "update" or "improve" the skill:

1. Check GitLab release and deprecation pages for changes since the last
   `Verified against` stamps.
2. Re-validate examples (CI Lint / `gitlab-ci-local`, `shellcheck`) and
   grep the `references/` files for `volatile:` markers and re-verify each
   one; bump stamps. Record the check in the Status log **even if nothing
   changed**.
3. When a new GitLab major ships (19+): keep the 18 baseline until the
   user raises it; document 19-only features with an explicit
   "introduced in 19.x" flag; propose a baseline bump to the user rather
   than deciding unilaterally.
4. Spot-check the least-recently-reviewed pillar (per the Status table's
   "Last reviewed" column) against current docs for rot.
5. Fold in any user feedback recorded in the Status log.

## Status log

### Reference files

Status values: `not started` → `researched` → `drafted` → `validated` →
`complete` (meets its pillar's "Done when" criteria). Readiness rule: the
README "(WIP)" marker is dropped once all reference files are
`validated` or better and the `SKILL.md` router body is written.

| File | Status | Verified against | Last reviewed | Open TODOs |
|---|---|---|---|---|
| `references/pipeline-structure.md` | complete | GitLab 18.11 | 2026-07-03 | None |
| `references/bash-in-ci.md` | complete | GitLab 18.11 / Runner 18.11 | 2026-07-03 | None |
| `references/readability.md` | complete | GitLab 18.11 | 2026-07-03 | None |
| `references/informative-logging.md` | complete | GitLab 18.11 | 2026-07-03 | None |
| `references/developer-experience.md` | complete | GitLab 18.11 | 2026-07-03 | None |
| `references/common-patterns.md` | complete | GitLab 18.11 | 2026-07-03 | None |
| `SKILL.md` router body | complete | N/A | 2026-07-03 | None |

### Session log

- **2026-07-03 (8):** Full independent re-verification of all six
  references against v18.11.0-ee raw docs; every load-bearing factual
  claim was re-fetched and checked, including cited archive URLs (all
  resolve). Nearly everything held, including previously suspicious
  claims (the Environments-UI stop action really does require deploy and
  stop jobs in the same `resource_group`; `!` negation really is
  18.11+). Four fixes applied: pattern 14 now flags
  `trigger:strategy: mirror` as introduced in GitLab 18.2 (it was
  presented as baseline but is invalid on 18.0-18.1, violating the
  introduced-in rule); pattern 7's cache-suffix bullet updated for the
  GitLab 18.4.5 change (Maintainer/Owner-started pipelines get the
  `-protected` cache key on any ref) and marked volatile; the
  `include:rules` variable list in `pipeline-structure.md` extended with
  the documented trigger/schedule/manual-run variables and
  `CI_PIPELINE_TRIGGERED` (the old list read as exhaustive and
  contradicted pattern 4's schedule-variable advice); the reserved
  job-name list in `readability.md` completed with `pages:deploy` and
  the quoted-only `true`/`false`/`nil` rule. Prose-only changes, no YAML
  or bash examples touched, so no re-validation was needed.
- **2026-07-03 (7):** Conservative deduplication pass; no factual content
  changed and no examples touched (no re-validation needed). Merged the
  two overlapping `artifacts:paths`-vs-reports bullets in pattern 8;
  removed pattern 13's duplicated GitLab 18.1 protected-variable MR
  conditions in favor of the existing pointer to the canonical
  `informative-logging.md` (drift-surface rule); split the unrelated
  `workflow:rules` instruction out of `SKILL.md` rules-of-engagement
  item 2 into its own item; fixed session-log formatting (entry
  2026-07-02 (4) punctuation, stray blank line). Reviewed all six
  references plus router for removable content; nothing else met the
  100%-certain bar.
- **2026-07-03 (6):** Dual independent review (Fable 5 + Sonnet 5
  subagents) of the developer-experience work in `036e890`, both
  re-verifying claims against v18.11.0-ee raw docs; no factual errors
  found. Applied their findings: reworded the unclosed-section note in
  `developer-experience.md` to guidance only (the renderer failure-mode
  behavior is not documented in 18.11); extended this brief's ownership
  paragraph to cover `developer-experience.md` and un-hardcoded the
  "five reference files" readiness rule; fixed a sentence fragment and a
  redundant `-E` re-explanation in `bash-in-ci.md`; added a forward link
  from informative-logging's context-on-failure bullet; replaced the
  folded-scalar `artifacts:name` with a plain quoted string; made the
  `SKILL.md` router row concrete; dropped the unsupported Runner stamp
  from `developer-experience.md`; corrected the stale "left uncommitted"
  line in entry (5). Validation-tooling note for future sessions:
  `gitlab-ci-local` merges `workflow:rules:variables` but does not
  enforce workflow-level `when: never`, so pipeline-suppression rules
  are never exercised by it; treat such rules as doc-verified only.
  Re-validated the changed YAML example (yamllint 1.38.0,
  gitlab-ci-local 4.73.0); `git diff --check` passes.
- **2026-07-03 (5):** Independent review of the new
  `developer-experience.md` against v18.11.0-ee raw docs; every cited
  claim held up (`workflow:name` variable-forwarding caveat,
  `artifacts:expose_as` limits, `annotations`/`external_link` report,
  JUnit screenshot attachments, the `<file>`-attribute prerequisite for
  copy-failed-tests, `CI_JOB_NAME_SLUG`, inputs descriptions on manual
  runs). One fix: the `workflow:name` example allowed duplicate branch +
  MR pipelines; added the pattern-1 suppression rule. Folded in the rest
  of the DX review: ERR-trap failure-location pattern in `bash-in-ci.md`
  (finally justifies `-E`), line timestamps (GA 18.9) and ANSI-color
  note in `informative-logging.md`, and a compiled-configuration pointer
  (pipeline editor **Full configuration** tab, `gitlab-ci-local --list`,
  CI Lint `merged_yaml`) under the debug-order table. Validation: both
  changed YAML examples pass `yamllint 1.38.0` and
  `gitlab-ci-local 4.73.0` (branch and MR simulation); the ERR-trap
  snippet passes `shellcheck 0.9.0`. (The user later committed this work
  as `036e890`.)
- **2026-07-03 (4):** Added a focused developer-experience reference
  covering pipeline identity, manual-run forms, execution headers and phase
  titles, actionable failures, GitLab-native reports and links, bounded
  diagnostics, local reproduction, and troubleshooting order. Added only
  the new reference plus router and brief bookkeeping; existing pattern and
  pillar content was not reorganized. Verified claims against the GitLab
  18.11 archive. Validation: YAML examples pass `yamllint 1.38.0` and
  `gitlab-ci-local 4.73.0`; `git diff --check` passes.
- **2026-07-03 (3):** Second independent review of the uncommitted changes;
  every doc-sourced claim re-verified against v18.11.0-ee raw docs and all
  example fixtures re-validated (yamllint 1.38.0, gitlab-ci-local 4.73.0).
  Two fixes applied: pattern 17 no longer combines `retry:when` with
  `retry:exit_codes` in one job (the 18.11 docs define them as separate
  filters and do not document combined semantics; example split into two
  jobs, both re-validated), and the multi-project permission bullet in
  pattern 14 now mirrors the docs' wording (trigger job fails when the
  upstream user cannot create pipelines in the downstream project) instead
  of the stronger "runs with the permissions of" paraphrase. Repo-level:
  added `.gitattributes` (`* text=auto eol=lf`) to stop LF-to-CRLF
  conversion warnings on WSL.
- **2026-07-03 (2):** Applied findings from an independent review of the
  uncommitted changes. Removed the invalid top-level input default that
  GitLab forbids alongside `spec:inputs:rules`; distinguished parent-child
  `trigger:include:inputs` from multi-project `trigger:inputs`; added the
  default 1,000 downstream-pipeline hierarchy cap; and added
  `include_jobs: true` to CI Lint job-list guidance. This exposed a
  `gitlab-ci-local 4.73.0` validation gap: it accepted the invalid input
  contract, then treated the corrected rule-level fallback default as
  missing when parsed without a consumer. The corrected contract passes
  in a paired include fixture with explicit inputs. The unauthenticated
  GitLab.com project CI Lint endpoint returned 403, so official keyword
  constraints remain authoritative.
- **2026-07-03:** Added advanced GitLab CI/CD guidance without changing
  the existing five-reference layout or adding files. Expanded
  `pipeline-structure.md` with conditional typed inputs, array access,
  input safety, and component API/versioning rules. Expanded
  `common-patterns.md` with downstream and dynamic child pipelines,
  matrix dependencies, `rules:needs`, auto-cancel, targeted retries,
  allowed exit codes, test sharding, clean checkout strategies, timeout
  budgeting, and compiled-config validation. Added `rules:changes` limits
  and `manual_confirmation`. OIDC is excluded as requested. Research used
  the GitLab 18.11 archive as authority and GitHub MCP searches of active
  configurations from projects including DataDog, Wireshark, Espressif,
  and GitLab itself as pattern candidates. Validation: all 37 YAML blocks
  pass `yamllint 1.38.0`; all 20 complete `common-patterns.md` examples
  and paired typed-input and component fixtures pass
  `gitlab-ci-local 4.73.0`; all five Bash blocks pass `shellcheck 0.9.0`.
  Volatile claims and referenced image tags were rechecked.
  `git diff --check` passes. `common-patterns.md` is 723 lines, above its
  preferred split threshold, retained because the user explicitly required the
  existing layout.
- **2026-07-02 (4):** Technical correctness audit against the GitLab
  18.11 archive. Fixed the duplicate-pipeline recipe so only push
  pipelines are suppressed; corrected artifact report upload and
  instance/keep-latest retention behavior; documented shell-executor
  Docker builds and unordered `resource_group` processing; made the
  staging gate actually manual; made the review-app stop job optional,
  compatible with its repo-based cleanup script, and usable from the
  Environments UI; added the GitLab 18.1 protected-variable MR
  conditions; fixed the unused npm cache path and replaced Alpine Node
  images where the handbook assumes bash. Narrowed the router's
  `workflow:rules` instruction so job-only requests do not overwrite an
  existing workflow. Scoped `set -x` guidance away from secrets and
  corrected the malformed YAML-indicator list. Added no files. Validated
  changed YAML with `gitlab-ci-local 4.73.0` and `yamllint 1.38.0`, and
  shell snippets with `shellcheck 0.9.0`; all pass.
- **2026-07-02 (3):** Dual independent review (Fable 5 + Sonnet 5
  subagents), both spot-checking claims against the v18.11.0-ee raw
  docs. Every checked factual claim held up except two version nits.
  Applied fixes: replaced the invented `CI_DEBUG_VERBOSE` variable with
  a non-`CI_` flag plus an accurate `CI_DEBUG_TRACE` warning (it exposes
  all variables and secrets; Developer+ can view); added the missing
  `volumes = ["/certs/client"]` requirement to the self-managed dind
  recipe; removed a redundant `extends:` from the version-banner example
  (it contradicted the array-replacement lesson); added the previously
  dangling shellcheck lint recipe as pattern 12 (secrets hygiene is now
  13); masking prerequisites now include the
  no-variable-name-collision rule; noted the 18.3 masked-by-default
  change; after_script-on-cancel corrected to GA 17.3; `interruptible`
  bullet added to pattern 1; components mentioned in the router row;
  why-comment on the tag rule in the pipeline-structure worked example.
  Rejected one Fable finding: the error literal `'job' does not exist in
  the pipeline` IS a documented 18.11 heading (yaml/needs.md
  troubleshooting), kept as-is. New examples re-validated
  (gitlab-ci-local + shellcheck). `common-patterns.md` is now 13
  patterns / ~450 lines, past its ~300-400 budget: split by theme when
  content is next added. Backlog idea from review: a `trigger:` /
  child-pipeline pattern (in scope for pillar 5, not built).
- **2026-07-02 (2):** Built all five pillars and the `SKILL.md` router;
  dropped "(WIP)" from the README. Sources: 18.11 archive (latest 18.x as
  of this session), raw doc markdown fetched from `gitlab-org/gitlab` at
  tag `v18.11.0-ee` and `gitlab-org/gitlab-runner` at `v18.11.0` (same
  content as the rendered archive, easier to consume). Validation: every
  YAML example ran through `gitlab-ci-local 4.73.0 --list` (schema +
  rules evaluation, including simulated tag/schedule pipelines) and
  `yamllint 1.38.0`; every bash example through `shellcheck 0.9.0`; all
  pass. Decisions: one `common-patterns.md` file (12 patterns, ~430
  lines) instead of a split; split by theme if it grows further.
  Volatile markers placed on: include limits (pipeline-structure),
  masking prerequisites (informative-logging), docker image major
  (common-patterns). Next actions: none for the pillars; backlog (API,
  permissions, releases, container registry) remains unbuilt. Open
  questions: none.
- **2026-07-02:** Repo skeleton, skill directory, frontmatter, this brief
  (refined through three independent reviews: Fable, Sonnet, Codex).
  Decisions: Linux/bash runners assumed, Windows/PowerShell out of scope;
  validation must be credential-free by default. Next actions: build
  Pillar 1. Open questions: none.
