# Resolving "Error acquiring the state lock" on DynamoDB

Your `terraform apply` is failing because a state lock is being held in your DynamoDB lock table and was never properly released. Since it has been 20 minutes, this is almost certainly a stale lock from a crashed or interrupted previous Terraform operation. Here is how to resolve it safely.

---

## Step 1: Confirm No One Else Is Running Terraform

Before force-unlocking, verify with your team that no other person or CI/CD pipeline is currently running `terraform plan` or `terraform apply` against this same state. A lock exists to prevent concurrent modifications, and force-unlocking while another operation is genuinely in progress will corrupt your state.

Check with your team through Slack, chat, or whatever communication channel you use. Also check your CI/CD system (GitHub Actions, GitLab CI, Jenkins, etc.) for any in-progress Terraform jobs targeting this workspace.

## Step 2: Inspect the Lock in DynamoDB (Optional but Recommended)

You can query the DynamoDB lock table directly to see the lock details and confirm it is stale:

```bash
aws dynamodb get-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "path/to/your/state/terraform.tfstate-md5"}}'
```

Replace `terraform-state-lock` with your actual DynamoDB table name, and adjust the key path to match your state file path. The output will show who created the lock, when it was created, and what operation was being performed. If the `Created` timestamp is 20+ minutes old and the `Who` field shows a process that is no longer running, the lock is stale and safe to remove.

## Step 3: Force-Unlock the State

Once you have confirmed no one else is running Terraform, force-unlock using the lock ID from the error message:

```bash
terraform force-unlock abc-123
```

Terraform will ask you to confirm. Type `yes` to proceed.

If you want to skip the confirmation prompt (e.g., in a CI/CD recovery script), you can use:

```bash
terraform force-unlock -force abc-123
```

## Step 4: Retry Your Operation

After the lock is released, retry your original command:

```bash
terraform apply
```

Or, if you had a saved plan file:

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

It is good practice to re-run `terraform plan` first to review the changes again, since the previous interrupted apply may have partially completed and changed the state.

## Step 5: Verify State Integrity

After the apply completes, verify your state is healthy:

```bash
# List all resources to confirm state is intact
terraform state list

# Run a plan to confirm there is no unexpected drift
terraform plan
```

If the plan shows no changes (or only the changes you expect), your state is in good shape.

You can also use the skill's state inspection script for a more thorough health check:

```bash
python3 scripts/inspect_state.py /path/to/your/terraform/directory --check-drift
```

This will report on resource counts, provider versions, backend configuration, any tainted resources, and configuration drift.

---

## Understanding What Happened

The state lock error occurs when Terraform acquires a lock in DynamoDB before modifying state, and that lock is not released. The three most common causes are:

1. **A previous `terraform apply` crashed or was interrupted** (e.g., Ctrl+C, network drop, CI/CD timeout, OOM kill). This is the most common cause and is almost certainly what happened in your case given the 20-minute age.
2. **Another user or pipeline is actively running Terraform** against the same state file. This is the legitimate use case for locks and is why you must verify first.
3. **The lock release failed** due to a transient network or DynamoDB error after the Terraform operation completed.

---

## Prevention: Avoid This in the Future

To reduce the likelihood of stale locks:

- **Implement timeouts in CI/CD pipelines.** Set reasonable timeouts on your Terraform jobs so that if a job hangs, the runner eventually kills it and the next run can detect the stale lock.
- **Always let Terraform complete or cancel gracefully.** Avoid hard-killing Terraform processes. If you must cancel, use Ctrl+C once and let Terraform perform its graceful shutdown, which includes releasing the lock.
- **Use remote state with proper locking** (which you already have with S3 + DynamoDB). This is the recommended backend configuration as documented in the skill's best practices:

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    kms_key_id     = "arn:aws:kms:us-east-1:ACCOUNT:key/KEY_ID"
  }
}
```

- **Pin your Terraform version** in `versions.tf` to prevent state corruption from version mismatches, which can compound lock issues:

```hcl
terraform {
  required_version = "~> 1.5.0"
}
```

- **Enable S3 versioning** on your state bucket so you can recover from state corruption if a partial write occurs during an interrupted apply.

---

## Quick Summary

| Step | Command | Purpose |
|------|---------|---------|
| 1 | Check with your team | Confirm no active Terraform runs |
| 2 | `aws dynamodb get-item ...` | Inspect the lock details (optional) |
| 3 | `terraform force-unlock abc-123` | Remove the stale lock |
| 4 | `terraform plan -out=tfplan && terraform apply tfplan` | Retry your operation |
| 5 | `terraform plan` (no changes expected) | Verify state integrity |
