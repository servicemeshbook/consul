# Consul - Commands 

Contents

- [Installing Consul](#Installing-Linkerd)
- [Service Discovery](#Service-Discovery)
- [Traffic Management](#Traffic-Management)

Copy and paste command as you practice.

## Installing Consul

### Prerequisite

### Check if keepalived is running
```
kubectl -n keepalived get pods
```

### Clone scripts
```
cd ~/ # Switch to home directory
git clone https://github.com/servicemeshbook/consul.git
cd consul
git checkout 1.6.1
cd scripts
```

### Download v1.6.1. package for Linux AMD64
```
wget https://releases.hashicorp.com/consul/1.6.1/consul_1.6.1_linux_amd64.zip
```

### Extract consul from the zip archive
```
unzip consul_1.6.1_linux_amd64.zip
sudo mv consul /bin
```

### Check the Consul version
```
consul version
```

### Installing Consul in Kubernetes

### Create persistent volumes directory
```
sudo mkdir -p /var/lib/consul{0,1,2}
```

### Create a consul namespace
```
kubectl create ns consul
```

### Grant cluster_admin to the namespace consul.
```
kubectl create clusterrolebinding consul-role-binding --clusterrole=cluster-admin --group=system:serviceaccounts:consul
```

### Create a storage class and 3 persistent volumes.
```
kubectl -n consul apply -f 01-create-pv-consul.yaml
```

### Downloading Consul Helm Chart

### Find out available versions of the Consul helm for Kubernetes
```
curl -L -s https://api.github.com/repos/hashicorp/consul-helm/tags | grep "name"
```

### Download consul-helm
```
cd # switch to home dir
export CONSUL_HELM_VERSION=0.11.0
curl -LOs https://github.com/hashicorp/consul-helm/archive/v${CONSUL_HELM_VERSION}.tar.gz
tar xfz v${CONSUL_HELM_VERSION}.tar.gz
```

### Modify two parameters
```
sed -i 's/failureThreshold:.*/failureThreshold: 30/g' \
~/consul-helm-${CONSUL_HELM_VERSION}/templates/server-statefulset.yaml

sed -i 's/initialDelaySeconds:.*/initialDelaySeconds: 60/g' \
~/consul-helm-${CONSUL_HELM_VERSION}/templates/server-statefulset.yaml 
```

### Installing Consul
```
cd ~/consul/scripts # Switch to scripts for this exercise
```

### Create a new Consul Cluster in Kubernetes
```
cat 02-consul-values.yaml
helm install ~/consul-helm-${CONSUL_HELM_VERSION}/ --name consul \
--namespace consul --set fullnameOverride=consul -f ./02-consul-values.yaml
```

### Make sure that the persistent volume claims are created 
```
kubectl -n consul get pvc
```
### Check if Consul servers are in a Ready 1/1
```
kubectl -n consul get pods
```

### Deploying Consul Server and Client
```
kubectl -n consul get sts
```

### Check the version of the Consul running in Kubernetes
```
kubectl -n consul exec -it consul-server-0 -- consul version
```

### Find out which server is the leader
```
kubectl -n consul logs consul-server-0 | grep -i leader
```

### Consul clients are installed as a DaemonSet
```
kubectl -n consul get ds
```

### Connecting Consul DNS to Kubernetes

### Consul runs its own DNS for service discovery
```
kubectl -n consul get svc
```
### We need to connect consul-dns service to Kubernetes DNS
```
cat 03-create-coredns-configmap.sh 
chmod +x 03-create-coredns-configmap.sh
./03-create-coredns-configmap.sh 
```

### Check addition of consul DNS server
```
kubectl -n kube-system get cm coredns -o yaml
```

### Consul server in VM

### Find out the end-points for the consul-server
```
kubectl -n consul get ep
```


### Query the node names using REST API.
```
curl -s localhost:8500/v1/catalog/nodes | json_reformat
```

### Check members of the Consul cluster using one of the Kubernetes Consul pods
```
kubectl -n consul exec -it consul-server-0 -- consul members
```

### Check the same from the VM
```
consul members
```

### Use consul info command
```
kubectl -n consul exec -it consul-server-0 -- consul info
```

### End of Consul Install commands

### Asciinema Consul - Install 

[![asciicast](https://asciinema.org/a/275098.svg)](https://asciinema.org/a/275098)

## Service Discovery

### End of Consul Service Discovery commands

## Traffic Management

### End of Consul Traffic Management commands