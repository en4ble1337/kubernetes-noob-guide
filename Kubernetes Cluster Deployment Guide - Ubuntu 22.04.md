# Kubernetes Cluster Deployment Guide - Ubuntu 22.04

A comprehensive guide for deploying and managing Kubernetes clusters from the ground up using containerd runtime.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Part 1: Initial Setup and Package Installation](#part-1-initial-setup-and-package-installation)
  - [System Preparation](#system-preparation)
  - [Installing containerd](#installing-containerd)
  - [Installing Kubernetes Packages](#installing-kubernetes-packages)
- [Part 2: Creating the Control Plane Node](#part-2-creating-the-control-plane-node)
  - [Initializing the Cluster](#initializing-the-cluster)
  - [Configuring kubectl Access](#configuring-kubectl-access)
  - [Deploying Pod Network (Calico)](#deploying-pod-network-calico)
- [Part 3: Adding Worker Nodes](#part-3-adding-worker-nodes)
  - [Node Setup](#node-setup)
  - [Joining Nodes to Cluster](#joining-nodes-to-cluster)
- [Part 4: Working with Your Cluster](#part-4-working-with-your-cluster)
  - [Cluster Inspection](#cluster-inspection)
  - [Deploying Applications](#deploying-applications)
  - [Imperative vs Declarative Deployments](#imperative-vs-declarative-deployments)
- [Part 5: Cloud Deployments](#part-5-cloud-deployments)
  - [Azure Kubernetes Service (AKS)](#azure-kubernetes-service-aks)
  - [Google Kubernetes Engine (GKE)](#google-kubernetes-engine-gke)
- [Daily Operations Commands](#daily-operations-commands)
- [Troubleshooting Guide](#troubleshooting-guide)

---

## Prerequisites

Before beginning the installation, ensure you have:

1. **4 Ubuntu 22.04 VMs**:
   - 1 Control Plane Node (c1-cp1)
   - 3 Worker Nodes (c1-node1, c1-node2, c1-node3)

2. **Network Configuration**:
   - Static IP addresses assigned to each VM
   - `/etc/hosts` file configured with hostname to IP mappings
   - All nodes can communicate with each other

3. **System Requirements**:
   - Swap disabled on all nodes
   - Root or sudo access
   - Take VM snapshots before installation for easy rollback

---

## Part 1: Initial Setup and Package Installation

### System Preparation

**Run on all nodes (Control Plane and Worker Nodes):**

#### 1. Disable Swap

Kubernetes requires swap to be disabled for proper operation.

```bash
# Disable swap immediately
sudo swapoff -a

# Edit fstab to disable swap permanently
sudo vi /etc/fstab
# Comment out any swap entries by adding # at the beginning of the line
```

#### 2. Load Kernel Modules

Configure required kernel modules for containerd:

```bash
# Create module configuration file
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load modules immediately
sudo modprobe overlay
sudo modprobe br_netfilter
```

#### 3. Configure System Parameters

Set up networking parameters required by Kubernetes:

```bash
# Create sysctl configuration
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters without reboot
sudo sysctl --system
```

### Installing containerd

**Run on all nodes:**

```bash
# Install containerd
sudo apt-get install -y containerd

# Create containerd configuration directory
sudo mkdir -p /etc/containerd

# Generate default configuration
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Configure systemd cgroup driver
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml

# Verify the change
grep 'SystemdCgroup = true' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
```

### Installing Kubernetes Packages

**Run on all nodes:**

```bash
# Install required dependencies
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes GPG key
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list
sudo apt-get update

# Check available versions
apt-cache policy kubelet | head -n 20

# Install specific version (recommended for learning/testing)
VERSION=1.29.1-1.1
sudo apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION

# Prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl containerd
```

**Alternative - Install Latest Version:**

```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl containerd
```

#### Verify Installation

```bash
# Check kubelet status (will be inactive until cluster is created)
sudo systemctl status kubelet.service

# Check containerd status (should be active)
sudo systemctl status containerd.service
```

---

## Part 2: Creating the Control Plane Node

### Initializing the Cluster

**Run only on the Control Plane Node (c1-cp1):**

#### 1. Download Calico Network Manifest

```bash
# Download Calico configuration
wget https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml

# Review and adjust Pod Network CIDR if needed
vi calico.yaml
# Look for CALICO_IPV4POOL_CIDR setting
```

#### 2. Initialize Kubernetes Cluster

```bash
# Initialize cluster with specific version
sudo kubeadm init --kubernetes-version v1.29.1

# For latest version, omit the version parameter:
# sudo kubeadm init
```

**Important:** Save the `kubeadm join` command output - you'll need it to add worker nodes!

### Configuring kubectl Access

**Run on Control Plane Node:**

```bash
# Set up kubectl for non-root user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Deploying Pod Network (Calico)

```bash
# Deploy Calico network plugin
kubectl apply -f calico.yaml

# Watch pods starting (use Ctrl+C to exit)
kubectl get pods --all-namespaces --watch

# Verify all system pods are running
kubectl get pods --all-namespaces

# Check node status (should show Ready)
kubectl get nodes
```

#### Understanding Static Pods

```bash
# View kubelet status (now active)
sudo systemctl status kubelet.service

# Check static pod manifests
ls /etc/kubernetes/manifests

# Examine API server manifest
sudo more /etc/kubernetes/manifests/kube-apiserver.yaml

# Examine etcd manifest
sudo more /etc/kubernetes/manifests/etcd.yaml

# View kubeconfig files location
ls /etc/kubernetes
```

---

## Part 3: Adding Worker Nodes

### Node Setup

**Run on each worker node (c1-node1, c1-node2, c1-node3):**

Follow the same steps from [Part 1](#part-1-initial-setup-and-package-installation):
1. Disable swap
2. Load kernel modules
3. Configure system parameters
4. Install containerd
5. Install Kubernetes packages

### Joining Nodes to Cluster

#### 1. Generate Join Command (on Control Plane)

```bash
# On Control Plane Node, generate join command
kubeadm token create --print-join-command
```

This will output something like:
```
kubeadm join 10.1.20.30:6443 --token abc123.xyz789 --discovery-token-ca-cert-hash sha256:hash_value_here
```

#### 2. Join Worker Node

**Run on each worker node:**

```bash
# SSH into worker node
ssh aen@c1-node1

# Run join command with sudo (use the actual command from previous step)
sudo kubeadm join 10.1.20.30:6443 --token abc123.xyz789 --discovery-token-ca-cert-hash sha256:hash_value_here

# Exit back to control plane
exit
```

#### 3. Verify Node Addition

**Run on Control Plane Node:**

```bash
# Check node status (may show NotReady initially)
kubectl get nodes

# Watch for Calico and kube-proxy pods on new nodes
kubectl get pods --all-namespaces --watch

# Verify node is Ready
kubectl get nodes
```

**Repeat the join process for c1-node2 and c1-node3**

---

## Part 4: Working with Your Cluster

### Cluster Inspection

```bash
# Display cluster information
kubectl cluster-info

# List all nodes
kubectl get nodes

# Get detailed node information
kubectl get nodes -o wide

# List pods in default namespace
kubectl get pods

# List system pods
kubectl get pods --namespace kube-system

# Get detailed pod information
kubectl get pods --namespace kube-system -o wide

# List all resources in all namespaces
kubectl get all --all-namespaces

# View available API resources
kubectl api-resources | more

# Filter resources by type
kubectl api-resources | grep pod

# Get detailed resource description
kubectl explain pod | more
kubectl explain pod.spec | more
kubectl explain pod.spec.containers | more

# Describe specific nodes
kubectl describe nodes c1-cp1
kubectl describe nodes c1-node1
```

### Deploying Applications

#### Imperative Deployment

```bash
# Create a deployment imperatively
kubectl create deployment hello-world --image=psk8s.azurecr.io/hello-app:1.0

# Deploy a standalone pod
kubectl run hello-world-pod --image=psk8s.azurecr.io/hello-app:1.0

# Check deployed resources
kubectl get pods
kubectl get pods -o wide

# View deployment details
kubectl get deployment hello-world
kubectl get replicaset
kubectl describe deployment hello-world

# Check pod logs
kubectl logs hello-world-pod

# Execute commands in pod
kubectl exec -it hello-world-pod -- /bin/sh

# Expose deployment as a service
kubectl expose deployment hello-world --port=80 --target-port=8080

# Check service details
kubectl get service hello-world
kubectl describe service hello-world

# Access service (replace with actual IP and port)
curl http://CLUSTER-IP:80

# Scale deployment
kubectl scale deployment hello-world --replicas=5
```

#### Declarative Deployment

```bash
# Generate deployment YAML
kubectl create deployment hello-world \
    --image=psk8s.azurecr.io/hello-app:1.0 \
    --dry-run=client -o yaml > deployment.yaml

# Review generated YAML
cat deployment.yaml

# Apply deployment
kubectl apply -f deployment.yaml

# Generate service YAML
kubectl expose deployment hello-world \
    --port=80 --target-port=8080 \
    --dry-run=client -o yaml > service.yaml

# Apply service
kubectl apply -f service.yaml

# Verify deployment
kubectl get all
```

#### Scaling Applications

```bash
# Edit deployment file
vi deployment.yaml
# Change replicas: 1 to replicas: 20

# Apply changes
kubectl apply -f deployment.yaml

# Verify scaling
kubectl get deployment hello-world
kubectl get pods

# Alternative: Edit resources directly
kubectl edit deployment hello-world
```

#### Cleanup

```bash
# Delete resources
kubectl delete deployment hello-world
kubectl delete service hello-world
kubectl delete pod hello-world-pod

# Verify deletion
kubectl get all
```

### Imperative vs Declarative Deployments

**Imperative (Command-based):**
- Quick for testing and development
- Commands are not tracked in version control
- Example: `kubectl create deployment`

**Declarative (YAML-based):**
- Recommended for production
- Configuration as code
- Version controlled
- Repeatable and auditable
- Example: `kubectl apply -f deployment.yaml`

---

## Part 5: Cloud Deployments

### Azure Kubernetes Service (AKS)

```bash
# Install Azure CLI
AZ_REPO=$(lsb_release -cs)
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | sudo tee /etc/apt/sources.list.d/azure-cli.list

curl -sL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null

sudo apt-get update
sudo apt-get install azure-cli

# Login to Azure
az login
az account set --subscription "Demonstration Account"

# Create resource group
az group create --name "Kubernetes-Cloud" --location centralus

# Check available versions
az aks get-versions --location centralus -o table

# Create AKS cluster
az aks create \
    --resource-group "Kubernetes-Cloud" \
    --generate-ssh-keys \
    --name CSCluster \
    --node-count 3

# Get cluster credentials
az aks get-credentials --resource-group "Kubernetes-Cloud" --name CSCluster

# Switch context
kubectl config use-context CSCluster

# Verify connection
kubectl get nodes
kubectl get pods --all-namespaces

# Delete cluster (when done)
# az aks delete --resource-group "Kubernetes-Cloud" --name CSCluster
```

### Google Kubernetes Engine (GKE)

```bash
# Install Google Cloud SDK
CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"
echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo apt-get update
sudo apt-get install google-cloud-sdk

# Initialize gcloud
gcloud init --console-only

# Create project
gcloud projects create psdemogke-1 --name="Kubernetes-Cloud"

# Set project context
gcloud config set project psdemogke-1

# Enable billing (via console)
# Visit https://console.cloud.google.com and enable billing for the project

# Create GKE cluster
gcloud container clusters create cscluster \
    --region us-central1-a \
    --no-enable-basic-auth

# Get credentials
gcloud container clusters get-credentials cscluster \
    --zone us-central1-a \
    --project psdemogke-1

# Switch context
kubectl config use-context gke_psdemogke-1_us-central1-a_cscluster

# Verify connection
kubectl get nodes

# Delete cluster (when done)
# gcloud container clusters delete cscluster --zone=us-central1-a
# gcloud projects delete psdemogke-1
```

---

## Daily Operations Commands

Essential commands for everyday Kubernetes operations:

### Cluster Information

```bash
# Get cluster info
kubectl cluster-info

# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Current context
kubectl config current-context
```

### Node Operations

```bash
# List all nodes
kubectl get nodes

# Detailed node view
kubectl get nodes -o wide

# Node details
kubectl describe node <node-name>

# Node resource usage
kubectl top nodes  # Requires metrics-server
```

### Pod Operations

```bash
# List pods (default namespace)
kubectl get pods

# List pods (all namespaces)
kubectl get pods --all-namespaces
kubectl get pods -A  # Short form

# Detailed pod view
kubectl get pods -o wide

# Watch pods in real-time
kubectl get pods --watch

# Pod details
kubectl describe pod <pod-name>

# Pod logs
kubectl logs <pod-name>

# Follow logs
kubectl logs -f <pod-name>

# Previous container logs
kubectl logs <pod-name> --previous

# Multi-container pod logs
kubectl logs <pod-name> -c <container-name>

# Execute command in pod
kubectl exec <pod-name> -- <command>

# Interactive shell
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- /bin/sh
```

### Deployment Operations

```bash
# List deployments
kubectl get deployments

# Deployment details
kubectl describe deployment <deployment-name>

# Scale deployment
kubectl scale deployment <deployment-name> --replicas=5

# Edit deployment
kubectl edit deployment <deployment-name>

# Rollout status
kubectl rollout status deployment/<deployment-name>

# Rollout history
kubectl rollout history deployment/<deployment-name>

# Rollback deployment
kubectl rollout undo deployment/<deployment-name>
```

### Service Operations

```bash
# List services
kubectl get services
kubectl get svc  # Short form

# Service details
kubectl describe service <service-name>

# List endpoints
kubectl get endpoints
```

### Resource Management

```bash
# Get all resources
kubectl get all

# Get all resources (all namespaces)
kubectl get all --all-namespaces

# Get specific resource types
kubectl get pods,services,deployments

# Output in YAML
kubectl get <resource> <name> -o yaml

# Output in JSON
kubectl get <resource> <name> -o json

# Wide output
kubectl get <resource> -o wide
```

### Namespace Operations

```bash
# List namespaces
kubectl get namespaces
kubectl get ns  # Short form

# Create namespace
kubectl create namespace <namespace-name>

# Set default namespace
kubectl config set-context --current --namespace=<namespace-name>

# Work with specific namespace
kubectl get pods -n <namespace-name>
```

### Apply and Create

```bash
# Apply configuration
kubectl apply -f <filename.yaml>

# Apply directory
kubectl apply -f <directory>/

# Create from file
kubectl create -f <filename.yaml>

# Generate YAML (dry-run)
kubectl create deployment <name> --image=<image> --dry-run=client -o yaml
```

### Delete Operations

```bash
# Delete resource
kubectl delete <resource-type> <resource-name>

# Delete from file
kubectl delete -f <filename.yaml>

# Force delete pod
kubectl delete pod <pod-name> --grace-period=0 --force

# Delete all pods in namespace
kubectl delete pods --all -n <namespace-name>
```

### Useful Aliases

```bash
# Add to ~/.bashrc for shortcuts
alias k='kubectl'
alias kg='kubectl get'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias kex='kubectl exec -it'

# Enable kubectl autocomplete
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

---

## Troubleshooting Guide

### Common Issues and Solutions

#### 1. Pods Not Starting

**Check pod status:**
```bash
kubectl get pods
kubectl describe pod <pod-name>
```

**Common causes:**
- Image pull errors
- Insufficient resources
- Configuration errors
- Node issues

**View events:**
```bash
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod-name> | grep -A 10 Events
```

#### 2. Node NotReady Status

**Check node status:**
```bash
kubectl get nodes
kubectl describe node <node-name>
```

**Check kubelet logs:**
```bash
# On the problematic node
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
```

**Check container runtime:**
```bash
sudo systemctl status containerd
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
```

#### 3. Networking Issues

**Check pod networking:**
```bash
kubectl get pods -o wide
kubectl exec -it <pod-name> -- ping <another-pod-ip>
```

**Check Calico pods:**
```bash
kubectl get pods -n kube-system | grep calico
kubectl logs -n kube-system <calico-pod-name>
```

**Verify DNS:**
```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

#### 4. Service Not Accessible

**Check service and endpoints:**
```bash
kubectl get service <service-name>
kubectl get endpoints <service-name>
kubectl describe service <service-name>
```

**Test from within cluster:**
```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- wget -O- <service-name>:<port>
```

#### 5. Cluster Component Issues

**Check system pods:**
```bash
kubectl get pods -n kube-system
kubectl describe pod -n kube-system <pod-name>
```

**Check API server logs:**
```bash
sudo journalctl -u kube-apiserver
kubectl logs -n kube-system kube-apiserver-<node-name>
```

**Check controller manager:**
```bash
kubectl logs -n kube-system kube-controller-manager-<node-name>
```

**Check scheduler:**
```bash
kubectl logs -n kube-system kube-scheduler-<node-name>
```

#### 6. Container Runtime Issues

**Check containerd status:**
```bash
sudo systemctl status containerd
sudo journalctl -u containerd -f
```

**List containers:**
```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a
```

**Inspect container:**
```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock inspect <container-id>
```

**Container logs:**
```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock logs <container-id>
```

#### 7. Configuration Issues

**Verify kubeconfig:**
```bash
kubectl config view
kubectl config current-context
echo $KUBECONFIG
```

**Check certificates:**
```bash
# On control plane
sudo kubeadm certs check-expiration
```

#### 8. Resource Constraints

**Check node resources:**
```bash
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
kubectl top nodes  # Requires metrics-server
kubectl top pods
```

**Check pod resource requests/limits:**
```bash
kubectl describe pod <pod-name> | grep -A 5 "Limits\|Requests"
```

### Diagnostic Commands

```bash
# Get cluster component status (deprecated in newer versions)
kubectl get componentstatuses

# Check API server accessibility
kubectl cluster-info

# View all events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Check for failed pods
kubectl get pods --all-namespaces --field-selector=status.phase!=Running

# View resource usage
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check persistent volume claims
kubectl get pvc --all-namespaces

# Verify RBAC permissions
kubectl auth can-i <verb> <resource>
kubectl auth can-i create pods
```

### Emergency Procedures

**Restart kubelet:**
```bash
sudo systemctl restart kubelet
```

**Restart containerd:**
```bash
sudo systemctl restart containerd
```

**Force delete stuck pod:**
```bash
kubectl delete pod <pod-name> --grace-period=0 --force
```

**Reset node (WARNING: destructive):**
```bash
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube/config
```

**Regenerate certificates:**
```bash
sudo kubeadm certs renew all
```

### Logs Location

```bash
# Kubelet logs
sudo journalctl -u kubelet

# Containerd logs
sudo journalctl -u containerd

# Pod logs directory
/var/log/pods/

# Container logs
/var/log/containers/
```

### Helpful Debug Tools

**Deploy debug pod:**
```bash
kubectl run debug --image=busybox --restart=Never -it --rm -- sh
```

**Network debug pod:**
```bash
kubectl run netdebug --image=nicolaka/netshoot --restart=Never -it --rm -- bash
```

**Check DNS resolution:**
```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

---

## Additional Resources

- **Official Kubernetes Documentation**: https://kubernetes.io/docs/
- **Kubernetes API Reference**: https://kubernetes.io/docs/reference/
- **kubectl Cheat Sheet**: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **Troubleshooting Guide**: https://kubernetes.io/docs/tasks/debug/

---

## Notes

- Always take snapshots before major changes
- Keep your cluster version consistent across all components
- Use declarative configuration (YAML files) for production
- Regular backups of etcd are critical
- Monitor cluster health regularly
- Keep Kubernetes and its components updated

---

**Version**: Kubernetes v1.29.1 | Ubuntu 22.04 | containerd runtime

**Last Updated**: October 2025
