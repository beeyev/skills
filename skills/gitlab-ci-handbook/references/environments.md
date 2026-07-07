# Environments and deployments

Verified against: GitLab 18.11
Sources:
- https://archives.docs.gitlab.com/18.11/ci/environments/
- https://archives.docs.gitlab.com/18.11/ci/environments/deployments/
- https://archives.docs.gitlab.com/18.11/ci/environments/deployment_safety/
- https://archives.docs.gitlab.com/18.11/ci/environments/protected_environments/
- https://archives.docs.gitlab.com/18.11/ci/environments/deployment_approvals/
- https://archives.docs.gitlab.com/18.11/ci/review_apps/
- https://archives.docs.gitlab.com/18.11/ci/resource_groups/
- https://archives.docs.gitlab.com/18.11/ci/environments/environments_dashboard/
- https://archives.docs.gitlab.com/18.11/ci/environments/external_deployment_tools/
- https://archives.docs.gitlab.com/18.11/ci/pipelines/downstream_pipelines/#downstream-pipelines-for-deployments
- https://archives.docs.gitlab.com/18.11/ci/yaml/#environment
- https://archives.docs.gitlab.com/18.11/ci/variables/predefined_variables/
- https://archives.docs.gitlab.com/18.11/api/deployments/
- https://archives.docs.gitlab.com/18.11/api/environments/
- https://gitlab.com/gitlab-org/gitlab/-/blob/v18.11.0-ee/.gitlab/ci/docs.gitlab-ci.yml
- https://gitlab.com/gitlab-org/gitlab/-/blob/v18.11.0-ee/.gitlab/ci/pages.gitlab-ci.yml
- https://gitlab.com/gitlab-com/www-gitlab-com/-/blob/1f99c88d674364fe21a581bed7570b3b5a479852/.gitlab-ci.yml

Read `pipeline-selection.md` before changing deployment rules or manual
gates. Read `security.md` before handling deployment credentials,
protected refs, or untrusted merge requests. Read `orchestration.md` for
downstream pipeline behavior and `pipeline-ui.md` when the pipeline graph,
not the Environments UI, is the problem.

## Contents

- [Mental model](#mental-model)
- [Design the environment model first](#design-the-environment-model-first)
- [Static environment pattern](#static-environment-pattern)
- [Review app pattern](#review-app-pattern)
- [Dynamic URLs](#dynamic-urls)
- [Environment-scoped variables and protection](#environment-scoped-variables-and-protection)
- [Deployment history and rollback](#deployment-history-and-rollback)
- [Operations, alerts, and API](#operations-alerts-and-api)
- [Downstream deployment project](#downstream-deployment-project)
- [Kubernetes metadata](#kubernetes-metadata)
- [Make the GitLab UI useful](#make-the-gitlab-ui-useful)
- [Failure signatures](#failure-signatures)
- [Review checklist](#review-checklist)

## Mental model

- An **environment** is the named target, such as `staging`, `production`,
  or `review/fix-login`. It persists across pipelines and has an ID, URL,
  tier, state, and deployment history.
- A **deployment** is a recorded attempt to change that target. GitLab normally
  creates it from an `environment:action: start` job; an external tool can
  create it explicitly through the Deployments API. Reusing `production`
  builds one useful history. Creating `production-$CI_PIPELINE_ID` creates
  unrelated environments and defeats that history.
- Environment state is `available`, `stopping`, or `stopped`. Stopping an
  environment is metadata unless its `on_stop` job actually removes the
  external resources. Deleting it requires it to be stopped first and also
  deletes its GitLab deployment records.
- Declaring `environment:` does not deploy anything. The job's `script` or
  downstream deployment pipeline owns the real operation. Conversely, a
  deploy script without `environment:` is invisible to GitLab's deployment
  history, controls, URLs, and environment-scoped variables unless an external
  tool reports the deployment through the API. API reporting adds history, not
  GitLab's execution-time deployment controls.

`environment:action` distinguishes the job's intent:

| Action | Creates a deployment | Changes lifecycle |
|---|---:|---|
| `start` | Yes | Creates or updates the environment and marks it available |
| `prepare` | No | Accesses it and resets `auto_stop_in` from the latest successful deployment |
| `access` | No | Accesses it and resets `auto_stop_in` from the latest successful deployment |
| `verify` | No | Accesses it without resetting `auto_stop_in` |
| `stop` | No | Runs teardown and marks it stopped |

Use `prepare`, `access`, or `verify` for non-deployment preparation, smoke
tests, health checks, or other environment-aware jobs that must receive
environment-scoped variables without polluting deployment history. A schema
or data migration that changes the target is normally part of an auditable
deployment, not a `prepare` action. Do not label a validation job `start`
merely to expose variables.

## Design the environment model first

Choose names, tiers, lifetime, ownership, and concurrency before writing
jobs.

- Use stable top-level names for shared targets: `development`, `staging`,
  and `production`. Use folders for dynamic fleets:
  `review/$CI_COMMIT_REF_SLUG`, `tenant/$TENANT_SLUG`, or
  `region/$REGION`. A shared prefix groups environments into a collapsible
  section in the UI.
- Prefer lowercase names. Environment names allow letters, digits, spaces,
  `-`, `_`, `/`, `$`, `{`, and `}`. Use slug variables for user-controlled
  refs. `CI_COMMIT_REF_NAME` can contain invalid characters. Empty variables
  can also leave an invalid trailing `/`, so verify every dynamic name input
  exists at pipeline creation.
- Do not treat `CI_ENVIRONMENT_SLUG` as a unique long identifier. GitLab
  truncates it to 24 characters and adds a random suffix when the environment
  name contains uppercase letters. Include a stable unique value such as the
  MR IID in infrastructure names when collisions matter.
- Set `deployment_tier` explicitly when a custom name represents a standard
  tier. Valid values are `production`, `staging`, `testing`, `development`,
  and `other`. Name-based inference can classify a custom target incorrectly.
  Group-level protected environments use tiers, and DORA deployment metrics
  depend on correct classification. Adding `deployment_tier` later does not
  update an existing environment; update it with the Environments API.
- Environment names cannot be renamed. Stop and delete the old environment,
  then create the new name. Plan names as durable identifiers, not display
  labels.
- Set a valid `url` whenever the target is browsable. GitLab then exposes it
  in merge requests, jobs, deployments, and the Environments UI.
- Keep deployment job names predictable: `deploy:staging`,
  `deploy:production`, `deploy:review`, and `stop:review`. The job answers
  what happens; the environment answers where.

## Static environment pattern

```yaml
deploy:production:
  stage: deploy
  interruptible: false
  resource_group: production
  rules:
    - if: $CI_DEPLOY_FREEZE
      when: never
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
      allow_failure: false
  manual_confirmation: "Deploy the default branch to production?"
  script:
    - ./scripts/ci/deploy.sh production "$CI_COMMIT_SHA"
  environment:
    name: production
    url: https://example.com
    deployment_tier: production
```

- `interruptible: false` keeps a newer pipeline from canceling a live
  deployment when automatic cancellation is enabled.
- `resource_group` serializes deployment jobs across pipelines for this
  project. It prevents concurrent writes, but its default `unordered` mode
  does not guarantee version order. Configure `oldest_first`,
  `newest_first`, or `newest_ready_first` through the Resource Groups API
  when order matters. The two newest-first modes require idempotent jobs.
- A deploy freeze only supplies the pre-pipeline `CI_DEPLOY_FREEZE` variable.
  The job must use a rule to respect it. Rules are fixed when the pipeline is
  created: a pipeline created before the freeze can still run a manual deploy
  during it, while one created during the freeze stays blocked after it ends.
  For strict enforcement, create the deployment pipeline just in time or add
  an execution-time policy check against the authoritative freeze schedule.
- A blocking manual job controls timing, not authorization. Protected
  environments control who can deploy. Deployment approvals add independent
  approval requirements.
- The deploy command should consume an immutable version, such as a commit
  SHA or image digest. Do not rebuild mutable application artifacts in the
  deploy job. See `data-flow.md` for artifact transfer.

For continuous delivery, also enable **Prevent outdated deployment jobs** in
project CI/CD settings. `resource_group` prevents overlap; the setting rejects
an older deployment after a newer deployment has started. They solve different
races. Job age is based on start time, not commit time, which can surprise
teams with manual deployment jobs. Decide whether rollback job retries are
allowed; sensitive projects should require a new pipeline for the old commit.

GitLab's own Pages job declares a stable `pages` environment and a matching
`resource_group: pages`. The general lesson is useful beyond Pages: serialize
every shared mutable target, not only targets named `production`.

## Review app pattern

```yaml
.review-environment:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  resource_group: review-$CI_MERGE_REQUEST_IID
  environment:
    name: review/$CI_MERGE_REQUEST_IID-$CI_COMMIT_REF_SLUG

deploy:review:
  extends: .review-environment
  stage: deploy
  script:
    - ./scripts/ci/deploy-review.sh "$CI_MERGE_REQUEST_IID" "$CI_COMMIT_SHA"
  environment:
    url: https://mr-$CI_MERGE_REQUEST_IID.review.example.com
    on_stop: stop:review
    auto_stop_in: 1 week

stop:review:
  extends: .review-environment
  stage: deploy
  variables:
    GIT_STRATEGY: empty
  script:
    - /usr/local/bin/destroy-review "$CI_MERGE_REQUEST_IID"
  environment:
    action: stop
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
      allow_failure: true
  manual_confirmation: "Delete this review environment?"
```

The hidden job centralizes the identity, selection rule, and lock so deploy
and stop jobs cannot drift. GitLab's own documentation review app uses the
same hidden-job pattern; `www-gitlab-com` also uses shared review environment
configuration and multiple stop actions for separate resources.

These official configurations also keep review environments under one prefix,
publish a direct URL, set finite lifetimes, and separate independent teardown
actions. Reuse those structural decisions, not their project-specific scripts,
images, or legacy conventions.

The example assumes teardown is baked into the job image. If cleanup lives
only in the repository, `GIT_STRATEGY: empty` removes it. Either keep checkout
and use merge request pipelines so the MR ref remains fetchable, or package
the cleanup tool outside the deleted branch. `empty` gives a clean build
directory before the job; `none` reuses the runner directory and can expose
stale files on persistent executors.

Review app invariants:

- Deploy and stop jobs must resolve to exactly the same environment name.
- They need equivalent selection rules so the stop job is present whenever
  the deploy job is present. Extending a common hidden job is safer than
  copying the rules.
- The stop job must define `when`, `environment:name`, and
  `environment:action: stop`. GitLab can run only non-archived stop jobs from
  the latest finished pipeline.
- Put them in the same stage, or give the stop job a runnable `needs` edge.
  A later cleanup stage can remain blocked when an earlier job fails.
- Use the same `resource_group` on both jobs. This is required for the
  Environments UI Stop action to run the `on_stop` job and prevents deploy
  and teardown from racing.
- `when: manual` plus `allow_failure: true` keeps the cleanup button available
  without holding the pipeline or merge request open.
- GitLab automatically stops branch-associated environments on branch delete
  or merge by default, even without `on_stop`. Use the Environments API
  `auto_stop_setting: with_action` if a static environment should stop only
  when an explicit stop action exists. This default is subject to a proposed
  GitLab change, so configure it rather than relying on inference for staging
  and production.
- `auto_stop_in` is approximate. Its worker runs hourly. A successful redeploy
  resets the deadline; `prepare` and `access` reset it; `verify` does not.
  `auto_stop_in: never` disables expiry.
- Multiple successful deploy jobs can each declare an `on_stop` job for the
  same environment. GitLab runs the matching stop actions from the latest
  finished pipeline in parallel and without ordering. Keep every action in
  that pipeline and make teardown independent and idempotent. For downstream
  deployments, declare the environment actions in the parent pipeline.
- Merge trains require duplicate pipelines to be prevented for automatic MR
  stop behavior to work reliably. Use the MR/branch workflow pattern from
  `pipeline-selection.md`.
- Fork merge requests should normally build and test without deploying.
  Restrict review-app deployment to the canonical project when it needs parent
  credentials or infrastructure. See `security.md` before enabling parent
  project pipelines for fork code.

Stopping changes GitLab state and can run cleanup. Deleting removes the
stopped environment and its deployment history. Scheduled hygiene should
normally stop stale resources first, verify external deletion, then delete
old GitLab records through the API. The UI's **Clean up environments** action
stops stale active environments but ignores protected environments.

## Dynamic URLs

When the platform assigns a URL during deployment, return it with a dotenv
report:

```yaml
deploy:review:
  script:
    - review_url="$(./scripts/ci/deploy-review.sh)"
    - printf 'DYNAMIC_ENVIRONMENT_URL=%s\n' "$review_url" > deploy.env
  artifacts:
    reports:
      dotenv: deploy.env
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: $DYNAMIC_ENVIRONMENT_URL
    on_stop: stop:review
```

GitLab reads the report after the job and updates the environment URL. Validate
the deploy command's output before publishing it. Do not repeat
`url: $DYNAMIC_ENVIRONMENT_URL` on the stop job: that job did not produce the
dotenv variable, so it cannot resolve the URL. Keep the URL and dotenv report
free of credentials; report artifacts and registered variables are not a
secret transport.

## Environment-scoped variables and protection

- Scope sensitive CI/CD variables to the smallest environment name or group,
  such as `production` or `review/*`. The default scope `*` exposes them to
  every job. An environment-scoped variable is available only to a job that
  declares the matching `environment:name`.
- Do not use environment-scoped variables in `rules` or `include`; they might
  not exist while GitLab validates and constructs the pipeline.
- Scope and masking reduce accidental exposure. They do not make hostile CI
  code safe. Combine protected variables, protected refs, protected
  environments, reviewed CI changes, and short-lived OIDC credentials as
  described in `security.md`.
- Protected environments are Premium/Ultimate. Configure **Allowed to
  deploy** for authorization. Group-level protected environments match
  deployment tiers, so set `deployment_tier` explicitly. Project and group
  restrictions combine; a user must satisfy both.
- Deployment-only access does not imply lifecycle administration. A user with
  Reporter-level access through an allowed group can deploy, but stopping or
  deleting a protected environment also depends on branch push or merge access.
  Design cleanup ownership separately from deployment permission.
- GitLab 18.11 has no `environment:` or job keyword that hides the Environments
  UI Stop action while leaving the user authorized to stop that environment.
  Removing `on_stop` changes the action from running teardown to metadata-only
  stopping; it does not make the button a reliable access control. Use
  protected-environment permissions to remove stop authorization, or make the
  teardown idempotent and safe when authorized users can stop it.
- Deployment approvals are Premium/Ultimate and apply to protected
  environments. Approval does not start the job. After all rules pass,
  someone must still run the deployment job. By default, the pipeline
  triggerer cannot self-approve. One user supplies only one approval even if
  they belong to several approver groups.
- A confirmation prompt is not an approval rule. A manual job is not an
  authorization boundary. State the intended control precisely in reviews.

## Deployment history and rollback

- One stable environment name produces an auditable sequence of deployments.
  Set the correct tier so GitLab can associate newly included merge requests
  even when a custom environment name contains `/`.
- UI rollback reruns only the selected previously successful deployment job.
  It does not rerun build or plan jobs. Rollback works only if the deploy script
  can reproduce the old state from immutable inputs that remain available.
  Terraform plans, expired artifacts, schema compatibility, and one-way
  migrations usually require a purpose-built rollback pipeline instead.
- GitLab stores deployment refs under `refs/environments/*`, but archives old
  refs for repository performance. It retains up to 50,000 recent deployment
  refs; archived deployment records remain available in the UI and API. Do not
  make operational recovery depend only on those refs.
- Use `action: verify` for post-deploy checks that should appear against the
  environment without creating fake deployments or extending a review app's
  lifetime.

## Operations, alerts, and API

- The Environments API can create, update, stop, and delete environments,
  correct existing tiers and URLs, and set `auto_stop_setting` to `always` or
  `with_action`. It cannot rename an environment.
- `force=true` on the stop API marks an environment stopped without running
  its `on_stop` action. Use it only when teardown is intentionally handled
  elsewhere or compute quota makes the action undesirable. It can orphan real
  resources.
- The cross-project Environments Dashboard is Premium/Ultimate and shows up to
  three environments per project. On GitLab.com, public projects can be added
  for free; private projects require the owning group to have Premium. The
  dashboard omits grouped environments such as review apps, so use it for
  shared promotion targets, not ephemeral fleet inventory.
- Environment alerts in the environment view and automatic rollback are
  Ultimate. Map the alert to the exact environment with
  `gitlab_environment_name`. Auto Rollback redeploys the latest successful
  deployment, skips rollback while another deployment is running, and runs at
  most once every three minutes. It is disabled by default.
- Treat API cleanup as reconciliation: list candidates, protect explicit
  exceptions, stop them, verify external resources are gone, then delete
  records. Do not use environment deletion as proof that infrastructure was
  removed.

External tools such as Argo CD or Heroku do not create GitLab deployment
history automatically. Their webhook should use the Deployments API to create
a `running` deployment and update it to `success` or `failed`. This restores
environment history, MR deployment visibility, and DORA data, but GitLab
cannot apply protected environments, deployment approvals, deployment safety,
or GitLab rollback to deployments it did not execute. Keep authorization and
rollback controls in the external delivery system.

## Downstream deployment project

Use a separate deployment project when application Maintainers must not read
production secrets or change deployment code. Put `environment` on the
upstream trigger job so the application project retains the environment and
deployment view:

```yaml
deploy:production:
  stage: deploy
  resource_group: production
  environment:
    name: production
    deployment_tier: production
  trigger:
    project: platform/deployments
    branch: main
    strategy: mirror
  variables:
    UPSTREAM_PROJECT_ID: $CI_PROJECT_ID
    UPSTREAM_COMMIT_SHA: $CI_COMMIT_SHA
    UPSTREAM_ENVIRONMENT_NAME: $CI_ENVIRONMENT_NAME
    UPSTREAM_ENVIRONMENT_ACTION: $CI_ENVIRONMENT_ACTION
```

- `strategy: mirror` keeps the trigger status aligned with the downstream
  pipeline. With `resource_group`, it also holds the lock until the downstream
  deployment finishes. On GitLab 18.0-18.1, use `strategy: depend` and accept
  its status limitations.
- Route downstream jobs on `CI_PIPELINE_SOURCE == "pipeline"` and the explicit
  upstream action. Implement deploy and stop paths separately.
- Do not forward masked secrets as ordinary variables. Masking metadata does
  not cross project boundaries. Store credentials in the deployment project
  or obtain them there with OIDC.
- Avoid requiring the same `oldest_first` resource group inside a child and
  its waiting parent trigger; that can deadlock. Own the lock at the parent
  trigger boundary.

## Kubernetes metadata

`environment:kubernetes` connects an environment to the agent-backed
Kubernetes dashboard. It does not deploy the application:

```yaml
environment:
  name: production
  kubernetes:
    agent: $CI_PROJECT_PATH:production
    managed_resources:
      enabled: false
    dashboard:
      namespace: application-production
      flux_resource_path: helm.toolkit.fluxcd.io/v2/namespaces/flux-system/helmreleases/application
```

In GitLab 18.4+, use `dashboard:namespace` and
`dashboard:flux_resource_path`. Their former direct locations under
`kubernetes` are deprecated. Set `managed_resources:enabled: false` when Flux
or another controller, rather than GitLab, owns the Kubernetes resources. The
agent must be installed, its `user_access` must authorize the environment
project or parent group, and the job user must be authorized for the agent.
Otherwise GitLab ignores the agent and dashboard metadata.

## Make the GitLab UI useful

- Use one shared environment per real target and one grouped prefix per
  dynamic fleet. Avoid pipeline IDs in shared environment names.
- Provide the canonical URL. For review apps, include a stable MR identifier
  in both the name and hostname so users can map the link back to the MR.
- Set tiers explicitly. This improves metrics, group protection, and
  cross-project dashboards without forcing names such as `production`.
- Add `manual_confirmation` to destructive deploy and stop buttons. Keep the
  message short and name the consequence.
- Add `.gitlab/route-map.yml` for content sites so MR file changes link
  directly to the corresponding review-app pages.
- Log the immutable version, target environment, and final URL, but never dump
  all environment variables. See `informative-logging.md`.
- Keep review app lifetimes finite and provide a working stop action. Hundreds
  of stale environments make search, cost, and ownership worse. Environment
  search matches prefixes and, for grouped names, the part after the folder;
  consistent naming makes it usable.
- Do not encode deployment history into dozens of job names. The Environments
  and Deployments views already own that history.

## Failure signatures

| Symptom | Likely cause and correction |
|---|---|
| `would create an environment with an invalid parameter` | A variable is empty, `CI_COMMIT_REF_NAME` supplied invalid characters, or `deployment_tier` is invalid. Use a guaranteed pipeline-time value, usually `CI_COMMIT_REF_SLUG`, and validate required variables before the job is created where possible. |
| Stop button changes state but resources remain | No valid `on_stop` job ran, or teardown is not implemented/idempotent. Inspect the latest finished pipeline and the stop job log. |
| `on_stop` job is absent | Deploy and stop `rules` differ, or the stop job was excluded from that pipeline. Share the same rule set. |
| `on_stop` job cannot start | It is blocked by stage order or `needs`. Put it in the deploy stage or give it a runnable dependency path. |
| UI Stop does not run teardown | Deploy and stop jobs do not share a `resource_group`, or the referenced job is archived/unavailable. Align the lock and inspect the current pipeline. |
| Review cleanup fails after branch deletion | Runner cannot fetch the deleted branch. Use MR pipelines, or package cleanup outside the repo and use `GIT_STRATEGY: empty`. |
| Environment URL is blank | URL is malformed, uses a job-script variable without a dotenv report, or the stop job repeats a dynamic URL it cannot resolve. |
| Deployment is `failed outdated deployment job` | Project outdated-deployment protection rejected a job older by start time. Start a new pipeline for intentional rollback or review the rollback-retry setting. |
| Job remains `Waiting for resource` | Another job holds the lock, process ordering waits behind an older pipeline, or parent/child locks deadlock. Inspect the current and upcoming jobs through the resource group UI/API. |
| Protected deployment trigger fails in a downstream workflow | The upstream trigger job lacks `environment`, so GitLab applies ordinary job permissions instead of protected-environment deployment permissions. Associate the trigger with the environment. |
| Environment-scoped secret is empty | The job lacks `environment`, the scope does not match the final name, or the variable was incorrectly expected during rules/include evaluation. |

## Review checklist

1. Does each real target have one stable name and the correct explicit tier?
2. Do deploy, verify, and stop jobs use the right `environment:action`?
3. Can two pipelines mutate the target concurrently or in the wrong order?
4. Are deployment jobs non-interruptible and protected from outdated runs?
5. Do deploy freezes, manual gates, protected environments, and approvals each
   implement the control the author claims?
6. Are production credentials short-lived, environment-scoped, and isolated
   from fork or unprotected pipelines?
7. Can a review app stop after branch deletion, failed earlier stages, and
   expiry? Are deploy and stop rules, names, and resource groups identical?
8. Can rollback reproduce the old version without expired or rebuilt inputs?
9. Does the UI expose a useful URL, grouped names, concise confirmations, and
   finite review-app lifetime?
10. For downstream deployment pipelines, does the parent retain the environment
    and lock until `strategy: mirror` reports completion?
