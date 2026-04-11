## Install the Semantic Router Helm chart

```
helm install semantic-router \
  oci://ghcr.io/vllm-project/charts/semantic-router \
  --version 0.2.0 \
  --namespace vllm-semantic-router-system \
  --create-namespace \
  -f https://raw.githubusercontent.com/vllm-project/semantic-router/refs/heads/main/deploy/kubernetes/ai-gateway/semantic-router-values/values.yaml

This will take a few minutes — it downloads the ModernBERT classifier model on startup.  
```

## Update the Semantic Router's backend config to point to your inference pipeline
```
kubectl edit configmap semantic-router-config -n vllm-semantic-router-system

Find the backend_refs section and change the endpoint:

models:
      - backend_refs:
        - endpoint: inference-gateway-istio.gateway-system.svc.cluster.local:80
          name: local-vllm
          weight: 1
        name: base-model
        reasoning_family: qwen3

Save and exit. Then restart the Semantic Router pod to pick up the new config:

kubectl rollout restart deployment/semantic-router -n vllm-semantic-router-system
```
# Test the Semantic Router

```
1. Get the application URL by running these commands:
  kubectl --namespace vllm-semantic-router-system port-forward $POD_NAME 8080:$CONTAINER_PORT

2. Test the Classification API:
  # Health check
  curl http://localhost:8080/health

  # Intent classification
  curl -X POST http://localhost:8080/api/v1/classify/intent \
    -H "Content-Type: application/json" \
    -d '{"text": "What is machine learning?"}'

3. Access metrics:
  kubectl --namespace vllm-semantic-router-system port-forward svc/semantic-router-metrics 9190:9190
  curl http://localhost:9190/metrics

4. Access gRPC API:
  kubectl --namespace vllm-semantic-router-system port-forward svc/semantic-router 50051:50051

5.  Send the request and check the Semantic Router logs to see if it classified the request

curl http://192.168.97.254/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"base-model","messages":[{"role":"user","content":"What is the derivative of x^3?"}]}'

# Semantic Router logs — see the classification happening
kubectl logs -n vllm-semantic-router-system -l app.kubernetes.io/name=semantic-router --tail=30

# EPP logs — see the routing decision
kubectl logs -n $NAMESPACE -l app.kubernetes.io/name=sim-pool-epp --tail=20

```
Client → Semantic Router (classifies intent)
                    → AgentGateway (traffic mgmt)
                        → EPP (picks best pod)
                            → Simulator Pod (returns response)

Client sends "What is the derivative of x^3?"
  → Istio Envoy receives request
    → ExtProc calls Semantic Router
      → Classifies as "math" domain ✓
      → Selects math-expert LoRA ✓  
      → Injects math system prompt ✓
      → Returns modified headers to Envoy
    → Envoy routes via HTTPRoute → InferencePool
      → EPP picks best simulator pod
        → Simulator returns fake response (ignores everything above)


 ## Add OpenAI as an option as well

 ```
# Update the routing decisions to use OpenAI for complex queries. For example, find math_decision and change its modelRefs to point to OpenAI:

- description: Mathematics and quantitative reasoning
        modelRefs:
        - model: openai-gpt4
          lora_name: math-expert
          use_reasoning: true
        name: math_decision

kubectl set env deployment/semantic-router \
  -n vllm-semantic-router-system \
  OPENAI_API_KEY=sk-your-key-here

kubectl rollout restart deployment/semantic-router -n vllm-semantic-router-system  

# This should route to OpenAI (math domain)
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"base-model","messages":[{"role":"user","content":"What is the derivative of x^3?"}]}'

# This should route to local simulator (general domain)
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"base-model","messages":[{"role":"user","content":"Hello, how are you?"}]}'
 ```       


 ## Create EnvoyFilter to plug in Semantic Router (Semantic rounter will be created later)

```
kubectl apply -f 02-semantic-router-envoyfilter.yaml

# What this does: Inserts the Semantic Router as an ExtProc filter into the Envoy proxy that Istio created for your inference-gateway. Every request now flows through the Semantic Router's classification pipeline before being routed to the EPP and simulator pods.

# Verify the EnvoyFilter
kubectl get envoyfilter -n gateway-system

```
