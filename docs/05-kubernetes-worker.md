# Bootstrapping Kubernetes Workers

In this lab you will bootstrap 3 Kubernetes worker nodes. The following virtual machines will be used:

* worker0
* worker1
* worker2

## Why

Kubernetes worker nodes are responsible for running your containers. All Kubernetes clusters need one or more worker nodes. We are running the worker nodes on dedicated machines for the following reasons:

* Ease of deployment and configuration
* Avoid mixing arbitrary workloads with critical cluster components. We are building machine with just enough resources so we don't have to worry about wasting resources.

Some people would like to run workers and cluster services anywhere in the cluster. This is totally possible, and you'll have to decide what's best for your environment.


## Provision the Kubernetes Worker Nodes

Run the following commands on `worker0`, `worker1`, `worker2`:

#### Move the TLS certificates in place

```
sudo mkdir -p /var/lib/kubernetes
```

```
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /var/lib/kubernetes/
```

#### rkt

Rkt will be the container runtime. We'll use the .deb packaged by CoreOS

```
wget https://github.com/coreos/rkt/releases/download/v1.15.0/rkt_1.15.0-1_amd64.deb
```

```
sudo dpkg -i rkt_1.15.0-1_amd64.deb
```

Rkt does not have a centralized runtime daemon. However, it does provide a local API service
for the kubelet to inspect running containers.  Enable it:

```
cat > rkt-api.service <<EOF
[Unit]
Description=rkt api service
Documentation=http://github.com/coreos/rkt
After=network.target rkt-api-tcp.socket
Requires=rkt-api.socket

[Service]
ExecStart=/usr/bin/rkt api-service

[Install]
WantedBy=multi-user.target
EOF
```

```
cat > rkt-api.socket <<EOF
[Unit]
Description=rkt api service socket
PartOf=rkt-api.service

[Socket]
ListenStream=127.0.0.1:15441
ListenStream=[::1]:15441
Service=rkt-api.service
BindIPv6Only=both

[Install]
WantedBy=sockets.target
EOF
```

```
sudo mv rkt-api.* /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable rkt-metadata.socket rkt-api.socket
sudo systemctl start rkt-metadata.socket rkt-api.socket
```

#### kubelet

The Kubernetes kubelet can now use [CNI - the Container Network Interface](https://github.com/containernetworking/cni) to manage machine level networking requirements.

Download and install CNI plugins

```
sudo mkdir -p /opt/cni
```

```
wget https://storage.googleapis.com/kubernetes-release/network-plugins/cni-07a8a28637e97b22eb8dfe710eeae1344f69d16e.tar.gz
```

```
sudo tar -xvf cni-07a8a28637e97b22eb8dfe710eeae1344f69d16e.tar.gz -C /opt/cni
```


Download and install the Kubernetes worker binaries:

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.4.0/bin/linux/amd64/kubectl
```
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.4.0/bin/linux/amd64/kube-proxy
```
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.4.0/bin/linux/amd64/kubelet
```

```
chmod +x kubectl kube-proxy kubelet
```

```
sudo mv kubectl kube-proxy kubelet /usr/bin/
```

```
sudo mkdir -p /var/lib/kubelet/
```

```
sudo sh -c 'echo "apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.pem
    server: https://10.240.0.10:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubelet
  name: kubelet
current-context: kubelet
users:
- name: kubelet
  user:
    token: chAng3m3" > /var/lib/kubelet/kubeconfig'
```

Create the kubelet systemd unit file:

```
sudo sh -c 'echo "[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kubelet \
  --allow-privileged=true \
  --api-servers=https://10.240.0.10:6443,https://10.240.0.11:6443,https://10.240.0.12:6443 \
  --cloud-provider= \
  --cluster-dns=10.32.0.10 \
  --cluster-domain=cluster.local \
  --configure-cbr0=true \
  --container-runtime=rkt \
  --rkt-path=/usr/bin/rkt \
  --rkt-api-endpoint=localhost:15441 \
  --network-plugin=kubenet \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --reconcile-cidr=true \
  --serialize-image-pulls=false \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2
  
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/kubelet.service'
```

```
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

```
sudo systemctl status kubelet --no-pager
```


#### kube-proxy


```
sudo sh -c 'echo "[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-proxy \
  --master=https://10.240.0.10:6443 \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --proxy-mode=iptables \
  --v=2
  
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/kube-proxy.service'
```

```
sudo systemctl daemon-reload
sudo systemctl enable kube-proxy
sudo systemctl start kube-proxy
```

```
sudo systemctl status kube-proxy --no-pager
```

> Remember to run these steps on `worker0`, `worker1`, and `worker2`
