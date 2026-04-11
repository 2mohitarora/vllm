# Full LLM inference stack on local Kubernetes

**Stack:** vLLM Semantic Router → Gateway (with Inference Extension EPP) → llm-d ModelService → vLLM (CPU) + DRA

Single Envoy-based gateway stack.
Gateway is both the traffic proxy AND llm-d's gateway provider.
The Inference Gateway Extension (EPP) plugs into the same Envoy proxy for KV-cache-aware routing to vLLM pods.

---

## Architecture

![alt text](image-3.png)

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
│  │ vLLM (CPU)   │ │
│  │ Qwen 0.5B    │ │
│  └──────────────┘ │
│  DRA: true        │
│  (simulated GPUs) │
└──────────────────┘
```

# Create namespace
  kubectl create namespace "${NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -

# ── Install Gateway API CRDs ──
  kubectl apply --server-side --force-conflicts \
    -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml  

# ── Install AgentGateway ──
helm upgrade -i agentgateway-crds \
    oci://cr.agentgateway.dev/charts/agentgateway-crds \
    --create-namespace \
    --namespace agentgateway-system \
    --version "${AGW_VERSION}"

# Control plane
  helm upgrade -i agentgateway \
    oci://cr.agentgateway.dev/charts/agentgateway \
    --namespace agentgateway-system \
    --version "${AGW_VERSION}" \
    --set controller.image.pullPolicy=Always
 
# Wait for controller
kubectl wait --for=condition=Available deployment/agentgateway \
    -n agentgateway-system --timeout=120s

# ── Create Gateway resource ──
  kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: inference-gateway
  namespace: ${NAMESPACE}
spec:
  gatewayClassName: agentgateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
EOF        


# Deploy llm-d simulator with AgentGateway