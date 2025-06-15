# Skube Build-Push-Deploy Action

A reusable GitHub Actions workflow for building a Docker image, pushing it to a Harbor registry, and deploying to a Kubernetes cluster. Found in the .github/workflows folder (This readme is written by ChatGPT and with revisions by myself, Too much writing for me)

---

## Table of Contents

- [Features](#features)  
- [Usage](#usage)  
  - [Calling the Workflow](#calling-the-workflow)  
  - [Inputs](#inputs)  
  - [Secrets](#secrets)  
- [How It Works](#how-it-works)  
- [Example](#example)  
- [Requirements](#requirements)  
- [License](#license)  

---

## Features

- **Build** your application container image using Docker Buildx  
- **Push** to a secure Harbor registry (supports insecure HTTP registries)  
- **Deploy** updated manifests to any Kubernetes cluster  
- **Parameterizable**: customize image name, Dockerfile path, manifest directory, namespace, and extra build args  
- **Reusable**: invoke via `workflow_call` from any repository  

---

## Usage

### Calling the Workflow

Create (or update) a GitHub Actions workflow in your repo that invokes this action:

```yaml
name: CI/CD

on:
  push:
    branches:
      - main

jobs:
  deploy:
    uses: skube-org/skube/.github/workflows/build-push-deploy.yml@main
    with:
      IMAGE_NAME:       my-app
      DOCKERFILE_PATH:  ./Dockerfile           # optional, defaults to ./Dockerfile
      MANIFEST_PATH:    ./k8s                   # directory or single YAML
      KUBE_NAMESPACE:   hobby                   # optional, defaults to "default"
      EXTRA_BUILD_ARGS: ARG1=foo ARG2=bar       # optional
    secrets:
      HARBOR_URL:       ${{ secrets.HARBOR_URL }}
      HARBOR_USERNAME:  ${{ secrets.HARBOR_USERNAME }}
      HARBOR_PASSWORD:  ${{ secrets.HARBOR_PASSWORD }}
      KUBE_CONFIG:      ${{ secrets.KUBE_CONFIG }}
```

### Inputs

| Name               | Required | Default        | Description                                                                        |
| ------------------ | :------: | :------------- | :--------------------------------------------------------------------------------- |
| `IMAGE_NAME`       |   true   | —              | Name (and optional path) of the Docker image in Harbor (e.g. `group/project/app`). |
| `DOCKERFILE_PATH`  |   false  | `./Dockerfile` | Path to the Dockerfile to build.                                                   |
| `MANIFEST_PATH`    |   true   | —              | Path (file or directory) containing your Kubernetes YAML manifests.                |
| `KUBE_NAMESPACE`   |   false  | `default`      | Kubernetes namespace to deploy into.                                               |
| `EXTRA_BUILD_ARGS` |   false  | —              | Any extra `--build-arg` values to pass to `docker build`.                          |

### Secrets

| Name              | Required | Description                                                          |
| ----------------- | :------: | :------------------------------------------------------------------- |
| `HARBOR_URL`      |   true   | Full URL of your Harbor registry (e.g. `my.harbor.local:5000`).      |
| `HARBOR_USERNAME` |   true   | Username for Harbor authentication.                                  |
| `HARBOR_PASSWORD` |   true   | Password (or robot account token) for Harbor authentication.         |
| `KUBE_CONFIG`     |   true   | Base64-encoded kubeconfig granting `kubectl` access to your cluster. |

---

## How It Works

1. **Checkout** your code
2. **Set up** Docker Buildx using the host daemon
3. **Log in** to Harbor (supports HTTP registries via `--tls-verify=false`)
4. **Build & push** your image for `linux/amd64`, tagging with both SHA and `latest`
5. **Configure** `kubectl` by writing the decoded kubeconfig
6. **Substitute** `IMAGE_PLACEHOLDER` in your Kubernetes YAMLs with the real image reference
7. **Apply** all manifests in your target namespace
8. **Wait** for your Deployment to successfully roll out

---

## Example

Assuming you have:

* A `Dockerfile` at your repo root
* A `k8s/` directory containing:

  * `configmap.yaml`
  * `secret.yaml`
  * `deployment.yaml` (uses `IMAGE_PLACEHOLDER`)
  * `service.yaml`

Your workflow might look like:

```yaml
name: CI/CD

on:
  workflow_dispatch:

jobs:
  release:
    uses: skube-org/skube/.github/workflows/build-push-deploy.yml@main
    with:
      IMAGE_NAME:     my-app
      MANIFEST_PATH:  ./k8s
      KUBE_NAMESPACE: production
    secrets:
      HARBOR_URL:       ${{ secrets.HARBOR_URL }}
      HARBOR_USERNAME:  ${{ secrets.HARBOR_USERNAME }}
      HARBOR_PASSWORD:  ${{ secrets.HARBOR_PASSWORD }}
      KUBE_CONFIG:      ${{ secrets.KUBE_CONFIG }}
```

---

## Requirements

* **Self-hosted runner** (with Docker & Buildx installed)
* **Docker Buildx** and **Docker CLI** on the runner
* **`kubectl`** on the runner (version pinned via `azure/setup-kubectl@v3`)
* Your Harbor registry added to Docker’s `insecure-registries` if using HTTP

---
