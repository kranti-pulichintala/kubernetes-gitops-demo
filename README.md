# Kubernetes GitOps Demo

A complete, working reference architecture that shows how to deliver a containerized
web application to Kubernetes using GitOps principles. It was built as a reference
architecture, so every manifest, chart, and pipeline step is realistic and runnable
as is.

## What this project is

A small nginx-based web application packaged two ways (a Helm chart and a Kustomize
base), deployed and kept in sync by ArgoCD using the app-of-apps pattern, and
validated on every push by a GitHub Actions workflow that lints, renders, and
schema-checks the manifests. There is no application source code here because the
focus is the delivery pipeline, not the workload.

## Why it matters

GitOps makes the cluster converge on the state described in Git instead of the
state someone clicked into place. That gives you a full audit trail, repeatable
deployments, easy rollbacks, and drift correction for free. This repo is a compact
example of that workflow that you can read end to end in one sitting, then adapt
for real services.

## Architecture

ArgoCD runs inside the cluster and continuously reconciles what it finds against
what Git declares. The layout follows the app-of-apps pattern:

1. A single root Application watches the `argocd/apps` directory in this repo.
2. Every child Application file in that directory becomes a managed app of its own.
3. ArgoCD then syncs each child: the `web-app` Helm chart into the `web`
   namespace, and the upstream `ingress-nginx` chart into `ingress-nginx`.

This means onboarding a new service is just adding one YAML file under
`argocd/apps`. The root app picks it up automatically, no dashboard clicking
required. Deleting a file prunes the app, because automated sync has prune and
self-heal enabled.

```
GitHub repo (this project)
  argocd/root-app.yaml          Root Application: watches argocd/apps
    -> argocd/apps/web-app.yaml            Deploys charts/web-app (Helm)
    -> argocd/apps/infra-ingress-nginx.yaml  Deploys upstream ingress-nginx chart
```

The web app itself exists in two packaging styles so you can compare approaches:

- `charts/web-app`: a Helm chart with parameterized image, replicas, probes,
  resources, ingress, and autoscaling values.
- `kustomize/base`: the same workload as plain manifests, plus a `production`
  overlay that bumps replicas to 4, pins the image to `nginx:1.25.3`, and adds
  an `env: production` label.

## Prerequisites

- A Kubernetes cluster (kind, k3d, minikube, or any cloud cluster) with
  `kubectl` configured
- Helm 3.14 or newer
- ArgoCD installed in the cluster (`kubectl create namespace argocd` then apply
  the official ArgoCD install manifest)
- Optional: the `kubeconform` binary for local manifest validation

## Step-by-step usage

1. Clone this repository and update the `repoURL` placeholders in
   `argocd/root-app.yaml` and `argocd/apps/web-app.yaml` to point at your own
   fork or copy.

2. Install ArgoCD if you have not already:

   ```
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

3. Apply the root application. This is the only manual step; everything else is
   Git-driven from here on:

   ```
   kubectl apply -f argocd/root-app.yaml
   ```

4. Watch ArgoCD discover the children and sync them:

   ```
   kubectl get applications -n argocd
   kubectl get pods -n web
   kubectl get pods -n ingress-nginx
   ```

5. Verify the web app serves traffic. Port-forward the service and curl it:

   ```
   kubectl port-forward -n web svc/release-web-app 8080:80
   curl http://localhost:8080
   ```

6. Update the deployment point of truth (for example bump
   `image.tag` in `charts/web-app/values.yaml`), commit, and push. ArgoCD
   detects the change and rolls the new image out automatically.

To render the chart or the Kustomize overlay locally without a cluster:

```
helm lint ./charts/web-app
helm template release ./charts/web-app
kustomize build ./kustomize/overlays/production
```

## The app-of-apps pattern

In ArgoCD, an Application is itself a Kubernetes resource, so a Git repo can
contain Applications that describe other Applications. The root app here points
at `argocd/apps` and tells ArgoCD to treat every file in that directory as an
Application definition. When you merge a new child file, the root app syncs
first, registers the child, and the child then syncs its own target (a Helm
chart, a Kustomize path, or an upstream chart). You get a self-bootstrapping
fleet: one `kubectl apply` installs the root, and the root installs everything
else. Removing a child file removes the app and its resources because automated
sync runs with prune enabled.

## Promotion flow from dev to production

There is no `kubectl set image` here. Every change flows through Git:

1. **Values change.** Edit `charts/web-app/values.yaml` (for example a new
   image tag) or adjust the Kustomize overlay (`kustomize/overlays/production`)
   on a feature branch.
2. **Pull request.** Open a PR. The `helm-lint` workflow lints the chart,
   renders it with `helm template`, and validates both the Helm output and the
   Kustomize production build against the Kubernetes schemas with kubeconform.
   Nothing merges unless the manifests are valid.
3. **Merge to main.** Once reviewed and merged, the repo state is the new
   desired state.
4. **ArgoCD sync.** The child Application polls Git, sees the drift, and
   applies the change automatically (automated sync with self-heal), rolling
   the update into the cluster.
5. **Overlay promotion.** For Kustomize-based promotion, verify the change in
   the base or a lower environment first, then promote by raising the replica
   count or pinning the new image tag in
   `kustomize/overlays/production/kustomization.yaml` in its own PR. Production
   only ever changes through that overlay file, so a rollback is simply
   reverting the PR commit.

## Continuous validation

The `.github/workflows/helm-lint.yml` workflow runs on every push and pull
request. It lints the Helm chart, renders it, downloads kubeconform, and checks
both the rendered Helm output and the built production Kustomize overlay against
the Kubernetes JSON schemas in strict mode. A schema violation fails the build
before it can ever reach the cluster.

## File layout

```
kubernetes-gitops-demo/
  README.md
  argocd/
    root-app.yaml                     Root app-of-apps Application (watches argocd/apps)
    apps/
      web-app.yaml                    Child app: this repo's Helm chart -> namespace web
      infra-ingress-nginx.yaml        Child app: upstream ingress-nginx chart -> ingress-nginx
  charts/
    web-app/
      Chart.yaml                      Chart metadata (v2 API)
      values.yaml                     Tunable values: image, replicas, probes, ingress, HPA
      templates/
        deployment.yaml               Deployment rendered from values
        service.yaml                  ClusterIP Service
        ingress.yaml                  Ingress (only when ingress.enabled is true)
  kustomize/
    base/
      kustomization.yaml              Base: deployment + service, common label app: web-app
      deployment.yaml                 Plain-manifest nginx Deployment, 2 replicas
      service.yaml                    Plain-manifest ClusterIP Service
    overlays/
      production/
        kustomization.yaml            Production: 4 replicas, nginx:1.25.3, env label
  .github/
    workflows/
      helm-lint.yml                   CI: helm lint, helm template, kubeconform validation
```

## Notes

- Replace `https://github.com/EXAMPLE/kubernetes-gitops-demo.git` with your real
  repo URL before applying anything.
- Change the ingress host `web.example.com` in `charts/web-app/values.yaml` to
  a domain you control, and point DNS at your ingress controller.
- The `ingress-nginx` child pins to the `4.x` chart series; pin to an exact
  version for production use.
