# EVMMv2 Setup (Tested Environment)

This document describes the minimum system setup that we used to run EVMMv2.
The setup targets Kubernetes + CRI-O on Debian/Ubuntu-based systems.

Verification  versions:
 - Ubuntu 20.04
 - Kubernetes: v1.30.x
 - CRI-O: v1.30.x
 - Buildah : 1.33.x
 - CRIU: 4.1.1
Notes
- This guide primarily targets Kubernetes v1.30.x. Other versions (e.g., v1.27.x) may require additional adjustments.
- Ensure that a recent CRIU version is used, as outdated versions may cause checkpoint/restore failures.
- EVMMv2 uses Buildah internally to construct checkpoint-based container images.


## Prerequisites
- Ubuntu 20.04 or later
- cgroup v2 enabled
- Passwordless (`NOPASSWD`) sudo access
- Standard GNU/Linux userland tools (curl, gpg, jq)
- buildah, criu

## Privilege Assumptions

EVMMv2 performs runtime control operations that require elevated privileges
(e.g., interactions with CRI-O, kubelet, and checkpoint/restore mechanisms).

For experimental simplicity and deterministic behavior, the implementation
**assumes passwordless (`NOPASSWD`) sudo access** for the executing user.

This design choice reflects the controlled experimental environment used in the evaluation 
and eliminates interactive privilege prompts during runtime control.

## 0) Install Required Packages
```bash
sudo apt update -y
sudo apt upgrade -y
sudo apt install curl gpg jq nano criu buildah -y
```

## 1) Get Kubernetes packages (kubelet/kubeadm/kubectl)

```bash
sudo swapoff -a # Required by Kubernetes default configuration

export KUBERNETES_VERSION=v1.30

sudo mkdir -p /etc/apt/keyrings

curl -fsSL "https://pkgs.k8s.io/core:/stable:/${KUBERNETES_VERSION}/deb/Release.key" \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBERNETES_VERSION}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

## 2) Get CRI-O package

```bash
export CRIO_VERSION=v1.30

curl -fsSL "https://pkgs.k8s.io/addons:/cri-o:/stable:/${CRIO_VERSION}/deb/Release.key" \
  | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/${CRIO_VERSION}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/cri-o.list
```

## 3) Install packages and start CRI-O serivce
   
```bash
sudo apt-get update
sudo apt-get install -y cri-o kubelet kubeadm kubectl
sudo systemctl enable --now crio.service
```

## 4) Kubernetes network setting

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

## 5) CRIU / Checkpoint configuration (CRI-O + runc)
EVMMv2 relies on checkpoint/restore functionality.
In some environments, `crun` may fail to locate CRIU libraries
(e.g., `failed: could not load libcriu.so.2`). In such cases, switching the default runtime to `runc` is required.

Additionally, `signature_validation` must be disabled to allow recovery from checkpoint-based container images.
Setting drop_infra_ctr = false is required to restore containers as pods; otherwise, `CreateContainerError` may occur.


### 5.1. Configure CRI-O:

Edit `/etc/crio/crio.conf.d/10-crio.conf`:

```
[crio.image]
signature_validation = false

[crio.runtime]
default_runtime = "runc"
drop_infra_ctr = false
```

### 5.2. Restart CRI-O:

```bash
sudo systemctl daemon-reload
sudo systemctl restart crio
```

### 5.3. Initialize:

```bash
sudo kubeadm init  --upload-certs | tee kubeadm-init.out
```

## 6) Configure kubeconfig:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 7) Install CNI (Calico)
```bash
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml
kubectl apply -f calico.yaml
```
Note: 
The Calico manifest above is verified to work with Kubernetes v1.30.x in our environment.
Depending on the target environment, users may need to select an appropriate Calico version.

## 8) Kubelet auth change for checkpoint API testing
Edit `/var/lib/kubelet/config.yaml` on each node and restart kubelet.

```
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: true
  webhook:
    cacheTTL: 0s
    enabled: false
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: AlwaysAllow
```

```bash
sudo systemctl restart kubelet
```

## (Optional) Checkpoint and Image Creation Verification:

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

Create a checkpoint:
```bash
curl -sk -X POST "https://localhost:10250/checkpoint/default/nginx/nginx"
```

Verify the checkpoint:

```bash
sudo ls -l /var/lib/kubelet/checkpoints
```

Create a checkpoint image using Buildah:

```bash
newcontainer=$(sudo buildah from scratch)
sudo buildah add $newcontainer /var/lib/kubelet/checkpoints/checkpoint-nginx_default-nginx-2026-00-00T00:00:00+00:00.tar /
sudo buildah config --annotation=io.kubernetes.cri-o.annotations.checkpoint.name=nginx $newcontainer
sudo buildah commit $newcontainer checkpoint-image:latest
sudo buildah rm $newcontainer
```
Note: 
The checkpoint tar filename varies depending on the pod name and creation time.
Replace the example filename with the actual file listed in `/var/lib/kubelet/checkpoints`.

Restore the container using the locally created image:
```bash
kubectl create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: restore-nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: localhost/checkpoint-image:latest
    imagePullPolicy: Never # If not set to # never, a connection refused error occurs (unable to retrieve internal images).
  nodeName: your-nodename
EOF

```
