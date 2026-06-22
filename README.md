# Spring Boot Dummy Application

This is a minimal Spring Boot web application that uses Thymeleaf to render an HTML page displaying the current environment name (read from `application.properties`) and the application version injected from `pom.xml` at build time using Maven resource filtering (`@project.version@`). Its primary purpose is to serve as a deployable target for demonstrating a GitOps workflow with ArgoCD, Kustomize, and Amazon ECR. The application is intentionally simple so the focus stays on the CI/CD pipeline and deployment automation.

## Pipeline Overview

Two CI pipelines are defined in this repository. Both share the same build and publish steps but differ in how they update the target GitOps repository.

### Pipeline 1: `.github/workflows/update_dev_kustomize.yml` — Kustomize-based deployment

This pipeline consists of two sequential jobs that run on every push or pull request to the `dev` branch.

**Job 1 — `build_scan_publish`:** Compiles, tests, and packages the application. It checks out the source, sets up JDK 17 with Maven caching, runs `mvn clean verify`, and scans both the source code and its dependencies with Trivy for HIGH and CRITICAL vulnerabilities. After passing the scan, it authenticates to AWS using encrypted GitHub Secrets and logs Docker into the private Amazon ECR registry. It extracts the application version from `pom.xml` (for example `0.0.7`) to use as the Docker image tag, builds a multi-stage Docker image, runs Trivy again on the final container image, and pushes it to ECR.

**Job 2 — `update_deployment_yaml_for_dev_overlay`:** Runs after the first job succeeds. It clones the infrastructure-as-code repository (`https://github.com/kolyaiks/argo-deployment-demo-infra-kustomize) at the `dev` branch using a fine-grained Personal Access Token, installs `kustomize`, re-authenticates to ECR, and runs `kustomize edit set image` in the `overlays/dev` directory to update the image reference to the newly pushed version. It then commits the change with a message such as "Bumping application version to 0.0.7" and pushes to the `dev` branch of the infra repo. This push triggers ArgoCD (via webhook or polling) to sync the new desired state to the Kubernetes cluster.

### Pipeline 2: `.github/workflows/update_dev_helm.yml` — Helm-based deployment (App-of-Apps)

This pipeline also consists of two sequential jobs. The first job is identical to Pipeline 1's build step.

**Job 1 — `build_scan_publish`:** Same as in Pipeline 1 — compiles, tests, scans, builds, and pushes a Docker image to ECR.

**Job 2 — `update_deployment_yaml_for_dev_env`:** Runs after the first job succeeds. It clones the GitOps repository `https://github.com/kolyaiks/argo-deployment-demo-gitops` at the `dev` branch using a fine-grained Personal Access Token, re-authenticates to ECR, and runs `sed` on `workloads/argo-deployment-demo-app/values-dev.yaml` to update the Helm values `image.tag` to the newly pushed version. It then commits the change with a message such as "Bumping application version to 0.0.7" and pushes to the `dev` branch of the GitOps repo. This push triggers ArgoCD (via its Application-of-Applications pattern) to sync the Helm-based deployment to the Kubernetes cluster.

## How to Run / Reproduce Locally

Build the Docker image:

```bash
docker build -t argo-deployment-demo-app .
```

Run the container:

```bash
docker run -p 8080:8080 argo-deployment-demo-app
```

Open [http://localhost:8080](http://localhost:8080) in your browser. You should see a page similar to this:

![Default output](https://github.com/kolyaiks/argo-deployment-demo-app/blob/main/argo-deployment-demo-app.drawio.png)

## Security Choices & Rationale

**Multi-stage Docker build.** The build stage uses `maven:3-openjdk-17-slim` which includes build tools and dependencies, while the runtime stage uses `amazoncorretto:17` which is a minimal JRE-only image. This reduces the final image size and attack surface by eliminating unnecessary tools and libraries from the build environment.

**Trivy security scanning.** Trivy is run on both the source code and dependencies (shift-left) and on the final container image. This catches vulnerabilities early in the pipeline, before they reach production.

**Non-blocking scans.** The scans report findings but do not fail the build (`exit-code: 0`). For this demo project this is a pragmatic choice — the team can see issues without blocking deployment velocity. In a production environment you would likely set `exit-code: 1` for CRITICAL findings and gate the pipeline.

**AWS credentials via GitHub Secrets.** Access keys and region are stored as encrypted GitHub Secrets and are never hardcoded in source files or visible in logs.

**PAT-scoped infra repo access.** The pipeline uses a dedicated Personal Access Token with minimal permissions — only write access to the single infrastructure repository. This follows the principle of least privilege and limits the blast radius if the token is compromised.

**Amazon Corretto base image.** An official, AWS-maintained OpenJDK distribution with long-term security support and regular patching. Using a trusted base image reduces supply-chain risk.

**Separation of application and infrastructure repositories.** The application source lives in this repository, while the Kubernetes manifests live in a separate repository (`https://github.com/kolyaiks/argo-deployment-demo-infra-kustomize/tree/feature/app-of-apps/k8s-yaml/app/argo-deployment-demo-app` or `https://github.com/kolyaiks/argo-deployment-demo-gitops`). This decouples application development from infrastructure changes and aligns with the GitOps pattern where ArgoCD watches the infrastructure repository.
