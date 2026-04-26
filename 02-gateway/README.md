## Install Gateway API CRDs
```
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml

kubectl get crd | grep gateway
```

## Install istio
```
istioctl install --set profile=minimal --set values.pilot.env.ENABLE_GATEWAY_API_INFERENCE_EXTENSION=true -y

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