# EVMMv2 Setup (Tested Environment)

This document describes the minimum system setup that we used to run EVMMv2.
The setup targets Kubernetes + CRI-O on Debian/Ubuntu-based systems.

> Tested versions:
> - Kubernetes: v1.27.x ~ v1.30.x
> - CRI-O: v1.30.x
>
> Note: This explanation is based primarily on v1.30.

## 0) Prerequisites

- Debian/Ubuntu-based Linux
- `curl`, `gpg`, `apt-transport-https`, `jq`
- Root or sudo privileges
- cgroup v2 enabled

## 1) Install Kubernetes packages (kubelet/kubeadm/kubectl) from pkgs.k8s.io
```bash
export KUBERNETES_VERSION=v1.30

sudo mkdir -p /etc/apt/keyrings

curl -fsSL "https://pkgs.k8s.io/core:/stable:/${KUBERNETES_VERSION}/deb/Release.key" \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBERNETES_VERSION}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

2) Install CRI-O from pkgs.k8s.io
```bash
export CRIO_VERSION=v1.30

curl -fsSL "https://pkgs.k8s.io/addons:/cri-o:/stable:/${CRIO_VERSION}/deb/Release.key" \
  | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/${CRIO_VERSION}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/cri-o.list
```

3) Install packages and start CRI-O
```bash
sudo apt-get update
sudo apt-get install -y cri-o kubelet kubeadm kubectl
sudo systemctl enable --now crio.service
```

4) Kernel modules and sysctl required by Kubernetes networking
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

5) CRIU / Checkpoint configuration (CRI-O + runc)
EVMMv2 uses checkpoint/restore functionality. On some environments, crun may fail to locate CRIU libraries
(e.g., failed: could not load libcriu.so.2). In that case, switching the default runtime to runc is required.

Configure CRI-O:

Edit: /etc/crio/crio.conf.d/10-crio.conf

```
[crio.image]
signature_validation = false

[crio.runtime]
default_runtime = "runc"
drop_infra_ctr = false
```

Install CRIU and restart CRI-O:

```bash
sudo apt-get install -y criu
sudo systemctl daemon-reload
sudo systemctl restart crio
```
Note: drop_infra_ctr=false is required to restore containers as pods. Otherwise, CreateContainerError may occur.

6) Bootstrap Kubernetes with kubeadm (feature gates)
Create kubeadm-config.yaml:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
failSwapOn: false
featureGates:
  InPlacePodVerticalScaling: true
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.30.14
apiServer:
  extraArgs:
    feature-gates: "InPlacePodVerticalScaling=true"
controllerManager:
  extraArgs:
    feature-gates: "InPlacePodVerticalScaling=true"
scheduler:
  extraArgs:
    feature-gates: "InPlacePodVerticalScaling=true"
```

Initialize:

```bash
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.out
```

Set kubeconfig:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

7) Install CNI (Calico)
```bash
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml
kubectl apply -f calico.yaml
```

8) Kubelet auth change for checkpoint API testing
Edit: /var/lib/kubelet/config.yaml (on each node) and restart kubelet.

```bash
sudo systemctl restart kubelet
```
Smoke test:
```bash
curl -sk https://127.0.0.1:10250/pods
```
Example pod:
```
kubectl create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: docker.io/nginx:latest
EOF
```
