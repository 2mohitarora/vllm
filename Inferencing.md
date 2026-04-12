# Full LLM inference stack on local Kubernetes

**Stack: vLLM Semantic Router → Istio Gateway (with Inference Extension EPP) → vLLM pods**

---

## Architecture

```
Client request (OpenAI-compatible)
    │
    ▼
┌────────────────────────────────────────────────────┐
│  vLLM Semantic Router (ExtProc on Envoy)           │
│  "Which MODEL should answer this?"                 │
│  • Classifies request intent (math, code, general) │
│  • Picks self-hosted model or cloud provider       │
│  • Rewrites model name + injects system prompt     │
│  • Semantic caching, jailbreak detection, PII      │
└──────────┬─────────────────────┬───────────────────┘
           │ self-hosted          │ cloud
           ▼                      ▼
┌──────────────────────────┐  ┌──────────────────────┐
│  Istio Gateway (Envoy)   │  │  OpenAI / Anthropic  │
│  • TLS, auth, HTTPRoute  │  │  (external APIs)     │
│  • Routes by model name  │  │  Semantic Router     │
│  • Standard Gateway API  │  │  forwards directly   │
└──────────┬───────────────┘  └──────────────────────┘
           │
           ▼
┌────────────────────────────────────────────────────┐
│  Inference Gateway Extension / EPP (ExtProc)       │
│  "Which POD of this model?"                        │
│  • KV-cache-aware endpoint selection               │
│  • Load-aware, prefix-cache-aware routing          │
│  • Plugs into same Envoy via ext_proc              │
│  • One EPP per InferencePool (per model)           │
│                                                    │
│  llm-d Inference Scheduler (plugins inside EPP):   │
│  • Disaggregated prefill/decode routing            │
│  • LoRA-aware routing                              │
│  • KV-cache transfer orchestration (NIXL)          │
└──────────┬─────────────────────────────────────────┘
           │
           ▼
┌────────────────────────────────────────────────────┐
│  vLLM Model Server pods (via llm-d ModelService)   │
│  • Actual inference (GPU or CPU)                   │
│  • OpenAI-compatible API                           │
│  • Reports metrics to EPP (/metrics)               │
│  • DRA allocates GPU devices (on real clusters)    │
│                                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ vLLM     │ │ vLLM     │ │ vLLM     │           │
│  │ Qwen3    │ │ Qwen3    │ │ Qwen3    │           │
│  │ replica 1│ │ replica 2│ │ replica 3│           │
│  └──────────┘ └──────────┘ └──────────┘           │
└────────────────────────────────────────────────────┘
```

## What each component does (and does NOT do)

| Component | What it does | What it does NOT do |
|---|---|---|
| **Semantic Router** | Classifies intent, picks model/provider, rewrites request, caching, safety | Does not route to specific pods |
| **Istio Gateway** | TLS termination, auth, HTTPRoute matching, traffic management | Does not understand models or inference |
| **EPP / Inference Scheduler** | Picks the optimal pod within an InferencePool based on metrics | Does not pick which model to use |
| **llm-d** | Orchestration framework — wires Gateway + EPP + vLLM via Helm charts. Adds advanced scheduling plugins (P/D disagg, KV-cache, LoRA-aware) | Does not route to external providers (OpenAI, etc.) |
| **vLLM** | Runs the actual model, generates tokens, serves OpenAI API | Does not route, schedule, or classify |
| **DRA** | Allocates GPU devices to pods on real clusters | Does not exist on Mac (no GPU driver) |

## What runs where on your Mac

| Component | Runs where | Purpose |
|---|---|---|
| **Istio (minimal)** | Inside K8s | Gateway + Envoy proxy |
| **Semantic Router** | Inside K8s | Intent classification via ExtProc |
| **EPP (per pool)** | Inside K8s | Smart pod-level routing |
| **Inference Simulator** | Inside K8s | Fake vLLM — tests orchestration without GPUs |
| **vLLM CPU** | Inside K8s | Real inference on CPU (slow but real) |
| **DRA manifests** | Files only | Ready for K8s 1.34+ GPU clusters |
| **vllm-metal** | Native macOS | Real inference on Apple Metal GPU (optional) |

## Key relationships

- **One Gateway** serves all models (single entry point)
- **One InferencePool + one EPP** per model (each model gets its own pool)
- **HTTPRoutes** connect the Gateway to InferencePools (by model name or header)
- **Semantic Router** sits in front via ExtProc — classifies and routes before the Gateway makes its HTTPRoute decision
- **llm-d** is not a single binary — it's the orchestration framework (Helm charts + configs) that wires all of the above together