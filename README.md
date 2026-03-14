# Pipeline Factory

A centralized, secure set of GitHub reusable workflows and composite actions that hundreds of services can consume to lint, test, scan, build, push, and optionally deploy containerized apps without managing cloud secrets.

## How it works

- **OIDC Authentication**: Uses GitHub OIDC to assume AWS roles only in AWS-bound jobs (no long-lived credentials)
- **Security Scanning**: CodeQL SARIF results and Trivy scans fail the workflow on HIGH/CRITICAL findings
- **Container Build**: Build and push images to ECR with GitHub Actions cache-backed Docker layers
- **Reusable Components**: Composite actions for common operations

## Repository Structure

```
pipeline-factory/
  .github/
    actions/
      aws-oidc-credentials/
        action.yml
      trivy-scan/
        action.yml
    workflows/
      reusable-aws-cicd.yml
    dependabot.yml
  README.md
```

## Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `aws-role-to-assume` | string | ✅ | - | ARN of IAM role to assume via OIDC |
| `aws-region` | string | ❌ | `us-east-1` | AWS region for operations |
| `ecr-repository-name` | string | ✅ | - | Target ECR repository name |
| `dockerfile-path` | string | ❌ | `./Dockerfile` | Path to Dockerfile |
| `context` | string | ❌ | `.` | Docker build context |
| `image-tag` | string | ❌ | - | Optional explicit image tag (defaults to short SHA) |
| `codeql-languages` | string | ❌ | `javascript,python,go` | Comma-separated list of CodeQL languages |
| `test-command` | string | ❌ | `''` | Optional test command to run |
| `enable-ecs-deploy` | boolean | ❌ | `false` | If true, update ECS service to new image |
| `ecs-cluster` | string | ❌ | - | ECS cluster name (required if deploy enabled) |
| `ecs-service` | string | ❌ | - | ECS service name (required if deploy enabled) |
| `ecs-container-name` | string | ❌ | - | Container name in task definition (required if deploy enabled) |

## Outputs

| Output | Description |
|--------|-------------|
| `image_tag` | Final image tag used |
| `image_uri` | ECR image URI with tag (`account.dkr.ecr.region.amazonaws.com/repo:tag`) |
| `image_digest` | Pushed image digest |

## Secrets

- No static AWS secrets required for AWS authentication; the workflow uses OIDC only
- Callers should not use `secrets: inherit` by default
- If tests need secrets, pass only the specific secrets required by the caller workflow

## Usage Example

```yaml
name: CI/CD

on:
  push:
    branches: [ main ]

permissions:
  contents: read
  id-token: write
  security-events: write

jobs:
  build-and-deploy:
    uses: your-github-org/pipeline-factory/.github/workflows/reusable-aws-cicd.yml@<immutable-release-sha>
    with:
      aws-role-to-assume: arn:aws:iam::123456789012:role/PipelineFactoryDeployerRole
      aws-region: us-east-1
      ecr-repository-name: my-web-app
      dockerfile-path: ./Dockerfile
      context: .
      test-command: 'npm ci && npm test'
      enable-ecs-deploy: false
```

### Deploy to ECS

To enable ECS deployment, add these parameters:

```yaml
with:
  enable-ecs-deploy: true
  ecs-cluster: my-cluster
  ecs-service: my-service
  ecs-container-name: web
```

## Security Posture

- **No long-lived credentials**: Uses OIDC for AWS authentication and only requests AWS credentials in build/deploy jobs
- **Mandatory security scans**: CodeQL and Trivy both run before release or deployment
- **Fail-fast enforcement**: The workflow parses CodeQL SARIF locally and fails on HIGH/CRITICAL findings; Trivy also exits non-zero on HIGH/CRITICAL findings
- **Immutable dependencies**: Third-party actions are pinned to full commit SHAs to reduce supply-chain risk
- **Standardized pipeline**: Reduces configuration drift across services

## Prerequisites

- GitHub organization with Actions enabled
- AWS account with:
  - OIDC identity provider configured for `token.actions.githubusercontent.com`
  - IAM role permitting ECR push and optional ECS deploy
- ECR repository per service or permissions to create them

## Versioning

- Create release tags after testing (e.g., `v1.0.0`)
- Create moving major tags `v1` pointing to the latest stable release
- Prefer pinning callers to an immutable release commit SHA
- Use moving tags like `@v1` only when you explicitly accept the trade-off of automatic updates

## Contributing

1. Make changes to workflows or actions
2. Test thoroughly with sample applications
3. Create a release tag
4. Update the moving major tag

## License

[Add your license here]
