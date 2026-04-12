## Create namespace

```
kubectl apply -f 00-namespace.yaml
```

## Install fake vLLM

```
kubectl apply -f 01-vllm-cpu.yaml
```

# Quick test
```
kubectl get pods -n vllm-cpu -o wide

curl http://192.168.194.14:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"What is 2+2?"}]}'
```  
