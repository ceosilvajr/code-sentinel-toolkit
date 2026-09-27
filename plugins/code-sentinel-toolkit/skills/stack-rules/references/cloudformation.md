# AWS CloudFormation

Covers CloudFormation templates (YAML/JSON), SAM templates (`Transform: AWS::Serverless-2016-10-31`)
and their per-environment parameter files.

## Naming
- Logical IDs `PascalCase` and descriptive of the resource (`OrdersTable`, not `Table1`).
- Renaming a logical ID replaces the resource: treat any logical-ID rename on a stateful
  resource as Critical unless the PR states the replacement is intended.
- Export names and physical names that include the stack or environment name, so two stacks
  cannot collide.

## Typing
Not applicable. `type-design-reviewer` skips template files. Parameter hygiene lives under
Boundaries.

## Error handling
- Stateful resources (databases and tables, buckets, file systems, KMS keys, user pools,
  queues holding data, log groups kept for audit) without `DeletionPolicy` and
  `UpdateReplacePolicy` set to `Retain` or `Snapshot`.
- A property change that forces replacement of a stateful resource (check the resource's
  "Update requires: Replacement" properties), with no migration plan in the PR.
- Alarms whose `TreatMissingData` hides an outage (`notBreaching` on a metric that stops
  reporting when the thing is down).
- Dead-letter queues, failure destinations or retry settings missing on asynchronous
  invocations and event rules where a dropped event is a data loss.
- Custom resources that do not signal failure back to CloudFormation on error (the stack hangs
  until timeout instead of rolling back).

## Boundaries
- IAM least privilege: flag `Action: "*"`, `service:*`, and `Resource: "*"` where the action
  supports resource-level permissions. Flag `iam:PassRole` without a resource restriction.
- No hardcoded account IDs, regions or ARNs: use `AWS::AccountId`, `AWS::Region`,
  `AWS::Partition`, `!Sub`, parameters or imports.
- Secrets never as plain `String` parameters or literal values; use dynamic references
  (`{{resolve:secretsmanager:...}}`, `{{resolve:ssm-secure:...}}`) or `NoEcho` plus a secret store.
- Public exposure: security groups open to `0.0.0.0/0` / `::/0` on non-HTTP(S) ports, bucket
  policies or ACLs granting public access, missing `PublicAccessBlockConfiguration`.
- Encryption at rest and in transit enabled where the service supports it.
- Parameters that differ per environment belong in parameter files, not as template edits
  per environment.
- N+1: not applicable.

## State and lifecycle
Not applicable.

## Testing
- Line coverage does not apply. The bar is: template passes `cfn-lint`; policy-as-code rules
  (`cfn-guard`, `checkov`, `cfn-nag`) pass if the repo uses them; the change was previewed as
  a change set, and the PR shows resources marked `Replacement: True` if any.
- Flag a template change with no lint step in CI as a Suggestion; flag a replacement of a
  stateful resource with no change-set evidence as Critical.
- Custom resource or macro code follows its own language pack for tests.

## PR size exclusions
Packaged templates written by `aws cloudformation package` / `sam package`, `.aws-sam/**`,
`cdk.out/**`, generated parameter files.

## Sensitive paths
Any IAM resource (`AWS::IAM::*`, inline `Policies`, resource policies), KMS keys and key
policies, security groups and network ACLs, user pools and identity providers, WAF rules,
bucket policies, anything with `DeletionPolicy` or a replacement-forcing change.
