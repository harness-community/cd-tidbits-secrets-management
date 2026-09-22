# CD | Tidbits | Secrets Management

> **Bite-sized how-to** | ~15 min setup

A Harness **secret** stores a credential. YAML only holds the secret **id**. At runtime Harness injects the value and masks it in logs.

This tidbit uses one secret: a Docker Hub token for a **private** image. The cluster cannot pull that image anonymously. Harness creates a Kubernetes image-pull secret from the Docker connector so the deploy succeeds.

| Piece | Role |
|---|---|
| Text secret `dockerhub-pat` | Docker Hub access token |
| Docker Hub connector | Uses that secret to create the pull secret |
| Manifest `imagePullSecrets` | `<+artifacts.primary.imagePullSecret>` |

Docs: [text secrets](https://developer.harness.io/docs/platform/secrets/add-use-text-secrets), [connectors](https://developer.harness.io/harness-platform/3.0/in-harness-3.0/connectors).

---

## Prerequisites

- Harness **Project** (org + project identifiers).
- Private Docker Hub repo with an image you can pull (e.g. `<user>/podinfo:latest`) and an **access token** (not your account password).
- Kubernetes cluster with a Harness **Delegate**, namespace `podinfo`.

---

## Setup

1. **Secret.** Project Settings → Secrets → Text. Id `dockerhub-pat`. Paste the Hub token. Never put the token in Git or pipeline YAML.

2. **Docker Hub connector.** [`connectors/dockerhub.yaml`](./connectors/dockerhub.yaml): URL `https://index.docker.io/v2/`, username + password → `dockerhub-pat`. Test connection. The Hub repo must be **Private**.

3. **Kubernetes connector.** [`connectors/k8s.yaml`](./connectors/k8s.yaml) (`InheritFromDelegate`).

4. **Env, infra, service, pipeline.** Paste these and replace every `# REPLACE:` line. Service `imagePath` must be the **private** image. Manifests are inline on the service (readable copies in [`manifests/`](./manifests/)).

| File | Creates |
|---|---|
| [`.harness/environment.yaml`](./.harness/environment.yaml) | Environment `podinfoenv` |
| [`.harness/infrastructure.yaml`](./.harness/infrastructure.yaml) | Infra on `k8sconnector`, namespace `podinfo` |
| [`.harness/service.yaml`](./.harness/service.yaml) | Service `podinfo` + inline manifests |
| [`.harness/pipeline.yaml`](./.harness/pipeline.yaml) | Deploy-only rolling pipeline |

The Deployment references the injected pull secret:

```yaml
spec:
  imagePullSecrets:
    - name: <+artifacts.primary.imagePullSecret>
  containers:
    - name: podinfod
      image: <+artifacts.primary.image>
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

**401 / ImagePullBackOff.** Check secret id, connector Test Connection, private Hub repo, and `imagePullSecrets` on the live Deployment.

**Deploy forbidden.** Delegate service account needs rights in namespace `podinfo`.

Rotate `dockerhub-pat` in Secrets and re-run; pipeline YAML does not change.
