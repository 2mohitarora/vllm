## Introduction

Gateway API Inference Extension (EPP) is an upstream Kubernetes SIG project `(kubernetes-sigs/gateway-api-inference-extension)`. It's part of the Gateway API ecosystem. It provides the generic InferencePool CRD and the reference EPP implementation that works with any Envoy-based gateway (Istio, Envoy Gateway, GKE Gateway, NGINX).

Gateway API Inference Extension optimizes self-hosting Generative Models on Kubernetes. More details can be found here: https://gateway-api-inference-extension.sigs.k8s.io/ 

The overall resource model focuses on 2 new inference-focused personas and corresponding resources that they are expected to manage:

![alt text](../assets/inference_extension.png)

*llm-d* builds on top of the Inference Extension. It has its own fork/extension of the EPP `(llm-d/llm-d-inference-scheduler)` that adds llm-d-specific plugins: KV-cache aware routing, disaggregated prefill/decode scheduling, prefix-cache scoring, LoRA-aware routing. These are plugins that plug into the EPP's extensible architecture.

## Important thing to note

Inference Scheduler is *not* a separate pod. It runs inside the EPP pod. They're the same thing, just different names for different layers of the same process.

Think of it this way:
```
EPP pod (one process, one container)
├── ExtProc server (gRPC layer — talks to Envoy)
├── Inference Scheduler (decision logic)
│   ├── Queue scorer plugin
│   ├── KV-cache utilization scorer plugin
│   ├── Prefix-cache scorer plugin
│   └── (llm-d adds: P/D disaggregation, LoRA-aware routing)
└── Data layer (watches pods, scrapes metrics)
```
The EPP is the container/binary — it receives gRPC ExtProc calls from Envoy, processes them, and returns routing decisions. The Inference Scheduler is the decision engine inside the EPP that scores available pods and picks the best one.

## Creating the InferencePool + EPP that routes traffic to pods serving the model

```
# Install Inference Extension CRDs:

kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.4.0/manifests.yaml

```

## Deploy InferenceObjective (Optional)


## Deploy the Body Based Router Extension (Optional)


## Latency-Based Routing


## Explore more features of llm-d InferenceScheduler

