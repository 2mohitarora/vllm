## Install Tools
```
brew install kubectl helm docker k9s
```

## Install Orbstack
```
brew install --cask orbstack

# Start Orbstack
# Enable k8s on Orbstack
```

## Configure docker
```
docker context use orbstack
export DOCKER_HOST="unix:///Users/mua0008/.orbstack/run/docker.sock"
docker context list
```

## Configure kubectl
```
kubectl config use-context orbstack
```

## Install cert-manager, various components need it
```
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# After cert-manager, wait for pods to become Ready:
kubectl get pods --all-namespaces -w -o wide
```