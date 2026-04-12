## Install Semantic Router

```
helm install semantic-router \
  oci://ghcr.io/vllm-project/charts/semantic-router \
  --version 0.2.0 \
  --namespace vllm-semantic-router-system \
  --create-namespace \
  --set persistence.storageClassName=local-path \
  --set config.classifier.pii_model.pii_mapping_path="models/mom-jailbreak-classifier/jailbreak_type_mapping.json" \
  -f semantic-router-values.yaml

This will take a few minutes — it downloads the ModernBERT classifier model on startup.  

There is a known bug in the Helm chart. The mom-pii-classifier model (which contains pii_type_mapping.json) is a separate HuggingFace model repo that the chart references but never downloads. The chart only downloads pii_classifier_modernbert-base_presidio_token_model (the weights), not mom-pii-classifier (the mapping file). That's why we did override pii_model.pii_mapping_path to a wrong file that exists to avoid the error.
```

# Verify 
```
kubectl --namespace vllm-semantic-router-system get pods
```

# Test the Semantic Router

```
1. Forward port:

  kubectl -n vllm-semantic-router-system port-forward svc/semantic-router 8081:8080

2. Test the Classification API:

  # Health check

  curl http://localhost:8081/health

  # Intent classification

  curl -X POST http://localhost:8081/api/v1/classify/intent \
    -H "Content-Type: application/json" \
    -d '{"text": "What is machine learning?"}'

  curl -X POST http://localhost:8081/api/v1/classify/intent \
    -H "Content-Type: application/json" \
    -d '{"text": "What is the derivative of x^3?"}'   

3. Access metrics:
  kubectl -n vllm-semantic-router-system port-forward svc/semantic-router-metrics 9190:9190
  curl http://localhost:9190/metrics
```

## Explain `semantic-router-values.yaml`

```
 1. Downloaded this file: https://raw.githubusercontent.com/vllm-project/semantic-router/refs/heads/main/deploy/kubernetes/ai-gateway/semantic-router-values/values.yaml 

 2. Changed: inference-gateway-istio.gateway-system.svc.cluster.local:80
```

## Create EnvoyFilter to plug in Semantic Router

```
kubectl apply -f 02-semantic-router-envoyfilter.yaml

# What this does: Inserts the Semantic Router as an ExtProc filter into the Envoy proxy that Istio created for your inference-gateway. Every request now flows through the Semantic Router's classification pipeline before being routed to the EPP and simulator pods.

# Verify the EnvoyFilter
kubectl get envoyfilter -n gateway-system

```

## Send inferencing request via gateway

```
kubectl get svc -n gateway-system

curl -v http://192.168.97.254/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-model-name: simulator" \
  -d '{"model":"base-model","messages":[{"role":"user","content":"What is the derivative of x^3?"}]}'

curl -v http://192.168.97.254/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-model-name: simulator" \
  -d '{"model":"auto","messages":[{"role":"user","content":"What is the derivative of x^3?"}]}'
 
```

## What is happening
```
Client sends "What is the derivative of x^3?"
  → Istio Gateway's Envoy receives request
    → ExtProc calls Semantic Router
      → Classifies as "math" domain ✓
      → Selects math-expert LoRA ✓  
      → Injects math system prompt ✓
      → Returns modified headers and body to Envoy
    → Envoy routes via HTTPRoute based on header match → InferencePool
      → EPP picks best model pod
        → Model pod returns the response
```
 ## Add External LLM as an option as well (HAEVN'T TRIED)

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


 