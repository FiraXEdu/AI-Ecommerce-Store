# Step-by-step lab guide

Do **one step at a time**. After each step, run the verify command and explain to the class what happened
before moving on. Every step lists *Concept*, *Apply*, *Verify*.

## Step 0 — Prerequisites and variables

- Azure subscription, [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) (or Cloud Shell), `kubectl`
- Rights to create AKS and Azure OpenAI resources

```bash
export RG=pet-store-rg
export LOCATION=eastus
export AKS_NAME=pet-store-aks
export AOAI_NAME=petstore-aoai-$RANDOM      # must be globally unique
git clone <your-repo-url> && cd aks-store-demo-lab
```

## Step 1 — Create the AKS cluster

*Concept:* a resource group holds Azure resources; AKS gives a managed Kubernetes control plane plus worker nodes.

```bash
az group create --name $RG --location $LOCATION
az aks create --resource-group $RG --name $AKS_NAME --generate-ssh-keys
az aks get-credentials --resource-group $RG --name $AKS_NAME
```
*Verify:* `kubectl get nodes` -> all nodes `Ready`.

## Step 2 — Namespace

*Concept:* a namespace isolates the app's objects from the rest of the cluster.

```bash
kubectl apply -f manifests/00-namespace/
kubectl config set-context --current --namespace=pet-store
```
*Verify:* `kubectl get ns pet-store`

## Step 3 — Database: DocumentDB

*Concept:* **StatefulSet** (stable identity, for stateful workloads) vs Deployment; **ClusterIP Service** gives a stable
internal DNS name (`documentdb`); startup/readiness/liveness **probes**; resource requests and limits.

Read `manifests/01-documentdb/statefulset.yaml`, then apply each file separately:

```bash
kubectl apply -f manifests/01-documentdb/statefulset.yaml
kubectl get pods -w                 # wait for 1/1 Running, then Ctrl+C
kubectl apply -f manifests/01-documentdb/service.yaml
```
*Verify:* `kubectl get statefulset,svc,pods` and `kubectl logs documentdb-0 --tail=20`

## Step 4 — Message queue: RabbitMQ

*Concept:* a **Secret** stores credentials (base64 is encoding, not encryption); `envFrom` injects it into the pod;
a queue decouples order placement from order processing.

```bash
kubectl apply -f manifests/02-rabbitmq/secret.yaml
kubectl get secret rabbitmq-secrets -o jsonpath='{.data.RABBITMQ_DEFAULT_USER}' | base64 -d; echo
kubectl apply -f manifests/02-rabbitmq/statefulset.yaml
kubectl apply -f manifests/02-rabbitmq/service.yaml
```
*Verify:* `kubectl get pods` -> `rabbitmq-0` Running. Optional UI: `kubectl port-forward svc/rabbitmq 15672:15672`,
open <http://localhost:15672> (login `username` / `password`).

## Step 5 — order-service (Node.js)

*Concept:* **ConfigMap** (non-secret config) + Secret; **initContainer** `wait-for-rabbitmq` delays the app until the
queue is reachable; this service **publishes** orders to RabbitMQ.

```bash
kubectl apply -f manifests/03-order-service/configmap.yaml
kubectl apply -f manifests/03-order-service/secret.yaml
kubectl apply -f manifests/03-order-service/deployment.yaml
kubectl get pods -w
kubectl apply -f manifests/03-order-service/service.yaml
```
*Verify:* run a throw-away pod to call the internal service (works from any terminal):
```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.37.0 -- wget -qO- http://order-service:3000/health
```
*Experiment:* delete `rabbitmq` (`kubectl delete -f manifests/02-rabbitmq/statefulset.yaml`) and recreate
`order-service` — watch the initContainer wait. Re-apply the statefulset afterwards.

## Step 6 — makeline-service (Go)

*Concept:* the **consumer**: reads orders from RabbitMQ and writes them to DocumentDB. Look at its `env:` (queue URI,
DB URI). Source: `src/makeline-service/` (`consumer.go`, `orderqueue.go`, `mongodb.go`).

```bash
kubectl apply -f manifests/04-makeline-service/deployment.yaml
kubectl apply -f manifests/04-makeline-service/service.yaml
```
*Verify:* `kubectl get pods` and
`kubectl run tmp --rm -it --restart=Never --image=busybox:1.37.0 -- wget -qO- http://makeline-service:3001/health`

## Step 7 — Azure OpenAI and ai-service (Python)

### 7.1 Create the Azure OpenAI resource and a model deployment

```bash
az cognitiveservices account create --name $AOAI_NAME --resource-group $RG --location $LOCATION \
  --kind OpenAI --sku S0 --custom-domain $AOAI_NAME

az cognitiveservices account deployment create --name $AOAI_NAME --resource-group $RG \
  --deployment-name gpt-4o-mini --model-name gpt-4o-mini --model-version "2024-07-18" \
  --model-format OpenAI --sku-name GlobalStandard --sku-capacity 10
```
Model, version and SKU vary by region. Check availability first, or create the deployment in the Foundry portal
and note its exact **deployment name**.

```bash
az cognitiveservices account deployment list --name $AOAI_NAME --resource-group $RG --query "[].name" -o tsv
az cognitiveservices account show --name $AOAI_NAME --resource-group $RG --query properties.endpoint -o tsv
az cognitiveservices account keys list --name $AOAI_NAME --resource-group $RG --query key1 -o tsv
```

### 7.2 Configure ai-service

*Concept:* config in a **ConfigMap**, the key in a **Secret**. The service uses the `AzureOpenAI` client, so:

- `AZURE_OPENAI_DEPLOYMENT_NAME` = the **deployment name** (not necessarily the model name)
- `AZURE_OPENAI_ENDPOINT` = the resource endpoint, e.g. `https://<name>.openai.azure.com/`
- `USE_AZURE_AD` = `"False"` for API-key auth (upstream's sample sets `"True"` for workload identity)

Edit the two placeholders in `manifests/05-ai-service/configmap.yaml`, then:

```bash
kubectl apply -f manifests/05-ai-service/configmap.yaml
kubectl create secret generic ai-service-secret --from-literal=OPENAI_API_KEY='<your-key>'
kubectl apply -f manifests/05-ai-service/deployment.yaml
kubectl apply -f manifests/05-ai-service/service.yaml
```
Never commit the key to Git.

*Verify:*
```bash
kubectl get pods                      # ai-service can take ~60 s (startup probe)
kubectl logs deploy/ai-service --tail=20
kubectl run tmp --rm -it --restart=Never --image=busybox:1.37.0 -- \
  wget -qO- --header='Content-Type: application/json' \
  --post-data='{"name":"Tea bags","tags":["tea"]}' http://ai-service:5001/generate/description
```
You should get JSON with a `description`. **404 "Resource not found"?** See Troubleshooting.

## Step 8 — product-service (Rust)

*Concept:* serves the product catalog and has `AI_SERVICE_URL=http://ai-service:5001/`, so AI description requests
reach ai-service through it. Source: `src/product-service/src/`.

```bash
kubectl apply -f manifests/06-product-service/
```
*Verify:* `kubectl run tmp --rm -it --restart=Never --image=busybox:1.37.0 -- wget -qO- http://product-service:3002/health`

## Step 9 — store-front (Vue.js)

*Concept:* **Service type LoadBalancer** creates an Azure public IP for the customer web app.

```bash
kubectl apply -f manifests/07-store-front/
kubectl get svc store-front -w        # wait for EXTERNAL-IP, then Ctrl+C
```
*Verify:* open `http://<EXTERNAL-IP>`; browse products, add to cart, place an order.

## Step 10 — store-admin (Vue.js)

```bash
kubectl apply -f manifests/08-store-admin/
kubectl get svc store-admin -w
```
*Verify:* open `http://<EXTERNAL-IP>` -> **Orders** shows the order you placed (it travelled store-front ->
order-service -> RabbitMQ -> makeline-service -> DocumentDB). **Products -> Add Product -> Ask AI Assistant**
generates a description via Azure OpenAI.

## Step 11 — Simulators

```bash
kubectl apply -f manifests/09-virtual-customer/
kubectl apply -f manifests/10-virtual-worker/
```
*Verify:* refresh **Orders** in store-admin: new orders appear and are completed over time. Check
`kubectl logs deploy/virtual-customer --tail=5` and the RabbitMQ UI queue counts.

## Step 12 — Explore (class discussion)

```bash
kubectl get all
kubectl scale deploy/store-front --replicas=3          # scaling
kubectl delete pod -l app=product-service              # self-healing
kubectl top pods                                       # resource use (metrics-server is built in to AKS)
kubectl exec -it deploy/order-service -- env | sort    # see injected config
```
Ask: what breaks if RabbitMQ is deleted? If DocumentDB restarts (no persistent storage)? Why is `ai-service` ClusterIP?

## Troubleshooting

| Symptom | Check |
|---|---|
| Pod `Pending` / `ImagePullBackOff` | `kubectl describe pod <name>`; node capacity, image tag, outbound access to `ghcr.io` |
| `CrashLoopBackOff` | `kubectl logs <pod> --previous` |
| `EXTERNAL-IP` `<pending>` | wait a few minutes; `kubectl describe svc store-front` |
| AI error **404 Resource not found** | Deployment name must match exactly (`az cognitiveservices account deployment list ...`). Use the `*.openai.azure.com` endpoint of the **same resource**, not a Foundry project endpoint. Test the call with curl: `curl -i "<ENDPOINT>openai/deployments/<DEPLOYMENT>/chat/completions?api-version=2024-12-01-preview" -H "api-key: <KEY>" -H "Content-Type: application/json" -d '{"messages":[{"role":"user","content":"hi"}]}'` |
| AI error 401 / 403 | wrong key, or `USE_AZURE_AD` is `"True"` while using a key |
| AI error 400 / unsupported parameter | model/API-version mismatch; try `AZURE_OPENAI_API_VERSION` or a `gpt-4o-mini` deployment |
| After editing the ConfigMap or Secret | `kubectl rollout restart deploy/ai-service` |

## Cleanup

```bash
az group delete --name $RG --yes --no-wait
# Azure OpenAI is soft-deleted; to reuse the name:
az cognitiveservices account purge --name $AOAI_NAME --resource-group $RG --location $LOCATION
```

## Where to read the code

| Folder | Language | Look at |
|---|---|---|
| `src/store-front`, `src/store-admin` | Vue.js | `src/`, `nginx.conf`, `Dockerfile` |
| `src/order-service` | Node (Fastify) | `app.js`, `routes/` |
| `src/makeline-service` | Go | `main.go`, `consumer.go`, `mongodb.go` |
| `src/product-service` | Rust | `src/config.rs` (`AI_SERVICE_URL`) |
| `src/ai-service` | Python (FastAPI) | `routers/description_generator.py` |
| `src/virtual-customer`, `src/virtual-worker` | Rust | `src/` |
