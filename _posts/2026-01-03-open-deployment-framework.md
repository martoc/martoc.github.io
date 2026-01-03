---
title: Building an Open Deployment Framework with GitHub Actions
subtitle: A modular approach to CI/CD for cloud-native applications
layout: post
author: martoc
image: https://martoc.github.io/blog/images/github.webp
---

Managing CI/CD pipelines across multiple repositories can quickly become unwieldy. Each project needs versioning, container builds, deployments, and releases—often with subtle variations that lead to duplicated workflow code. This post introduces an open deployment framework built entirely on GitHub Actions, designed to bring consistency and reusability to cloud-native deployments.

## The Problem

When scaling from a handful of repositories to dozens, maintaining individual CI/CD configurations becomes a significant burden:

- **Duplication**: Similar workflow logic copied across repositories
- **Inconsistency**: Different versioning schemes, build processes, and deployment patterns
- **Maintenance**: Updating a common process requires changes in every repository
- **Onboarding**: New projects require significant setup effort

## The Solution: A Modular Framework

The framework addresses these challenges through three layers of abstraction:

```text
┌─────────────────────────────────────────────────────────┐
│                  Reusable Workflows                     │
│  (workflow-github-actions, workflow-infrastructure,     │
│   workflow-container-image, workflow-helm-chart,        │
│   workflow-workload)                                    │
├─────────────────────────────────────────────────────────┤
│                   GitHub Actions                        │
│  (action-tag, action-release, action-container-build,   │
│   action-helm-build, action-deploy, etc.)               │
├─────────────────────────────────────────────────────────┤
│                  Tool Containers                        │
│  (container-terraform-tools, container-cloudformation-  │
│   tools)                                                │
└─────────────────────────────────────────────────────────┘
```

## Tool Containers

At the foundation are purpose-built Docker images containing validated tool combinations.

### Terraform Tools

The `container-terraform-tools` image provides a complete Terraform development environment:

| Tool | Purpose |
|------|---------|
| Terraform 1.14.0 | Infrastructure as code |
| TFLint 0.54.0 | Configuration linting |
| Checkov 3.2.495 | Security scanning |

```bash
docker run --rm -v $(pwd):/workspace -w /workspace \
  martoc/terraform-tools:latest terraform validate
```

### CloudFormation Tools

Similarly, `container-cloudformation-tools` bundles AWS CloudFormation tooling:

| Tool | Purpose |
|------|---------|
| Rain 1.8.5 | Template management |
| cfn-lint 1.34.0 | Template validation |
| Checkov 3.2.495 | Security scanning |

```bash
docker run --rm -v $(pwd):/workspace -w /workspace \
  martoc/cloudformation-tools:latest cfn-lint template.yaml
```

## GitHub Actions

The second layer provides individual, composable actions for specific CI/CD tasks.

### Versioning: action-tag

Automatically calculates semantic versions from [Conventional Commits](https://www.conventionalcommits.org/):

```yaml
- uses: martoc/action-tag@v0
```

Features:
- Automatic version calculation from commit messages
- Floating tags for major/minor versions (v1, v1.2)
- SemVer and PEP440 format support
- Release candidate tags for pull requests

### Container Operations

**action-container-build** supports multi-registry, multi-architecture builds:

```yaml
- uses: martoc/action-container-build@v0
  with:
    registry: gcp  # or docker.io, aws
    region: europe-west2
    gcp_project_id: my-project
    platforms: linux/arm64,linux/amd64
```

**action-container-distribute** copies images between registries, useful for promoting images across environments:

```yaml
- uses: martoc/action-container-distribute@v0
  with:
    source_registry: gcp
    source_region: us-central1
    target_registry: gcp
    target_region: europe-west1
    container_image: myapp:1.0.0
```

### Helm Chart Operations

**action-helm-build** packages and pushes Helm charts to OCI registries:

```yaml
- uses: martoc/action-helm-build@v0
  with:
    registry: gcp
    region: europe-west2
    repository_name: helm-charts
    gcp_project_id: my-project
```

**action-helm-deploy** deploys charts to Kubernetes clusters:

```yaml
- uses: martoc/action-helm-deploy@v0
  with:
    registry: gcp
    chart_name: my-application
    chart_version: 1.0.0
    chart_value_file: values/production.yaml
```

### AWS Deployment: action-deploy

Handles complete AWS deployments including ECR repository creation and CloudFormation stack deployment:

```yaml
- uses: martoc/action-deploy@v0
  with:
    region: us-east-2
    environment: production
    workload-name: my-service
```

## Reusable Workflows

The top layer combines actions into complete, opinionated workflows that can be called with minimal configuration.

### Basic CI/CD: workflow-github-actions

For simple projects needing only versioning and releases:

```yaml
jobs:
  publish:
    uses: martoc/workflow-github-actions/.github/workflows/publish.yml@v0
    permissions:
      contents: write
```

Prerequisites: A Makefile with `init` and `validate` targets.

### Infrastructure: workflow-infrastructure

Terraform and CloudFormation validation workflows using containerised tools:

```yaml
jobs:
  terraform:
    uses: martoc/workflow-infrastructure/.github/workflows/terraform.yml@v0
    permissions:
      contents: write
    secrets: inherit
```

The workflow runs validation inside the appropriate tool container, ensuring consistent tool versions across all infrastructure projects.

### Container Images: workflow-container-image

Complete container build pipelines for Docker Hub or GCP:

```yaml
jobs:
  build:
    uses: martoc/workflow-container-image/.github/workflows/gcp.yml@v0
    with:
      region: europe-west2
      repository_name: containers
      gcp_project_id: my-project
      workload_identity_provider: ${{ vars.WIF_PROVIDER }}
      service_account: ${{ vars.SERVICE_ACCOUNT }}
    permissions:
      contents: write
      id-token: write
```

### Helm Charts: workflow-helm-chart

Build and deploy workflows for Helm charts:

```yaml
jobs:
  build:
    uses: martoc/workflow-helm-chart/.github/workflows/build.yml@v0
    with:
      registry: gcp
      region: europe-west2
      repository_name: helm-charts
      gcp_project_id: my-project
```

### Application Workloads: workflow-workload

Complete build-test-deploy pipelines for Go and Node.js applications:

```yaml
jobs:
  build-and-deploy:
    uses: martoc/workflow-workload/.github/workflows/go.yml@v0
    with:
      region: eu-west-1
      environment: production
      workload-name: my-service
      workload-type: lambda
    secrets: inherit
```

This workflow handles:
1. Semantic versioning
2. Build and test execution
3. Code coverage upload to Codecov
4. Multi-architecture container builds
5. GitHub release creation
6. CloudFormation-based deployment to AWS

## Design Principles

### Make-Based Conventions

All workflows expect projects to implement standard Makefile targets:

```makefile
.PHONY: init validate build

init:
    # Install dependencies

validate:
    # Run linting and validation

build:
    # Build the project
```

This convention allows workflows to remain generic whilst projects define their specific tooling.

### Workload Identity Federation

GCP integrations use Workload Identity Federation rather than service account keys, following security best practices:

```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: ${{ vars.WIF_PROVIDER }}
    service_account: ${{ vars.SERVICE_ACCOUNT }}
```

### Multi-Cloud Support

Actions support multiple cloud providers where applicable:
- **Container registries**: Docker Hub, AWS ECR, GCP Artifact Registry
- **Deployments**: AWS (CloudFormation), GCP (GKE via Helm)

## Getting Started

To adopt the framework for a new project:

1. **Create a Makefile** with `init` and `validate` targets
2. **Choose the appropriate workflow** based on project type
3. **Configure secrets** for your target registries/cloud providers
4. **Reference the reusable workflow** in your repository's workflow file

Example for a Terraform module:

```yaml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    uses: martoc/workflow-infrastructure/.github/workflows/terraform.yml@v0
    permissions:
      contents: write
    secrets: inherit
```

## Repository Links

All components are open source and available on GitHub:

**Tool Containers**
- [container-terraform-tools](https://github.com/martoc/container-terraform-tools)
- [container-cloudformation-tools](https://github.com/martoc/container-cloudformation-tools)

**GitHub Actions**
- [action-tag](https://github.com/martoc/action-tag)
- [action-release](https://github.com/martoc/action-release)
- [action-container-build](https://github.com/martoc/action-container-build)
- [action-container-distribute](https://github.com/martoc/action-container-distribute)
- [action-helm-build](https://github.com/martoc/action-helm-build)
- [action-helm-distribute](https://github.com/martoc/action-helm-distribute)
- [action-helm-deploy](https://github.com/martoc/action-helm-deploy)
- [action-deploy](https://github.com/martoc/action-deploy)
- [action-setup-mo](https://github.com/martoc/action-setup-mo)

**Reusable Workflows**
- [workflow-github-actions](https://github.com/martoc/workflow-github-actions)
- [workflow-infrastructure](https://github.com/martoc/workflow-infrastructure)
- [workflow-container-image](https://github.com/martoc/workflow-container-image)
- [workflow-helm-chart](https://github.com/martoc/workflow-helm-chart)
- [workflow-workload](https://github.com/martoc/workflow-workload)

## Conclusion

This framework demonstrates how GitHub Actions' reusable workflow feature can eliminate CI/CD duplication whilst maintaining flexibility. By layering tool containers, individual actions, and complete workflows, projects can adopt as much or as little of the framework as needed.

The key benefits are consistency across repositories, reduced maintenance burden, and faster onboarding for new projects. All whilst remaining fully open source and customisable.
