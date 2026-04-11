## Install Gateway API CRDs
```
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml

kubectl get crd | grep gateway
```

## Install AgentGateway control plane
```
# Install AgentGateway CRDs:

helm upgrade -i agentgateway-crds \
  oci://cr.agentgateway.dev/charts/agentgateway-crds \
  --create-namespace \
  --namespace agentgateway-system \
  --version v1.1.0

# Install control plane

helm upgrade -i agentgateway \
  oci://cr.agentgateway.dev/charts/agentgateway \
  --namespace agentgateway-system \
  --version v1.1.0 \
  --set controller.image.pullPolicy=Always

# Check gateway class 
kubectl get gatewayclasses -o wide

```

## Create Gateway resource
```
kubectl apply -f 01-inference-gateway.yaml

kubectl get gateway -n agentgateway-system

# You should see inference-gateway with a PROGRAMMED status of True.
```