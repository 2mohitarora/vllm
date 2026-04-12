# vllm

### Create kubernetes cluster

[./01-k8s/README.md](./01-k8s/README.md)

### Install Istio Gateway

[./02-gateway/README.md](./02-gateway/README.md)

### Install Inference Extension and llm-d InferenceScheduler (extension of the EPP)

[./03-gateway-api-inferencing-extension/README.md](./03-gateway-api-inferencing-extension/README.md)

### Install vLLM simulator

This simulator pretends to be a vLLM server serving a model called simulator. They expose /v1/chat/completions, /v1/models, /health, and /metrics exactly like real vLLM would. It is used to test the inference scheduler without needing a real GPU.

[./04-vllm-simulator/README.md](./04-vllm-simulator/README.md)

### Install Semantic Router

[./05-vllm-semantic-router/README.md](./05-vllm-semantic-router/README.md)

### Add another model to the mix running on vllm (CPU)

[./06-vllm-cpu/README.md](./06-vllm-cpu/README.md)

### Test DRA with example-driver that provides access to a set of mock GPU devices

Repo: https://github.com/kubernetes-sigs/dra-example-driver

[./07-dra/README.md](./07-dra/README.md)

- Couldn't test because image doesn't support ARM64 : `docker manifest inspect registry.k8s.io/dra-example-driver/dra-example-driver:v0.2.1`
- Could have tried on Orbstack but Orbstack k8s version < 1.34