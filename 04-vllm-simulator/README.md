## Create namespace

```
kubectl apply -f 00-namespace.yaml
```

## Create resource claim template

```
kubectl apply -f 01-resource-claim-template.yaml
```

## Install fake vLLM

```
kubectl apply -f 02-fake-vllm.yaml
```

## Quick test (direct pod)
```
kubectl get pods -n vllm-simulator -o wide

kubectl -n vllm-simulator port-forward deployment/vllm-simulator 8000:8000

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-model-name: simulator" \
  -d '{"model":"base-model","messages":[{"role":"user","content":"Hello"}]}'
```  

# Create Inference Pool and deploy EPP for Simulator mode

```
helm install sim-pool \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool \
  --namespace vllm-simulator \
  --set inferencePool.modelServers.matchLabels.app=vllm-simulator \
  --set inferencePool.targetPortNumber=8000 \
  --dependency-update \
  --set provider.name=istio \
  --version v1.4.0 \
  --set inferenceExtension.image.hub=ghcr.io/llm-d \
  --set inferenceExtension.image.name=llm-d-inference-scheduler \
  --set inferenceExtension.image.tag=latest \
  --set inferenceExtension.resources.requests.cpu=50m \
  --set inferenceExtension.resources.requests.memory=128Mi \
  --set inferenceExtension.resources.limits.memory=256Mi
```

What this just created:

- An InferencePool custom resource that says "look for pods with label app=vllm-simulator on port 8000"
- An EPP Deployment (a Go binary) that watches those pods, collects their load/cache metrics, and makes routing decisions
- A Service for the EPP's gRPC endpoint that the Gateway will call for each incoming request 
 
## Verify

```
kubectl get inferencepool -n vllm-simulator -o yaml

kubectl get svc -n vllm-simulator
```

## Create HTTPRoute pointing to Simulator InferencePool

```
kubectl apply -f sim-http-route.yaml
```

## Verify the HTTPRoute

```
kubectl get httproute -n vllm-simulator -o yaml

# Get Gateway IP
kubectl get svc -n gateway-system

curl -v http://192.168.97.254/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-model-name: simulator" \
  -d '{"model":"base-model","messages":[{"role":"user","content":"Hello"}]}'

If you get a streamed response back, you've got the full flow working: Client → Istio Gateway → EPP (smart routing) → Simulator Pod  

Few things to note:

1. The model field in the request body is standard OpenAI API format. Every component in the chain understands it.
2. When your request hits Istio → EPP, the EPP reads the request body, extracts the model field, and uses it to route. The EPP knows about the simulator pods because the InferencePool's selector matches them (app: vllm-simulator).
3. EPP also knows what model name the simulator is serving. For example, in our case, the simulator pods are serving the model named "base-model". The simulator was started with --model base-model, so it reports itself as serving a model called base-model. The EPP discovers the model name by scraping each pod's /v1/models endpoint
4. When you send "model": "base-model": EPP sees the model name matches what the simulator pods advertise → routes to a simulator pod → 200 OK
5. When you send "model": "non-existent-model": EPP sees no pod advertises "non-existent-model" → returns 404 Not Found
