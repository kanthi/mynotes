---
title: '17. Project: Scalr-Style Drift Detection & OPA Policy-as-Code Pipeline'
---

# Project: Scalr-Style Drift Detection & OPA Policy-as-Code Pipeline

Build a modular **OpenTofu** repository with automated PR `tofu plan` comments, **OPA / Conftest** policy guardrails that block insecure infrastructure (public S3 buckets, wildcard IAM `*`, unencrypted volumes), and a scheduled drift-detection workflow that catches out-of-band console changes and opens an alert or remediation issue.

## Policy-as-Code Rego Rule (`policy/iam_and_s3.rego`)

```rego
package main

deny[msg] {
  rc := input.resource_changes[_]
  rc.type == "aws_iam_policy"
  contains(rc.change.after.policy, "\"Action\":\"*\"")
  msg := sprintf("IAM policy %s must not use wildcard Action '*'", [rc.address])
}

deny[msg] {
  rc := input.resource_changes[_]
  rc.type == "aws_s3_bucket_public_access_block"
  rc.change.after.block_public_acls != true
  msg := sprintf("S3 bucket %s must block public ACLs", [rc.address])
}
```

## PR Validation & Scheduled Drift Detection

```bash
# In PR CI:
tofu plan -out=tfplan.binary
tofu show -json tfplan.binary > tfplan.json
conftest test tfplan.json --policy policy/

# In Scheduled Drift Cron (every 6h):
tofu plan -detailed-exitcode -no-color
# Exit code 0 = no drift; 2 = drift detected -> alert Slack/PagerDuty & open GitHub Issue
```

## Acceptance Drills

- Open a PR adding an IAM policy with `"Action": "*"` and verify Conftest fails the PR before `tofu apply` can run.
- Manually tag or modify a managed resource out-of-band and verify the scheduled drift job exits with code `2` and posts a structured diff.
