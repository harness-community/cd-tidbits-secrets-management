# CD | Tidbits | Secrets Management

> **Bite-sized how-to** | ~15 min setup

---

## What is a Harness secret?

A **secret** is a credential Harness stores for you. Pipelines and connectors only hold the secret **id**. At runtime Harness injects the value and masks it in logs.

This tidbit uses one secret for one job: a **private Docker Hub** image. The cluster cannot pull that image anonymously. Harness injects the registry password as a Kubernetes image-pull secret so the deploy succeeds.

Auth, scopes, and other secret types: [Connectors](https://developer.harness.io/harness-platform/3.0/in-harness-3.0/connectors) (credentials live on the connector; the **value** is a Harness secret).

| Piece | Role |
|---|---|
| Text secret `dockerhub-pat` | Docker Hub access token |
| Docker Hub connector | Uses that secret to push and to mint a pull secret |
| Manifest `imagePullSecrets` | `<+artifacts.primary.imagePullSecret>` — injected on the cluster |

---

## Prerequisites

Before you start, make sure you have:

- A Harness account with a **Project** (note its org + project identifiers).
- Harness Cloud build credits.
- A Docker Hub user and a **private** repository you can push to (e.g. `<user>/podinfo`).
- A Docker Hub **access token** (not your account password).
- A Kubernetes cluster with a Harness **Delegate** in it, and namespace `podinfo`.
- A GitHub PAT that can clone [harness-community/podinfo](https://github.com/harness-community/podinfo) and this tidbit repo.

If you already ran [cd-tidbits-connector-usage](https://github.com/harness-community/cd-tidbits-connector-usage), reuse those GitHub and Kubernetes connectors. Make the Hub repo **private** and keep going from the secret + pull-secret step.

---

## Step 1 — Create the registry secret

1. **Project Settings → Secrets → + New Secret → Text**.
2. Id: `dockerhub-pat`. Paste the Docker Hub access token.
3. Save. Confirm the **Id** is `dockerhub-pat` (underscores/hyphens as you typed — copy the Id, not the name).

Do **not** put the token in Git or in pipeline YAML.

You also need a GitHub PAT secret (`github-pat`) so CI can clone podinfo. That is not the lesson; it is only how the pipeline gets source.

---

## Step 2 — Docker Hub connector (points at the secret)

1. **Connectors → Docker Registry**.
2. Name `dockerhub-connector`, id `dockerhubconnector`.
3. URL `https://index.docker.io/v2/`. Username + password → secret `dockerhub-pat`.
4. Test connection. Save.

YAML: [`connectors/dockerhub.yaml`](./connectors/dockerhub.yaml).

On the Docker Hub website, set `<YOUR_DOCKERHUB_USER>/podinfo` (or whatever image path you use) to **Private**.

---

## Step 3 — GitHub + Kubernetes connectors, env, infra, service

Create (or reuse) GitHub and Kubernetes connectors: [`connectors/github.yaml`](./connectors/github.yaml), [`connectors/k8s.yaml`](./connectors/k8s.yaml).

Paste environment, infrastructure, and service YAML. Edit every `# REPLACE:` line. The service artifact must be the **private** image path. Manifests in this repo already request the injected pull secret:

| File | Creates |
|---|---|
| [`.harness/environment.yaml`](./.harness/environment.yaml) | Environment `podinfoenv` |
| [`.harness/infrastructure.yaml`](./.harness/infrastructure.yaml) | KubernetesDirect on `k8sconnector` |
| [`.harness/service.yaml`](./.harness/service.yaml) | Service `podinfo` + `manifests/` |

---

## Step 4 — Pipeline

Paste [`.harness/pipeline.yaml`](./.harness/pipeline.yaml). Set org, project, and the same private image path as the service. Save.

Build clones podinfo and **pushes to the private repo**. Deploy applies manifests that pull with the Harness-injected secret.

---

## Step 5 — Run (expect GREEN)

**Run**, branch `master`.

Without the pull secret (or with a public-only connector), the pod stays `ImagePullBackOff`. With `dockerhub-pat` on the connector and `<+artifacts.primary.imagePullSecret>` in the Deployment, the pod reaches **Running**.

```sh
kubectl -n podinfo get po
kubectl -n podinfo describe po <pod>   # should show a pull secret, not 401
```

**Green plus Running is the correct outcome.** The cluster used a credential it never saw in Git.

---

## Pipeline YAML reference

[`.harness/pipeline.yaml`](./.harness/pipeline.yaml) — same shape as the connector tidbit: CI build/push, then a Kubernetes rolling deploy. The difference is the **private** image and this in [`manifests/deployment.yaml`](./manifests/deployment.yaml):

```yaml
spec:
  imagePullSecrets:
    - name: <+artifacts.primary.imagePullSecret>
  containers:
    - name: podinfod
      image: <+artifacts.primary.image>
```

---

## Common Issues & Tips

**`ImagePullBackOff` / 401.** Repo is private and the pull secret was not created. Check secret id `dockerhub-pat`, connector Test Connection, and `imagePullSecrets` on the Deployment.

**Clone fails.** GitHub PAT / `githubconnector` — not this lesson; test that connector.

**Deploy forbidden.** Delegate service account needs rights in namespace `podinfo`.

---

## What's next?

- Rotate `dockerhub-pat` in Secrets; re-run. The pipeline YAML does not change.
- Store `.harness/` as Remote so reviews see secret *ids*, never values.

---

## Resources

- [Add and reference text secrets](https://developer.harness.io/docs/platform/secrets/add-use-text-secrets)
- [Connectors](https://developer.harness.io/harness-platform/3.0/in-harness-3.0/connectors)
- [podinfo](https://github.com/harness-community/podinfo)
- [Connector usage tidbit](https://github.com/harness-community/cd-tidbits-connector-usage)
