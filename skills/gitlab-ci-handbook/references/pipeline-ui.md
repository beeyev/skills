# Pipeline UI: make large graphs navigable

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/pipelines/ (pipeline detail, mini, and downstream graphs)
- https://archives.docs.gitlab.com/18.11/ci/jobs/ (job order and grouped jobs)
- https://archives.docs.gitlab.com/18.11/ci/pipeline_editor/ (configuration visualization)
- https://archives.docs.gitlab.com/18.11/ci/pipelines/downstream_pipelines/ (downstream cards and child reports)
- https://archives.docs.gitlab.com/18.11/ci/pipelines/pipeline_architectures/ (architecture choices)

Use only functionality built into GitLab. The goal is not the smallest
possible picture. The graph must preserve execution semantics, expose
failures, and let a reader find the relevant domain quickly.

## Understand which view hurts

GitLab renders the same pipeline differently across its UI. Identify the
problematic view before changing YAML:

| View | Layout | What makes it large |
|---|---|---|
| Pipeline list, commit, and MR mini graph | Always grouped by stage | Many stages and downstream pipelines |
| Pipeline details, Stage | One column per stage, jobs alphabetical within it | Many stages or many jobs in one stage |
| Pipeline details, Job dependencies | Columns derived from `needs`; dependency lines optional | Many jobs, dense dependencies, large fan-out or fan-in |
| Downstream pipeline cards | Cards to the right of the parent graph | Many child or multi-project pipelines |
| Pipeline Editor, Visualize | Static stages, jobs, and `needs` lines | The compiled configuration itself |

Mini graphs expand a stage into its jobs but do not collapse similar job
groups. The regular pipeline graph can collapse specially named similar
jobs. A change that helps one view can leave another unchanged.

## Choose the smallest semantic change

| Symptom | Prefer | Avoid |
|---|---|---|
| Mini graph is too wide | Remove stages that do not express a real ordering boundary | Merging stages when stage barriers are required for correctness |
| One stage is too tall | Collapse homogeneous shards with GitLab's grouped-job naming | Grouping jobs that have different purposes or failure actions |
| Dependency view is a hairball | Turn off Show dependencies; hover the job whose ancestry matters | Removing real `needs` or adding fake `needs` to improve layout |
| Root graph mixes independent domains | Static parent-child pipelines by domain | Child pipelines whose only purpose is cosmetic grouping |
| Unchanged monorepo domains appear | Conditional trigger jobs or includes | Skipping required checks merely to reduce node count |
| A matrix dominates the graph | Keep only supported combinations; split independently owned domains | Reducing required platform coverage for presentation |
| Pipeline list entries are ambiguous | `workflow:name` from `developer-experience.md` | Encoding information GitLab already displays |

Do not use a universal job or stage threshold. Viewport size, job-name
length, fan-out, and dependency density matter more than one count. Treat
repeated scrolling, truncated names, slow graph loading, and inability to
find the failing domain as evidence that structure needs attention.

## 1. Keep stages coarse

Stages serve two purposes: execution barriers and UI grouping. Use a stage
only when its jobs form a recognizable phase or later work must wait for the
whole phase. A stage per job makes the full graph wide and makes every mini
graph longer.

Use `needs` for dependencies between individual jobs while retaining coarse
stages such as `build`, `test`, and `deploy`. In Pipeline details:

- Stage view tells the high-level story.
- Job dependencies view shows actual DAG ordering. It appears once jobs
  are configured with `needs`.
- Show dependencies draws all edges. Disable it on a dense graph and hover
  one job to highlight only its prerequisite chain.

Never change `needs` only to rearrange the graph. It controls scheduling and
artifact download, so a visual cleanup can otherwise change correctness or
duration. DAG constraints are in `data-flow.md`.

## 2. Collapse homogeneous parallel jobs

GitLab collapses similar jobs in the regular pipeline graph when names end
with an index and total separated by `/`, `:`, `\`, or a space:

```yaml
test:unit 1/3:
  stage: test
  script:
    - ./scripts/ci/test-unit.sh 1 3

test:unit 2/3:
  stage: test
  script:
    - ./scripts/ci/test-unit.sh 2 3

test:unit 3/3:
  stage: test
  script:
    - ./scripts/ci/test-unit.sh 3 3
```

`parallel: 3` produces the same indexed naming shape automatically and is
preferable when every shard has the same configuration. Base-name style
lives in `readability.md`. The test runner must
partition work correctly; grouping does not justify duplicating the full
test suite.

Limits:

- Grouping works in regular pipeline graphs, not mini graphs.
- The group hides repetition, not complexity. Every job still exists, runs,
  consumes a runner, and contributes status.
- Use it only when one failure response applies to every member. Distinct
  jobs need distinct names.

## 3. Use parent-child pipelines for real domains

Parent-child pipelines are GitLab's native structural boundary for a large
same-project pipeline. Use them when jobs form independently understandable
domains such as frontend, backend, mobile, documentation, or infrastructure.
The parent graph shows trigger jobs and downstream cards instead of every
child job. A reader can expand one downstream card at a time.

Use the static child pattern in `orchestration.md` by default. Add dynamic
generation only when topology genuinely depends on generated data. Pair each
trigger with `strategy: mirror` so its status matches the child pipeline.

Accept the costs before splitting:

- The reader navigates between parent and child graphs.
- Artifact and variable flow crosses a pipeline boundary.
- With `mirror` or `depend`, the trigger job waits for the child, so its
  stage stays occupied until the child completes; a default trigger
  succeeds as soon as the child is created. A cosmetic split can
  therefore change stage timing and total pipeline duration.
- Child jobs see `CI_PIPELINE_SOURCE=parent_pipeline`.
- Child pipelines are not separate entries in the project's pipeline list.
- Trigger and child `rules` must preserve MR, branch, tag, and schedule
  behavior.

GitLab 18.6+ can surface JUnit, code quality, Terraform, and metrics reports
from child pipelines in the parent's MR widgets. Security report
support starts in GitLab 18.9. The report-producing child must use
`strategy: mirror` or `strategy: depend` so the parent waits for it. Prefer
`mirror` on GitLab 18.2+. <!-- volatile: re-verify on version bump -->

Do not create a child for a handful of closely coupled jobs. The extra graph
and data-flow boundary is worse than scrolling.

## 4. Remove jobs only when they are irrelevant

The best node is a job that has no reason to exist in that pipeline. In a
monorepo, use `rules:changes` on domain trigger jobs or `include:rules` for a
fixed set of domains. This reduces work and UI size together.

Selection is a correctness decision, not a display preference. Scheduled,
tag, manual, and first-branch pipelines have different change-comparison
behavior. Apply the guarded patterns from `pipeline-selection.md`; do not add
a bare `changes` rule while refactoring the graph.

Avoid combining unrelated commands into one job to reduce node count. That
removes parallelism, makes retries coarser, hides which check failed, and can
increase the critical path.

## 5. Keep matrix fan-out bounded

Every `parallel:matrix` permutation is a real job and appears in the graph.
Use matrices for a bounded, intentional support set. Keep matrix values short
because GitLab includes them in generated job names.

When a matrix becomes difficult to navigate, first remove unsupported or
duplicate combinations. If separate teams own coherent subsets, place those
subsets in child pipelines. Do not split one tightly coupled matrix into
children solely to change its rendering.

When each consumer needs only its matching producer, use
`needs:parallel:matrix`; that removes fan-in from the dependency view and
scopes artifact downloads. Keep full fan-in when the consumer genuinely
requires every permutation; narrowing `needs` to thin the graph changes
scheduling and artifacts. Matrix mechanics are in `orchestration.md`.

## 6. Surface results outside the graph

A large graph should not be the only way to understand a pipeline:

- Use `workflow:name` to distinguish MR, release, scheduled, and deployment
  pipelines in the pipeline list.
- Publish JUnit and other supported report artifacts so failures appear in
  MR widgets and pipeline tabs.
- Use environments for deployment state instead of encoding the entire
  deployment history in job names.
- Use actionable job names. Full graphs sort alphabetically; mini graphs sort
  by status severity first, then alphabetically.
- Use the project's Build > Jobs page with its job-name filter
  (GitLab 18.5+) to find one job across pipelines without scanning any
  graph.

These features do not compact the graph, but they reduce how often users must
scan it. Details are in `developer-experience.md` and `readability.md`.

## Native review workflow

Before changing YAML:

1. Record the problematic surface: mini graph, Stage view, Job dependencies,
   or downstream cards.
2. Count stages, jobs per stage, grouped shards, matrix permutations, and
   downstream pipelines. Identify whether width, height, edge density, or
   mixed ownership is the actual problem.
3. Choose the smallest pattern above that matches that problem.

After changing YAML:

1. Open Pipeline Editor > Visualize to inspect all stages, jobs, and `needs`
   lines. Also inspect Full configuration when includes or inheritance changed.
2. Run CI Lint simulation (`dry_run` with `ref`) for the relevant branch
   or tag. Lint cannot simulate MR or schedule pipeline sources; only a
   real pipeline proves their job selection. The Visualize tab alone does
   not prove conditional job selection either.
3. Compare the simulated job list and compiled configuration with the original
   to confirm that only intended topology changed.
4. Run a real pipeline for every affected source: a branch push at
   minimum; an MR pipeline when MR rules, widgets, or child reports
   changed; tag or schedule pipelines when their selection changed. A
   branch pipeline proves nothing about MR-only rules or MR report
   widgets. Inspect the mini graph, both grouping modes in Pipeline
   details, downstream cards, and MR reports in the MR pipeline.
5. Report any semantic changes separately from presentation improvements.

Validation levels and their limits are in `orchestration.md`.
