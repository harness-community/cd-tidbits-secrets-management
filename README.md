# CD | Tidbits | Secrets Management

> **Bite-sized how-to** | ~15 min setup

A Harness **secret** stores a credential. YAML only holds the secret **id**. At runtime Harness injects the value and masks it in logs.

This tidbit uses one secret: a Docker Hub token for a **private podinfo image**. The cluster cannot pull that image anonymously. Harness creates a Kubernetes image-pull secret from the Docker connector so the deploy succeeds.

This walkthrough does **not** build or push. You need a **podinfo** image already in a **private Docker Hub repository** you own, tagged `latest` (for example `<YOUR_DOCKERHUB_USER>/podinfo:latest`). To build and push that image, use the [connector usage tidbit](https://university-registration.harness.io/self-paced-training-tidbit-introduction-to-cd-connector-usage), then set the Hub repo to **Private**. This pipeline only deploys that existing tag.

| Piece | Role |
|---|---|
| Text secret `dockerhub-pat` | Docker Hub access token |
| Docker Hub connector | Uses that secret to create the pull secret |
| GitHub connector | At deploy time, clones **this** tidbit repo so Harness can apply [`manifests/`](./manifests/) |
| Manifest values | [`manifests/values.yaml`](./manifests/values.yaml) — image and dockercfg expressions |
| Kubernetes Secret | [`manifests/docker-secret.yaml`](./manifests/docker-secret.yaml) — `podinfo-dockerhub-pull` |

Docs: [text secrets](https://developer.harness.io/docs/platform/secrets/add-use-text-secrets), [connectors](https://developer.harness.io/harness-platform/3.0/in-harness-3.0/connectors).

---

## Prerequisites

- Harness **Project** (org + project identifiers).
- A **podinfo** image already pushed to **your private Docker Hub repository**, tagged `latest` (or change the pipeline tag to match). Build and push with the [connector usage tidbit](https://university-registration.harness.io/self-paced-training-tidbit-introduction-to-cd-connector-usage), then mark the Hub repo **Private**. This tidbit does not build or push.
- A Docker Hub **access token** (not your account password) that can pull that private image.
- Kubernetes cluster with a Harness **Delegate**, namespace `podinfo`.
- A GitHub PAT that can read this tidbit repo (Harness fetches [`manifests/`](./manifests/) from Git).

---

## Setup

1. **Secret.** Project Settings → Secrets → Text. Id `dockerhub-pat`. Paste the Hub token. Never put the token in Git or pipeline YAML.

2. **Docker Hub connector.** [`connectors/dockerhub.yaml`](./connectors/dockerhub.yaml): URL `https://index.docker.io/v2/`, username + password → `dockerhub-pat`. Test connection. The Hub repository that holds **your podinfo image** must be **Private**.

3. **Kubernetes connector.** [`connectors/k8s.yaml`](./connectors/k8s.yaml) (`InheritFromDelegate`).

4. **GitHub connector.** [`connectors/github.yaml`](./connectors/github.yaml), secret `github-pat`. The pipeline has no clone step. At **Deploy**, Harness uses this connector to clone **this** tidbit repo and apply [`manifests/`](./manifests/). Without it, Rolling Deploy cannot load the manifests.

5. **Env, infra, service, pipeline.** Paste these and replace every `# REPLACE:` line. Set service `imagePath` to the private Hub repo where you **already pushed podinfo** (for example `<YOUR_DOCKERHUB_USER>/podinfo`). The service must set `connectorRef: githubconnector`, manifest paths, and `valuesPaths: [manifests/values.yaml]`.

| File | Creates |
|---|---|
| [`.harness/environment.yaml`](./.harness/environment.yaml) | Environment `podinfoenv` |
| [`.harness/infrastructure.yaml`](./.harness/infrastructure.yaml) | Infra on `k8sconnector`, namespace `podinfo` |
| [`.harness/service.yaml`](./.harness/service.yaml) | Service `podinfo`; manifests from this repo |
| [`.harness/pipeline.yaml`](./.harness/pipeline.yaml) | Deploy-only rolling pipeline |

Harness renders expressions in [`manifests/values.yaml`](./manifests/values.yaml). The image goes on the container; the dockercfg value is **Secret data**, not the Secret name:

```yaml
# values.yaml
image: <+artifacts.primary.image>
dockercfg: <+artifacts.primary.imagePullSecret>
```

```yaml
# docker-secret.yaml
type: kubernetes.io/dockercfg
data:
  .dockercfg: {{.Values.dockercfg}}
```

```yaml
# deployment.yaml
imagePullSecrets:
  - name: podinfo-dockerhub-pull
containers:
  - name: podinfod
    image: {{.Values.image}}
```

---

## Run

**Run** the pipeline.

Without the pull secret, the pod stays `ImagePullBackOff`. With `dockerhub-pat` on the connector and `imagePullSecrets` on the Deployment, the pod is **Running**:

```sh
kubectl -n podinfo get po
kubectl -n podinfo describe po <pod>   # ImagePullSecrets present; no 401
```

Green plus Running means the cluster used a credential that never appeared in Git.

**401 / ImagePullBackOff.** Confirm podinfo exists on the private Hub repo (`docker pull <user>/podinfo:latest` after `docker login`). Check secret id, connector Test Connection, and `imagePullSecrets` on the live Deployment.

**Deploy forbidden.** Delegate service account needs rights in namespace `podinfo`.

Rotate `dockerhub-pat` in Secrets and re-run; pipeline YAML does not change.
