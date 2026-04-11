# Resolving "Error Acquiring the State Lock" on DynamoDB

## Diagnosis

Your `terraform apply` is stuck because Terraform cannot acquire the state lock. This is a **State Lock Error** -- one of the most common Terraform issues when using an S3 + DynamoDB backend. After 20 minutes, the lock is almost certainly stale (a normal apply rarely takes that long unless you are provisioning very large infrastructure).

**Common causes** (from the troubleshooting guide):
1. A previous `terraform apply` or `terraform plan` crashed or was interrupted (Ctrl+C, CI/CD timeout, network drop).
2. Another user or CI/CD pipeline is currently running Terraform against the same state.
3. The state lock was not released properly after a failed operation.

---

## Step-by-Step Resolution

### Step 1: Confirm No One Else Is Running Terraform

**This is critical -- do not skip it.** Force-unlocking while another process is legitimately holding the lock will cause state corruption.

- Check with your team (Slack, chat, etc.) -- is anyone running `terraform apply` or `terraform plan` right now?
- Check your CI/CD system (GitHub Actions, GitLab CI, Jenkins) for any in-progress Terraform pipeline runs against this workspace.

### Step 2: Inspect the Lock in DynamoDB (Optional but Recommended)

You can verify the lock details directly in the DynamoDB table to see who created it and when:

```bash
aws dynamodb get-item \
  --table-name <your-lock-table-name> \
  --key '{"LockID": {"S": "<your-state-path>/terraform.tfstate-md5"}}'
```

This will return the lock metadata including:
- **Who** created the lock (user and hostname)
- **When** it was created
- **What operation** was in progress (plan or apply)

If the timestamp is 20+ minutes old and no one on your team claims the operation, the lock is stale.

### Step 3: Force Unlock the State

Once you have confirmed no legitimate process holds the lock, run:

```bash
terraform force-unlock abc-123
```

Terraform will ask you to confirm. Type `yes` to proceed.

**Important notes:**
- Use the exact lock ID from the error message (`abc-123` in your case).
- This only removes the lock entry from DynamoDB -- it does not modify your state file.
- If you are using Terragrunt, run `terragrunt force-unlock abc-123` instead.

### Step 4: Retry Your Operation

After the lock is released, re-run your original command:

```bash
terraform apply
```

If you had a saved plan file:

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

### Step 5: Verify State Health

After successfully applying, confirm your state is healthy:

```bash
# List resources to verify state is readable
terraform state list

# Optionally run the state inspector script for a full health check
python3 scripts/inspect_state.py /path/to/your/terraform/directory --check-drift
```

This script will check resource counts, provider versions, backend configuration, and optionally detect any drift between your configuration and actual infrastructure.

---

## Prevention Going Forward

To avoid this problem recurring:

1. **Always let Terraform complete or cancel gracefully.** Avoid killing processes with `kill -9` or abrupt CI/CD timeouts. If you must cancel, use Ctrl+C once and let Terraform clean up.

2. **Set CI/CD pipeline timeouts appropriately.** Ensure your pipeline timeout exceeds the longest expected Terraform operation, and add a cleanup step that runs `terraform force-unlock` on failure (with appropriate safeguards).

3. **Use proper state locking backend.** You already have S3 + DynamoDB, which is the recommended setup per best practices. Make sure:
   - The DynamoDB table has `LockID` as the partition key (type String).
   - All team members and CI/CD systems reference the same `dynamodb_table` in their backend configuration.

4. **Pin Terraform versions.** Version mismatches between team members can cause unexpected failures that leave locks behind:
   ```hcl
   terraform {
     required_version = "~> 1.5.0"
   }
   ```

5. **Enable S3 versioning on your state bucket.** This allows recovery if state corruption ever occurs alongside a lock issue:
   ```bash
   aws s3api list-object-versions \
     --bucket your-terraform-state-bucket \
     --prefix path/to/terraform.tfstate
   ```

6. **Enable debug logging when investigating issues:**
   ```bash
   export TF_LOG=DEBUG
   export TF_LOG_PATH=terraform-debug.log
   terraform apply
   ```

---

## Summary

For your immediate situation: verify no one else is running Terraform, then run `terraform force-unlock abc-123` and retry your apply. The lock has been stale for 20 minutes, so this is safe to do after confirming no active operations.
