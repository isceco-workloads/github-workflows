# Controlled decommissioning of a chatbot (tofu destroy)

Applies equally to all environments. There is no retention and no
backup: after the destroy, chat histories, RAG data and buckets are irrevocably gone (user decision).

Components:

- `github-workflows/.github/workflows/tf_destroy.yaml`: central reusable workflow containing all the logic.
- `<environment-repo>/.github/workflows/destroy.yml`: caller (`workflow_dispatch`).
- `<environment-repo>/.github/workflows/deploy.yml`, job `guard_deleted_chatbots`: prevents a chatbot directory
  from being deleted while its state still contains resources.

`tf_execute.yaml` is unchanged (the `|| true` there stays as it is). The new workflows deliberately contain
no `|| true` and no `continue-on-error`.

## Procedure

1. A person starts the workflow "Decommission Chatbot" in Actions with `mode=plan`, `chatbot=<name>`, `confirm=<name>`.
   Nothing is changed. The job summary shows the resource list, the plan for deletion protection and the
   destroy plan. Plan files are kept for 1 day as the artifact `decommission-<name>-<run_id>`.
2. Deliberate approval in the second run, the same person is allowed: the person reads the summary and starts the workflow with `mode=apply`,
   `chatbot=<name>`, `confirm=<name>`, `plan_run_id=<run ID from step 1>`.
   There is no approver list and no comparison of persons. Phase 1 changes nothing, only phase 2 changes something.
   Job `approve` checks via the GitHub API (fail-closed): plan run successful, `workflow_dispatch`,
   `main`, same commit, younger than 24 h (`max_plan_age_hours`), first attempt (no re-run), same workflow,
   artifact matches the chatbot and has not expired.
3. Then the following run one after another: disable deletion protection, empty S3 buckets, `tofu destroy`, check "state empty".
4. Second pull request: delete `chatbots/<name>/`. The guard checks that the state is empty (or does not exist).

If aborted midway in phase 2, the chatbot may be partially torn down (an abort in phase 1 changes nothing). Then fix the cause and restart with `mode=plan`.

## Configuration

| What | Where | Note |
|------|-------|------|
| Call SHA | `destroy.yml`, line `uses: ...tf_destroy.yaml@000...` | **Important:** use the current commit SHA. |
| OIDC role | Secret `AWS_OIDC_ROLE_ARN` | Same as for `deploy.yml`. |

## No four-eyes requirement (protection model)

Required reviewers on environments are only available in private repositories with GitHub Team/Enterprise. Therefore there is no four-eyes requirement: the same person may plan and approve. Protection consists of two deliberate runs, plan review in the summary, confirmation by input (`confirm`) and audit log (run history, approval summary with person). `environment:` is not set in the workflow.
Alternative when a licence is available: GitHub Environment with required reviewers (additionally add `environment: <name>` to the jobs `protection_off` and `destroy`, only as an addition).

Limits: anyone who can write to `main` can change the workflow.
Branch protection on `main` with pull request review by persons (not teams) for changes to the workflow and the callers is recommended and should be kept. The guard job
`guard_deleted_chatbots` should be added as a required status check so that the merge itself is blocked.

## Why "exact plan" only partly applies

`tofu apply <planfile>` requires unchanged state. Disabling deletion protection and emptying the buckets
change state or reality, so a destroy plan created in advance would be stale afterwards. Therefore:

- The plan for deletion protection is applied exactly as created (stale = abort).
- The destroy plan is regenerated in the apply run. It is only applied if the list of resources to delete
  is identical to the approved `destroy_addresses.txt`. Otherwise abort without apply.

## Deletion protection

The root module `chatbots/<name>/main.tf` declares no variables for this; the module reads the values from `settings.json`.
`-var` or `TF_VAR_*` would therefore have no effect. The workflow instead overwrites, in the runner checkout (never committed),
`cognito_deletion_protection = "INACTIVE"` and `pgvector_deletion_protection = false` via `jq`, identically in every job.
The protection plan is limited via `-target` to the `aws_cognito_user_pool` and `aws_rds_cluster` resources of the state.
The addresses (including module paths and `for_each` keys) are read dynamically from the state (`tofu show -json`) and
stored as `protection_targets.txt` in the plan artifact. If none is found, the plan job aborts (a plan without `-target` would cover the whole state).
The protection plan may only contain `update` changes to these two types that affect only `deletion_protection`.
Anything else aborts with `Disallowed changes in protection plan`. Possible causes: a change in a dependency of the targeted
resources (`-target` includes dependencies), or drift of the targeted resources themselves. Check the plan output of the step
and `protection.json`; if it is drift, complete a normal deploy first.
Drift on other resources no longer blocks the decommissioning.
The step "Apply protection plan" applies exactly the stored `protection.tfplan` (unchanged).
DynamoDB tables have no deletion protection in the code.

### Decision: protection plan per `-target` instead of full plan

- Context: the full plan failed on drift unrelated to the decommissioning, e.g. `null_resource.build_and_upload`
  (replaced on every plan, trigger paths depend on the build drive) and 20+ unconfirmed
  `aws_sns_topic_subscription` (AWS deletes them after 48 h, the plan shows `create`).
- Decision: limit the plan to the protection resources via `-target`, keep the allow list strict.
- Rejected alternative: extend the allow list with `null_resource` and SNS subscriptions. The apply of the stored plan
  would then trigger the build and send 20+ confirmation emails during a decommissioning.
- Consequences: unrelated drift no longer blocks. Residual risk: `-target` also plans the dependencies of the targeted
  resources; drift there still aborts (see above). The modules (`null_resource`, SNS) are fixed separately.

## Failure notice

If a job fails in phase 1 or in the approval (`validate`, `plan`, `approve`), nothing was changed in the real system and the summary says so.
The notice "may be partially torn down" only appears when `protection_off`, `empty_buckets` or `destroy` failed.

## S3 buckets

The modules do not set `force_destroy`. The workflow reads all `aws_s3_bucket` from the state (recursively), deletes all
versions and delete markers in batches, aborts open multipart uploads and then checks that the bucket is empty.
Not covered: S3 Vectors (`aws_s3vectors_*`), Object Lock, MFA Delete.

## Guard in deploy.yml

The job `guard_deleted_chatbots` runs on push and pull request. It compares the chatbots (directories with
`chatbots/<name>/settings.json`) in the merge base and in the new state. If a chatbot is missing, it reads the state
`<name>_chatbot.tfstate` from the state bucket (`shared_config.yaml`):

- State does not exist or `resources` is empty: deletion allowed.
- State contains resources: job fails, with a pointer to the decommission workflow.
- Any other error (access, KMS, network, invalid JSON): job fails (fail-closed).

Renaming the directory (`a` to `b`) counts as deleting `a`, because the state key depends on the directory name
and `a_chatbot.tfstate` would otherwise be orphaned. Renamed files inside the chatbot (settings.json stays) are not a finding.
`Execute` waits for the guard. In addition, the matrix now only contains directories with `settings.json`, so that deleted
chatbots no longer trigger a plan/apply.
