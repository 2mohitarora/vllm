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


kubectl edit configmap semantic-router-config -n vllm-semantic-router-system

Find this block

models:
      - backend_refs:
        - endpoint: vllm-llama3-8b-instruct.default.svc.cluster.local:8000
          name: local-vllm
          weight: 1
        name: base-model
        reasoning_family: qwen3

Change the endpoint to your AgentGateway service:

models:
      - backend_refs:
        - endpoint: inference-gateway.agentgateway-system.svc.cluster.local:80
          name: local-vllm
          weight: 1
        name: base-model
        reasoning_family: qwen3

Save and exit. Then restart the Semantic Router pod to pick up the new config:

kubectl rollout restart deployment/semantic-router -n vllm-semantic-router-system

Client :8080 → Semantic Router (classifies intent)
                    → AgentGateway (traffic mgmt)
                        → EPP (picks best pod)
                            → Simulator Pod (returns response)