# Install kubernetes on ubuntu:

------------

### basic steps:
Firstly we should disable the swap feature on all nodes:
```bash
swapoff -a; sed -i '/swap/d' /etc/fstab
```
Then we need to verify the mac is unique:
```bash
sudo cat /sys/class/dmi/id/product_uuid
```
Letting iptables see bridged traffic:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

and for containerd we should set these up:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
Apply sysctl params without reboot:
```bash
sudo sysctl --system
```

------------
### Installing docker and containerd :
Lets update the repo first:
```bash
sudo apt-get update
```
now add some basic packages:
```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    unzip
```
Add Docker official GPG key:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
Add Docker repo:
```bash
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Install Docker:
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
Configure containerd:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```
Using the systemd cgroup driver
To use the systemd cgroup driver in /etc/containerd/config.toml with runc, set
```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```
Configure the Docker daemon, in particular to use systemd for the management of the container’s cgroups.
```bash
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Restart containerd:
```bash
sudo systemctl restart containerd
```

Enable Docker:
```bash
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

------------

### Installing kubernetes tools:
Install basic packages:
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl
```
Download the Google Cloud public signing key:
```bash
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
Add the Kubernetes apt repository:
```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

------------
### Configuring kubernetes cluster:
#### on the master node:
pull the requierd docker images first (Optional):
```bash
kubeadm config images pull
```
Provisioning  the master node:

```bash
kubeadm init --apiserver-advertise-address=<master_node_ip> --pod-network-cidr=192.168.0.0/16  --ignore-preflight-errors=all
```
Run kubectl from none root user:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Deploy calico networking model:
```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```




