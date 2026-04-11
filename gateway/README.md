## Install Gateway API CRDs
```
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml

kubectl get crd | grep gateway
```

## Install istio
```
istioctl install --set profile=minimal --set values.pilot.env.ENABLE_GATEWAY_API_INFERENCE_EXTENSION=true

# Check pods
kubectl get pods -n istio-system

# Check gateway class 
kubectl get gatewayclasses -o wide

```

## Create Gateway resource
```
kubectl create namespace gateway-system

kubectl apply -f 01-inference-gateway.yaml

kubectl get gateway -n gateway-system

# You should see inference-gateway with a PROGRAMMED status of True.
```

## Create EnvoyFilter to plug in Semantic Router (Semantic rounter will be created later)

```
kubectl apply -f 02-semantic-router-envoyfilter.yaml

# What this does: Inserts the Semantic Router as an ExtProc filter into the Envoy proxy that Istio created for your inference-gateway. Every request now flows through the Semantic Router's classification pipeline before being routed to the EPP and simulator pods.

# Verify the EnvoyFilter
kubectl get envoyfilter -n gateway-system

```
