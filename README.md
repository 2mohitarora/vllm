# vllm

### Create kubernetes cluster

[./01-k8s/README.md](./01-k8s/README.md)

### Install Istio Gateway

[./02-gateway/README.md](./02-gateway/README.md)

### Install Inference Extension and llm-d InferenceScheduler (extension of the EPP)

[./03-gateway-api-inference-extension/README.md](./03-gateway-api-inference-extension/README.md)

### Install vLLM simulator

This simulator pretends to be a vLLM server serving a model called simulator. They expose /v1/chat/completions, /v1/models, /health, and /metrics exactly like real vLLM would. It is used to test the inference scheduler without needing a real GPU.

[./04-vllm-simulator/README.md](./04-vllm-simulator/README.md)

### Install Semantic Router

[./05-vllm-semantic-router/README.md](./05-vllm-semantic-router/README.md)

### Add another model to the mix running on vllm (CPU)

[./06-vllm-cpu/README.md](./06-vllm-cpu/README.md)

### Install DRA (Make sure k8s 1.34+)

```
kubectl version -o json | grep -E "gitVersion|platform"
```

[./07-dra/README.md](./07-dra/README.md)