## Install Tools
```
brew install kubectl helm docker k9s istioctl vcluster
```

## Install Orbstack
```
brew install --cask orbstack
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
sudo vcluster create cluster-1 --driver docker --values ./vcluster/cluster.yaml
```