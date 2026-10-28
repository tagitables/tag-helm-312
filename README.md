# Custom Helm Chart — Kubernetes Deployment & Release Management

A hands-on implementation of a custom Helm chart for NGINX, demonstrating Kubernetes application deployment, Helm release management, infrastructure-as-code security scanning, and OCI-based chart publishing to Amazon Elastic Container Registry (ECR).

## Tech Stack

`Helm` · `Kubernetes` · `NGINX` · `Docker Desktop` · `Minikube` · `WSL2 Ubuntu` · `AWS ECR` · `AWS CLI` · `Snyk IaC` · `YAML` · `Git` · `GitHub`

## 1. Project Overview

This project demonstrates the creation, configuration, packaging, deployment, and lifecycle management of a custom Helm chart.

The implementation covers:

- Custom Helm chart creation using Kubernetes YAML templates.
- Kubernetes Deployment and LoadBalancer Service configuration.
- Helm installation, upgrade, rollback, and uninstallation.
- Application accessibility and deployment health verification.
- Infrastructure-as-code security scanning using Snyk.
- Helm chart packaging and publishing to private Amazon ECR.

## 2. Deployment Environment

### Amazon EKS Availability

The original activity specified deployment to Amazon EKS.

However, the previously configured classroom cluster, `shared-eks-cluster` in `us-east-1`, was no longer available during implementation.

A separate cluster, `cilium-eks-cluster`, was discovered in `ap-southeast-1`, but its intended purpose and deployment authorization were not established.

To avoid modifying an unfamiliar shared cluster or interfering with classmates' resources, the Kubernetes deployment was completed locally using **Docker Desktop with WSL2 integration and Minikube**.

### Local Kubernetes Environment

| Component | Configuration |
|---|---|
| Host environment | Windows 11 |
| Linux environment | WSL2 Ubuntu |
| Container platform | Docker Desktop |
| Kubernetes distribution | Minikube |
| Kubernetes version | v1.37.0 |
| Kubernetes context | `minikube` |
| Namespace | `tag-helm-312` |

**Deployment scope:** Kubernetes release-management operations were completed on Minikube. Helm chart publishing was completed on AWS ECR. No deployment to Amazon EKS was performed.

## 3. Repository Structure

```text
tag-helm-312/
├── .gitignore
├── README.md
└── tag-helm-app/
    ├── .helmignore
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── _helpers.tpl
        ├── deployment.yaml
        └── service.yaml
```

The chart contains the required Deployment and Service templates. Unnecessary default templates were removed to retain a minimal application configuration.

## 4. Helm Chart Configuration

| Property | Configuration |
|---|---|
| Chart name | `tag-helm-app` |
| Chart version | `1.0.0` |
| Application version | `1.0.0` |
| Container image | `nginx:latest` |
| Initial replica count | 2 |
| Service type | `LoadBalancer` |
| Service port | 80 |
| Container port | 80 |
| Image pull policy | `IfNotPresent` |

The chart uses Helm template expressions to configure the Deployment and Service using values defined in `values.yaml`.

### Chart Validation

```bash
helm lint ./tag-helm-app
```

Validation result:

```text
1 chart(s) linted, 0 chart(s) failed
```

## 5. Kubernetes Deployment

### Install the Helm Chart

From the repository root:

```bash
helm install tag-helm-app ./tag-helm-app \
  --create-namespace \
  --namespace tag-helm-312
```

### Verify Deployment

```bash
helm list -n tag-helm-312
kubectl get all -n tag-helm-312
```

The initial deployment successfully created:

- One Kubernetes Deployment.
- Two running NGINX Pods.
- One Kubernetes Service of type `LoadBalancer`.

The Deployment reported `2/2` available replicas.

### Access the Application

Because the application was deployed on Minikube rather than Amazon EKS, an external AWS Load Balancer was not provisioned.

Application access was verified through Kubernetes port forwarding:

```bash
kubectl port-forward \
  -n tag-helm-312 \
  svc/tag-helm-app-service \
  8080:80
```

Local endpoint:

```text
http://localhost:8080
```

The default NGINX welcome page was successfully displayed in the browser.

## 6. Helm Release Lifecycle

The release was upgraded, rolled back, and subsequently uninstalled.

| Revision | Operation | Replicas | Result |
|---|---|---|---|
| 1 | Initial installation | 2 | Successful |
| 2 | Helm upgrade | 3 | Successful |
| 3 | Rollback to revision 1 | 2 | Successful |

### Upgrade

The `replicaCount` in `values.yaml` was changed from `2` to `3`.

```bash
helm upgrade tag-helm-app ./tag-helm-app \
  -f ./tag-helm-app/values.yaml \
  --namespace tag-helm-312
```

Verification:

```bash
kubectl get deployments,pods -n tag-helm-312
```

The Deployment reported `3/3` available replicas, with all three NGINX Pods running.

### Rollback

```bash
helm rollback tag-helm-app 1 \
  -n tag-helm-312
```

Release history:

```bash
helm history tag-helm-app \
  -n tag-helm-312
```

The rollback created revision 3 and restored the original two-replica configuration.

### Uninstall

```bash
helm uninstall tag-helm-app \
  -n tag-helm-312
```

Result:

```text
release "tag-helm-app" uninstalled
```

The Helm release was successfully removed from the local Kubernetes environment.

## 7. Infrastructure-as-Code Security Scanning

Snyk IaC was used to inspect the rendered Kubernetes manifests.

### Render Helm Templates

From the chart directory:

```bash
helm template tag-helm-app . \
  --output-dir ./output
```

### Run Snyk Scan

```bash
snyk iac test ./output
```

### Scan Results

| Severity | Findings |
|---|---|
| Critical | 0 |
| High | 0 |
| Medium | 4 |
| Low | 6 |
| **Total** | **10** |

The scan identified opportunities to improve container security configuration, resource management, health checks, and LoadBalancer access restrictions.

Examples include:

- Explicit non-root container execution.
- Disabling privilege escalation.
- Dropping unnecessary Linux capabilities.
- Defining CPU and memory limits.
- Configuring liveness probes.
- Restricting LoadBalancer source ranges.

These findings were documented but not remediated, as the implementation retained the original assignment configuration.

## 8. Helm Chart Packaging and Amazon ECR

The custom Helm chart was packaged and successfully published to a private Amazon ECR repository.

| Property | Configuration |
|---|---|
| AWS service | Amazon ECR |
| AWS region | `us-east-1` |
| Repository | `tag-helm-app` |
| Chart version | `1.0.0` |
| Registry visibility | Private |
| Publishing format | OCI Helm artifact |
| Publication status | Successful |

### Package the Chart

From the chart directory:

```bash
helm package .
```

Generated artifact:

```text
tag-helm-app-1.0.0.tgz
```

### Authenticate Helm to ECR

The registry endpoint uses `<MY_AWS_ACCOUNT_ID>` as a placeholder rather than exposing the classroom AWS account identifier.

```bash
aws ecr get-login-password \
  --region us-east-1 | \
helm registry login \
  --username AWS \
  --password-stdin \
  <MY_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

### Push the Helm Chart

```bash
helm push tag-helm-app-1.0.0.tgz \
  oci://<MY_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/
```

The chart was successfully published to:

```text
<MY_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/tag-helm-app:1.0.0
```

Published artifact digest:

```text
sha256:1af7ae86204ef87e6c7a91e883c85758e22d47c3f159d4ecbc3d91e1df9cd66f
```

The published chart can be retrieved from the private registry by an authenticated user with appropriate ECR permissions.

## 9. Implementation Outcome

This project successfully demonstrated the custom Helm chart lifecycle, from YAML-based application configuration to Kubernetes deployment, scaling, rollback, security scanning, and OCI artifact publication.

**Completed:**
- Helm chart creation and validation.
- Local Kubernetes deployment using Docker Desktop and Minikube.
- NGINX application verification.
- Helm upgrade from two to three replicas.
- Rollback to the original two-replica configuration.
- Helm release uninstallation.
- Snyk IaC security assessment.
- Helm chart packaging and private Amazon ECR publication.

**Environment limitation:** The original classroom Amazon EKS cluster was unavailable. Minikube was therefore used to demonstrate the Kubernetes and Helm operations in an isolated local environment. This does not constitute an Amazon EKS deployment.

The project illustrates how Helm supports repeatable application deployment and release management across compatible Kubernetes environments.
