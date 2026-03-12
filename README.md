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

- **`kind-config.yaml`**: kind cluster configuration for local testing.
  - Labels the control-plane node with `ingress-ready=true` (commonly used by Ingress controller installs on kind).
  - Maps kind node ports `80` and `443` to your machine’s `80` and `443`, so you can reach Ingress via `http://localhost` / `https://localhost`.

## Prerequisites

- A Kubernetes cluster and `kubectl` configured
- The `libary-prod` namespace must exist (this repo does not include a Namespace manifest)
- An Ingress controller installed (e.g. NGINX Ingress) for `Ingress` to work

## Local cluster with kind (recommended)

Create a kind cluster using the provided config:

```bash
kind create cluster --name libary --config kind-config.yaml
```

Install an Ingress controller (example: ingress-nginx for kind):

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl -n ingress-nginx wait --for=condition=Available deploy/ingress-nginx-controller --timeout=180s
```

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
kubectl rollout status deployment/libary-app -n libary-prod
```

## Access the app (Ingress)

The Ingress uses host **`libary.local`**.

- If you’re using **kind + `kind-config.yaml`**, ports `80/443` are mapped to your machine, so you can point `libary.local` to `127.0.0.1`.
- Otherwise (other clusters), map `libary.local` to your Ingress controller / LoadBalancer IP.

Example `/etc/hosts` entry:

```bash
# kind (with this repo’s kind-config.yaml)
127.0.0.1 libary.local
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

