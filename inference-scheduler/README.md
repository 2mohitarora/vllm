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
  --set experimentalHttpRoute.enabled=true \
  --version v1.4.0 \
  --set nodeSelector."kubernetes\.io/arch"=amd64

```

What this just created:

An InferencePool custom resource that says "look for pods with label app=llm-d-sim on port 8000"
An EPP Deployment (a Go binary) that watches those pods, collects their load/cache metrics, and makes routing decisions
A Service for the EPP's gRPC endpoint that the Gateway will call for each incoming request  

## Verify

```
kubectl get inferencepool -n llm-d -o yaml

kubectl get httproute llm-d-sim -n llm-d -o yaml
```

## Deploy InferenceObjective (Optional)


## Deploy the Body Based Router Extension (Optional)


## Latency-Based Routing

