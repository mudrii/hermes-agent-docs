# AWS Bedrock

Hermes Agent supports Amazon Bedrock as a native provider through the Bedrock Converse API. This is not the OpenAI-compatible adapter path. Use it when you want IAM authentication, native Bedrock model discovery, inference profiles, and Guardrails support.

## Prerequisites

- Install Bedrock support:

```bash
pip install hermes-agent[bedrock]
```

- Make AWS credentials available through the standard boto3 credential chain:
  - IAM role on EC2, ECS, or Lambda
  - `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
  - `AWS_PROFILE`
  - `aws configure`

- Grant at least these Bedrock permissions:
  - `bedrock:InvokeModel`
  - `bedrock:InvokeModelWithResponseStream`
  - `bedrock:ListFoundationModels`
  - `bedrock:ListInferenceProfiles`

## Quick start

```bash
hermes model
```

Choose `AWS Bedrock`, then select your region and model.

## Config example

```yaml
model:
  default: us.anthropic.claude-sonnet-4-6
  provider: bedrock
  base_url: https://bedrock-runtime.us-east-2.amazonaws.com

bedrock:
  region: us-east-2
```

## Region precedence

Hermes resolves the region in this order:

1. `bedrock.region` in `config.yaml`
2. `AWS_REGION`
3. `AWS_DEFAULT_REGION`
4. fallback default `us-east-1`

## Guardrails

You can attach Bedrock Guardrails globally:

```yaml
bedrock:
  region: us-east-2
  guardrail:
    guardrail_identifier: "abc123def456"
    guardrail_version: "1"
    stream_processing_mode: "async"
    trace: "disabled"
```

## Model discovery

Hermes discovers available Bedrock models from the control plane and prefers inference profile IDs in the model picker.

Typical examples:

- `us.anthropic.claude-sonnet-4-6`
- `us.anthropic.claude-opus-4-6-v1`
- `us.amazon.nova-pro-v1:0`
- `deepseek.v3.2`

## Diagnostics

```bash
hermes doctor
```

The doctor checks whether AWS credentials are available, whether `boto3` is installed, and whether the Bedrock control plane is reachable.

## Troubleshooting

### No AWS credentials found

Make sure one of the standard AWS credential sources is configured. On AWS compute, attaching an IAM role is usually enough.

### On-demand model invocation is not supported

Use an inference profile ID such as `us.anthropic.claude-sonnet-4-6` instead of a bare foundation model ID.

### ThrottlingException

You have hit the Bedrock quota for that model or inference profile. Hermes retries with backoff, but sustained failures usually require a quota increase in AWS.

