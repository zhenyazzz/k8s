# Kubernetes: project deployment

This guide covers deploying the microservices from the `k8s/` directory to a local cluster (**Minikube**) or any Kubernetes cluster with similar capabilities.

All manifests assume the **`default` namespace** and **Ingress NGINX** (`ingressClassName: nginx`).

---

## Layer layout

| Directory | Contents |
|-----------|----------|
| `01-shared` | ConfigMap, Infisical SecretStore, ExternalSecrets for shared secrets, NetworkPolicy, PodDisruptionBudget |
| `02-infrastructure` | Redis, Kafka, MongoDB |
| `03-databases` | PostgreSQL (auth-db, user-db, order-db) |
| `04-apps` | auth, user, order, payment |
| `05-gateway` | gateway-service, Ingress |

The application services and gateway are scaled to **2 replicas**; database StatefulSets, Kafka, MongoDB, and Redis use **a single replica** (lab setup without data-tier HA).

---

## 1. Prerequisites

Install and verify:

- **Docker** (for the Minikube driver)
- **kubectl**
- **Minikube**
- **Helm** (for External Secrets Operator)

```bash
kubectl version --client
kubectl config current-context
helm version
```

---

## 2. Starting Minikube

Recommended resources for several databases and double app replicas (tune to your machine):

```bash
minikube start --driver=docker --cpus=4 --memory=6144 --disk-size=40g
```

Enable Ingress (the **ingress-nginx** controller):

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

Check API access:

```bash
kubectl cluster-info
kubectl get nodes
```

---

## 3. External Secrets Operator (ESO)

Application secrets are delivered via **External Secrets Operator** and **Infisical**, not `stringData` in git.

Add the chart repository and install the release (pick a chart version that matches your ESO image; list available pairs with):

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm search repo external-secrets/external-secrets --versions
```

Example install (chart **2.4.1** matches app **v2.4.1**):

```bash
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace \
  --version 2.4.1
```

Verify:

```bash
kubectl get pods -n external-secrets
```

Upgrade the release if needed:

```bash
helm upgrade external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --version 2.4.1
```

---

## 4. Infisical and bootstrap secret

The operator uses Infisical **Universal Auth**. Before applying the `SecretStore`, create a cluster secret for the machine identity (use values from Infisical):

```bash
kubectl create secret generic infisical-universal-auth \
  --from-literal=clientId='<INFISICAL_CLIENT_ID>' \
  --from-literal=clientSecret='<INFISICAL_CLIENT_SECRET>'
```

Ensure the project and environment referenced in `01-shared/infisical-secret-store.yaml` contain the folders and keys the manifests expect, including:

- `shared-secrets` (e.g. JWT)
- `auth-db`, `user-db`, `order-db`
- `mongodb`, `payment-mongo`

Key naming examples are in `k8s/infisical-import/*.env` (for manual upload to Infisical, not for automatic injection into the cluster).

---

## 5. Apply order

Apply in order: shared config and policies → infrastructure → databases → apps → gateway and Ingress.

From the **repository root**:

```bash
kubectl apply -f k8s/01-shared/
kubectl apply -f k8s/02-infrastructure/
kubectl apply -f k8s/03-databases/
kubectl apply -f k8s/04-apps/
kubectl apply -f k8s/05-gateway/
```

Wait until pods are ready (gateway and apps should show **`2/2`** when two replicas are configured):

```bash
kubectl get pods
kubectl get pods -w
```

Check secret sync:

```bash
kubectl get externalsecret
kubectl get secret
```

Status of a specific ExternalSecret:

```bash
kubectl describe externalsecret shared-secrets
```

---

## 6. Accessing the API

### Via Ingress

`05-gateway/ingress.yaml` uses host **`localhost`** and class **nginx**. After the ingress controller is ready:

```bash
kubectl get ingress
```

On Windows, you may need a separate terminal:

```bash
minikube tunnel
```

Then test:

```bash
curl http://localhost/
```

### Via port-forward (reliable for debugging)

```bash
kubectl port-forward svc/gateway-service 8080:8080
```

Gateway: `http://localhost:8080`

---

## 7. Logs and troubleshooting

Deployment logs:

```bash
kubectl logs deploy/gateway-service --tail=100
kubectl logs deploy/user-service --tail=100
```

StatefulSet logs (pod name ends with `-0`):

```bash
kubectl logs user-db-0 --tail=80
kubectl logs mongodb-0 --tail=80
```

For `CrashLoopBackOff`:

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
```

Common issues:

| Symptom | What to check |
|---------|----------------|
| ExternalSecret does not create a Secret | `kubectl describe externalsecret ...`, `infisical-universal-auth` secret, Infisical connectivity |
| App pods not starting | deployment logs, database / Redis / Kafka readiness |
| Out of RAM/CPU | increase Minikube resources or lower limits in manifests |
| Ingress 404 / no response | `kubectl get pods -n ingress-nginx`, host and annotations on the Ingress |

---

## 8. Updating manifests

After editing YAML:

```bash
kubectl apply -f k8s/01-shared/
kubectl apply -f k8s/02-infrastructure/
kubectl apply -f k8s/03-databases/
kubectl apply -f k8s/04-apps/
kubectl apply -f k8s/05-gateway/
```

Force a new rollout if the image tag was unchanged but you need a fresh pull:

```bash
kubectl rollout restart deployment/gateway-service
kubectl rollout status deployment/gateway-service
```

---

## 9. Removing project resources

Reverse order (avoids leaving dangling dependencies):

```bash
kubectl delete -f k8s/05-gateway/
kubectl delete -f k8s/04-apps/
kubectl delete -f k8s/03-databases/
kubectl delete -f k8s/02-infrastructure/
kubectl delete -f k8s/01-shared/
```

Stop Minikube:

```bash
minikube stop
```

Delete the Minikube profile entirely (wipes cluster data):

```bash
minikube delete
```

---

## Quick pre-demo checklist

1. Minikube is running and the **ingress** addon is enabled.
2. ESO is installed; pods in `external-secrets` are healthy.
3. **infisical-universal-auth** secret exists; required Infisical folders are filled in.
4. `kubectl apply` was run for directories **01 → 05**.
5. `kubectl get pods`: critical services are **Running**; apps show **2/2** where two replicas are set.
6. Gateway is reachable via **port-forward** or **Ingress**.
