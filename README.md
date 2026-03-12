# Kubernetes manifests – Library app

This folder contains Kubernetes manifests to deploy the **Library app** container (`owolabitemi/libary-clean:v1.0.0`) into the `libary-prod` namespace, expose it internally via a ClusterIP Service, and route traffic via an Ingress host.

## What’s in this folder

- **`deployment.yaml`**: Creates a `Deployment` named `libary-app` (2 replicas) running on port `3000`.
  - **Rollout strategy**: RollingUpdate with `maxUnavailable: 0` and `maxSurge: 1` (aims for zero-downtime deploys).
  - **Health checks**: readiness + liveness probes on `GET /health` at port `3000`.
  - **Config/Secrets**: loads all env vars from:
    - `ConfigMap` `libary-config-map`
    - `Secret` `libary-secrets`

- **`service.yaml`**: Creates a `Service` named `libary-service` (type `ClusterIP`) exposing:
  - Service port `80` → Pod `targetPort: 3000`
  - Selects pods with label `app: library-app`

- **`ingress.yaml`**: Creates an `Ingress` named `libary-ingress` routing:
  - Host `libary.local`
  - Path `/` → `libary-service:80`

- **`configmap.yaml`**: Creates `ConfigMap` `libary-config-map` with:
  - `NODE_ENV=production`

- **`secrets.yaml`**: Creates `Secret` `libary-secrets` with `stringData` values (Kubernetes base64-encodes them at rest).

## Prerequisites

- A Kubernetes cluster and `kubectl` configured
- The `libary-prod` namespace must exist (this repo does not include a Namespace manifest)
- An Ingress controller installed (e.g. NGINX Ingress) for `Ingress` to work

## Deploy

Create the namespace (once):

```bash
kubectl create namespace libary-prod
```

Apply manifests (recommended order):

```bash
kubectl apply -f configmap.yaml
kubectl apply -f secrets.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

Check status:

```bash
kubectl -n libary-prod get deploy,po,svc,ingress
kubectl -n libary-prod describe ingress libary-ingress
```

## Access the app (Ingress)

The Ingress uses host **`libary.local`**. If you’re running locally (e.g. Docker Desktop Kubernetes, kind, minikube), map the Ingress controller IP to that host in `/etc/hosts`.

Example (replace `INGRESS_IP` with your controller’s address):

```bash
kubectl -n ingress-nginx get svc
# then add: INGRESS_IP  libary.local
```

Then browse:

- `http://libary.local/`

## Notes / gotchas

- **Secrets are currently plaintext**: `secrets.yaml` contains a real `DATABASE_URL`. Avoid committing real credentials. Prefer one of:
  - a local-only `secrets.yaml` not committed to VCS
  - an external secret manager (External Secrets Operator)
  - Sealed Secrets / SOPS
  - injecting at deploy time via CI/CD

- **Label spelling**: the `Deployment` is named `libary-app` but uses the label `app: library-app` (with an extra “r” in “library”). This is fine because `Service` selects the label, not the resource name.

