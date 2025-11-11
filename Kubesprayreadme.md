# Kubernetes Cluster Deployment with Kubespray, MetalLB, NGINX Ingress, and Cert-Manager

This guide provides a step-by-step tutorial to deploy a Kubernetes cluster using **Kubespray** on two private VMs (one master and one worker node) via a **jump server**, with **MetalLB** providing external IPs for NGINX ingress and **Cert-Manager** issuing Let's Encrypt certificates. The DNS name points to the jump server, and port-forwarding routes traffic to the application.

---
## Table of Contents

[Overview & Architecture] 
1. [Important Note — SSH access]  
2. [Commands for Master & Worker Nodes (Pre-requisites)]  
3. [Jump Server Setup] 
4. [Complete Master node — MetalLB, Ingress, Cert-Manager, App Deploy]
5. [Jump Server — kubeconfig, kubectl, persistent port-forwarding] 
6. [Screenshots]
7. [CI/CD Integration] 
    

## Architecture Overview
The following diagram explains how external traffic flows securely from the public internet to the private Kubernetes cluster through the Jump Server and Ingress Controller:

![kubespray-architecture](https://github.com/user-attachments/assets/0c8455fb-85c0-4492-99d1-1a87fb0810ff)


```
        Internet
            |
        DNS points
            |
 [Jump Server] (public IP)
            |
Port Forwarding to Ingress-NGINX
            |
      +-----+-----+
      |           |
[Master Node]  [Worker Node]
(K8s Control Plane)  (K8s Worker)

```
- **Jump Server**: Public access point, manages Ansible/Kubespray deployment and port forwarding.  
- **Master Node**: Runs Kubernetes control plane components.  
- **Worker Node**: Runs workloads.  
- **MetalLB**: Provides external IPs to services like NGINX Ingress.  
- **Cert-Manager**: Issues TLS certificates via Let's Encrypt.  

---
## Important Note — SSH access
Before you start, **enable passwordless SSH from the Jump Server to both private VMs**

## Prerequisites

### Master & Worker Nodes

**1. Disable swap**
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
**2. Update OS & install Python**
```
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-apt apt-transport-https ca-certificates curl gnupg lsb-release
which python3
```

**3. Enable required kernel modules**
```
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/99-kubernetes-cri.conf <<'EOF'
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

**4. Ensure time synchronization**
Ubuntu 20.04/22.04 use systemd-timesyncd by default; optionally install ntp
```
sudo timedatectl status
```
if not synced:
[Install Chrony](https://infotechys.com/chrony-commands-for-system-administrators/)

**5. Install helpful tools (optional)**
```
sudo apt install -y jq net-tools curl vim git
```
### Jump Server Setup ###

**1. Update & install dependencies**
```
sudo apt update && sudo apt upgrade -y
sudo apt install -y git python3-pip sshpass curl python3-venv
```

**2. Set up Python virtual environment**

[Creating and Activating a Python Virtual Environment](https://docs.python.org/3/library/venv.html)

[Ansible Installation via pip](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#pip-install)

Make local binaries accessible and Verify  Installation
```
ansible --version
```

**3. Clone Kubespray & install requirements**

Clone the Kubespray Repository:
```
git clone https://github.com/kubernetes-sigs/kubespray.git
```
```
cd kubespray
sudo pip3 install -r requirements.txt
```

**4. Create inventory**
```
cp -r inventory/sample inventory/mycluster
```

Edit inventory/mycluster/hosts.yaml with your nodes’ IPs and users.

**5. Test connectivity**
```
ansible -i inventory/mycluster/hosts.yaml all -m ping
```

**6. Deploy Kubernetes cluster**
```
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml -v
```
**7.Configure Kubeconfig**

***Copy kubeconfig from Master to Jump Server***

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
**8. **Kubectl:** [Install kubectl on Jump Server](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)**


Verify connectivity:
```
KUBECONFIG=/root/.kube/config kubectl get nodes
# or
kubectl get nodes
```

***Merging with existing kubeconfig (if needed):***
```
export KUBECONFIG=~/.kube/config:~/admin.conf
kubectl config view --merge --flatten > ~/.kube/config.merged
mv ~/.kube/config.merged ~/.kube/config
kubectl config get-contexts
kubectl config use-context <context-name>
```
### Master Node Setup
**1.Helm: [Helm Installation](https://helm.sh/docs/intro/install/)**

**2.MetalLB: [Install MetalLB](https://metallb.universe.tf/installation/)**
Apply MetalLB address pool configuration (file exists in repo and make changes to it accordingly)
This repo contains metallb-config.yaml — apply that file:

kubectl apply -f metallb-config.yaml

The metallb-config.yaml in this repo defines the IPAddressPool and L2Advertisement used by MetalLB. Edit it if you need a different range for your environment.

***Note: The repository contains the configuration files metallb-config.yaml, cluster-issuer.yaml, and ingress.yaml. You can use them as templates and modify them according to your environment (e.g., IP ranges, DNS names, service names).***

**3.NGINX Ingress Controller: [Deploy Ingress NGINX](https://platform9.com/learn/v1.0/tutorials/nginix-controller-via-yaml)**
```
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```
The ingress-nginx-controller service should get an EXTERNAL-IP assigned by MetalLB.

**4.Deploy Application:**
Deploy your application using the Helm chart in the repo:
```
helm install <your_helm_chart_name> ./<path_to_chart> --namespace default --create-namespace
```
**5.Cert-Manager: [Install Cert-Manager](https://cert-manager.io/docs/installation/)**
```
kubectl get pods -n cert-manager
```
Apply ClusterIssuer (file exists in repo make changes to it accordingly)
This repo contains clusterissuer.yaml (configures ACME/Let’s Encrypt). Apply it:
```
kubectl apply -f cluster-issuer.yaml
```
**6. Apply the Ingress resource (file exists in repo)**
This repo contains ingress.yaml (Update the placeholder values accordingly). Apply it:

kubectl apply -f ingress.yaml


ingress.yaml references the ClusterIssuer letsencrypt-prod (cert-manager) and requests a TLS certificate. cert-manager will create a Certificate and attempt ACME HTTP01 validation.

## Configure Port-Forward on Jump Server
Final step: configure the Jump Server so the cluster is accessible externally.
```
sudo kubectl port-forward svc/ingress-nginx-controller -n ingress-nginx 80:80 443:443 --address 0.0.0.0
```
****Make it persistent with systemd***
#### Output will show:
#### Forwarding from 0.0.0.0:80 -> 80
#### Forwarding from 0.0.0.0:443 -> 443

### Make the port-forward persistent using systemd
Create /etc/systemd/system/k8s-kubespray-portforward.service with the following content (on Jump Server):
```
sudo nano /etc/systemd/system/k8s-kubespray-portforward.service
```
Now paste this entire block inside:
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
sudo systemctl enable --now k8s-kubespray-portforward
sudo systemctl status k8s-kubespray-portforward
```
Open firewall ports (ufw example (if your ufw is active)
```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```
Access the Application
Verify Cert-Manager & Ingress
```
kubectl get certificate -A
kubectl describe certificate myapi-cert -n default   # or name used in repo
kubectl describe ingress fastapi-ingress -n default  # or the ingress name in repo
```
If Certificate shows Ready=True, the TLS secret exists and you should be able to access the app via HTTPS.

Visit:

https://myapi.servebeer.com

If cert-manager successfully issued the certificate and Ingress is configured properly, https://DNS_HostName should now serve your application with a valid TLS certificate.

## Screenshots:
### Application Accessible via DNS Hostname
<p align="center">
<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/f0931074-d3dd-45cc-9bdb-15b2ec47a14b" />
" />
</p>

### Verified HTTPS Connection
<p align="center">
<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/693de705-1e37-4169-ab9e-cdb3a24554b7" />
</p>








## CI/CD Integration — Automated Build & Deployment via GitHub Actions
This project integrates a fully automated CI/CD pipeline to streamline Docker image creation, Helm upgrades, and cluster deployment on the private Kubernetes setup built using kubeadm.

### Overview

Whenever code is pushed to the main branch, the pipeline automatically:
Builds and pushes the Docker image to Docker Hub.
Triggers deployment on the Kubernetes master node (via Jump Server) using Helm upgrade to roll out the updated image.
This ensures a continuous delivery loop: every code change is built, containerized, and deployed securely to the private cluster.

---

### 🚀 1. Continuous Integration (CI) — Build and Push Docker Image

**Workflow File:** `.github/workflows/ci.yml`

This workflow automates the build and image-push process every time a commit is pushed to the **main branch**.

#### 🔧 Steps

**1. Checkout Repository**  
   Fetches the latest code from the repository.

**2. Docker Hub Authentication**  
   Logs in to Docker Hub using stored **GitHub Secrets**:  
   - `DOCKERHUB_USERNAME`  
   - `DOCKERHUB_TOKEN`

**3. Build Docker Image**  
   Builds a Docker image of the FastAPI application and tags it with the unique Git commit SHA for version traceability:
   ```bash
   docker build -t <DOCKERHUB_USERNAME>/fastapi-demo:<GITHUB_SHA> .
   ```
**4. Push Image to Docker Hub**
   Pushes the built image to Docker Hub:
   ```
   docker push <DOCKERHUB_USERNAME>/fastapi-demo:<GITHUB_SHA>
   ```

**5. Expose Image Tag for CD Pipeline**
   Exports the built image tag (GITHUB_SHA) so the CD workflow can use it during deployment.

## 🧩 2. Continuous Deployment (CD) — Helm-Based Deployment via Jump Server

**Workflow File:** `.github/workflows/cd.yml`

This workflow triggers automatically after the **CI workflow** completes successfully.  
It securely connects to the **Jump Server**, which has SSH access to the **Kubernetes master node**, and deploys the latest Docker image using **Helm**.

---

### 🔧 Steps

**1. Checkout Repository**  
   Clones the repository to access the Helm chart.

**2. Set Up SSH Authentication**  
   Configures the SSH agent using a private key stored in **GitHub Secrets**:  
   - `SSH_PRIVATE_KEY`

**3. Deploy via Jump Server**  
   Establishes a secure SSH connection to the Jump Server, then connects to the Kubernetes master node to perform the Helm upgrade.  

   Uses these environment variables (defined as GitHub Secrets):  
   - `JUMP_USER`, `JUMP_HOST` — Jump Server credentials  
   - `MASTER_USER`, `MASTER_IP` — Master node credentials  
   - `DOCKERHUB_USERNAME` — Docker Hub username  

**4. Helm Upgrade Execution**  
   On the master node, the workflow ensures the Helm chart exists at:  
   `/home/<MASTER_USER>/FastAPI-CI-CD/demoapp-chart`  

   Then executes:
   ```
   helm upgrade --install fastapi-release /home/<MASTER_USER>/FastAPI-CI-CD/demoapp-chart \
     --reuse-values \
     --set image.repository=<DOCKERHUB_USERNAME>/fastapi-demo \
     --set image.tag=<GITHUB_SHA> \
     --set image.pullPolicy=Always \
     --namespace default \
     --atomic
   ```
  
  The --reuse-values flag ensures existing service configurations (like NodePort) are preserved while updating only the image

  ## 🔐 3. Secrets Used

| Secret Name | Description |
|--------------|-------------|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `SSH_PRIVATE_KEY` | Private key for SSH access |
| `JUMP_USER`, `JUMP_HOST` | Jump Server SSH credentials |
| `MASTER_USER`, `MASTER_IP` | Kubernetes Master Node credentials |

---

## 🧠 4. Workflow Trigger

The **CD workflow** runs only after the **CI workflow** (`CI- Build and push the Docker Image`) completes successfully:

```yaml
on:
  workflow_run:
    workflows: ["CI- Build and push the Docker Image"]
    types:
      - completed
```
This ensures only a successfully built and pushed image is deployed to the cluster.

## 📊 5. Summary

| Stage | Tool | Description |
|--------|------|-------------|
| **CI** | Docker + GitHub Actions | Builds and pushes the FastAPI image to Docker Hub |
| **CD** | Helm + SSH + Jump Server | Securely deploys the updated image to the Kubernetes cluster |
| **Security** | GitHub Secrets | Manages all sensitive credentials safely |

---

## End Result

Each new commit on the **main branch** automatically:  
- Builds a Docker image  
- Pushes it to **Docker Hub**  
- Triggers a **Helm-based deployment** to your private Kubernetes cluster  

> Achieving a **zero-touch, fully automated CI/CD pipeline** — secure, version-controlled, and production-ready.
