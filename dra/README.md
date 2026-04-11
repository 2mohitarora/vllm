## Create DeviceClass

```
kubectl apply -f 00_deviceclass.yaml
```

## Create ResourceClaimTemplate

```
kubectl apply -f 01_resourceclaim_template.yaml

kubectl get resourceclaimtemplate -n default
```

## Create an example pod that references this claim to see DRA in action
```
kubectl apply -f 02_example_pod.yaml

kubectl get pod dra-test-pod

# It should show Pending — because there's no DRA driver providing ResourceSlices with devices matching gpu.walmart.com

kubectl describe pod dra-test-pod

# Look for the Events section
```
