# AKS Store Demo Lab — deploy one component at a time

A teaching version of [Azure-Samples/aks-store-demo](https://github.com/Azure-Samples/aks-store-demo).
The upstream project ships one big `aks-store-all-in-one.yaml`. Here it is **split into one file per
Kubernetes object, grouped by component and numbered in deployment order**, so students apply and verify
each piece separately and see what it does.

```
manifests/
  00-namespace/          Namespace "pet-store"
  01-documentdb/         statefulset, service          (order database)
  02-rabbitmq/           secret, statefulset, service  (order queue)
  03-order-service/      configmap, secret, deployment, service   (Node.js)
  04-makeline-service/   deployment, service           (Go, queue -> database)
  05-ai-service/         configmap, deployment, service (Python, calls Azure OpenAI)
  06-product-service/    deployment, service           (Rust)
  07-store-front/        deployment, service           (Vue.js, LoadBalancer)
  08-store-admin/        deployment, service           (Vue.js, LoadBalancer)
  09-virtual-customer/   deployment                    (simulates orders)
  10-virtual-worker/     deployment                    (simulates order completion)
src/                     Full application source code (copied from upstream)
docs/STEPS.md            The step-by-step lab guide  <-- start here
```

## Architecture

| Layer | Components |
|---|---|
| Frontend (LoadBalancer) | `store-front`, `store-admin` (Vue.js) |
| Backend (ClusterIP) | `order-service` (Node), `product-service` (Rust), `makeline-service` (Go), `ai-service` (Python) |
| Data and messaging | `rabbitmq` (order queue), `documentdb` (MongoDB-compatible order DB) |
| External | Azure OpenAI (product descriptions) |
| Simulators | `virtual-customer`, `virtual-worker` |

**Order flow:** store-front / virtual-customer -> order-service -> RabbitMQ -> makeline-service -> DocumentDB.
**AI flow:** store-admin -> product-service -> ai-service -> Azure OpenAI.

## Quick start

Follow [docs/STEPS.md](docs/STEPS.md). The deployment commands in short:

```bash
kubectl apply -f manifests/00-namespace/
kubectl config set-context --current --namespace=pet-store
kubectl apply -f manifests/01-documentdb/        # then verify, then continue with 02, 03 ...
```

> **Demo only.** DocumentDB and RabbitMQ run without persistent storage and with default credentials
> (`username` / `password`). Use managed services and Key Vault in production.

## Source and license

Application code and original manifests: Microsoft, MIT licensed (see `LICENSE.md`), from
<https://github.com/Azure-Samples/aks-store-demo> (images used: tag `2.2.0`). Manifests here are split and
reordered for teaching; `05-ai-service` is rewritten to use a ConfigMap and a Secret.
