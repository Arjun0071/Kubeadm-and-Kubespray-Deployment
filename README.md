# 🚀 Secure Kubernetes Deployment (kubeadm) — End-to-End Tutorial
**Kubernetes (kubeadm) + MetalLB + NGINX Ingress + Cert-Manager (Let’s Encrypt)**
  
This README describes a hands-on, end-to-end setup used to deploy a Python web application (packaged via Helm and hosted on Docker Hub) on a small, self-managed Kubernetes cluster. The cluster lives on **private VMs** (Master + Worker) and is exposed securely via a **public Jump Server** using persistent port-forwarding.

> **Public test URL used in this repo (example):** `https://myapi.servebeer.com`  
> **Placeholders you must replace:** `<MASTER_IP>`, `<token>`, `<hash>`, `<your_helm_chart_name>`, `<your_namespace>`, `<your_dockerhub_repo/image:tag>`, `<your_email@example.com>`

---

## Table of Contents
1. [Overview & Architecture] 
2. [Important Note — SSH access]  
3. [Commands for Master & Worker Nodes (common)]  
4. [Master node — initialize control plane] 
5. [Worker node — join the cluster]
6. [Complete Master node — MetalLB, Ingress, Cert-Manager, App Deploy]
7. [Jump Server — kubeconfig, kubectl, persistent port-forwarding] 
8. [Screenshots (placeholders)]  


---

## Overview & Architecture

The following diagram explains how external traffic flows securely from the public internet to the private Kubernetes cluster through the Jump Server and Ingress Controller:

Internet  
↓  
DNS → `myapi.servebeer.com`  
↓  
**Jump Server (Public IP)** — persistent `kubectl port-forward` → **Ingress Controller (NGINX)**  
↓  
**Kubernetes Cluster (Private VMs)**  
├─ **Master (Control Plane)** — initialized via `kubeadm`, hosts MetalLB controller, NGINX Ingress, and Cert-Manager  
└─ **Worker Node(s)** — host application pods  

- **MetalLB** allocates `LoadBalancer` IPs from the **private subnet**, simulating cloud-style load balancing for on-prem environments.  
- **Cert-Manager** automates SSL certificate management by obtaining and renewing **Let’s Encrypt** certificates via the **HTTP-01 challenge** handled through the Ingress.

## Important Note — SSH access
Before you start, **enable passwordless SSH from the Jump Server to both private VMs**

## ⚙️ Commands for Master & Worker Nodes (Common)

Run the following commands on **both Master and Worker nodes** (❌ not on the Jump Server unless explicitly mentioned).

---

### 1. 🧩 Update OS Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. 🐋 Install & Configure containerd

```
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
```
# Generate default config
```
sudo containerd config default | sudo tee /etc/containerd/config.toml
```
# Restart & enable
```
sudo systemctl restart containerd
sudo systemctl enable containerd
```
# Check status
```
sudo systemctl status containerd
```
### ✅ Ensure systemd cgroup is used
```
containerd config dump | grep SystemdCgroup
# Expected: SystemdCgroup = true
```

### If the output is false, edit:
```
sudo vim /etc/containerd/config.toml
# In the section:
# [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
# add/update:
SystemdCgroup = true

sudo systemctl restart containerd
```
### 3. Install kubeadm, kubelet, kubectl
```
sudo apt install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
### 4. Disable swap and enable kernel settings
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```
## Master node — initialize control plane
Run the following on the Master node only.

### 1. Initialize Kubernetes master
(Using Flannel pod network example CIDR)
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Save the kubeadm output — it contains the join command used by Workers (kubeadm join <MASTER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>).

### 2. Set up kubectl for your user
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### 3. Optional: install net-tools & verify API server listening
```
sudo apt install -y net-tools
sudo netstat -tulnp | grep 6443   # Kubernetes API server should be listening on 6443
```

### 4. Apply Flannel CNI (example)
```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### 5. Verify Master node state
```
kubectl get nodes
# Note: Master may show NotReady until CNI is up
```

## Worker node — join the cluster
Run on each Worker node using the join command produced by kubeadm init
```
sudo kubeadm join <MASTER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
If the token expired, create a new one on the master:
```
# Run on Master
kubeadm token create --print-join-command
```
Verify on the Master:

kubectl get nodes
#### After a few moments, nodes should show as Ready

## Complete Master node — install MetalLB, Ingress, Cert-Manager, and deploy app
After Worker nodes have joined, finish the control-plane configuration and deploy network services & the app.
### 1. Install Helm (if not already installed)
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
### 2. Install MetalLB (Helm)
```
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb -n metallb-system --create-namespace
kubectl get pods -n metallb-system
```
Apply MetalLB address pool configuration (file exists in repo and make changes to it accordingly)
This repo contains metallb-config.yaml — apply that file:

kubectl apply -f metallb-config.yaml

The metallb-config.yaml in this repo defines the IPAddressPool and L2Advertisement used by MetalLB. Edit it if you need a different range for your environment.

### 3. Deploy NGINX Ingress Controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```
The ingress-nginx-controller service should get an EXTERNAL-IP assigned by MetalLB.

### 4. Install Cert-Manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl get pods -n cert-manager
```
Apply ClusterIssuer (file exists in repo make changes to it accordingly)
This repo contains clusterissuer.yaml (configures ACME/Let’s Encrypt). Apply it:
```
kubectl apply -f clusterissuer.yaml
```

### 5. Deploy your application using Helm
```
helm install <your_helm_chart_name> ./<path_to_chart> --namespace default --create-namespace
```
Make sure the service exposed by your chart uses the port expected by your ingress (example in your setup: port 8000).

### 6. Apply the Ingress resource (file exists in repo)
This repo contains ingress.yaml (Update the placeholder values accordingly). Apply it:

kubectl apply -f ingress.yaml


ingress.yaml references the ClusterIssuer letsencrypt-prod (cert-manager) and requests a TLS certificate. cert-manager will create a Certificate and attempt ACME HTTP01 validation.

### 7. Verify Cert-Manager & Ingress
```
kubectl get certificate -A
kubectl describe certificate myapi-cert -n default   # or name used in repo
kubectl describe ingress fastapi-ingress -n default  # or the ingress name in repo
```
If Certificate shows Ready=True, the TLS secret exists and you should be able to access the app via HTTPS.


## Jump Server — kubeconfig, kubectl & persistent port-forwarding
Final step: configure the Jump Server so the cluster is accessible externally.

### 1. Copy kubeconfig from Master to Jump Server

On Jump Server:
```
scp <MASTER_USER>@<MASTER_IP>:/etc/kubernetes/admin.conf /root/.kube/config
```
Set proper permissions:
```
sudo mkdir -p /root/.kube
sudo chown root:root /root/.kube/config
sudo chmod 600 /root/.kube/config
ls -l /root/.kube/config
```
### 2. Install kubectl on Jump Server
```
sudo apt-get update
sudo apt-get install -y ca-certificates curl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

Verify connectivity:
```
KUBECONFIG=/root/.kube/config kubectl get nodes
# or
kubectl get nodes
```

### 3. Port-forward ingress to public ports 80/443 (quick test)
```
sudo kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 80:80 443:443 --address 0.0.0.0
```
#### Output will show:
#### Forwarding from 0.0.0.0:80 -> 80
#### Forwarding from 0.0.0.0:443 -> 443

### 4. Make the port-forward persistent using systemd

Create /etc/systemd/system/k8s-portforward.service with the following content (on Jump Server):
```
[Unit]
Description=Kubernetes port-forward for ingress controller (80/443)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Environment=KUBECONFIG=/root/.kube/config
ExecStart=/usr/local/bin/kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 80:80 443:443 --address 0.0.0.0
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable & start it:
```
sudo systemctl daemon-reload
sudo systemctl enable --now k8s-portforward.service
sudo systemctl status k8s-portforward.service
```
5. Open firewall ports (ufw example (if your ufw is active)
```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```
If cert-manager successfully issued the certificate and Ingress is configured properly, https://DNS_HostName should now serve your application with a valid TLS certificate.

## Screenshots:
### Application Accessible via DNS Hostname
<p align="center">
<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/e756f246-445d-493e-93fd-e028ac6601e4" />
</p>

### Verified HTTPS Connection
<p align="center">
<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/8292b8f0-b444-4e3f-986f-84ff7d83d287" />
</p>









