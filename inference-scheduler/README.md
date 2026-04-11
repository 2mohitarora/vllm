## Creating the InferencePool + EPP that routes traffic to pods serving the model

This is the smart routing layer. It has two parts:

- InferencePool — a custom resource that tells the EPP "find all pods matching these labels, they're serving this model"
- EPP (Endpoint Picker Proxy) — the actual process that receives requests from the Gateway, picks the best pod, and forwards the request

```
# Install Inference Extension CRDs:

kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.4.0/manifests.yaml

# Install the InferencePool chart, configured to discover our simulator pods:

helm install sim-pool \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool \
  --namespace llm-d \
  --set inferencePool.modelServers.matchLabels.app=llm-d-sim \
  --set inferencePool.targetPortNumber=8000 \
  --dependency-update \
  --version v1.4.0

```

What this just created:

- An InferencePool custom resource that says "look for pods with label app=llm-d-sim on port 8000"
- An EPP Deployment (a Go binary) that watches those pods, collects their load/cache metrics, and makes routing decisions
- A Service for the EPP's gRPC endpoint that the Gateway will call for each incoming request  

## Verify

```
kubectl get inferencepool -n llm-d -o yaml

kubectl get svc -n llm-d
```

## Create HTTPRoute to point to InferencePool

```
kubectl apply -f sim-http-route.yaml
```

## Verify the HTTPRoute

```
kubectl get httproute -n llm-d -o yaml

curl http://192.168.97.254/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"simulator","messages":[{"role":"user","content":"Hello"}]}'

If you get a streamed response back, you've got the full flow working: Client → AgentGateway → EPP (smart routing) → Simulator Pod  
```

## Deploy InferenceObjective (Optional)


## Deploy the Body Based Router Extension (Optional)


## Latency-Based Routing

