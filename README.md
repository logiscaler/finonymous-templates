# Finonymous — public CloudFormation templates

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![AWS CloudFormation](https://img.shields.io/badge/AWS-CloudFormation-orange.svg)](https://aws.amazon.com/cloudformation/)
[![Deploy region: us-east-1](https://img.shields.io/badge/region-us--east--1-lightgrey.svg)](#region-requirement)

Open-source CloudFormation templates for onboarding to the [Finonymous](https://logiscaler.com.au) FinOps platform.

We publish the YAML here, in a public repo, so **your security team can audit exactly what runs in your AWS account before you click Launch Stack**. Every IAM grant is documented in the template's own comments — no opaque agent, no hidden permissions.

---

## Quick start

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https%3A%2F%2Flogiscaler.github.io%2Ffinonymous-templates%2Fcustomer-foundational%2Fv0.1.0.yaml&stackName=finonymous-foundational)

One click takes you to the AWS CloudFormation console (us-east-1) with the template pre-loaded. You only need to provide one parameter that has no default — see [Parameters](#parameters) below.

Prefer the CLI?

```bash
aws cloudformation deploy \
  --region us-east-1 \
  --stack-name finonymous-foundational \
  --template-file customer-foundational/v0.1.0.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ExternalId="$(openssl rand -base64 48 | tr -d '/+=' | head -c 64)"
```

---

## What's in this repo

| Path | What it is | When you'd deploy it |
|---|---|---|
| [`customer-foundational/v0.1.0.yaml`](customer-foundational/v0.1.0.yaml) | The one-time foundational stack — CUR v2 export, cross-account read role for Finonymous, manifest-watcher Lambda, shared execution role for future capability Lambdas, SSM capability registry, SNS notifications topic. | Once per AWS account you want Finonymous to observe. |

Each template is published under a stable, version-pinned filename (`v<MAJOR>.<MINOR>.<PATCH>.yaml`) so you can lock to an exact reviewed version. See [Versioning](#versioning).

---

## About Finonymous

Finonymous is a FinOps platform for AWS, with two operating modes:

- **SaaS mode** — auto-ingested CUR data, automated anomaly detection, RI/SP recommendations, cost-allocation chargeback, idle-resource finders. Open dashboards, no ceremony.
- **AI mode** — a chat-first surface over the same data and actions. Ask "where am I overspending this week" and it walks you through it.

The platform lives at [logiscaler.com.au](https://logiscaler.com.au). Trial customers run at [trial.logiscaler.com.au](https://trial.logiscaler.com.au).

This repo is the **customer-side** of the system — what your AWS account hosts to let Finonymous see your data.

---

## The `customer-foundational` stack

### What it does

The foundational stack is the **single template every customer deploys to onboard**. It:

1. **Configures AWS Data Exports (CUR v2)** to write your Cost & Usage Report in Parquet format to an S3 bucket, hourly granularity.
2. **Creates a cross-account IAM role** that the Finonymous platform Lambda can assume — scoped to read-only access to your CUR objects only.
3. **Installs the capability platform** — a `manifest-watcher` Lambda that pulls a signed manifest from Finonymous daily, a shared execution role for future capability Lambdas, an SSM-based capability registry, and an SNS topic for admin notifications.

After deploying once, **adding new Finonymous capabilities** (idle-EC2 finder, orphaned-EBS finder, tagging compliance, etc.) only requires you to update the stack's `EnabledCapabilities` parameter — you stay in full control of which capabilities ever run.

### What deploys to your account

Eleven resources, all `us-east-1`, all tagged `Service=finonymous`:

| Resource | Why it exists |
|---|---|
| S3 bucket | Receives CUR v2 parquet files (encrypted, versioned, private) |
| Bucket policy | Allows ONLY the AWS Billing service to write CUR objects |
| CUR v2 export | Tells AWS Billing to publish your CUR to that bucket, hourly |
| Cross-account read role | What the Finonymous platform Lambda assumes to read CUR |
| Shared capability execution role | Execution role for future in-account capability Lambdas |
| SNS topic | Where the manifest-watcher posts pending-capability notifications |
| SSM parameter | Tracks which capabilities you have approved |
| Manifest-watcher role | Execution role for the watcher Lambda (narrow scope) |
| Manifest-watcher Lambda | Polls the Finonymous manifest endpoint daily, updates approved capability code |
| EventBridge rule | Triggers the watcher Lambda on schedule |
| Lambda permission | Lets EventBridge invoke the watcher |

The full IAM contract for each one is documented in the YAML itself — open [`customer-foundational/v0.1.0.yaml`](customer-foundational/v0.1.0.yaml) and read the inline comments.

### Trust model

We follow least-privilege everywhere. Highlights your security team will care about:

**Cross-account CUR read role**
- Trust `Principal` is a **specific** IAM role ARN in the Finonymous platform account (`762233727016`) — not account-root, not `"*"`.
- `sts:ExternalId` condition with a value YOU generate and supply — defeats the confused-deputy attack.
- Inline policy: ONLY `s3:GetObject` on `<your-bucket>/<your-prefix>/*` and `s3:ListBucket` on the bucket with an `s3:prefix` condition. No `s3:*`. No `Resource: "*"`.

**Manifest-watcher Lambda**
- Cannot create IAM roles. Cannot attach policies. Cannot modify IAM in any way.
- Can only update the **code** of Lambda functions whose name matches `finonymous-capability-*` AND whose tag `Service=finonymous` — enforced via an IAM condition.
- Cannot update itself. Updates to the watcher require a CFN stack update YOU run.
- Can read/write SSM parameters under `/finonymous/capabilities/*` only.
- Can publish to one specific SNS topic only.

**Shared capability execution role**
- Starts with only CloudWatch Logs and SNS publish to the notifications topic.
- When you add a new capability, the CFN stack update attaches **only** the IAM grants that capability needs (e.g., `ec2:DescribeInstances` for idle-EC2). Each grant goes through CFN, never through the watcher at runtime.

**S3 bucket policy on your CUR bucket**
- Single statement allowing the `billingreports.amazonaws.com` service principal to write CUR objects.
- Conditioned on `aws:SourceAccount` = your account AND `aws:SourceArn` matching the CUR export ARN pattern.
- No anonymous access. No third parties.

If anything in the YAML doesn't match this description, **please open a security issue** (see [Reporting issues](#reporting-issues-and-security)).

### Region requirement

Deploy this stack in **`us-east-1`**. AWS Data Exports (CUR v2) has its control plane only in `us-east-1` per the [AWS endpoint registry](https://docs.aws.amazon.com/general/latest/gr/billing.html#bcm_data_exports). The stack will fail to deploy in other regions.

Our ingestion Lambda in `ap-southeast-2` reads your S3 bucket cross-region — costs about $0.06/month per customer in transfer fees, paid by Finonymous.

### Deployment modes

The template supports three modes via the `DeploymentMode` parameter, for different starting points:

| Mode | Use when | What the stack creates |
|---|---|---|
| `NewBucketAndExport` (default) | First-time setup, no existing CUR | Bucket + bucket policy + CUR export + cross-account role + capability platform |
| `ExistingBucketNewExport` | You have an S3 bucket you want CUR to write to (e.g., a central audit-data bucket) but no CUR v2 export yet | CUR export pointing at your bucket + cross-account role + capability platform. **You must manually append a statement to your existing bucket policy** — we emit the exact JSON in the `RequiredBucketPolicyStatement` output. |
| `ExistingBucketAndExport` | You already run CUR v2 for your own analytics | Cross-account role + capability platform only. No bucket policy changes needed. |

### Parameters

| Parameter | Default | Notes |
|---|---|---|
| `FinonymousPlatformAccountId` | `762233727016` | The Finonymous platform AWS account. Only change if your account manager has given you a different value (dedicated single-tenant). |
| `FinonymousIngestionRoleArn` | `arn:aws:iam::762233727016:role/finonymous-cur-ingestion-execution` | The specific platform Lambda role permitted to assume the cross-account read role. |
| `ExternalId` | *(no default — required)* | A 32–128 character random string YOU generate. Used in the cross-account trust to defeat confused-deputy. Generate with `openssl rand -base64 48 \| tr -d '/+=' \| head -c 64`. Share with your Finonymous account manager. |
| `DeploymentMode` | `NewBucketAndExport` | See [Deployment modes](#deployment-modes) above. |
| `ExistingBucketName` | *(empty)* | Required if `DeploymentMode` is `ExistingBucketNewExport` or `ExistingBucketAndExport`. |
| `ExistingBucketPrefix` | `cur-export` | Prefix under which CUR objects land in your bucket. |
| `EnabledCapabilities` | `cur-ingestion` | Comma-separated capability IDs you have approved. Only listed capabilities will be applied by the watcher. |
| `ManifestUrl` | `https://platform.finonymous.com.au/manifest.json` | Signed manifest endpoint. Should not normally need changing. |
| `ManifestWatcherScheduleExpression` | `rate(24 hours)` | How often the watcher polls the manifest. Daily is plenty. |

### Outputs

When the stack finishes, the **Outputs** tab in the CloudFormation console will show:

| Output | Send to your Finonymous account manager? | Notes |
|---|---|---|
| `CurReadRoleArn` | **Yes** | The ARN our ingestion Lambda will assume. |
| `CurBucketName` | **Yes** | The bucket containing your CUR. |
| `CurPrefix` | **Yes** | Prefix under which CUR objects land. |
| `CurBucketRegion` | **Yes** | Always `us-east-1`. |
| `ManifestWatcherFunctionArn` | No (informational) | Manually invoke to force an immediate manifest pull after adding a capability. |
| `NotificationsTopicArn` | No (informational) | Subscribe your admin email / Slack / PagerDuty to receive notifications. |
| `CapabilityExecutionRoleArn` | No (informational) | Role used by future in-account capability Lambdas. |
| `RequiredBucketPolicyStatement` | No (only emitted in `ExistingBucketNewExport` mode) | Append this JSON statement to your existing bucket policy. |
| `DeploymentModeReminder` | No (informational) | Reminder of which mode this stack was deployed in. |

The `ExternalId` you supplied is **not** emitted as an output (we mark the parameter `NoEcho: true`). Keep your own copy of the value you generated — you'll need to share it with your Finonymous account manager so they can store it in their connection record.

---

## For your security team

A checklist your security reviewer can walk through, with grep-friendly anchors to the YAML:

- [ ] **No wildcard IAM actions.** `grep "Action.*'\*'" customer-foundational/v0.1.0.yaml` → no matches.
- [ ] **No wildcard resources.** `grep "Resource.*'\*'" customer-foundational/v0.1.0.yaml` → no matches.
- [ ] **Cross-account trust is specific.** The `CurReadRole` trust policy `Principal` is the platform's specific Lambda execution role ARN, not account-root, not `"*"`.
- [ ] **`sts:ExternalId` is required** on the cross-account assume. You can verify by trying to assume the role without it; it should be denied.
- [ ] **Bucket policy allows ONE principal**: `billingreports.amazonaws.com`. No `Principal: "*"`. Conditioned on `aws:SourceAccount` + `aws:SourceArn`.
- [ ] **Watcher Lambda has no IAM-write actions** — review the `ManifestWatcherRole` inline policy. No `iam:*`, no `lambda:CreateFunction`, no `lambda:AddPermission`.
- [ ] **Watcher can only update tagged Lambdas** — the `lambda:UpdateFunctionCode` action is conditioned on `aws:ResourceTag/Service: finonymous`.
- [ ] **Inline Python in the watcher is auditable** — read the `Code: ZipFile:` block in the YAML. ~100 lines, no external libraries beyond `boto3` / `urllib` / `hashlib` (all in the Python 3.12 Lambda runtime).
- [ ] **Code-content integrity** — before updating a capability Lambda's code, the watcher SHA-256s the downloaded zip and compares against the value in the signed manifest. Mismatch = refuse to update.
- [ ] **No network access from the watcher** to anywhere outside `platform.finonymous.com.au` (the manifest endpoint). The Lambda runs without a VPC config — egress is via the public internet, but the URL is fixed in code and configurable only via a CFN parameter you control.
- [ ] **Stack tear-down** removes everything except the S3 bucket (which is `Retain` so your CUR data isn't lost). See [Tear-down](#tear-down).

Open an issue if anything in the YAML doesn't pass this checklist.

---

## Versioning

We follow semantic versioning on stable, **immutable** filenames:

```
customer-foundational/v0.1.0.yaml
customer-foundational/v0.1.1.yaml   ← bug-fix release
customer-foundational/v0.2.0.yaml   ← additive, backward-compatible
customer-foundational/v1.0.0.yaml   ← breaking changes (e.g., parameter rename)
```

Each version stays available forever — you can pin to an exact SHA-reviewed version for compliance. **The Launch Stack button at the top of this README always points at the latest stable version.**

Release notes live in [`CHANGELOG.md`](CHANGELOG.md) (once we have more than one release).

---

## Tear-down

To remove Finonymous from your account:

```bash
aws cloudformation delete-stack \
  --region us-east-1 \
  --stack-name finonymous-foundational
```

This deletes every resource except the S3 CUR bucket (which has `DeletionPolicy: Retain` so your historical CUR data isn't lost). To delete the bucket too:

```bash
# Replace <bucket> with the value of the CurBucketName output
aws s3 rb s3://<bucket> --force --region us-east-1
```

After tear-down, the Finonymous platform's next ingestion attempt fails (the cross-account role no longer exists), and your customer record will be flagged disconnected. No data lingers on our side beyond what was already ingested.

---

## Reporting issues and security

- **Found a bug or typo?** [Open an issue](https://github.com/logiscaler/finonymous-templates/issues/new).
- **Found a security concern?** Please email `security@logiscaler.com.au` rather than opening a public issue. We respond within 1 business day.

---

## Related

- **Main product**: [logiscaler.com.au](https://logiscaler.com.au)
- **Trial signup**: [trial.logiscaler.com.au](https://trial.logiscaler.com.au)
- **Manifest endpoint** (signed): `https://platform.finonymous.com.au/manifest.json`

---

## License

Apache License 2.0 — see [LICENSE](LICENSE).
