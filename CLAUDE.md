# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation template repository** that provides a reusable nested stack template for deploying a Kinesis Data Stream with configurable capacity mode, retention and KMS encryption. The template follows the nested stack pattern and is designed to be referenced by parent CloudFormation stacks.

**Key characteristics:**

- Nested CloudFormation template (referenced via `TemplateURL`)
- Stream name built from project, base name, account ID, environment and region
- ON_DEMAND or PROVISIONED capacity mode, configurable retention (24 to 8760 hours)
- Optional KMS encryption; an empty `KmsKeyId` disables it
- Explicit `DeletionPolicy` and `UpdateReplacePolicy` on the stream
- Automated semantic versioning and releases
- AWS OIDC authentication for CI/CD deployments

## Project Structure

```text
cloudformation/
├── template.yaml                    # Nested template: Kinesis Data Stream
├── parameters.json                  # Parameter values used for deployment
└── stack-config.json                # Stack name, template and parameter file paths (read by CI)

.github/workflows/
├── ci.yaml                          # CI build (calls a reusable workflow)
├── release.yaml                     # Semantic release on push to main
├── create-branch.yaml               # Auto-create feature branches from issues
├── claude.yaml                      # Claude Code workflow
├── claude-code-review.yaml          # Claude Code review workflow
├── notify.yaml                      # Notifications
└── setup-environments.yaml          # GitHub environment setup

scripts/plugins/
├── release.config.js                # Semantic-release configuration
└── (other release plugins)          # Commit analysis, notes generation, publishing

.claude/
└── .skills/                         # Project skills

.devcontainer/
└── devcontainer.json                # Dev container setup (Node.js 20)

package.json                         # Dependencies: semantic-release, commitizen
README.md                            # Template documentation and usage examples
CHANGELOG.md                         # Generated release changelog
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Trigger semantic release (usually automatic on main)

```bash
npm run release
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` type. Only `feat` and `fix` trigger releases.

## Key Architecture Concepts

### Nested Stack Pattern

This repo provides a **nested stack template**, referenced from parent/root CloudFormation stacks via `TemplateURL`.

- **Parent stack** calls `AWS::CloudFormation::Stack` with `TemplateURL` pointing to the uploaded template in S3
- **Nested template** returns values through its `Outputs` section (no `Export`, so values are passed explicitly)
- **Parent** reads outputs with `!GetAtt NestedStack.Outputs.OutputKey`

### Stream Naming Convention

```bash
{ProjectName}-{StreamBaseName}-{AccountId}-{Environment}-{Region}[-{CiSuffix}]
```

Example: `cfn-nested-cfn-stream-123456789012-devl-us-east-1`

### Parameter Groups

Parameters are grouped in the `AWS::CloudFormation::Interface` metadata of `template.yaml`:

- **Naming:** `ProjectName`, `StreamBaseName`, `environment`, `CiSuffix`
- **Capacity:** `StreamMode`, `ShardCount`, `RetentionPeriodHours`
- **Encryption:** `KmsKeyId`

When adding a parameter, add it to a group and to `ParameterLabels`.

### KMS Key Handling

- `KmsKeyId` accepts a key ID, key ARN, alias name (`alias/...`) or alias ARN, or an empty string
- `AllowedPattern` enforces these forms; bare alias names without the `alias/` prefix are rejected
- Encryption is enabled when `KmsKeyId` is non-empty (`IsEncrypted` condition)
- The value is passed unchanged to `StreamEncryption.KeyId`

### Resource Policies

- The stream has `DeletionPolicy: Delete` and `UpdateReplacePolicy: Delete`, so it suits ephemeral CI stacks
- For long-lived production streams, switch both to `Retain`; deleting the stack then leaves the stream in place
- Resource-level metadata (`com.aws.cloudformation.Context` with `why` and `must`) describes the intent of each stateful resource

## Key Files to Understand

### `cloudformation/template.yaml`

**Purpose:** Creates a Kinesis Data Stream with configurable capacity, retention and encryption

**Key inputs:**

- `ProjectName` (required): Stream name prefix, lowercase letters, numbers and hyphens, max 20 characters
- `StreamBaseName` (default: `cfn-nested-stream`): Base name component
- `environment`: Environment label (`devl`, `prod`, `test`)
- `CiSuffix`: Optional suffix for CI/CD unique deployments
- `StreamMode`: `ON_DEMAND` (default) or `PROVISIONED`
- `ShardCount`: Used only when `StreamMode` is `PROVISIONED`
- `RetentionPeriodHours`: 24 to 8760
- `KmsKeyId` (default: `alias/aws/kinesis`): KMS key ID, ARN or alias; empty disables encryption

**Key outputs:**

- `KinesisStreamName`: Stream name
- `KinesisStreamArn`: Stream ARN

### `.github/workflows/ci.yaml`

**Triggered on:** Manual `workflow_dispatch`. The `push` and `pull_request` triggers are currently commented out.

**Configuration:** The `load-config` job reads `cloudformation/stack-config.json` for the stack name and template and parameter file paths. The build itself runs in the reusable workflow `subhamay-bhattacharyya-gha/cfn-ci-build-reusable-wf`, which owns the validation, deployment and cleanup steps.

**Environment setup:**

- Reads config from GitHub environment variables: `AWS_REGION`, `AWS_ACCOUNT_ID`, `OIDC_ROLE_NAME`, `CFN_TEMPLATES_S3_BUCKET`
- Uses AWS OIDC for keyless authentication
- Requires GitHub environment `ci` with OIDC trust configured

### `.github/workflows/release.yaml`

**Triggered:** On push to main

**Process:**

1. Analyze commits (conventional format: `feat:`, `fix:`, `BREAKING CHANGE:`)
2. Generate release notes
3. Update CHANGELOG.md
4. Create GitHub release and tag
5. Commit version bump

**Release rules:**

- `feat:` → MINOR bump (0.1.0 → 0.2.0)
- `fix:` → PATCH bump (0.1.0 → 0.1.1)
- `BREAKING CHANGE:` → MAJOR bump (0.1.0 → 1.0.0)
- Other commits → no release

## Testing & Validation

**Lint the template** (the same check the IDE runs, including informational rules):

```bash
cfn-lint cloudformation/template.yaml
cfn-lint --include-checks=I cloudformation/template.yaml
```

**Validate with AWS:**

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
```

**Manual stack deployment:**

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nested-kinesis-devl \
  --parameter-overrides file://cloudformation/parameters.json \
  --region us-east-1
```

## AWS Credentials & Environment Variables

**GitHub environment variables required in `ci` environment:**

- `AWS_REGION`: CloudFormation deployment region
- `AWS_ACCOUNT_ID`: AWS account to deploy into
- `OIDC_ROLE_NAME`: IAM role name for OIDC trust (uses `arn:aws:iam::{ACCOUNT_ID}:role/{ROLE_NAME}`)
- `CFN_TEMPLATES_S3_BUCKET`: S3 bucket where templates are stored

**OIDC setup:** The CI workflow uses AWS OIDC for keyless auth. The GitHub OIDC provider must trust the specified role.

## Conventional Commits & Release Flow

This repo enforces conventional commits to drive semantic versioning:

```bash
npx cz commit
```

Commit types:

- `feat: add support for X` → triggers MINOR release
- `fix: correct behavior of Y` → triggers PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits to `main` trigger releases. Feature branches use this format but releases happen on merge to main.

## When Modifying the Template

1. **Edit `cloudformation/template.yaml`**
2. **Update `cloudformation/parameters.json`** if parameters are added, renamed or removed. A stale key makes deployment fail.
3. **Update the parameter groups and labels** in the interface metadata when adding a parameter
4. **Lint and validate** with `cfn-lint` and `aws cloudformation validate-template`
5. **Update `README.md`** if parameters, outputs or behavior change
6. **Create a PR** with conventional commit message (e.g., `feat: add shard count parameter`)
7. **Merge to main** → release workflow creates version tag and GitHub release

## Dev Container

Pre-configured with:

- Node.js 20
- GitHub Copilot extension

Use via VS Code: `code --remote-container-url <repo-url>`

## Current Branch

Main branch is the release branch. Feature work branches from here and merges back via PR. Branch naming follows: `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-encryption`).
