# Lab 2 Guide — GitOps with ArgoCD + Helm

**Lab 1 must be complete before starting.** Your EKS cluster should have the `dev` namespace, RBAC, and secrets (`db-credentials`, `jwt-secret`) already applied.

Verify before starting:
```bash
kubectl get namespace dev
kubectl get secret db-credentials -n dev
kubectl get secret jwt-secret -n dev
```

All three must exist. If not, complete Lab 1 first.

---

# Part 1 — Day 2: ArgoCD with Raw Manifests

## What We Are Building

In Lab 1 you deployed 9 services by running `kubectl apply` manually — one command per resource, per service. Today you hand that job to ArgoCD. ArgoCD watches this Git repo and applies changes automatically. You will never run `kubectl apply` in production again.

By the end of Day 2 you will have:
- ArgoCD installed on your EKS cluster
- A pharma AppProject scoping what ArgoCD can touch
- All 9 services deployed via ArgoCD watching `lab2/manifests/`
- Experienced configuration drift and ArgoCD's response to it

---

## Step 1 — Fork and Configure the Repository

Every student needs their own copy of zen-gitops so ArgoCD can watch **your** repo and pick up **your** changes (including your RDS endpoint).

**1a. Fork on GitHub**

1. Go to `https://github.com/DPP-2026/zen-gitops`
2. Click **Fork** → create under your GitHub account
3. Clone your fork locally:

```bash
git clone https://github.com/<YOUR_GITHUB_USERNAME>/zen-gitops.git
cd zen-gitops
```

**1b. Update all ArgoCD app files to point to your fork**

The `day2-raw` apps reference the instructor org (`DPP-2026`). The `day3-helm` apps use a placeholder. Replace both:

```bash
export GH_USER=<your-actual-github-username>

# macOS
find lab2/argocd-apps -name "*.yaml" \
  -exec sed -i '' "s|DPP-2026/zen-gitops|$GH_USER/zen-gitops|g" {} + \
  -exec sed -i '' "s|<YOUR_GITHUB_USERNAME>|$GH_USER|g" {} +

# Linux / Cloud9
find lab2/argocd-apps -name "*.yaml" \
  -exec sed -i "s|DPP-2026/zen-gitops|$GH_USER/zen-gitops|g" {} + \
  -exec sed -i "s|<YOUR_GITHUB_USERNAME>|$GH_USER|g" {} +

# Verify — every line should show your username
grep -r "github.com" lab2/argocd-apps/ | grep -v ".gitkeep"
```

Expected output shows `github.com/<YOUR_USERNAME>/zen-gitops` everywhere — no `DPP-2026` or `<YOUR_GITHUB_USERNAME>` remaining.

**1c. Add your fork to the AppProject source whitelist**

ArgoCD's AppProject enforces which repos are allowed. Your fork must be listed:

```bash
# macOS
sed -i '' "s|DPP-2026/zen-gitops|$GH_USER/zen-gitops|g" \
  lab2/argocd-apps/project/pharma-project.yaml

# Linux / Cloud9
sed -i "s|DPP-2026/zen-gitops|$GH_USER/zen-gitops|g" \
  lab2/argocd-apps/project/pharma-project.yaml

grep "sourceRepos" -A3 lab2/argocd-apps/project/pharma-project.yaml
# Should show your fork URL
```

---

## Step 2 — Set Your RDS Endpoint

The ConfigMaps in `lab2/manifests/` contain a `DB_HOST` that must match **your** RDS instance. ArgoCD applies what is in Git — if the endpoint is wrong, ArgoCD will keep applying the broken ConfigMap and pods will keep crashing even after you fix the cluster manually.

```bash
export DB_HOST=$(aws rds describe-db-instances \
  --query 'DBInstances[?DBInstanceIdentifier==`pharma-dev-postgres`].Endpoint.Address' \
  --output text)
echo $DB_HOST
# Expected: pharma-dev-postgres.<unique-id>.us-east-1.rds.amazonaws.com
```

```bash
# macOS
find lab2/manifests/ -name "configmap.yaml" \
  -exec sed -i '' "s|DB_HOST:.*rds\.amazonaws\.com|DB_HOST: $DB_HOST|g" {} +

# Linux / Cloud9
find lab2/manifests/ -name "configmap.yaml" \
  -exec sed -i "s|DB_HOST:.*rds\.amazonaws\.com|DB_HOST: $DB_HOST|g" {} +
```

Verify:
```bash
grep "DB_HOST" lab2/manifests/*/configmap.yaml
# Every line should show your actual RDS endpoint
```

---

## Step 3 — Commit and Push Everything

```bash
git add lab2/
git commit -m "lab2: configure my GitHub username and RDS endpoint"
git push
```

> ArgoCD clones your repo directly from GitHub. Changes that are not pushed are invisible to it. Always push before expecting ArgoCD to act.

---

## Step 4 — Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for the server to be ready (2-3 minutes)
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s

kubectl get pods -n argocd
```

Expected — all pods Running and Ready:
```
NAME                                                READY   STATUS    RESTARTS
argocd-application-controller-0                     1/1     Running   0
argocd-applicationset-controller-xxx                1/1     Running   0
argocd-dex-server-xxx                               1/1     Running   0
argocd-notifications-controller-xxx                 1/1     Running   0
argocd-redis-xxx                                    1/1     Running   0
argocd-repo-server-xxx                              1/1     Running   0
argocd-server-xxx                                   1/1     Running   0
```

---

## Step 5 — Access the ArgoCD UI

Get the initial admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

**Option A — Port-forward (local machine):**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Open: `https://localhost:8080`

**Option B — LoadBalancer (Cloud9 / remote):**
```bash
kubectl get svc argocd-server -n argocd
# Copy the EXTERNAL-IP from the LoadBalancer line
```
Open: `https://<EXTERNAL-IP>` (you will see a TLS warning — click through, it is expected for a self-signed cert).

Login: username `admin`, password from the command above.

---

## Step 6 — Install and Login via ArgoCD CLI

```bash
# Mac
brew install argocd

# Linux / Cloud9
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

Login (port-forward):
```bash
argocd login localhost:8080 \
  --username admin \
  --password $(kubectl get secret argocd-initial-admin-secret -n argocd \
    -o jsonpath='{.data.password}' | base64 -d) \
  --insecure
```

Login (LoadBalancer — replace with your LB address):
```bash
export ARGOCD_LB=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

argocd login $ARGOCD_LB \
  --username admin \
  --password $(kubectl get secret argocd-initial-admin-secret -n argocd \
    -o jsonpath='{.data.password}' | base64 -d) \
  --insecure
```

Expected: `'admin:login' logged in successfully`

---

## Step 7 — Key ArgoCD Concepts

**AppProject** — A security boundary. Defines which Git repos ArgoCD can use as sources, which namespaces it can deploy to, and which Kubernetes resource types it can create. Think of it as IAM for ArgoCD.

**Application** — The core ArgoCD object. Links a Git source (repo + path + branch) to a Kubernetes destination (cluster + namespace). ArgoCD continuously compares what is in Git with what is in the cluster and reports the difference.

**Sync Status:**
- `Synced` — cluster matches Git exactly
- `OutOfSync` — something differs (Git changed, or cluster was modified manually)
- `Unknown` — ArgoCD cannot determine state yet

**Health Status:**
- `Healthy` — all Kubernetes resources are in their desired state
- `Progressing` — pods are starting or updating
- `Degraded` — something is wrong (pods crashing, etc.)

**Reconciliation loop:** ArgoCD polls Git every 3 minutes. When it detects a difference between Git and the cluster, it marks the Application as OutOfSync. With `automated: true` in syncPolicy, it applies the diff automatically.

---

## Step 8 — Create the pharma AppProject

```bash
kubectl apply -f lab2/argocd-apps/project/pharma-project.yaml

# Verify
argocd proj list
```

Expected:
```
NAME    DESCRIPTION                  DESTINATIONS   SOURCES  CLUSTER-RESOURCE-WHITELIST  ...
pharma  Pharma microservices project  3              1        */*                         ...
```

In the UI: click **Settings** → **Projects** → **pharma**. You can see the allowed source repos (your fork), destination namespaces (dev, qa, prod), and the resource whitelist.

---

## Step 9 — Deploy auth-service via ArgoCD (Raw Manifests)

Do this one first and understand every step before scaling to all 9.

```bash
kubectl apply -f lab2/argocd-apps/day2-raw/auth-service-app.yaml

# Check the application status
argocd app list

# Get detailed status
argocd app get auth-service-dev-raw
```

Watch the pod come up:
```bash
kubectl get pods -n dev -w
```

Press `Ctrl+C` when the pod reaches `Running`.

**What just happened?**
1. ArgoCD cloned your zen-gitops fork
2. Found all YAML files in `lab2/manifests/auth-service/`
3. Applied them to the `dev` namespace
4. Is now watching for any difference between that directory and the cluster

---

## Step 10 — Explore the ArgoCD UI

In the browser, click on the `auth-service-dev-raw` application.

**Resource Tree** — The full hierarchy ArgoCD manages:
```
Application (auth-service-dev-raw)
├── ServiceAccount (auth-service)
├── ConfigMap (auth-service)
├── Deployment (auth-service)
│   └── ReplicaSet (auth-service-xxx)
│       └── Pod (auth-service-xxx-yyy)  ← click to see logs
└── Service (auth-service)
```

Click any resource to see its YAML, events, and logs.

**App Diff** — Click to see what ArgoCD would change if you synced right now. Should be empty (Synced).

**Sync History** — Click **History** to see every previous sync with its Git commit hash.

---

## Step 11 — Simulate Configuration Drift

Someone bypasses GitOps and manually scales auth-service:

```bash
kubectl scale deployment auth-service --replicas=3 -n dev
kubectl get pods -n dev
```

Watch ArgoCD detect the drift (within 3 minutes, or force a refresh):
```bash
argocd app get auth-service-dev-raw
# STATUS: OutOfSync

argocd app diff auth-service-dev-raw
```

In the UI the app shows an orange `OutOfSync` badge.

Sync back to Git state:
```bash
argocd app sync auth-service-dev-raw
kubectl get pods -n dev
# Back to 1 pod — drift reverted
```

> **Key insight:** With `selfHeal: true` added to syncPolicy, ArgoCD would revert this automatically without you triggering it. Git always wins.

---

## Step 12 — Deploy All 9 Services

```bash
for app in lab2/argocd-apps/day2-raw/*.yaml; do
  kubectl apply -f $app
done

# Watch all applications appear
argocd app list

# Watch all pods come up
kubectl get pods -n dev -w
```

Wait for all pods to reach `1/1 Running`. Services that connect to the database (auth, drug-catalog, inventory, manufacturing, supplier, notification) need 60-90 seconds for Flyway DB migrations on first startup.

---

## Step 13 — Verify the Full Stack

```bash
kubectl get all -n dev
```

Expected:
- 9 Deployments
- 9 ReplicaSets
- 9 Pods (`1/1 Running`)
- 9 Services
- 2 Ingresses (api-gateway: `/api`, pharma-ui: `/`)

Get the nginx LoadBalancer address:
```bash
kubectl get svc -n ingress-nginx
# Copy the EXTERNAL-IP of the ingress-nginx-controller LoadBalancer
```

Open in your browser:
```
http://<EXTERNAL-IP>/
```

You should see the Zen Pharma UI load. If you see `404 Not Found` from nginx, the pharma-ui ingress is not applied — check `argocd app get pharma-ui-dev-raw` for sync errors.

Test the API gateway:
```bash
curl http://<EXTERNAL-IP>/api/actuator/health
# Expected: {"status":"UP"}
```

Check that all 9 ArgoCD apps are Synced and Healthy:
```bash
argocd app list
# All STATUS: Synced, HEALTH: Healthy
```

---

## Step 14 — Feel the Pain of Raw Manifests

```bash
find lab2/manifests -name "*.yaml" | wc -l
# ~40 files — 4-5 per service × 9 services
```

Now imagine: the team releases a new Docker image. You need to update the image tag. That means editing `deployment.yaml` in 9 directories. Add a new environment variable? Edit 9 `configmap.yaml` files. Change the readiness probe delay? Edit 9 `deployment.yaml` files.

**This is what Helm solves. See you in Day 3.**

---

# Part 2 — Day 3: Migration to Helm

## Prerequisites

- Day 2 complete: ArgoCD installed, pharma AppProject created, all 9 day2-raw apps running
- `helm` CLI installed

```bash
# Mac
brew install helm

# Linux / Cloud9
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:
```bash
helm version
argocd app list | grep "dev-raw"
# Should show 9 apps running
```

---

## Step 1 — What is Helm?

Helm is a package manager for Kubernetes. Instead of 9 × 4 = 36 YAML files with identical structure, you have one chart that templates everything:

```
helm-charts/              ← one chart for ALL 9 services
├── Chart.yaml
├── values.yaml           ← defaults
└── templates/
    ├── deployment.yaml   ← {{ .Values.image.tag }}, {{ .Values.replicaCount }}
    ├── service.yaml      ← {{ .Values.service.port }}
    ├── configmap.yaml    ← iterates over {{ .Values.configmap }}
    ├── serviceaccount.yaml
    ├── hpa.yaml          ← {{- if .Values.autoscaling.enabled }}
    └── ingress.yaml      ← {{- if .Values.ingress.enabled }}
```

Combined with per-service, per-environment values files:
```
envs/dev/values-auth-service.yaml   ← auth-service's port, image, config for dev
envs/qa/values-auth-service.yaml    ← same service, different values for qa
envs/prod/values-auth-service.yaml  ← production values
```

**One chart + one values file = all the manifests for one service in one environment.**

---

## Step 2 — Walk Through the Helm Chart

```bash
ls helm-charts/
ls helm-charts/templates/

# Read the deployment template — notice the Go template syntax
cat helm-charts/templates/deployment.yaml
```

Key template patterns:
```yaml
name: {{ include "pharma-service.fullname" . }}
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
{{- if .Values.autoscaling.enabled }}
# HPA is only rendered when autoscaling.enabled: true
{{- end }}
```

---

## Step 3 — Preview What Helm Renders

Before ArgoCD uses Helm, see what it produces:

```bash
helm template auth-service ./helm-charts \
  -f envs/dev/values-auth-service.yaml \
  -n dev | less
```

Compare with what you wrote manually in `lab2/manifests/auth-service/` — same Deployment structure, same ConfigMap keys, same Service spec.

**This is what ArgoCD runs internally every 3 minutes.** It renders the template, compares the output to the cluster, and applies the diff.

---

## Step 4 — Migrate auth-service from Raw to Helm

Delete the Day 2 raw app:
```bash
argocd app delete auth-service-dev-raw --yes

kubectl delete -f lab2/manifests/auth-service/ -n dev
```

Apply the Day 3 Helm app:
```bash
kubectl apply -f lab2/argocd-apps/day3-helm/dev/auth-service-app.yaml

argocd app get auth-service-dev
# SOURCE: helm-charts (Helm)
# STATUS: Synced
```

Watch pods:
```bash
kubectl get pods -n dev -w
```

The pod is identical to what the raw manifest deployed — same image, same config, same probes. But now the source is a Helm chart + a values file instead of 4 separate manifests.

```bash
# Confirm ArgoCD is using Helm
argocd app get auth-service-dev -o yaml | grep -A5 "source:"
```

---

## Step 5 — Migrate All 9 Services to Helm (dev)

Delete all Day 2 raw apps:
```bash
for app in $(argocd app list -o name | grep "dev-raw"); do
  argocd app delete $app --yes
done

for svc in auth-service api-gateway drug-catalog-service inventory-service \
           manufacturing-service supplier-service qc-service notification-service pharma-ui; do
  kubectl delete -f lab2/manifests/$svc/ -n dev --ignore-not-found
done
```

Apply all Day 3 Helm apps for dev:
```bash
for app in lab2/argocd-apps/day3-helm/dev/*.yaml; do
  kubectl apply -f $app
done

argocd app list
kubectl get pods -n dev -w
```

Wait for all 9 apps to show `Synced / Healthy`.

---

## Step 6 — Deploy QA Environment

First, make sure the `qa` namespace and its secrets exist:
```bash
kubectl get namespace qa || kubectl create namespace qa
kubectl get secret db-credentials -n qa 2>/dev/null || \
  kubectl get secret db-credentials -n dev -o yaml \
    | sed 's/namespace: dev/namespace: qa/' \
    | kubectl apply -f -
kubectl get secret jwt-secret -n qa 2>/dev/null || \
  kubectl get secret jwt-secret -n dev -o yaml \
    | sed 's/namespace: dev/namespace: qa/' \
    | kubectl apply -f -
```

Deploy:
```bash
for app in lab2/argocd-apps/day3-helm/qa/*.yaml; do
  kubectl apply -f $app
done

argocd app list | grep "\-qa"
kubectl get pods -n qa -w
```

QA uses automated sync — ArgoCD applies changes from `envs/qa/` automatically when you push to main.

---

## Step 7 — Deploy Production Environment (Manual Sync Gate)

First, ensure the `prod` namespace and secrets exist (same pattern as qa, replacing `qa` with `prod`):
```bash
kubectl get namespace prod || kubectl create namespace prod
kubectl get secret db-credentials -n prod 2>/dev/null || \
  kubectl get secret db-credentials -n dev -o yaml \
    | sed 's/namespace: dev/namespace: prod/' \
    | kubectl apply -f -
kubectl get secret jwt-secret -n prod 2>/dev/null || \
  kubectl get secret jwt-secret -n dev -o yaml \
    | sed 's/namespace: dev/namespace: prod/' \
    | kubectl apply -f -
```

Deploy the app specs:
```bash
for app in lab2/argocd-apps/day3-helm/prod/*.yaml; do
  kubectl apply -f $app
done

argocd app list | grep "\-prod"
# STATUS: OutOfSync  ← waiting for human approval
```

The prod apps are OutOfSync — ArgoCD knows what to apply but will not do it without a human trigger. **This is your compliance gate.**

Approve in the UI: click any prod application → **Sync** → review the diff → **Synchronize**.

Or via CLI:
```bash
for app in $(argocd app list -o name | grep "\-prod"); do
  argocd app sync $app
done
kubectl get pods -n prod
```

---

## Step 8 — Simulate a CI Image Tag Update (Full GitOps Loop)

This is the end-to-end GitOps workflow. In real life your CI pipeline does this after a successful build and test run.

```bash
# Simulate CI: update auth-service image tag in dev
# If you have yq installed:
yq e '.image.tag = "sha-classdemo"' -i envs/dev/values-auth-service.yaml

# Or with sed:
# macOS
sed -i '' 's/tag: sha-.*/tag: sha-classdemo/' envs/dev/values-auth-service.yaml
# Linux
sed -i 's/tag: sha-.*/tag: sha-classdemo/' envs/dev/values-auth-service.yaml

git add envs/dev/values-auth-service.yaml
git commit -m "ci(dev): update auth-service → sha-classdemo"
git push
```

ArgoCD detects the change within 3 minutes (or force a refresh now):
```bash
argocd app get auth-service-dev --watch
# STATUS changes: Synced → OutOfSync → Synced (automated sync fires)

kubectl get pods -n dev
kubectl describe pod -l app=auth-service -n dev | grep Image
# Image: 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-classdemo
```

**You just completed the full GitOps CD pipeline:**
1. Value changed in Git
2. ArgoCD detected the change
3. Helm rendered new manifests with the updated tag
4. Kubernetes performed a rolling update
5. Zero manual `kubectl` commands

---

## Step 9 — Final Verification

```bash
argocd app list
```

Expected:
```
NAME                          CLUSTER    NAMESPACE  STATUS  HEALTH     ...
api-gateway-dev               in-cluster dev        Synced  Healthy
auth-service-dev              in-cluster dev        Synced  Healthy
drug-catalog-service-dev      in-cluster dev        Synced  Healthy
inventory-service-dev         in-cluster dev        Synced  Healthy
manufacturing-service-dev     in-cluster dev        Synced  Healthy
notification-service-dev      in-cluster dev        Synced  Healthy
pharma-ui-dev                 in-cluster dev        Synced  Healthy
qc-service-dev                in-cluster dev        Synced  Healthy
supplier-service-dev          in-cluster dev        Synced  Healthy
... (qa and prod apps)
```

Confirm the UI is accessible:
```bash
kubectl get svc -n ingress-nginx
# Get the EXTERNAL-IP of ingress-nginx-controller
```

`http://<EXTERNAL-IP>/` should load the Zen Pharma UI. `http://<EXTERNAL-IP>/api/actuator/health` should return `{"status":"UP"}`.

---

## What You Built

```
EKS Cluster
├── dev namespace    — 9 services, Helm-managed, auto-sync
├── qa namespace     — 8 services, Helm-managed, auto-sync
└── prod namespace   — 8 services, Helm-managed, MANUAL sync gate

ArgoCD
└── pharma project
    ├── 9 × dev apps
    ├── 8 × qa apps    (qc-service excluded — no qa values file)
    └── 8 × prod apps  (qc-service excluded, manual sync gate)

zen-gitops repo
├── helm-charts/           ← one chart for all services
├── envs/dev|qa|prod/      ← per-service per-env values files
└── lab2/argocd-apps/      ← Application specs you applied
```

**Interview showcase:**
- Open ArgoCD UI, show all apps Synced/Healthy
- Update a values file, push, watch ArgoCD sync dev automatically
- Show a prod app OutOfSync waiting for the manual gate
- Explain: "This is how a code change travels from developer commit to production — with a human approval gate before prod"

---

# Part 3 — External Secrets Operator

## The Problem With Lab 1 Secrets

In Lab 1 you ran `kubectl create secret` by hand. Someone had to know the plaintext value, type it into a terminal, and hope it never ended up in a `.env` file or Slack message. That works for a lab. It does not work in production for three reasons:

1. **Secrets live only in etcd.** Base64-encoded, not encrypted by default. Anyone with `kubectl get secret` in that namespace can read the value. If the cluster is destroyed, the secrets are gone.
2. **Rotation is fully manual.** Changing a password means `kubectl delete secret` + `kubectl create secret` + pod restarts, repeated in every namespace that needs it.
3. **GitOps breaks down.** Your cluster state should be reproducible from Git. But you cannot commit plaintext secrets to Git. You are left with a gap — the cluster cannot be rebuilt from Git alone.

---

## What External Secrets Operator Does

ESO bridges **AWS Secrets Manager** (the encrypted source of truth) and **Kubernetes Secrets** (what pods actually consume). It watches for `ExternalSecret` CRDs in the cluster and materializes them into native K8s Secrets automatically.

```
AWS Secrets Manager                (encrypted, audited, IAM-gated)
  /pharma/dev/db-credentials  →  { "username": "...", "password": "..." }
  /pharma/dev/jwt-secret       →  { "secret": "..." }
        │
        │  ESO polls every refreshInterval via IRSA
        ▼
ClusterSecretStore                 (cluster-wide AWS connector)
        │
        │  ExternalSecret CRD maps remote keys → local K8s Secret keys
        ▼
K8s Secret: db-credentials         (materialized in dev/qa/prod namespace)
  DB_USERNAME: pharmaadmin
  DB_PASSWORD: ***
        │
        │  envFrom: secretRef in Deployment spec
        ▼
Pod reads DB_PASSWORD as an environment variable
```

The pod cannot tell the difference from a hand-created secret. The entire advantage is in everything that happens before and after.

---

## Step 1 — Prerequisites

**Terraform must have run Stage 1 first.** The setup script relies on an IAM role that Terraform creates. Confirm:

```bash
# Verify the ESO IAM role exists (default naming convention: pharma-<env>-eso-role)
aws iam get-role --role-name pharma-dev-eso-role \
  --query 'Role.Arn' --output text
# Expected: arn:aws:iam::<account-id>:role/pharma-dev-eso-role
```

If this returns an error, run `terraform apply` in `zen-infra` before continuing.

Also confirm the Secrets Manager paths exist:
```bash
aws secretsmanager get-secret-value \
  --secret-id /pharma/dev/db-credentials \
  --query 'SecretString' --output text
# Expected: {"username":"pharmaadmin","password":"..."}

aws secretsmanager get-secret-value \
  --secret-id /pharma/dev/jwt-secret \
  --query 'SecretString' --output text
# Expected: {"secret":"..."}
```

If either is missing, create it in the AWS console (Secrets Manager → Store a new secret → Other type of secret) before continuing.

---

## Step 2 — Install External Secrets Operator

ESO is installed via Helm into its own namespace:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true

# Wait for ESO pods to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=external-secrets \
  -n external-secrets --timeout=120s

kubectl get pods -n external-secrets
```

Expected — all pods Running:
```
NAME                                               READY   STATUS    RESTARTS
external-secrets-xxx                               1/1     Running   0
external-secrets-cert-controller-xxx               1/1     Running   0
external-secrets-webhook-xxx                       1/1     Running   0
```

---

## Step 3 — Run the Setup Script

The script at `zen-infra/scripts/03-setup-external-secrets.sh` does four things in order:
1. Annotates the ESO service account with your IRSA role ARN
2. Creates the `ClusterSecretStore` pointing to AWS Secrets Manager
3. Creates `ExternalSecret` resources for `db-credentials` and `jwt-secret` in the target namespace
4. Polls until both secrets are confirmed synced

Run it from the **zen-infra** directory:

```bash
cd /path/to/zen-infra
bash scripts/03-setup-external-secrets.sh
```

The script will prompt you for four values:

```
1. Target environment    → dev  (or qa / prod)
2. AWS region            → us-east-1
3. AWS account ID        → your 12-digit account number
4. ESO IAM role name     → pharma-dev-eso-role  (press Enter for default)
```

**What IRSA means (the script explains this too):** IRSA lets a Kubernetes ServiceAccount assume an AWS IAM role. The ESO pod exchanges its ServiceAccount JWT token for short-lived AWS credentials via EKS's OIDC provider. No passwords or access keys are stored anywhere in the cluster.

Successful output ends with:
```
OK  Both secrets synced successfully into namespace 'dev'.
OK  External Secrets setup complete.
```

Repeat for each environment you deployed:
```bash
# Run once per environment — the script prompts for which one
bash scripts/03-setup-external-secrets.sh   # choose qa
bash scripts/03-setup-external-secrets.sh   # choose prod
```

---

## Step 4 — Verify the Sync

```bash
# Check ExternalSecret status in the dev namespace
kubectl get externalsecret -n dev
```

Expected:
```
NAME             STORE                 REFRESH INTERVAL   STATUS         READY
db-credentials   aws-secrets-manager   1h                 SecretSynced   True
jwt-secret       aws-secrets-manager   1h                 SecretSynced   True
```

Confirm the K8s Secrets were materialized:
```bash
kubectl get secret db-credentials jwt-secret -n dev
```

Inspect the keys ESO created (base64-encoded values, not plaintext):
```bash
kubectl get secret db-credentials -n dev -o jsonpath='{.data}' | python3 -m json.tool
```

You should see four keys: `DB_USERNAME`, `DB_PASSWORD`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`. The Spring Boot services use `SPRING_DATASOURCE_*` while others use `DB_*` — ESO materializes all four from the same Secrets Manager entry.

---

## Step 5 — Replace Manual Secrets With ESO-Managed Ones

The Lab 1 secrets you created with `kubectl create secret` were created with `creationPolicy: Owner` by ESO — ESO will not overwrite secrets it does not own. Delete the old hand-created ones so ESO takes ownership:

```bash
# Only if your secrets were created manually in Lab 1 (not by a previous ESO run)
kubectl delete secret db-credentials jwt-secret -n dev

# ESO detects the deletion and immediately re-creates them
# (it reconciles continuously — no need to restart anything)
kubectl get secret db-credentials jwt-secret -n dev
# They re-appear within seconds, now owned by ESO
```

Restart the deployments so pods pick up the freshly-materialized secrets:
```bash
for svc in auth-service api-gateway drug-catalog-service inventory-service \
           manufacturing-service supplier-service qc-service notification-service; do
  kubectl rollout restart deployment/$svc -n dev
done

kubectl get pods -n dev -w
```

Wait for all pods to come back to `1/1 Running`. Verify auth-service is healthy:
```bash
kubectl exec -n dev deploy/auth-service -- \
  curl -s http://localhost:8081/actuator/health
# Expected: {"status":"UP"}
```

---

## The Three ESO Objects Explained

### 1. ClusterSecretStore

Cluster-wide connector to AWS. Defined once, used by all namespaces. The script creates this pointing at the `external-secrets` ServiceAccount (in the `external-secrets` namespace) which holds the IRSA annotation:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

### 2. ExternalSecret per namespace

One per secret per environment. Safe to commit to Git — contains only AWS paths, never values.

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: dev
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: /pharma/dev/db-credentials
        property: username
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /pharma/dev/db-credentials
        property: password
    - secretKey: SPRING_DATASOURCE_USERNAME
      remoteRef:
        key: /pharma/dev/db-credentials
        property: username
    - secretKey: SPRING_DATASOURCE_PASSWORD
      remoteRef:
        key: /pharma/dev/db-credentials
        property: password
```

The `/pharma/dev/` path prefix is how environment isolation is enforced — different IAM policies can restrict which roles can read which paths.

### 3. The materialized K8s Secret

ESO creates this automatically. Deployments consume it exactly as they consumed the Lab 1 hand-created secret:

```yaml
envFrom:
  - secretRef:
      name: db-credentials    # DB_USERNAME, DB_PASSWORD, SPRING_DATASOURCE_* become env vars
  - secretRef:
      name: jwt-secret        # JWT_SECRET becomes an env var
```

---

## ESO vs Native K8s Secrets

| Concern | Lab 1 (native kubectl) | ESO + AWS Secrets Manager |
|---|---|---|
| Source of truth | etcd (base64, not encrypted by default) | AWS Secrets Manager (encrypted, CloudTrail-audited) |
| Who can read plaintext | Anyone with `kubectl get secret` in that namespace | Only IAM roles you explicitly grant |
| Rotation | Manual kubectl + pod restart | Update in AWS SM → ESO picks it up on next `refreshInterval` |
| Audit trail | None | Every read/write logged in AWS CloudTrail |
| Git safety | Cannot commit secrets; cluster not reproducible from Git | `ExternalSecret` CRD committed (path pointer only); values never touch Git |
| Cross-env isolation | You manage 3 copies manually | Same ExternalSecret pattern, different AWS path, IAM enforces isolation |
| Disaster recovery | Secrets gone if cluster is destroyed | Secrets survive cluster deletion; reapply ExternalSecret and they re-materialize |

---

## The GitOps-Specific Reason

The entire cluster state should be reproducible by running `kubectl apply` against the Git repo. With native secrets you always have a gap — someone must run the manual `kubectl create secret` step first, out of band.

With ESO, you commit the `ExternalSecret` CRD to Git (safe — it contains only paths). ArgoCD applies it. ESO reads it and fetches the actual values from AWS. The cluster rebuilds itself completely from Git + AWS, with no manual secret injection step.

`k8s/external-secrets/dev-external-secrets.yaml` is committed to this repo. It contains zero sensitive data — just the AWS Secrets Manager paths like `/pharma/dev/db-credentials`. The actual credentials never leave AWS.

---

## Troubleshooting ESO

### ExternalSecret status is `SecretSyncedError`

```bash
kubectl describe externalsecret db-credentials -n dev
# Look at the Events section at the bottom
```

Common causes and fixes:

| Error in Events | Cause | Fix |
|---|---|---|
| `AccessDeniedException` | IAM role missing `secretsmanager:GetSecretValue` | Check the IAM policy attached to `pharma-dev-eso-role` in AWS console |
| `ResourceNotFoundException` | Secret path does not exist in Secrets Manager | Create `/pharma/dev/db-credentials` in AWS Secrets Manager |
| `WebIdentityErr` | OIDC provider not configured or IRSA annotation missing | Re-run the setup script; check `kubectl get sa external-secrets -n external-secrets -o yaml` for the annotation |
| `unable to get provider` | ClusterSecretStore not created yet | `kubectl get clustersecretstore aws-secrets-manager` — if absent, re-run the script |

### Check IRSA annotation is present

```bash
kubectl get sa external-secrets -n external-secrets \
  -o jsonpath='{.metadata.annotations}'
# Expected: {"eks.amazonaws.com/role-arn":"arn:aws:iam::<account>:role/pharma-dev-eso-role"}
```

If missing, re-run the setup script or annotate manually:
```bash
kubectl annotate sa external-secrets -n external-secrets \
  eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/pharma-dev-eso-role \
  --overwrite
kubectl rollout restart deployment/external-secrets -n external-secrets
```

---

# Troubleshooting

## ArgoCD app stuck in OutOfSync after fork

**Symptom:** App shows OutOfSync with error `repository not permitted`.

**Cause:** The AppProject `sourceRepos` does not include your fork URL.

**Fix:**
```bash
kubectl get appproject pharma -n argocd -o yaml | grep sourceRepos -A5
# If your fork URL is not listed, re-apply after running Step 1c again
kubectl apply -f lab2/argocd-apps/project/pharma-project.yaml
```

## Pods in CrashLoopBackOff — `UnknownHostException`

**Symptom:**
```
Caused by: java.net.UnknownHostException: pharma-dev-postgres.<id>.us-east-1.rds.amazonaws.com
```

**Cause:** The `DB_HOST` ConfigMap entry points to a wrong or stale endpoint.

**Fix:**
```bash
export DB_HOST=$(aws rds describe-db-instances \
  --query 'DBInstances[?DBInstanceIdentifier==`pharma-dev-postgres`].Endpoint.Address' \
  --output text)

# Update configmaps (see Step 2 above), commit, push
# ArgoCD will apply the corrected ConfigMap on the next sync cycle
argocd app sync <service-name>-dev-raw   # or -dev for Helm apps
kubectl rollout restart deployment/<service-name> -n dev
```

## 404 Not Found on the LoadBalancer URL

**Symptom:** `http://<LB>/` returns nginx 404.

**Cause:** The pharma-ui ingress resource was not applied. ArgoCD only manages what is explicitly in the Application's source path.

**Fix:**
```bash
argocd app get pharma-ui-dev-raw
# Look for sync errors — if the ingress.yaml is missing from the repo, add it
kubectl get ingress -n dev
# If pharma-ui ingress is absent, check that lab2/manifests/pharma-ui/ingress.yaml exists and was pushed
```

## Pods in ImagePullBackOff

**Symptom:** Events show `failed to pull image ... not found`

**Fix:**
```bash
aws ecr describe-images \
  --repository-name auth-service \
  --query 'sort_by(imageDetails,&imagePushedAt)[-5:].imageTags' \
  --output table
```

Update the image tag in the relevant values file (Helm) or `deployment.yaml` (raw), commit, push.

## Pending pods — insufficient resources

**Symptom:** Pods stuck in `Pending`. Events: `Insufficient memory` or `Too many pods`.

**Cause:** During a rolling update, old and new pods coexist. If the cluster is near capacity, new pods cannot be scheduled.

**Fix:**
```bash
kubectl get pods -n dev
kubectl delete pod <pod-name-in-crashloopbackoff> -n dev
```

---

## Cleanup

To remove a specific environment:
```bash
# Delete all ArgoCD apps for dev
for app in $(argocd app list -o name | grep "\-dev"); do
  argocd app delete $app --yes
done

# Or delete the namespace entirely
kubectl delete namespace dev
```

To remove ArgoCD:
```bash
kubectl delete namespace argocd
```
