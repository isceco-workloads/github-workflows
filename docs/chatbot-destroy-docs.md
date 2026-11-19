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

If aborted midway, the chatbot may be partially torn down. Then fix the cause and restart with `mode=plan`.

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
The protection plan may only contain `update` changes to `aws_cognito_user_pool` and `aws_rds_cluster` that affect only
`deletion_protection`. Anything else (drift, pending changes) aborts: complete a normal deploy first.
DynamoDB tables have no deletion protection in the code.

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

## Assumptions and residual risks (not verified against real AWS/GitHub)

- IAM of the OIDC role (assumption): `s3:ListBucket`, `s3:ListBucketVersions`, `s3:GetBucketVersioning`, `s3:DeleteObject`,
  `s3:DeleteObjectVersion`, `s3:AbortMultipartUpload`, `s3:ListBucketMultipartUploads` on the chatbot buckets,
  `s3:GetObject` on the state bucket plus `kms:Decrypt` (guard), plus the usual destroy permissions of the
  `system/<env>-*-role` roles.
- Aurora creates a final snapshot `<env>-pgvector-final` on destroy (`skip_final_snapshot = false` in the module
  `bedrock-chatbot-base/modules/aurora_pgvector`). This contradicts "no backup", but cannot be
  switched off via `settings.json`. The snapshot remains and must be deleted manually before re-creating with the same name.
- Bedrock guardrail (`skip_destroy`), KMS keys (deletion period) and the empty state file remain.
- Plan files may contain sensitive values. Artifact: 1 day, private repository.
- `gh` is not required (the GitHub API is called via `curl`). `jq` and GNU `date` on the runner are assumed.
- The protection plan ignores attributes that are only "known after apply".
- The behaviour of the CodeBuild runners with several consecutive jobs of the same label has not been verified.
- The guard does not run for fork pull requests (no secrets); the repositories are private.
- A direct push to `main` without review can only be prevented by branch protection (organizational).
