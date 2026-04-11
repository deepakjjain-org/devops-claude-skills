# Fixing "Error acquiring the state lock" in Terraform with DynamoDB

## Understanding the Problem

When Terraform runs `apply` (or `plan`), it acquires a lock on the state file to prevent concurrent modifications. With an S3 backend, this lock is stored as an item in a DynamoDB table. The error means another process (or a previously crashed process) is holding that lock.

Since it has been 20 minutes, this is almost certainly a **stale lock** -- the process that originally acquired it has either crashed, been killed, timed out, or lost connectivity before it could release the lock.

---

## Step 1: Verify the Lock Is Actually Stale

Before force-unlocking, confirm that no legitimate Terraform operation is still running. Check with your team:

- Is anyone else running `terraform apply` or `terraform plan` against this workspace?
- Was there a CI/CD pipeline (e.g., GitHub Actions, GitLab CI, Jenkins) that may still be running?
- Did a previous run crash or get interrupted (Ctrl+C, killed terminal, OOM, network timeout)?

If you are confident no legitimate operation is holding the lock, proceed to Step 2.

## Step 2: Force-Unlock the State

Terraform provides a built-in command for this. Run the following from the same directory where your Terraform configuration lives:

```bash
terraform force-unlock abc-123
```

Terraform will ask for confirmation. Type `yes` to proceed.

If you want to skip the confirmation prompt (e.g., in a CI/CD context):

```bash
terraform force-unlock -force abc-123
```

This removes the lock entry from the DynamoDB table, freeing the state for new operations.

## Step 3: Verify the Lock Is Released

After force-unlocking, verify that Terraform can now acquire the lock:

```bash
terraform plan
```

If the plan runs without the lock error, the issue is resolved.

---

## Alternative: Manually Remove the Lock from DynamoDB

If `terraform force-unlock` fails for any reason (e.g., backend misconfiguration, permissions issues), you can manually delete the lock item directly from DynamoDB.

### Using the AWS CLI

First, identify your DynamoDB table name and the lock item. The partition key is typically `LockID` and its value is the S3 bucket path to your state file (not the lock ID `abc-123`). You can scan or query the table:

```bash
aws dynamodb scan \
  --table-name <your-lock-table-name> \
  --filter-expression "contains(Info, :lockid)" \
  --expression-attribute-values '{":lockid": {"S": "abc-123"}}'
```

Once you have the `LockID` value (which looks like `<bucket-name>/<path-to>/terraform.tfstate`), delete it:

```bash
aws dynamodb delete-item \
  --table-name <your-lock-table-name> \
  --key '{"LockID": {"S": "<bucket-name>/<path-to>/terraform.tfstate"}}'
```

### Using the AWS Console

1. Open the DynamoDB console.
2. Navigate to your lock table.
3. Find the item where the `Info` field contains `abc-123`.
4. Delete that item.

---

## Step 4: Retry Your Terraform Apply

Once the lock is cleared:

```bash
terraform apply
```

This should now proceed without the lock error.

---

## Important Warnings

1. **Never force-unlock while a legitimate operation is running.** This can cause state corruption if two processes write to the state file simultaneously. Always confirm that no other operation is active first.

2. **Check for state corruption after force-unlock.** After clearing a stale lock, it is good practice to run `terraform plan` before `terraform apply`. Review the plan carefully to ensure the state is consistent and Terraform is not proposing unexpected changes.

3. **If state corruption has occurred**, you may need to restore from a previous state version. If your S3 bucket has versioning enabled (which it should for Terraform state), you can restore a previous version of the `.tfstate` file.

---

## Preventing This in the Future

1. **Enable S3 bucket versioning** on your state bucket so you can recover from state corruption.

2. **Use `-lock-timeout` flag** to make Terraform wait before failing on lock contention:
   ```bash
   terraform apply -lock-timeout=5m
   ```
   This tells Terraform to retry acquiring the lock for up to 5 minutes before giving up.

3. **Ensure CI/CD pipelines handle interruptions gracefully.** If your pipeline runner can be preempted or killed, make sure the cleanup step releases the Terraform lock (or set reasonable timeouts).

4. **Set a DynamoDB TTL** (optional, advanced). Some teams configure a TTL attribute on the DynamoDB lock table to automatically expire stale locks after a certain period. This is not a built-in Terraform feature and requires custom implementation.

5. **Use `terraform plan -out=plan.tfplan`** and then `terraform apply plan.tfplan` to separate planning from applying, reducing the time the lock is held during apply.

---

## Quick Summary

For your specific situation with lock ID `abc-123` stuck for 20 minutes:

```bash
# 1. Confirm no one else is running Terraform
# 2. Force-unlock the state
terraform force-unlock abc-123

# 3. Verify the lock is cleared
terraform plan

# 4. Proceed with your apply
terraform apply
```

This should resolve the issue immediately.
