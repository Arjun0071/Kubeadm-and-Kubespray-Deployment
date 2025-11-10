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
6. [Screenshots (placeholders)]  


## Architecture Overview
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

1.[Creating and Activating a Python Virtual Environment](https://docs.python.org/3/library/venv.html)

2. [Ansible Installation via pip](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#pip-install)

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
" />
</p>
