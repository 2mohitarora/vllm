## Install Tools
```
brew install kubectl helm docker k9s istioctl
```

## Install Orbstack
```
brew install --cask orbstack

orb config set rosetta false
orb config set k8s.enable true
orbctl stop

# Start Orbstack and Mark sure rosetta config is disabled and k8s is enabled

```

## Configure docker
```
docker context use orbstack
export DOCKER_HOST="unix:///Users/mua0008/.orbstack/run/docker.sock"
docker context list
```

## Configure Kubeconfig
```
kubectl config use-context orbstack
```