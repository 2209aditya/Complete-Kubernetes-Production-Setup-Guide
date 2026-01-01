# 🚀 Complete Kubernetes Production Setup Guide

> A comprehensive, production-ready Kubernetes guide for DevOps Engineers and SREs. This runbook covers everything from cluster setup to disaster recovery.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.29-326CE5?logo=kubernetes)](https://kubernetes.io/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [System Configuration](#system-configuration)
- [Container Runtime Setup](#container-runtime-setup)
- [Kubernetes Installation](#kubernetes-installation)
- [Cluster Initialization](#cluster-initialization)
- [Networking Setup](#networking-setup)
- [Node Management](#node-management)
- [Workload Deployment](#workload-deployment)
- [Networking Deep Dive](#networking-deep-dive)
- [RBAC Configuration](#rbac-configuration)
- [Security Hardening](#security-hardening)
- [Cluster Upgrades](#cluster-upgrades)
- [Troubleshooting](#troubleshooting)
- [Production Best Practices](#production-best-practices)
- [etcd Backup & Restore](#etcd-backup--restore)
- [Logging Setup (EFK)](#logging-setup-efk)
- [Monitoring (Prometheus + Grafana)](#monitoring-prometheus--grafana)
- [Ingress & TLS](#ingress--tls)
- [Storage Management](#storage-management)
- [Resource Governance](#resource-governance)
- [High Availability](#high-availability)
- [Disaster Recovery](#disaster-recovery)

---

## 🏗️ Architecture Overview

### Cluster Design

```
┌─────────────────────────────────────────────┐
│           Control Plane (Master)            │
│  ┌──────────┐ ┌──────┐ ┌──────────────┐   │
│  │ API      │ │ etcd │ │ Scheduler    │   │
│  │ Server   │ │      │ │ Controller   │   │
│  └──────────┘ └──────┘ └──────────────┘   │
└─────────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
┌───────▼────────┐      ┌──────▼─────────┐
│   Worker-1     │      │   Worker-2     │
│  ┌──────────┐  │      │  ┌──────────┐  │
│  │ Pods     │  │      │  │ Pods     │  │
│  │ kubelet  │  │      │  │ kubelet  │  │
│  │ kube-proxy│ │      │  │ kube-proxy│ │
│  └──────────┘  │      │  └──────────┘  │
└────────────────┘      └────────────────┘
```

### Components Stack

| Component | Technology |
|-----------|-----------|
| **OS** | Ubuntu 20.04/22.04 |
| **Container Runtime** | containerd |
| **Cluster Tool** | kubeadm |
| **CNI Plugin** | Calico |
| **Monitoring** | Prometheus + Grafana |
| **Logging** | EFK Stack |
| **Ingress** | NGINX Ingress Controller |

---

## ✅ Prerequisites

### Hardware Requirements

| Node Type | CPU | RAM | Disk |
|-----------|-----|-----|------|
| **Master** | 2 cores | 4 GB | 30 GB |
| **Worker** | 2 cores | 4 GB | 30 GB |

### Network Requirements

- ✅ Static IP addresses (recommended)
- ✅ All nodes can communicate with each other
- ✅ Internet access for package downloads
- ✅ Ports 6443, 2379-2380, 10250-10252 open

### System Requirements

- ✅ Passwordless sudo access
- ✅ Swap disabled
- ✅ Unique hostname per node
- ✅ Unique MAC address per node

### Set Hostnames

Execute on respective nodes:

```bash
# On master node
hostnamectl set-hostname master-node

# On worker nodes
hostnamectl set-hostname worker-1
hostnamectl set-hostname worker-2
```

### Configure DNS Resolution

Edit `/etc/hosts` on **ALL nodes**:

```bash
cat <<EOF >> /etc/hosts
10.0.0.10 master-node
10.0.0.11 worker-1
10.0.0.12 worker-2
EOF
```

---

## ⚙️ System Configuration

> **Execute on ALL nodes**

### Disable Swap (Mandatory)

```bash
# Disable swap immediately
swapoff -a

# Disable swap permanently
sed -i '/ swap / s/^/#/' /etc/fstab

# Verify
free -h
```

**Why?** Kubernetes scheduler requires swap to be disabled for proper memory management.

### Configure Kernel Modules

```bash
# Load required kernel module
modprobe br_netfilter

# Verify module is loaded
lsmod | grep br_netfilter
```

### Set Kernel Parameters

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Apply settings
sysctl --system

# Verify
sysctl net.bridge.bridge-nf-call-iptables
```

---

## 🐳 Container Runtime Setup

### Install containerd

```bash
# Update package index
apt update

# Install containerd
apt install -y containerd

# Verify installation
containerd --version
```

### Configure containerd

```bash
# Generate default configuration
containerd config default | tee /etc/containerd/config.toml

# Edit configuration
vi /etc/containerd/config.toml
```

Find and set:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

### Restart and Enable containerd

```bash
systemctl restart containerd
systemctl enable containerd
systemctl status containerd
```

---

## 📦 Kubernetes Installation

> **Execute on ALL nodes**

### Add Kubernetes Repository

```bash
# Install prerequisites
apt install -y apt-transport-https ca-certificates curl

# Add Kubernetes GPG key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | apt-key add -

# Add Kubernetes repository
cat <<EOF | tee /etc/apt/sources.list.d/kubernetes.list
deb https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /
EOF
```

### Install Kubernetes Components

```bash
# Update package index
apt update

# Install kubelet, kubeadm, and kubectl
apt install -y kubelet kubeadm kubectl

# Prevent automatic updates
apt-mark hold kubelet kubeadm kubectl

# Verify installation
kubeadm version
kubectl version --client
```

---

## 🎯 Cluster Initialization

> **Execute on MASTER node only**

### Initialize Control Plane

```bash
kubeadm init --pod-network-cidr=192.168.0.0/16
```

**What this creates:**
- API Server (cluster gateway)
- etcd (cluster database)
- Scheduler (pod placement)
- Controller Manager (cluster state reconciliation)

### Configure kubectl Access

```bash
# Setup kubectl config
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Verify cluster access
kubectl get nodes
```

**Expected output:** Master node in `NotReady` state (normal until CNI is installed)

---

## 🌐 Networking Setup

### Install Calico CNI

```bash
# Deploy Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Monitor installation
kubectl get pods -n kube-system -w
```

**Wait until all pods are in `Running` state.**

### Verify Network Setup

```bash
# Check system pods
kubectl get pods -n kube-system

# Check nodes (should now be Ready)
kubectl get nodes
```

---

## 🔗 Node Management

### Join Worker Nodes

**On MASTER node:**

```bash
# Generate join command
kubeadm token create --print-join-command
```

**On WORKER nodes:**

```bash
# Execute the join command (example)
kubeadm join 10.0.0.10:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify Cluster

```bash
# Check all nodes
kubectl get nodes

# Check node details
kubectl get nodes -o wide
```

### Node Maintenance Operations

```bash
# Mark node as unschedulable
kubectl cordon worker-1

# Drain node (evacuate pods)
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# Make node schedulable again
kubectl uncordon worker-1
```

---

## 🚀 Workload Deployment

### Create Deployment

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

Deploy:

```bash
kubectl apply -f deployment.yaml

# Verify deployment
kubectl get deployments
kubectl get pods -l app=nginx
```

### Expose Service

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Deploy:

```bash
kubectl apply -f service.yaml

# Verify service
kubectl get svc
kubectl describe svc nginx-service
```

### Auto Scaling (HPA)

```bash
# Create horizontal pod autoscaler
kubectl autoscale deployment nginx-deployment \
  --cpu-percent=70 \
  --min=2 \
  --max=10

# Verify HPA
kubectl get hpa
```

---

## 🔌 Networking Deep Dive

### Networking Layers

| Layer | Component | Purpose |
|-------|-----------|---------|
| **Pod-to-Pod** | CNI (Calico) | Container networking |
| **Pod-to-Service** | kube-proxy | Service discovery & load balancing |
| **DNS** | CoreDNS | Internal DNS resolution |
| **External Access** | Ingress | HTTP/HTTPS routing |

### Test Pod Networking

```bash
# Create test pods
kubectl run test-1 --image=busybox --command -- sleep 3600
kubectl run test-2 --image=busybox --command -- sleep 3600

# Get pod IPs
kubectl get pods -o wide

# Test connectivity
kubectl exec test-1 -- ping <test-2-ip>
```

---

## 🔐 RBAC Configuration

### Create Role

Create `dev-role.yaml`:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: dev-role
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
```

### Create RoleBinding

Create `dev-rolebinding.yaml`:

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-binding
  namespace: default
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f dev-role.yaml
kubectl apply -f dev-rolebinding.yaml

# Verify
kubectl get roles
kubectl get rolebindings
```

---

## 🛡️ Security Hardening

### Pod Security Standards

Create `secure-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx:1.25
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

### Network Policy

Create `network-policy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Secrets Management

```bash
# Create secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secretpass123

# Use secret in pod
kubectl create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
EOF
```

---

## 🔄 Cluster Upgrades

### Upgrade Master Node

```bash
# Check upgrade availability
kubeadm upgrade plan

# Upgrade kubeadm
apt-mark unhold kubeadm
apt update && apt install -y kubeadm=1.29.x-00
apt-mark hold kubeadm

# Apply upgrade
kubeadm upgrade apply v1.29.x

# Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt update && apt install -y kubelet=1.29.x-00 kubectl=1.29.x-00
apt-mark hold kubelet kubectl

# Restart kubelet
systemctl daemon-reload
systemctl restart kubelet
```

### Upgrade Worker Nodes

```bash
# On master: Drain the worker node
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# On worker: Upgrade packages
apt-mark unhold kubeadm
apt update && apt install -y kubeadm=1.29.x-00
apt-mark hold kubeadm

# Upgrade node
kubeadm upgrade node

# Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt update && apt install -y kubelet=1.29.x-00 kubectl=1.29.x-00
apt-mark hold kubelet kubectl

# Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# On master: Uncordon the node
kubectl uncordon worker-1
```

---

## 🐛 Troubleshooting

### Common Issues

| Issue | Diagnosis | Resolution |
|-------|-----------|------------|
| **Pod CrashLoopBackOff** | `kubectl logs <pod>` | Check application logs |
| **Node NotReady** | `journalctl -u kubelet` | Check kubelet service |
| **ImagePullBackOff** | `kubectl describe pod <pod>` | Verify image name/credentials |
| **DNS Resolution Fails** | `kubectl logs -n kube-system coredns-xxx` | Check CoreDNS pods |
| **Network Issues** | `kubectl get pods -n kube-system` | Verify CNI pods |

### Debug Commands

```bash
# Check pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> --previous

# Describe resources
kubectl describe pod <pod-name>
kubectl describe node <node-name>

# Execute commands in pod
kubectl exec -it <pod-name> -- /bin/bash

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check system pods
kubectl get pods -n kube-system

# Check kubelet logs (on node)
journalctl -u kubelet -f

# Check container runtime (on node)
systemctl status containerd
journalctl -u containerd
```

---

## 🎖️ Production Best Practices

### Essential Practices Checklist

- ✅ **PodDisruptionBudgets** for high availability
- ✅ **ResourceQuotas** for namespace isolation
- ✅ **LimitRanges** for default resource limits
- ✅ **Network Policies** for traffic control
- ✅ **RBAC** with least privilege principle
- ✅ **Pod Security Standards** enforcement
- ✅ **Secrets encryption at rest**
- ✅ **Image scanning** with Trivy
- ✅ **Regular etcd backups**
- ✅ **Monitoring & Alerting** setup
- ✅ **Centralized logging**
- ✅ **Ingress with TLS**
- ✅ **GitOps** with ArgoCD
- ✅ **Multi-AZ deployment**

### Distroless Images

Use Google's distroless images for enhanced security:

```yaml
containers:
- name: app
  image: gcr.io/distroless/nodejs:latest
```

---

## 💾 etcd Backup & Restore

### Why etcd Backup?

etcd stores the entire cluster state. If etcd is lost, the cluster is unrecoverable.

### Backup etcd

```bash
# Install etcdctl
apt install -y etcd-client

# Create backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-*.db
```

### Automate Backups

Create `/usr/local/bin/etcd-backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/backup/etcd"
RETENTION_DAYS=7

mkdir -p $BACKUP_DIR

ETCDCTL_API=3 etcdctl snapshot save \
  $BACKUP_DIR/etcd-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Cleanup old backups
find $BACKUP_DIR -name "etcd-*.db" -mtime +$RETENTION_DAYS -delete
```

Add to crontab:

```bash
# Backup every 6 hours
0 */6 * * * /usr/local/bin/etcd-backup.sh
```

### Restore etcd

```bash
# Stop kubelet
systemctl stop kubelet

# Restore from backup
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20240101-120000.db \
  --data-dir=/var/lib/etcd-from-backup

# Update etcd manifest
vi /etc/kubernetes/manifests/etcd.yaml

# Change data-dir to:
# --data-dir=/var/lib/etcd-from-backup

# Start kubelet
systemctl start kubelet

# Verify cluster
kubectl get nodes
```

---

## 📊 Logging Setup (EFK)

### Architecture

```
Pod Logs → Fluent Bit → Elasticsearch → Kibana
```

### Install Fluent Bit

```bash
# Create namespace
kubectl create namespace logging

# Deploy Fluent Bit
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-service-account.yaml
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role.yaml
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-binding.yaml
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-ds.yaml

# Verify
kubectl get pods -n logging
```

### Why Centralized Logging?

- Pod restarts cause log loss
- Required for Root Cause Analysis (RCA)
- Compliance and audit requirements
- Aggregated view across all containers

---

## 📈 Monitoring (Prometheus + Grafana)

### Install Using Helm

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add Prometheus repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Verify installation
kubectl get pods -n monitoring
```

### Access Grafana

```bash
# Port forward Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

# Get admin password
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

Access at: `http://localhost:3000`

### Key Metrics to Monitor

| Metric | Purpose |
|--------|---------|
| **CPU/Memory Usage** | Resource planning & autoscaling |
| **Pod Restart Count** | Application stability |
| **Node Pressure** | Infrastructure health |
| **API Server Latency** | Control plane performance |
| **etcd Performance** | Cluster database health |
| **Network Traffic** | Bandwidth usage |

---

## 🌐 Ingress & TLS

### Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Create TLS Certificate

```bash
# Generate self-signed certificate (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=app.example.com/O=MyOrg"

# Create Kubernetes secret
kubectl create secret tls app-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

### Secure Ingress Example

Create `ingress-tls.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f ingress-tls.yaml

# Verify
kubectl get ingress
kubectl describe ingress app-ingress
```

---

## 💿 Storage Management

### Persistent Volume (PV)

Create `persistent-volume.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/data"
```

### Persistent Volume Claim (PVC)

Create `persistent-volume-claim.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

### Use in Pod

Create `pod-with-storage.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: pvc-data
```

Deploy:

```bash
kubectl apply -f persistent-volume.yaml
kubectl apply -f persistent-volume-claim.yaml
kubectl apply -f pod-with-storage.yaml

# Verify
kubectl get pv
kubectl get pvc
kubectl describe pod app-with-storage
```

---

## ⚖️ Resource Governance

### ResourceQuota

Create `resource-quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    persistentvolumeclaims: "5"
    services: "10"
    pods: "20"
```

### LimitRange

Create `limit-range.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: development
spec:
  limits:
  - max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

Apply:

```bash
kubectl create namespace development
kubectl apply -f resource-quota.yaml
kubectl apply -f limit-range.yaml

# Verify
kubectl describe quota -n development
kubectl describe limitrange -n development
```

---

## 🎯 High Availability

### PodDisruptionBudget

Create `pod-disruption-budget.yaml`:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: critical-app
```

Apply:

```bash
kubectl apply -f pod-disruption-budget.yaml

# Verify
kubectl get pdb
kubectl describe pdb nginx-pdb
```

**Benefits:**
- Ensures minimum availability during voluntary disruptions
- Protects against cascading failures
- Safe node maintenance and upgrades

---

## 🚨 Disaster Recovery

### Failure Scenarios & Actions

| Failure Type | Detection | Action | Recovery Time |
|--------------|-----------|--------|---------------|
| **Pod Crash** | Container restart count | Check logs, fix code | Minutes |
| **Node Failure** | Node NotReady status | Pods reschedule automatically | 5-10 minutes |
| **AZ Failure** | Multiple node failures | Traffic routes to healthy AZ | Seconds |
| **etcd Corruption** | API server errors | Restore from backup | 10-30 minutes |
| **Control Plane Failure** | API unavailable | HA setup or rebuild | Variable |
| **Complete Cluster Loss** | Total failure | Rebuild + restore etcd | Hours |

### Disaster Recovery Checklist

- ✅ Regular etcd backups (automated)
- ✅ Backup storage in different region
- ✅ Documented restore procedure
- ✅ Multi-AZ deployment
- ✅ Control plane HA (3+ masters)
- ✅ GitOps for declarative config
- ✅ Disaster recovery drills (quarterly)
- ✅ RTO/RPO defined and tested

---

## 🔍 Advanced Troubleshooting Flow

### Systematic Debugging Approach

```
1. IDENTIFY
   ↓
   What's broken? (Pod/Node/Network/Storage/API)
   ↓
2. COLLECT DATA
   ↓
   Logs, Events, Metrics, Describe output
   ↓
3. ANALYZE
   ↓
   Find root cause
   ↓
4. REMEDIATE
   ↓
   Fix the issue
   ↓
5. VERIFY
   ↓
   Confirm resolution
   ↓
6. DOCUMENT
   ↓
   Update runbook
```

### Issue-Specific Debugging

#### Pod Issues

```bash
# Get pod status
kubectl get pod <pod-name> -o wide

# Check events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# Check resource usage
kubectl top pod <pod-name>

# Execute into pod
kubectl exec -it <pod-name> -- /bin/sh
```

#### Node Issues

```bash
# Check node status
kubectl get nodes -o wide
kubectl describe node <node-name>

# Check kubelet (on node)
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# Check container runtime (on node)
sudo systemctl status containerd
sudo crictl ps
```

#### Network Issues

```bash
# Check CNI pods
kubectl get pods -n kube-system | grep calico

# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system coredns-xxx

# Test DNS resolution
kubectl run test --image=busybox:1.28 --rm -it -- nslookup kubernetes.default

# Check services
kubectl get svc --all-namespaces
kubectl get endpoints
```

####
