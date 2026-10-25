# release-demo-gitops

GitOps repository for the **release-demo** portfolio project. It describes what runs in each environment (dev, staging, prod). **Argo CD** deploys it, and **Kargo** promotes new app versions from one environment to the next by committing to this repo.

The app itself lives in **[release-demo-app](https://github.com/ksycz/release-demo-app)**, a tiny Node.js service with `/` and `/health`, built into `ghcr.io/ksycz/release-demo-app` by GitHub Actions.

## How a release flows

```
release-demo-app                        release-demo-gitops                  cluster
────────────────                        ───────────────────                  ───────
git tag v0.2.0 ──▶ CI builds image
                   ghcr.io/…:0.2.0 ──▶ Kargo Warehouse finds it → Freight
                                        Promotion to a Stage:
                                          edit env/<stage>/values.yaml
                                          commit + push ───────────────────▶ Argo CD syncs
                                          ask Argo CD to sync                 the Application
```

- **dev** gets every new version automatically.
- **staging** only accepts versions that passed dev; **prod** only versions that passed staging. Both are promoted by hand in Kargo.
- A **rollback** is a promotion of an older Freight, so it's a normal commit in this repo too.

## Repository layout

```
chart/                    Helm chart for the app (Deployment with readiness probe, Service)
env/<dev|staging|prod>/   values per environment: replica count and image tag (Kargo edits the tag)
argocd/                   one Argo CD Application per environment
bootstrap/root.yaml       root Application ("app of apps"): creates everything in argocd/
kargo/                    Kargo Project, Warehouse (watches the image) and Stages
```

## Running it locally

Everything runs on a laptop in a [kind](https://kind.sigs.k8s.io/) cluster. Tested on macOS (Apple Silicon) with Docker Desktop, kind 0.33, kubectl 1.36, Helm 4.3 and Kargo 1.12.

### Every day: start working

1. Start **Docker Desktop** and wait for "Engine running".
2. Check the cluster:

   ```bash
   kubectl config current-context        # should print kind-release-demo
   kubectl get nodes                     # release-demo-control-plane, STATUS Ready
   ```

   If `get nodes` can't connect, start the cluster's container again and wait a minute:

   ```bash
   docker start release-demo-control-plane
   ```

   If it doesn't exist anymore, see [From scratch](#from-scratch) below.

3. Open the web pages. Each `port-forward` keeps running, so use one terminal tab per command:

   | What | Command | Address |
   | --- | --- | --- |
   | Argo CD | `kubectl port-forward svc/argocd-server -n argocd 8443:443` | https://localhost:8443 (user `admin`) |
   | Kargo | `kubectl port-forward svc/kargo-api -n kargo 8081:80` | http://localhost:8081 (Admin) |
   | App, dev | `kubectl port-forward service/release-demo 8080:80 -n release-demo-dev` | http://localhost:8080 |
   | App, staging | `kubectl port-forward service/release-demo 8082:80 -n release-demo-staging` | http://localhost:8082 |
   | App, prod | `kubectl port-forward service/release-demo 8083:80 -n release-demo-prod` | http://localhost:8083 |

   A port-forward connects to one Pod, so it stops when that Pod is replaced (for example during a promotion). Just run it again.

4. Log the Kargo CLI in (needed after a restart):

   ```bash
   kargo login http://localhost:8081 --admin
   ```

**Argo CD admin password:**

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo
```

### Health check

```bash
kubectl get pods -A | grep -E "argocd-server|kargo-api|release-demo"   # all Running
kubectl get applications -n argocd                                    # all Synced, Healthy
kargo get stages --project release-demo                               # dev, staging, prod
```

### From scratch

Rebuilds the whole setup on an empty machine. Everything about the app comes back from Git; only Kargo's promotion history is lost.

**Tools** (macOS): Docker Desktop, then

```bash
brew install kind kubectl helm kargo
```

**1. Cluster**

```bash
kind create cluster --name release-demo
```

**2. Argo CD and the apps**

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=300s
kubectl apply -f bootstrap/root.yaml
```

The root Application creates the three environment Applications, which deploy the chart into `release-demo-dev`, `release-demo-staging` and `release-demo-prod`.

**3. cert-manager** (needed by Kargo)

```bash
helm install cert-manager cert-manager \
  --repo https://charts.jetstack.io \
  --namespace cert-manager --create-namespace \
  --version 1.19.3 \
  --set crds.enabled=true \
  --wait
```

**4. Kargo.** Run these in **one** terminal tab, because the variables only exist there. Save the printed password.

```bash
pass=$(openssl rand -base64 48 | tr -d "=+/" | head -c 32)
echo "Kargo admin password: $pass"
hashed_pass=$(htpasswd -bnBC 10 "" $pass | tr -d ':\n')
signing_key=$(openssl rand -base64 48 | tr -d "=+/" | head -c 32)

helm install kargo oci://ghcr.io/akuity/kargo-charts/kargo \
  --namespace kargo --create-namespace \
  --set api.adminAccount.passwordHash=$hashed_pass \
  --set api.adminAccount.tokenSigningKey=$signing_key \
  --set api.tls.enabled=false \
  --wait
```

`api.tls.enabled=false` serves the Kargo page over plain HTTP. That's only acceptable on a local machine.

Then start the Kargo port-forward and log in (see [Every day](#every-day-start-working)).

**5. Kargo project and Git credentials**

```bash
kargo apply -f kargo/project.yaml
```

Kargo pushes to this repo, so it needs a GitHub **fine-grained token** with access to this repository only and **Contents: Read and write**. Store it as a Secret in the project's namespace:

```bash
read -s GITHUB_TOKEN                     # paste the token, press Enter (hidden)
echo -n "$GITHUB_TOKEN" | wc -c          # must print a number > 0

kubectl create secret generic gitops-repo -n release-demo \
  --from-literal=repoURL=https://github.com/ksycz/release-demo-gitops.git \
  --from-literal=username=ksycz \
  --from-literal=password="$GITHUB_TOKEN"
kubectl label secret gitops-repo -n release-demo kargo.akuity.io/cred-type=git
```

Never commit the token.

**6. Warehouse and Stages**

```bash
kargo apply -f kargo/warehouse.yaml
kargo apply -f kargo/stages.yaml
```

The Warehouse finds the image versions, and dev promotes the newest one automatically.

## Everyday tasks

**Release a new app version** (in release-demo-app, after merging to `main`):

```bash
git tag -a v0.3.0 -m "Short description"
git push origin v0.3.0
```

CI publishes `ghcr.io/ksycz/release-demo-app:0.3.0`, and dev picks it up. To skip the Warehouse's polling interval:

```bash
kargo refresh warehouse --project release-demo release-demo
```

**Promote:** drag the Freight onto a Stage in the Kargo page, or

```bash
kargo get freight --project release-demo --warehouse release-demo
kargo promote --project release-demo --freight <name> --stage staging
```

**Roll back:** promote an older Freight to the Stage, the same way. Don't use `helm rollback` or Argo CD's Rollback button: Git would still hold the new version, and Argo CD would restore it.

## Design decisions

- **Helm chart + one values file per environment.** The templates exist once; environments only differ in `env/<stage>/values.yaml`.
- **One name per environment everywhere:** folder `env/prod`, namespace `release-demo-prod`, Application `release-demo-prod`, Kargo Stage `prod`. The Kargo Stages build paths from the Stage name (`${{ ctx.stage }}`).
- **App of apps.** `bootstrap/root.yaml` is the only thing applied by hand; all other Applications come from Git.
- **Fixed image tags only.** Environments run versions like `0.2.0`, never `main` or `0.2`, so what passed staging is exactly what reaches prod.
- **All environments on one branch (`main`).** Simple, but Kargo's `argocd-update` step doesn't pin `desiredRevision`: with a pin, each promotion commit makes the other Stages look out of date. Kargo therefore trusts Argo CD's Synced/Healthy status instead of an exact commit, and chart changes reach every environment at once. A branch per environment would isolate them; planned as a later improvement.
