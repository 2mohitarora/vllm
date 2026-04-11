![alt text](image-1.png)

## Training

The two core open-source projects that make this work

- Kubeflow Trainer is the training equivalent of what vLLM is for inference. It's a Kubernetes-native distributed AI platform for scalable LLM fine-tuning and training across frameworks including PyTorch, HuggingFace, DeepSpeed, Megatron, JAX, and more. It handles the hard distributed training orchestration — splitting a job across multiple GPUs/nodes, coordinating gradient synchronization, managing MPI communication — so your data scientists don't have to.
- Kueue is the multi-tenancy layer. It provides Kubernetes with additional job queueing capabilities necessary for efficiently scheduling batch AI/ML workloads. This is the piece that solves the "how do 5 teams share 20 GPUs fairly" problem.

## Here's what the platform team vs. tenant team split looks like in practice:
  
- Platform team
  - Install Kubeflow Trainer operator
  - Install + configure Kueue
  - Set up GPU node pools + drivers
  - Define quotas per team (ClusterQueue)
  - Configure fair-share + preemption policies
  - Manage storage (PVCs, NFS, model cache)
  - RBAC: namespace-per-team isolation
  - Monitoring dashboards (GPU util, queue)
  - Base container images with frameworks
  - Cost attribution + chargeback

- Tenant team
  - Write training script (PyTorch / HF)
  - Submit a TrainJob YAML (or Python SDK)
  - Pick number of GPUs + workers
  - Point to their dataset
  - Monitor their job's logs + metrics
  - Download checkpoints when done

## How Kueue solves multi-tenancy
![alt text](image-2.png)  

## The recommended open-source stack

| Layer | Tool | Role | Job submission |
| --- | --- | --- | --- |
| **Job submission** | Kubeflow Trainer | Distributed training operator (PyTorch, DeepSpeed, etc.) |  |
| **Queuing + quotas** | Kueue | Multi-tenant job queuing, fair sharing, preemption |  |
| **Orchestration** | JobSet / LeaderWorkerSet | Co-scheduling multi-pod training jobs |  |
| **Experiment tracking** | MLflow or Kubeflow Model Registry | Track runs, metrics, model versions |  |
| **Notebooks** | Kubeflow Notebooks | Jupyter/VS Code in K8s pods |  |
| **GPU management** | NVIDIA GPU Operator | Device plugin, drivers, MIG, DCGM monitoring |  |
| **Monitoring** | Prometheus + Grafana | GPU utilization, queue depth, job metrics |  |
| **Fault recovery** | AppWrapper (optional) | Auto-retry failed training jobs |  |
| **Inference (post-train)** | vLLM + llm-d | Serve the trained model |  |