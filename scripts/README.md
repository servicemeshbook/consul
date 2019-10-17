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

### Asciinema recording - Consul - Install 

[![asciicast](https://asciinema.org/a/275098.svg)](https://asciinema.org/a/275098)

## Service Discovery

### Installing Demo Application

### Let's create backend counting and frontend dashboard services

```
cat 04-counting-demo.yaml
kubectl -n consul apply -f 04-counting-demo.yaml
```

### Check counting and dashboard pods
```
kubectl -n consul get pods
```

### Describe one of the microservice and check the sidecar proxy injection
```
kubectl -n consul describe pod counting
```

### Defining Ingress for Consul dashboard
```
export INGRESS_HOST=$(kubectl -n kube-system get service nginx-controller \
-o jsonpath='{.status.loadBalancer.ingress..ip}') ; echo $INGRESS_HOST

sudo sed -i '/webconsole.consul.local/d' /etc/hosts
echo "$INGRESS_HOST webconsole.consul.local" | sudo tee -a /etc/hosts

cat 05-create-ingress.yaml
kubectl apply -f 05-create-ingress.yaml
```

### Service Discovery

### Find out the node port for the demo application's dashboard-service
```
kubectl -n consul get svc dashboard-service
```

### Note the node port number from the above command and open URL http://192.168.142.101:<port> in browser


### Open URL http://webconsole.consul.local from another tab

### Service discovery - command line

```
consul config list -kind service-defaults
```

### Mutual TLS

### Check the log of the sidecar proxy for TLS
```
kubectl -n consul logs counting -c consul-connect-envoy-sidecar | grep tls
```

### The leaf certificates time to live is 72 hours. Verify this by
```
curl -s http://consul-server.consul.svc.cluster.local:8500/v1/connect/ca/configuration | json_reformat
```

### The root certificates can be viewed as through the following REST API call
```
curl -s http://consul-server.consul.svc.cluster.local:8500/v1/connect/ca/roots | json_reformat
```

### Key-Value store

```
consul kv put redis/config/minconns 1
consul kv put redis/config/maxconns 25
consul kv put redis/config/users/admin password
```

### Extract key from the store along with other metadata. 
```
consul kv get --detailed redis/config/minconns
```

### Get all values from the key-value store recursively
```
consul kv get -recurse
```

### The keys can also be obtained through REST API
```
curl -s http://localhost:8500/v1/kv/redis/config/minconns | json_reformat 
```

### Monitoring and Metrics
```
consul monitor
```

### watch for changes in a given data view from Consul
```
consul watch -type=service -service=consul
```

### Metrics Collection
```
curl -s http://localhost:8500/v1/agent/metrics | json_reformat
```

### Registering External Service
```
kubectl -n consul -c counting cp counting:counting-service ~/counting-service
chmod +x ~/counting-service
sudo cp ~/counting-service /bin
```

### Define a systemd service in the local VM to run the counting service
```
cat 06-create-systemd-service.sh
chmod +x 06-create-systemd-service.sh
sudo ./06-create-systemd-service.sh
```

### Enable and start external-counting service
```
sudo systemctl enable external-counting
sudo systemctl start external-counting
sudo systemctl status external-counting
```

### Test external counting service
```
curl http://localhost:10001/health
```

### we will register this service with the Consul agent
```
cat 07-define-external-service-json.sh
chmod +x 07-define-external-service-json.sh
./07-define-external-service-json.sh 
curl -X PUT -d @external-counting.json http://localhost:8500/v1/agent/service/register
```

### The external service should show in the web console of Consul.

### Asciinema recording - Consul Service Discovery
[![asciicast](https://asciinema.org/a/275154.svg)](https://asciinema.org/a/275154)

### End of Consul Service Discovery commands

## Traffic Management



### End of Consul Traffic Management commands