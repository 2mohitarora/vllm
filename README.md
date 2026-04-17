# vllm

### Create kubernetes cluster

[./00-k8s/README.md](./00-k8s/README.md)

### Test DRA with example-driver that provides access to a set of mock GPU devices

Repo: https://github.com/kubernetes-sigs/dra-example-driver

[./01-dra/README.md](./01-dra/README.md)

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

---

## Training (parallel stack, same constructs — K8s + DRA)

See [./Training.md](./Training.md) for the architecture overview. The numbered directories below mirror the inferencing walkthrough and reuse `00-k8s/` and `01-dra/`.

### Install Kueue (multi-tenant queues + quotas)

[./07-kueue/README.md](./07-kueue/README.md)

### Install Kubeflow Trainer + ClusterTrainingRuntimes

[./08-kubeflow-trainer/README.md](./08-kubeflow-trainer/README.md)

### Shared training storage (HF cache + checkpoints)

[./09-training-storage/README.md](./09-training-storage/README.md)

### Submit a fake TrainJob (training equivalent of vllm-simulator)

[./10-trainjob-simulator/README.md](./10-trainjob-simulator/README.md)

### Submit a real HF LoRA fine-tune on CPU (hands off to 06-vllm-cpu)

[./11-trainjob-hf-cpu/README.md](./11-trainjob-hf-cpu/README.md)

### Install MLflow (experiment tracking + model registry)

[./12-mlflow/README.md](./12-mlflow/README.md)

### Install Volcano (gang scheduling + topology-aware GPU packing)

[./13-volcano/README.md](./13-volcano/README.md)

### Install Argo Workflows (pipeline DAG: prep → train → eval → register → promote)

[./14-argo-workflows/README.md](./14-argo-workflows/README.md)

### Eval gate (mandatory governance step before Production promotion)

[./15-eval-gate/README.md](./15-eval-gate/README.md)

### End-to-end training demo (scripted walk-through)

[./demo-training-e2e.md](./demo-training-e2e.md)