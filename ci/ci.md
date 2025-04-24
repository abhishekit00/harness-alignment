# Harness CI Pipeline for Building and Pushing Docker Images to GAR

This document explains the setup and configuration of a **generic, reusable, and production-grade Harness CI pipeline** that builds Docker images and pushes them to **Google Artifact Registry (GAR)**.

---

## ✨ Highlights

- ✅ Reusable across **multiple microservices**
- ⚖️ Supports multi-region & multi-registry GAR pushes
- 🚀 Dynamically configurable using **runtime inputs**
- 📄 Can be saved as a template for other teams to reuse quickly

---

## 🔧 Use Case: Assignment Requirement

> "Create a CI pipeline that builds a Docker image and pushes it to Google Artifact Registry (GAR). The pipeline should be generic enough for multiple microservices."

This Harness pipeline fulfills that requirement and is **easily extendable** to multi-region/multi-registry scenarios.

---

## ⚙️ Pipeline YAML

```yaml
pipeline:
  name: BuildAndDeployImage
  identifier: BuildAndDeployImage
  projectIdentifier: harnesscicd
  orgIdentifier: default
  tags: {}
  properties:
    ci:
      codebase:
        connectorRef: githubconnectordemo
        build: <+input>   # Runtime input: Choose branch/tag
  stages:
    - stage:
        name: build
        identifier: build
        type: CI
        spec:
          cloneCodebase: true
          caching:
            enabled: false
            override: true
            paths: []
          execution:
            steps:
              - step:
                  type: BuildAndPushGAR
                  name: BuildAndPushGAR
                  identifier: BuildAndPushGAR_1
                  spec:
                    connectorRef: gcpconnectordemo
                    host: <+input>              # Runtime input for GAR host (e.g., us-east1-docker.pkg.dev)
                    projectID: <+input>         # Runtime input for GCP Project ID
                    imageName: <+input>         # Runtime input for image name e.g., harness-docker-repo/voting-app
                    tags:
                      - <+pipeline.executionId>  # Automatically tags image with unique execution ID
                    caching: true
                    dockerfile: vote/Dockerfile  # Default Dockerfile path (can be parameterized)
                    context: <+input>           # Runtime input for build context (e.g., vote)
          platform:
            os: Linux
            arch: Amd64
          runtime:
            type: Cloud
            spec: {}
```

---

## 🔄 Reusability & Modularity

### ✅ Why it's Generic:
- You can reuse this pipeline **without editing YAML** — just provide values at runtime.
- Supports **any Dockerfile path**, build context, GAR host, project ID, and image name.

### ⚖️ Multi-Registry/Multi-Region Support:
- To support multi-region or backup registries, simply **add another `BuildAndPushGAR` step** with different `host` and `projectID`.

```yaml
- step:
    type: BuildAndPushGAR
    name: PushToSecondaryRegion
    spec:
      host: europe-west1-docker.pkg.dev
      projectID: another-project-id
      imageName: <+input>
      tags:
        - latest
```

---

## 📗 Best Practices

| Practice | Description |
|---------|-------------|
| **Runtime Inputs** | Use `<+input>` to keep pipeline flexible for different services. |
| **Tagging** | Use `<+pipeline.executionId>` or semantic versioning to uniquely tag images. |
| **Artifact Repo Naming** | Use a common prefix like `harness-docker-repo/service-name`. |
| **Cloud Runtime** | Prefer `Cloud` runtime to avoid GKE/privileged container issues. |
| **Pipeline Templates** | Save this as a template in Harness to allow org-wide reuse. |

---

## 🔄 Successful Build Example

![Successful Pipeline](./Screenshot%202025-04-22%20at%204.13.34%E2%80%AFPM.png)

> ✅ As shown above, the pipeline executed successfully with all dynamic inputs resolved at runtime.

---

## 🎯 Conclusion

This pipeline design is:
- ✅ Scalable for teams managing 10s or 100s of microservices.
- 📄 Easy to replicate across environments and teams.
- ⚖️ Aligned with **best CI/CD practices** for Google Cloud.

You can now extend this foundation for deployment, security scans, or quality gates.

---

