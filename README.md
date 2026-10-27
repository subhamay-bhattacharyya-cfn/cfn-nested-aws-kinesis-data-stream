# CloudFormation Nested Stack Template : Kinesis Data Stream

<!-- Row 1: Status - Most Important -->
[![Release](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream/actions/workflows/ci.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)&nbsp;[![GitHub Repo](https://img.shields.io/badge/GitHub-Repository-blue?logo=github)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/github/languages/top/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazon&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-kinesis-data-stream/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/39107784178fdf53c4082802434203a8/raw/cfn-nested-aws-kinesis-data-stream.json)](https://gist.github.com/subhamay-bhattacharyya/39107784178fdf53c4082802434203a8)

## Overview

[`cloudformation/template.yaml`](cloudformation/template.yaml) is a reusable nested stack template that creates a Kinesis Data Stream with configurable capacity mode, retention and KMS encryption. It is designed to be invoked from a parent/root CloudFormation stack.

### Features

- ✅ ON_DEMAND or PROVISIONED capacity mode
- ✅ Configurable retention period (24 to 8760 hours)
- ✅ Optional KMS encryption (key ID, key ARN, alias name or alias ARN; empty disables encryption)
- ✅ Stream name built from project, base name, account ID, environment and region
- ✅ Optional CI suffix for unique deployments
- ✅ `DeletionPolicy` and `UpdateReplacePolicy` set to `Delete`

## Parameters

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `ProjectName` | String | `cfn-nested` | Stream name prefix (lowercase letters, numbers, hyphens; max 20) |
| `StreamBaseName` | String | `cfn-nested-stream` | Base name for the stream |
| `environment` | String | `devl` | Deployment environment (`devl`, `prod`, `test`) |
| `CiSuffix` | String | `""` | Optional CI suffix appended to the stream name |
| `StreamMode` | String | `ON_DEMAND` | `ON_DEMAND` or `PROVISIONED` |
| `ShardCount` | Number | `1` | Number of shards (used only when `StreamMode` is `PROVISIONED`) |
| `RetentionPeriodHours` | Number | `24` | Retention period in hours (24 to 8760) |
| `KmsKeyId` | String | `alias/aws/kinesis` | KMS key ID, key ARN, alias (`alias/...`) or alias ARN. Leave empty to disable encryption |

## Outputs

- `KinesisStreamName` — Name of the Kinesis Data Stream
- `KinesisStreamArn` — ARN of the Kinesis Data Stream

## Stream Naming Convention

```bash
{ProjectName}-{StreamBaseName}-{AccountId}-{Environment}-{Region}[-{CiSuffix}]
```

Example: `cfn-nested-cfn-stream-123456789012-devl-us-east-1`

## Usage

### 1. Upload the Template to S3

Parent stacks reference nested templates by URL, so upload the template to an S3 bucket first:

```bash
aws s3 cp cloudformation/template.yaml s3://your-cfn-templates-bucket/templates/kinesis-stream.yaml
```

### 2. Reference from a Parent Stack

```yaml
KinesisNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-templates-bucket/templates/kinesis-stream.yaml
    Parameters:
      ProjectName: cfn-nested
      StreamBaseName: cfn-stream
      environment: !Ref Environment
      StreamMode: ON_DEMAND
      KmsKeyId: alias/aws/kinesis
    Tags:
      - Key: Environment
        Value: !Ref Environment

Outputs:
  StreamName:
    Value: !GetAtt KinesisNestedStack.Outputs.KinesisStreamName
  StreamArn:
    Value: !GetAtt KinesisNestedStack.Outputs.KinesisStreamArn
```

### 3. Deploy Directly with AWS CLI

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nested-kinesis-devl \
  --parameter-overrides file://cloudformation/parameters.json \
  --region us-east-1
```

## Best Practices Implemented

- ✅ Explicit `DeletionPolicy` and `UpdateReplacePolicy` on the stateful stream resource
- ✅ KMS encryption enabled by default, with validated key ID, ARN and alias inputs
- ✅ Parameter groups and labels in the CloudFormation console
- ✅ Export-free outputs, so values are passed to parent stacks explicitly
- ✅ Stream name prefixed with project, account ID, environment and region

## License

MIT
