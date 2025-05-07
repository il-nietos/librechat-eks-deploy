# librechat-eks-deploy

This repository contains a Helm chart to deploy [LibreChat](https://github.com/danny-avila/LibreChat) and its MongoDB backend into a Kubernetes cluster, specifically targeting AWS EKS deployments.

It is part of a larger project to self-host multi-model AI chat interfaces without relying on centralized SaaS infrastructure. See [librechat-eks-infra](https://github.com/your-org/librechat-eks-infra) for infrastructure provisioning using Terraform.

---

## Repository Structure

librechat-eks-deploy
├── .github/workflows # CI workflow for Helm packaging and publishing
├── librechat-chart # Helm chart for LibreChat + MongoDB
│ ├── envs # Per-environment values
│ ├── templates # Kubernetes resources
│ └── values.yaml # Base values


---

## Overview

The Helm chart deploys:

- **LibreChat**: The frontend and backend React and Node.js application
- **MongoDB**: Deployed as a subchart from Bitnami (version `13.6.5`)
- **Kubernetes Service**: Exposes LibreChat externally via a `LoadBalancer`-type service. This service is annotated for automatic NLB provisioning by the AWS Load Balancer Controller.

Each environment (`dev`, `stage`, `prod`) is configured through its corresponding `values-{env}.yaml` file. The differences are:

| Parameter         | `dev`        | `stage`      | `prod`       |
|------------------|--------------|--------------|--------------|
| MongoDB storage  | `8Gi`        | `20Gi`       | `50Gi`       |
| maxReplicas      | `2`          | `3`          | `5`          |

---

## Secrets and Configuration

Sensitive runtime values (e.g., API keys for external LLMs) are injected into the deployment through GitHub Actions using repository secrets and Helm `--set` overrides.

The chart includes templated `ConfigMap` and `Secret` definitions which populate the runtime configuration of LibreChat from both the environment-specific values and secure variables.

---

## AWS Integration

The `Service` created for LibreChat is of type `LoadBalancer`, and includes annotations to ensure the AWS Load Balancer Controller automatically provisions a Network Load Balancer (NLB) and attaches it to the service. This controller must be installed in the target EKS cluster (see librechat-eks-infra).

The MongoDB pod uses a persistent volume provisioned via EBS. This requires the EBS CSI driver, which must also be installed in the cluster. When the MongoDB StatefulSet requests persistent storage, the CSI driver provisions and attaches an EBS volume automatically.

## CI/CD Pipeline
The `.github/workflows/helmchart.yml` workflow:

1. Packages the Helm chart.

2. Tags it according to the latest commit or version.

3. Pushes it to the GitHub Container Registry under ghcr.io/il-nietos/librechat-chart.

This Helm chart is then consumed by the librechat-eks-infra Terraform repository, which pulls the chart and installs it into the appropriate namespace in EKS using a Helm release per environment.

## Namespaces and Environment Management
Each environment is deployed in a dedicated Kubernetes namespace (librechat-dev, librechat-prod, etc.). These namespaces are expected to exist prior to Helm installation. The namespace creation is typically handled by the Terraform setup in the infra repo.

Customising the Chart
You can override configuration parameters using your own values.yaml or by setting CLI flags. Examples:

# Install into an existing namespace

```helm upgrade --install librechat ./librechat-chart \
  --namespace librechat-dev \
  -f librechat-chart/envs/values-dev.yaml \
  --set librechat.apiKey=$OPENAI_API_KEY
```


## Assumptions and Requirements
You are deploying into an EKS cluster with the AWS Load Balancer Controller installed and IAM permissions configured

The EBS CSI driver is installed in the cluster

Namespaces per environment are pre-created

This Helm chart is not meant to be deployed outside the context of its infrastructure companion repo