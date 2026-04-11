# Resolving "Error acquiring the state lock" on DynamoDB

Your `terraform apply` is stuck because a previous Terraform operation either crashed, was interrupted, or is still running, and the state lock in DynamoDB was never released. Since it has been 20 minutes, this is almost certainly a stale lock rather than an active operation. Here is the systematic resolution following the skill's troubleshooting workflow.

---

## Step 1: Confirm No One Else Is Running Terraform

Before force-unlocking, **check with your team** to make absolutely sure no one else has an active `terraform apply` or `terraform plan` running against this same state. A force-unlock while another operation is genuinely in progress can corrupt your state file.

If you use CI/CD pipelines (GitHub Actions, GitLab CI, etc.), also check for any running or queued pipeline jobs that target this Terraform workspace.

## Step 2: Inspect the Lock in DynamoDB (Optional but Recommended)

You can verify the stale lock directly in DynamoDB to see who created it and when:

```bash
aws dynamodb get-item \
  --table-name <your-lock-table-name> \
  --key '{"LockID": {"S": "<state-file-path>/terraform.tfstate-md5"}}'
```

Replace `<your-lock-table-name>` with your DynamoDB table name (commonly `terraform-state-lock` or `terraform-locks`) and `<state-file-path>` with the S3 key path of your state file. The returned item will show the `Who` field (user/hostname), `Operation` (e.g., `OperationTypeApply`), and `Created` timestamp. If the `Created` timestamp is 20+ minutes old and the `Who` field matches a process you know is no longer running, it is safe to proceed.

## Step 3: Force-Unlock the State

Run the following command using the lock ID from your error message:

```bash
terraform force-unlock abc-123
```

Terraform will ask you to confirm. Type `yes` to proceed.

This removes the lock entry from the DynamoDB table, allowing new operations to acquire the lock normally.

## Step 4: Verify the State Is Healthy

After unlocking, confirm your state file was not corrupted by the interrupted operation:

```bash
# List resources to verify state is readable
terraform state list

# Run a plan to verify state consistency
terraform plan
```

You can also use the skill's state inspection script for a more comprehensive health check:

```bash
python3 scripts/inspect_state.py /path/to/your/terraform/directory --check-drift
```

This script checks state health, resource counts, provider versions, backend configuration, and optionally detects drift between your configuration and actual infrastructure.

If `terraform plan` shows unexpected changes or errors, your state may have been partially written before the crash. In that case, restore from the previous S3 version:

```bash
# List available state versions
aws s3api list-object-versions \
  --bucket <your-state-bucket> \
  --prefix <path/to/terraform.tfstate>

# Restore a known-good version
aws s3api get-object \
  --bucket <your-state-bucket> \
  --key <path/to/terraform.tfstate> \
  --version-id <VERSION_ID> \
  terraform.tfstate.backup
```

## Step 5: Re-run Your Apply

Once the state is confirmed healthy:

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

---

## Prevention: Stop This From Happening Again

Based on the skill's best practices (`references/best_practices.md`) and troubleshooting guide (`references/troubleshooting.md`):

1. **Always let Terraform complete or cancel gracefully.** Avoid killing Terraform processes with `Ctrl+C` twice or `kill -9`. A single `Ctrl+C` sends a graceful shutdown signal that releases the lock.

2. **Set timeouts in CI/CD pipelines.** If your pipeline kills the Terraform process on timeout, the lock will be left behind. Configure your pipeline to send a graceful signal first, or add a cleanup step that runs `terraform force-unlock` on timeout.

3. **Enable S3 versioning on your state bucket.** This ensures you can always recover from a corrupted state caused by an interrupted write. Your `backend.tf` should look like:

   ```hcl
   terraform {
     backend "s3" {
       bucket         = "company-terraform-state"
       key            = "prod/networking/terraform.tfstate"
       region         = "us-east-1"
       encrypt        = true
       dynamodb_table = "terraform-state-lock"
     }
   }
   ```

4. **Use consistent Terraform versions across your team.** Version mismatches can cause state compatibility issues. Pin your version in `versions.tf`:

   ```hcl
   terraform {
     required_version = "~> 1.7.0"
   }
   ```

5. **Consider a lock monitoring alert.** Set up a CloudWatch alarm or simple cron job that checks for DynamoDB lock entries older than a threshold (e.g., 30 minutes) and notifies the team via Slack or email.

---

## Summary

| Step | Action | Command |
|------|--------|---------|
| 1 | Verify no active operations | Check with team, check CI/CD pipelines |
| 2 | Inspect the DynamoDB lock | `aws dynamodb get-item ...` |
| 3 | Force-unlock | `terraform force-unlock abc-123` |
| 4 | Verify state health | `terraform state list` and `terraform plan` |
| 5 | Re-run apply | `terraform plan -out=tfplan && terraform apply tfplan` |

---

*Guidance sourced from the iac-terraform skill: `SKILL.md` (Section 4: Troubleshooting Issues), `references/troubleshooting.md` (State Lock Error), `references/best_practices.md` (State Management), and `scripts/inspect_state.py` (state health verification).*
