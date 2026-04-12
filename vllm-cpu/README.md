## Create namespace

```
kubectl apply -f 00-namespace.yaml
```

## Install fake vLLM

```
kubectl apply -f 01-vllm-cpu.yaml
```

## Quick test
```
kubectl get pods -n vllm-cpu -o wide

curl http://192.168.194.14:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"What is 2+2?"}]}'
```  

## Create inference pool and deploy EPP for cpu model

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
  --set inferenceExtension.image.tag=latest


  
```