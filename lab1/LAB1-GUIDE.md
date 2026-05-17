# Session 1 Lab Guide — Deploy Zen Pharma Platform Manually

**Prerequisites:** Your EKS cluster is up and `kubectl` is configured against it.

Verify before starting:
```bash
kubectl cluster-info
kubectl get nodes
```

You should see your nodes in `Ready` state. If not, fix your kubeconfig before continuing.

---

## What We Are Deploying

The Zen Pharma platform has 9 microservices. Today you will deploy all of them to the `dev` namespace using raw Kubernetes manifests — no automation, no templating, just `kubectl apply`.

| Service | Port | Language | Secrets Needed |
|---|---|---|---|
| auth-service | 8081 | Spring Boot | db-credentials, jwt-secret |
| api-gateway | 8080 | Spring Boot | db-credentials, jwt-secret |
| drug-catalog-service | 8082 | Spring Boot | db-credentials |
| inventory-service | 8083 | Spring Boot | db-credentials |
| manufacturing-service | 8085 | Spring Boot | db-credentials |
| supplier-service | 8084 | Spring Boot | db-credentials |
| qc-service | 8086 | Spring Boot | none |
| notification-service | 3000 | Node.js | db-credentials |
| pharma-ui | 80 | React / nginx | none |

By the end of this lab, all 9 services will be running in the `dev` namespace.

---

## Step 1 — Clone the Repository

```bash
git clone https://github.com/<your-org>/zen-gitops.git
cd zen-gitops
```

All manifest files for this lab are in the `lab1/` directory.

---

## Step 2 — Create the Namespace

```bash
kubectl apply -f lab1/00-namespace.yaml
```

Verify:
```bash
kubectl get namespace dev
```

Expected output:
```
NAME   STATUS   AGE
dev    Active   5s
```

---

## Step 3 — Apply RBAC

```bash
kubectl apply -f lab1/01-rbac.yaml
```

This creates a `pharma-deployer` Role in the `dev` namespace that allows deployment operations.

Verify:
```bash
kubectl get role pharma-deployer -n dev
kubectl get rolebinding pharma-deployer-binding -n dev
```

---

## Step 4 — Create Secrets

Secrets must exist **before** pods start. Pods that reference a missing secret will fail immediately with `CreateContainerConfigError`.

```bash
# Database credentials — used by 7 of the 9 services
kubectl create secret generic db-credentials \
  --from-literal=DB_USERNAME=pharma_user \
  --from-literal=DB_PASSWORD=pharmaPass123 \
  -n dev

# JWT signing key — used by auth-service and api-gateway
kubectl create secret generic jwt-secret \
  --from-literal=JWT_SECRET=mysupersecretjwtkey256bitslongkey \
  -n dev
```

Verify:
```bash
kubectl get secrets -n dev
```

Expected output:
```
NAME             TYPE     DATA   AGE
db-credentials   Opaque   2      5s
jwt-secret       Opaque   1      3s
```

Peek at the values (note: base64 is NOT encryption):
```bash
kubectl get secret db-credentials -n dev \
  -o jsonpath='{.data.DB_USERNAME}' | base64 -d && echo
```

> **Discussion point:** We are typing passwords directly into the terminal. The secret is stored in etcd as base64. Anyone with `kubectl get secret` access can decode it. In Session 3, External Secrets Operator will replace this step entirely — secrets will be pulled from AWS Secrets Manager automatically.

---

## Step 5 — Deploy auth-service First

We start with auth-service because it is the most complete service — it uses a ConfigMap, two Secrets, a ServiceAccount, liveness and readiness probes, and a read-only filesystem with a `/tmp` volume.

```bash
kubectl apply -f lab1/auth-service.yaml
```

This creates 4 resources at once:
- `ServiceAccount/auth-service`
- `ConfigMap/auth-service`
- `Deployment/auth-service`
- `Service/auth-service`

Watch the pod come up:
```bash
kubectl get pods -n dev -w
```

Press `Ctrl+C` when the pod reaches `Running`.

Describe the pod to understand every field:
```bash
kubectl describe pod -l app=auth-service -n dev
```

Read through the output. Focus on:
- `Events` section at the bottom — this is your first debug tool
- `Containers.auth-service.Liveness` and `Readiness` — see the probe config
- `Volumes` — note the `tmp` emptyDir

Check the logs:
```bash
kubectl logs -l app=auth-service -n dev
```

> **Expected:** The pod will be `Running` but may show `0/1 READY` if it cannot reach the RDS database. This is intentional — the readiness probe checks `/actuator/health/readiness` which returns `DOWN` when the database is unreachable. The pod gets no traffic until the probe passes. We will revisit this in Session 3 when we wire up secrets from AWS Secrets Manager.

Test the health endpoint from inside the pod:
```bash
kubectl exec -n dev deploy/auth-service -- \
  curl -s http://localhost:8081/actuator/health
```

Check Service endpoints (will be empty if pod is not ready):
```bash
kubectl get endpoints auth-service -n dev
```

---

## Step 6 — Deploy the Remaining 8 Services

Now deploy all remaining services one by one. After each one, observe what happens.

```bash
kubectl apply -f lab1/api-gateway.yaml
kubectl apply -f lab1/drug-catalog-service.yaml
kubectl apply -f lab1/inventory-service.yaml
kubectl apply -f lab1/manufacturing-service.yaml
kubectl apply -f lab1/supplier-service.yaml
kubectl apply -f lab1/qc-service.yaml
kubectl apply -f lab1/notification-service.yaml
kubectl apply -f lab1/pharma-ui.yaml
```

Watch all pods come up together:
```bash
kubectl get pods -n dev -w
```

---

## Step 7 — Verify Everything

Check all resources in the dev namespace:
```bash
kubectl get all -n dev
```

You should see:
- 9 Deployments
- 9 ReplicaSets
- 9 Pods
- 9 Services
- 2 Ingresses (api-gateway and pharma-ui)

Check ConfigMaps and ServiceAccounts:
```bash
kubectl get configmaps -n dev
kubectl get serviceaccounts -n dev
```

List only the pods and their status:
```bash
kubectl get pods -n dev -o wide
```

---

## Step 8 — Explore and Troubleshoot

### View logs for any service
```bash
kubectl logs -l app=<service-name> -n dev

# Example:
kubectl logs -l app=api-gateway -n dev
```

### Describe a pod to see events and config
```bash
kubectl describe pod -l app=<service-name> -n dev
```

### Execute a command inside a running pod
```bash
# Test internal DNS — api-gateway should be able to reach auth-service by name
kubectl exec -n dev deploy/api-gateway -- \
  curl -s http://auth-service:8081/actuator/health
```

### Check that the Service has endpoints
```bash
# An empty ENDPOINTS column means the pod is not passing readiness
kubectl get endpoints -n dev
```

### View events for the whole namespace (sorted by time)
```bash
kubectl get events -n dev --sort-by='.lastTimestamp'
```

---

## Step 9 — See the Pain Point

You have just deployed 9 services. Count what you did manually:

| What | Count |
|---|---|
| Secrets created by hand | 2 |
| Files applied | 11 |
| Resources created | ~40 (Deployments, Services, ConfigMaps, SAs, ReplicaSets, Pods, Ingresses) |
| Environments covered | 1 (dev only) |

Now imagine doing this for **qa** and **prod** as well — with different image tags, different replica counts, different resource limits, and different config values per environment.

That is 9 services × 3 environments = **27 deployments**, all maintained by hand, with no audit trail, no rollback, and no guarantee that dev and prod configs match.

This is exactly the problem that **Helm + ArgoCD** solves in Session 2.

---

## Cleanup (Optional)

To remove everything you deployed:

```bash
kubectl delete namespace dev
```

This deletes the namespace and everything inside it — all pods, services, configmaps, secrets, and deployments.

---

## Summary

| Step | Command |
|---|---|
| Create namespace | `kubectl apply -f lab1/00-namespace.yaml` |
| Apply RBAC | `kubectl apply -f lab1/01-rbac.yaml` |
| Create secrets | `kubectl create secret generic ...` |
| Deploy auth-service | `kubectl apply -f lab1/auth-service.yaml` |
| Deploy all others | `kubectl apply -f lab1/<service>.yaml` |
| Verify | `kubectl get all -n dev` |
| Debug | `kubectl describe pod / kubectl logs / kubectl get events` |
| Cleanup | `kubectl delete namespace dev` |
