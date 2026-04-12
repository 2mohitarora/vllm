# Full LLM inference stack on local Kubernetes

**Stack: Istio Gateway (Envoy) with Semantic Router + EPP as ExtProc filters → vLLM pods (launched via llm-d ModelService)**

---

## Architecture

```
Client request (OpenAI-compatible)
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│  Istio Gateway (Envoy proxy)                             │
│  THE single entry point for all traffic                  │
│  • TLS termination, auth, rate limiting                  │
│  • HTTPRoute matching (routes by model name / headers)   │
│  • Runs ExtProc filter chain on every request:           │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ ExtProc Filter 1: vLLM Semantic Router             │  │
│  │ "Which MODEL should answer this?"                  │  │
│  │  • Classifies request intent (math, code, general) │  │
│  │  • Rewrites model name + injects system prompt     │  │
│  │  • Semantic caching, jailbreak detection, PII      │  │
│  │  • Routes cloud queries to OpenAI/Anthropic        │  │
│  ├────────────────────────────────────────────────────┤  │
│  │ ExtProc Filter 2: Inference Scheduler (EPP)        │  │
│  │ "Which POD of this model?"                         │  │
│  │  • KV-cache-aware endpoint selection               │  │
│  │  • Load-aware, prefix-cache-aware routing          │  │
│  │  • One EPP per InferencePool (per model)           │  │
│  │  • llm-d plugins: P/D disagg, LoRA-aware routing   │  │
│  └────────────────────────────────────────────────────┘  │
└──────────┬──────────────────────────────┬────────────────┘
           │ self-hosted                   │ cloud
           ▼                               ▼
┌────────────────────────────┐  ┌──────────────────────────┐
│  vLLM Model Server pods    │  │  OpenAI / Anthropic      │
│  (via llm-d ModelService*) │  │  (external APIs)         │
│  • Actual inference        │  │  Semantic Router         │
│  • OpenAI-compatible API   │  │  forwards directly via   │
│  • Reports metrics to EPP  │  │  backend_refs config     │
│  • DRA allocates GPUs      │  └──────────────────────────┘
│                            │
│  ┌────────┐ ┌────────┐    │
│  │ vLLM   │ │ vLLM   │    │
│  │ pod 1  │ │ pod 2  │    │
│  └────────┘ └────────┘    │
└────────────────────────────┘

* ModelService is just a Helm chart — not a runtime component. It's llm-d's opinionated way to deploy vLLM pods with the right configuration.
```

## What each component does (and does NOT do)

| Component | What it does | What it does NOT do |
|---|---|---|
| **Istio Gateway (Envoy)** | Entry point. TLS, auth, HTTPRoutes. Hosts both ExtProc filters. | Does not understand models or make inference decisions |
| **Semantic Router** | Classifies intent, picks model/provider, rewrites request, caching, safety. Runs as ExtProc filter inside Gateway's Envoy. | Does not route to specific pods. Does not run standalone — needs Envoy. |
| **EPP / Inference Scheduler** | Picks the optimal pod within an InferencePool based on metrics. Runs as ExtProc filter inside Gateway's Envoy. | Does not pick which model to use. Does not route to external providers. |
| **llm-d** | Helm charts + configs that wire Gateway + EPP + vLLM together. Adds advanced scheduling plugins (P/D disagg, KV-cache, LoRA-aware). | Not a single binary. Does not route to external providers. |
| **vLLM** | Runs the actual model, generates tokens, serves OpenAI API, reports metrics. | Does not route, schedule, or classify. |
| **DRA** | Allocates GPU devices to pods using CEL expressions (K8s 1.34+). | Not available on Mac (no GPU driver). |

## Request flow in detail

1. **Client** sends `POST /v1/chat/completions` with `"model": "auto"` to the Gateway
2. **Gateway (Envoy)** receives the request and runs the ExtProc filter chain:
   - **Filter 1 (Semantic Router)**: reads the body, classifies "this is a math query", rewrites `"model": "auto"` → `"model": "Qwen/Qwen3-0.6B"`, injects math system prompt, returns modified request to Envoy
   - **Filter 2 (EPP)**: reads the model name, looks up the InferencePool for that model, scores available pods by KV-cache utilization + queue depth + prefix-cache match, picks the best pod, sets `x-gateway-destination-endpoint` header
3. **Gateway (Envoy)** routes the request to the selected vLLM pod via the HTTPRoute → InferencePool
4. **vLLM pod** runs inference on the GPU/CPU, streams tokens back through the Gateway to the client

For cloud queries: the Semantic Router classifies and forwards directly to the external API (e.g., `api.openai.com`) via its `backend_refs` config. The EPP is not involved — there are no pods to route between.

## Multiple models setup

Each model gets its own InferencePool + EPP. The Gateway routes between them:

```
Gateway (Envoy)
    │
    ├─ HTTPRoute: model=math    → InferencePool: math-pool    → EPP → vLLM pods (Qwen3-32B)
    ├─ HTTPRoute: model=code    → InferencePool: code-pool    → EPP → vLLM pods (DeepSeek)
    └─ HTTPRoute: model=general → InferencePool: general-pool → EPP → vLLM pods (Llama-8B)
```

| Resource | How many | Why |
|---|---|---|
| Gateway | 1 | Single entry point |
| Semantic Router | 1 | Classifies all requests |
| HTTPRoute | 1 per model (or 1 with multiple rules) | Maps model names to pools |
| InferencePool | 1 per model | Defines which pods serve that model |
| EPP | 1 per pool | Smart routing within that model's replicas |
| vLLM pods | N per model (scale independently) | Actual inference |

## What runs where on your Mac

| Component | Runs where | Purpose |
|---|---|---|
| **Istio (minimal profile)** | Inside K8s | istiod + per-Gateway Envoy proxy |
| **Semantic Router** | Inside K8s | Intent classification via ExtProc |
| **EPP (per pool)** | Inside K8s | Smart pod-level routing via ExtProc |
| **Inference Simulator** | Inside K8s | Fake vLLM — tests orchestration without GPUs |
| **vLLM CPU** | Inside K8s | Real inference on CPU (slow but real) |
| **DRA manifests** | Files only | Ready for K8s 1.34+ GPU clusters |
| **vllm-metal** | Native macOS | Real inference on Apple Metal GPU (optional) |

## Key things to remember

- **Gateway is always the front door** — nothing bypasses it. Both Semantic Router and EPP are ExtProc filters running inside the Gateway's Envoy.
- **Semantic Router and EPP are separate pods** — but they're called by the same Envoy process as part of its request processing pipeline.
- **llm-d is not a runtime component** — it's the orchestration framework (Helm charts, scheduling plugins, configs) that wires everything together.
- **Cloud routing bypasses EPP** — the Semantic Router forwards to external APIs directly. EPP only handles self-hosted pods.
- **One InferencePool + one EPP per model** — each model's replicas are managed independently.