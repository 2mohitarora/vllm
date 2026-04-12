# Full LLM inference stack on local Kubernetes

**Stack: vLLM Semantic Router → Gateway (Istio Gateway using Gateway API implementation of Kubernetes with Inference Extension EPP) → llm-d ModelService → vLLM + DRA**

- Single Envoy-based gateway stack.
- Gateway is both the traffic proxy AND llm-d's gateway provider.
- The Inference Gateway Extension (EPP) plugs into the same Envoy proxy for KV-cache-aware routing to vLLM pods.

---

## Architecture

![alt text](assets/inferencing.png)

```
Client request (OpenAI-compatible)
    │
    ▼
┌────────────────────────────────────────────────────┐
│  vLLM Semantic Router                              │
│  Classifies request → picks self-hosted or cloud   │
│  (CPU only, runs embedding classifier)             │
└──────────┬─────────────────────────────────────────┘
           │ routing decision (backend URL)
           ▼
┌────────────────────────────────────────────────────┐
│  llm-d (the full orchestration stack)              │
│  ┌──────────────────────────────────────────────┐  │
│  │ LLM features:                                │  │
│  │  • Token rate limiting                       │  │
│  │  • API format translation (OpenAI↔Anthropic) │  │
│  │  • Provider credential injection             │  │
│  │  • Provider failover                         │  │
│  └──────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │ Gateway API (Istio / AgentGateway):          │  │
│  │  • TLS, load balancing, retries              │  │
│  │  • Gateway API implementation                │  │
│  └──────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │ Inference Gateway Extension (EPP):           │  │
│  │  • KV-cache-aware endpoint selection         │  │
│  │  • Load-aware, criticality-aware routing     │  │
│  │  • Plugs into same Envoy via ext_proc        │  │
│  └──────────────────────────────────────────────┘  │
└──────┬──────────────────────────────┬──────────────┘
       │ self-hosted                   │ cloud
       ▼                               ▼
┌──────────────────┐          ┌──────────────────┐
│  llm-d            │          │  OpenAI API      │
│  ModelService     │          │  (external)      │
│  ┌──────────────┐ │          └──────────────────┘
│  │ vLLM         │ │
│  │ Qwen 0.5B    │ │
│  └──────────────┘ │
└──────────────────┘
```

## What's included

| Component | Runs where | Purpose |
|---|---|---|
| **Gateway** | Inside K8s | Gateway API implementation of Istio |
| **llm-d EPP** | Inside K8s | Inference scheduler — smart pod-level routing |
| **Inference Simulator** | Inside K8s | Fake vLLM — tests orchestration without GPUs |
| **DRA manifests** | Inside K8s | DeviceClass + ResourceClaimTemplate — ready for real GPUs |
| **Semantic Router** | Inside K8s | Intent classifier — routes to best-fit model |
| **vllm-metal** | Native macOS | Real inference on Apple Metal GPU |
