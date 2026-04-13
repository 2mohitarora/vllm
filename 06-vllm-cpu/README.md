## Create namespace

```
kubectl apply -f 00-namespace.yaml
```

## Install vLLM serving Qwen/Qwen3-0.6B on cpu

```
kubectl apply -f 01-vllm-cpu.yaml
```

## Quick test (direct pod once its up and running)
```
kubectl get pods -n vllm-cpu -o wide

kubectl -n vllm-cpu port-forward deployment/vllm-qwen3-cpu 8000:8000

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"What is 2+2?"}]}'
```  

## Create inference pool and deploy EPP for Qwen/Qwen3-0.6B on cpu model

```
helm install qwen3-cpu-pool \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool \
  --namespace vllm-cpu \
  --set inferencePool.modelServers.matchLabels.app=vllm-qwen3-cpu \
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

# Verify

```
kubectl get inferencepool -n vllm-cpu -o yaml

kubectl get svc -n vllm-cpu

```

## Create HTTPRoute pointing to CPU InferencePool

```
kubectl apply -f qwen3-cpu-http-route.yaml
```

## Verify the HTTPRoute

```
kubectl get httproute -n vllm-cpu -o yaml

# Get Gateway IP
kubectl get svc -n gateway-system

curl -v http://192.168.97.254/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "X-SELECTED-MODEL: Qwen/Qwen3-0.6B" \
  -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"What is 2+2?"}]}'
```