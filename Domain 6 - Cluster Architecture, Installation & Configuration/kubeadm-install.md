##### Documentation Link:

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

##### Step 1: Setup containerd
```sh
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```
```sh
modprobe overlay
modprobe br_netfilter
```
```sh
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
```sh
sysctl --system
```
```sh
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```
```sh
nano /etc/containerd/config.toml
```
  --> SystemdCgroup = true

```sh
systemctl restart containerd
```

##### Step 2: Kernel Parameter Configuration
```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```sh
sudo sysctl --system
```

##### Step 3: Configuring Repo and Installation
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```sh
sudo apt-get update
apt-cache madison kubeadm
sudo apt-get install -y kubelet=1.27.0-00 kubeadm=1.27.0-00 kubectl=1.27.0-00 cri-tools=1.26.0-00
apt search cri-tools  
sudo apt-mark hold kubelet kubeadm kubectl
```

#### Step 4: Initialize Cluster with kubeadm (Only master node)
```sh
kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=1.27.0
```
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


```
##### Step 4 b: crictl to default the runtime endpoint (master node)
```sh
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
EOF
```

##### Step 4 c: Install etcdctl  
```sh
export RELEASE=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
wget https://github.com/etcd-io/etcd/releases/download/${RELEASE}/etcd-${RELEASE}-linux-amd64.tar.gz
tar xvf etcd-${RELEASE}-linux-amd64.tar.gz
cd etcd-${RELEASE}-linux-amd64
sudo mv etcd etcdctl etcdutl /usr/local/bin 

$ etcd --version
etcd Version: 3.5.2
Git SHA: 99018a77b
Go Version: go1.16.3
Go OS/Arch: linux/amd64

$ etcdctl version
etcdctl version: 3.5.2
API version: 3.5

$ etcdutl version
etcdutl version: 3.5.2
API version: 3.5

```

##### Step 5: Install Network Addon (flannel) (master node)
```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
##### Step 6: Connect Worker Node (Only worker node)
```sh
kubeadm join 159.89.165.203:6443 --token qmw5dj.ljdh8r74ce3y85ad \
        --discovery-token-ca-cert-hash sha256:83374ec05088fa7efe9c31cce63326ae7037210ab049048ef08f8c961a048ddf
```
#### Step 7: Verification
```sh
kubectl get nodes
kubectl run nginx --image=nginx
kubectl get pods -o wide
```
