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