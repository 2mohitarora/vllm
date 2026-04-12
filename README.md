# vllm

### Create kubernetes cluster

[./k8s/README.md](./k8s/README.md)

### Install Istio Gateway with Inference Extension

[./gateway/README.md](./gateway/README.md)

### Install vLLM simulator published by llm-d

This simulator pretends to be a vLLM server serving a model called simulator. They expose /v1/chat/completions, /v1/models, /health, and /metrics exactly like real vLLM would. It is used to test the inference scheduler without needing a real GPU.

[./llm-d/README.md](./llm-d/README.md)

### Install InferenceScheduler

[./inference-scheduler/README.md](./inference-scheduler/README.md)

### Install Semantic Router

[./vllm-semantic-router/README.md](./vllm-semantic-router/README.md)

### Install DRA (TRY THIS WHEN Orbstack supports k8s 1.34)

```
kubectl version -o json | grep -E "gitVersion|platform"
```

[./dra/README.md](./dra/README.md)