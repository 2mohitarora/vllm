## Install Tools
```
brew install kubectl helm docker cilium-cli k9s
```

## Install Orbstack
```
brew install --cask orbstack

# Add local registries that will be created later
# Add registry to docker daemon in ~/.docker/daemon.json
{
  "insecure-registries": ["localhost:5050"]
}

# Start Orbstack
```

## Configure docker
```
docker context use orbstack
export DOCKER_HOST="unix:///Users/mua0008/.orbstack/run/docker.sock"
docker context list
```

## Create your first vcluster
```
sudo vcluster create cluster-1 --driver docker --values cluster-1.yaml

helm repo add cilium https://helm.cilium.io/
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cilium cilium/cilium --version 1.19.1 --set kubeProxyReplacement=true --namespace cilium --create-namespace --set ipam.operator.clusterPoolIPv4PodCIDRList=10.1.0.0/16

# After CNI is installed, wait for pods to become Ready:
kubectl get pods --all-namespaces -w

# Check cilium status
cilium status --namespace cilium

# Note: Make sure to configure the CNI plugin according to your cluster's pod CIDR
kubectl get configmap cilium-config -n cilium -o yaml | grep -i cidr
```

## Install cert-manager, various components need it
```
# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# After cert-manager, wait for pods to become Ready:
kubectl get pods --all-namespaces -w -o wide
```