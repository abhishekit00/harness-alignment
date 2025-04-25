# 🚀 End-to-End CI/CD Pipeline Using Harness for GKE-Based Microservices

## 📌 Overview

In this project, I’ve created a generic and reusable CI/CD pipeline setup using [Harness](https://harness.io/). The main idea was to design something flexible that can be used across multiple microservices with minimal changes.

This setup allows me to:

- Build and push Docker images to Google Artifact Registry (GAR)
- Deploy to Google Kubernetes Engine (GKE)
- Use both **Rolling** and **Blue-Green** deployment strategies
- Handle deployments for both **QA** and **Production** environments
- Optionally push the same image to multiple registries
- Pass inputs dynamically so the pipeline works for any microservice

---

## 📁 Folder Structure
```
/harness/
├── ci-pipeline.yaml                # CI pipeline to build & push Docker image
├── cd-pipeline-rolling.yaml       # CD pipeline for rolling deployment
├── cd-pipeline-bluegreen.yaml     # CD pipeline for blue-green deployment
├── trigger.yaml                   # Trigger: CI → CD post success
├── manifests/
│   ├── deployment.yaml            # Generic K8s deployment manifest
│   └── service.yaml               # LoadBalancer service definition
```

---

## ⚙️ CI Pipeline: Build & Push Docker Images
Manually triggered pipeline (`ci_build_push`) that:

- Accepts GitHub repo name, branch, registry host, and GCP project ID as inputs
- Builds the Docker image
- Pushes to **Primary Registry** and optionally to **Secondary Registry**
- Tags: `<+pipeline.sequenceId>`, `latest`

### 🔁 Multi-Registry Capability
Pushes the same image to multiple Google Artifact Registries with runtime inputs for connector, host, and project ID.

```yaml
[pipeline ci_build_push]
pipeline:
  name: CI-Build-Push
  identifier: ci_build_push
  projectIdentifier: Harnessdemo
  orgIdentifier: default
  tags: {}

  properties:
    ci:
      codebase:
        connectorRef: githubconnector
        repoName: <+input>
        build:
          type: branch
          spec:
            branch: <+input>

  stages:
    - stage:
        name: Build & Push Image
        identifier: BuildPush
        type: CI
        spec:
          cloneCodebase: true
          platform:
            os: Linux
            arch: Amd64
          runtime:
            type: Cloud
            spec: {}
          execution:
            steps:
              - step:
                  name: Build & Push to GAR (Primary)
                  identifier: BuildPushPrimary
                  type: BuildAndPushGAR
                  spec:
                    connectorRef: gcpconnector
                    host: us-east1-docker.pkg.dev
                    projectID: harness-457605
                    imageName: harness-demo/todo
                    tags:
                      - <+pipeline.sequenceId>
                      - latest
              - step:
                  name: Push to Secondary Registry (Optional)
                  identifier: PushSecondary
                  type: BuildAndPushGAR
                  when:
                    stageStatus: Success
                    condition: <+input>.isSecondary == true
                  spec:
                    connectorRef: gcpconnector_secondary
                    host: europe-docker.pkg.dev
                    projectID: harness-457605
                    imageName: harness-demo/todo
                    tags:
                      - <+pipeline.sequenceId>
                      - latest
```

---

## 🚀 CD Pipelines: Deploy to GKE
Triggered automatically after a successful CI run.

### 🧩 Rolling Deployment (Default)
- Uses `K8sRollingDeploy` to incrementally replace pods
- Stages:
  - **QA** deployment to `qa_namespace`
  - **PROD** deployment to `prod_namespace`
- Safe and zero-downtime

```yaml
[pipeline cd_rolling_deploy]
pipeline:
  name: CD-Rolling-Deploy
  identifier: cd_rolling_deploy
  projectIdentifier: Harnessdemo
  orgIdentifier: default
  stages:
    - stage:
        name: Deploy to QA
        identifier: deploy_qa
        type: Deployment
        spec:
          deploymentType: Kubernetes
          service:
            serviceRef: <+input>
          environment:
            environmentRef: qa_env
            infrastructureDefinitions:
              - identifier: qa_infra
          execution:
            steps:
              - step:
                  name: Rolling Deploy to QA
                  identifier: rolling_qa
                  type: K8sRollingDeploy
                  spec:
                    skipDryRun: false
    - stage:
        name: Deploy to PROD
        identifier: deploy_prod
        type: Deployment
        spec:
          deploymentType: Kubernetes
          service:
            serviceRef: <+input>
          environment:
            environmentRef: prod_env
            infrastructureDefinitions:
              - identifier: prod_infra
          execution:
            steps:
              - step:
                  name: Rolling Deploy to PROD
                  identifier: rolling_prod
                  type: K8sRollingDeploy
                  spec:
                    skipDryRun: false
```

### 🔄 Blue-Green Deployment (Optional)
```yaml
[pipeline cd_bluegreen_deploy]
pipeline:
  name: CD-BlueGreen-Deploy
  identifier: cd_bluegreen_deploy
  projectIdentifier: Harnessdemo
  orgIdentifier: default
  stages:
    - stage:
        name: Deploy to QA (BG)
        identifier: deploy_qa_bg
        type: Deployment
        spec:
          deploymentType: Kubernetes
          service:
            serviceRef: <+input>
          environment:
            environmentRef: qa_env
            infrastructureDefinitions:
              - identifier: qa_infra
          execution:
            steps:
              - step:
                  name: BlueGreen Deploy
                  identifier: bluegreen_deploy_qa
                  type: K8sBlueGreenDeploy
                  spec:
                    skipDryRun: false
              - step:
                  name: Swap Services
                  identifier: swap_services_qa
                  type: K8sBGSwapServices
                  spec: {}
    - stage:
        name: Deploy to PROD (BG)
        identifier: deploy_prod_bg
        type: Deployment
        spec:
          deploymentType: Kubernetes
          service:
            serviceRef: <+input>
          environment:
            environmentRef: prod_env
            infrastructureDefinitions:
              - identifier: prod_infra
          execution:
            steps:
              - step:
                  name: BlueGreen Deploy
                  identifier: bluegreen_deploy_prod
                  type: K8sBlueGreenDeploy
                  spec:
                    skipDryRun: false
              - step:
                  name: Swap Services
                  identifier: swap_services_prod
                  type: K8sBGSwapServices
                  spec: {}
```

---

## 🔁 CI → CD Trigger
```yaml
  trigger:
  name: CI-to-CD Trigger
  identifier: trigger_citocd
  projectIdentifier: Harnessdemo
  orgIdentifier: default
  enabled: true
  source:
    type: Pipeline
    spec:
      pipeline: ci_build_push
      stagesToExecute: [BuildPush]
  inputYaml: |
    pipeline:
      identifier: cd_rolling_deploy
      stages:
        - stage:
            identifier: deploy_qa
            spec:
              service:
                serviceRef: todo-service
              environment:
                environmentRef: qa_env
                infrastructureDefinitions:
                  - identifier: qa_infra
        - stage:
            identifier: deploy_prod
            spec:
              service:
                serviceRef: todo-service
              environment:
                environmentRef: prod_env
                infrastructureDefinitions:
                  - identifier: prod_infra

```

---

## 📦 Kubernetes Manifests
### 🧾 deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <+service.name>
  labels:
    app: <+service.name>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <+service.name>
  template:
    metadata:
      labels:
        app: <+service.name>
    spec:
      containers:
        - name: <+service.name>
          image: <+artifact.image>
          ports:
            - containerPort: 8000
```

### 🧾 service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: <+service.name>-svc
  labels:
    app: <+service.name>
spec:
  selector:
    app: <+service.name>
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer
```

---

## ✅ Summary Table
| Capability                             | Status ✅ |
|----------------------------------------|-----------|
| Docker Image Build & Push (GAR)        | ✅         |
| Multi-Registry Image Publishing        | ✅         |
| CI → CD Trigger (Automated)            | ✅         |
| Rolling Deployment (QA & PROD)         | ✅         |
| Blue-Green Deployment (Safe Release)   | ✅         |
| Generic Inputs for Microservice Support| ✅         |
| Reusable Manifest Structure            | ✅         |
| K8s Namespace Isolation (QA/PROD)       | ✅         |

---

## 📎 References
- 📘 [Harness Docs](https://developer.harness.io/)
- 🧠 [CI/CD as Code Blog](https://www.harness.io/blog/ci-cd-pipeline-as-code-with-harness)
- 🧪 [Sample App Repo](https://github.com/abhi-gaur/todo-django-app)

---
