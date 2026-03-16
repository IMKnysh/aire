# AIRE - AI Runtime Environment

A bare-metal Kubernetes setup with [agentgateway](https://agentgateway.dev) and [kagent](https://kagent.dev) for running AI agents on your own infrastructure.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [1. Kubernetes Cluster Setup](#1-kubernetes-cluster-setup)
  - [1.1 Initialize the cluster](#11-initialize-the-cluster)
  - [1.2 Install CNI (Flannel)](#12-install-cni-flannel)
  - [1.3 Single-node setup](#13-single-node-setup-control-plane--worker-on-same-vm)
  - [1.4 Install MetalLB](#14-install-metallb-bare-metal-load-balancer)
- [2. Install agentgateway](#2-install-agentgateway)
  - [2.1 Install Kubernetes Gateway API CRDs](#21-install-kubernetes-gateway-api-crds)
  - [2.2 Install agentgateway CRDs](#22-install-agentgateway-crds)
  - [2.3 Install agentgateway](#23-install-agentgateway)
  - [2.4 Deploy the agentgateway proxy](#24-deploy-the-agentgateway-proxy)
  - [2.5 Configure an LLM backend](#25-configure-an-llm-backend-google-gemini)
- [3. Install kagent](#3-install-kagent)
  - [3.1 Install kagent CRDs](#31-install-kagent-crds)
  - [3.2 Configure kagent](#32-configure-kagent)
  - [3.3 Install kagent](#33-install-kagent)
  - [3.4 Configure LLM provider credentials](#34-configure-llm-provider-credentials)
  - [3.5 Accessing the UI](#35-accessing-the-ui)
  - [3.6 Useful Commands](#36-useful-commands)
  - [3.7 Troubleshooting](#37-troubleshooting)

---

## Prerequisites

- A VM or bare-metal server with `kubeadm`, `kubectl`, and `helm` installed
- Docker or containerd as the container runtime
- Helm 3.x
- Access to a supported LLM provider (Google Gemini, AWS Bedrock, or OpenAI)

---

## 1. Kubernetes Cluster Setup

### 1.1 Initialize the cluster

```bash
kubeadm init --control-plane-endpoint ec2-100-52-110-239.compute-1.amazonaws.com
```

### 1.2 Install CNI (Flannel)

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

If nodes are stuck in `NotReady`, ensure the controller manager has CIDR allocation enabled.
Add the following flags to `/etc/kubernetes/manifests/kube-controller-manager.yaml`:

```yaml
- --allocate-node-cidrs=true
- --cluster-cidr=10.244.0.0/16
```

Then restart kubelet:

```bash
sudo systemctl restart kubelet
```

### 1.3 Single-node setup (control plane + worker on same VM)

Remove the control-plane taint and load balancer exclusion label so workloads can be scheduled:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl label nodes --all node.kubernetes.io/exclude-from-external-load-balancers-
```

### 1.4 Install MetalLB (bare-metal load balancer)

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

Configure the IP address pool:

```bash
kubectl apply -f network/ip-pool.yaml
```

---

## 2. Install agentgateway

agentgateway acts as an AI-aware proxy that routes traffic to LLM backends.

### 2.1 Install Kubernetes Gateway API CRDs

```bash
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
```

### 2.2 Install agentgateway CRDs

```bash
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v1.0.0-rc.1 \
  agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds
```

### 2.3 Install agentgateway

```bash
helm upgrade -i -n agentgateway-system \
  agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
  --version v1.0.0-rc.1
```

### 2.4 Deploy the agentgateway proxy

```bash
kubectl apply -f helm/agentgateway/agentgateway-proxy.yaml
```

Verify the deployment:

```bash
kubectl get gateway agentgateway-proxy -n agentgateway-system
kubectl get deployment agentgateway-proxy -n agentgateway-system
```

### 2.5 Configure an LLM backend (Google Gemini)

Create the API key secret (update the value with your key first):

```bash
kubectl apply -f helm/agentgateway/google-secret.yaml
```

Register the Gemini backend:

```bash
kubectl apply -f helm/agentgateway/gemini-backend.yaml
```

Route traffic to the backend:

```bash
kubectl apply -f helm/agentgateway/http-route.yaml
```

---

## 3. Install kagent

kagent provides a set of Kubernetes-native AI agents managed via Helm.

### 3.1 Install kagent CRDs

```bash
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --namespace kagent --create-namespace
```

### 3.2 Configure kagent

Edit `helm/kagent/values.yaml` to enable or disable agents and set your default provider:

```yaml
agents:
  k8s-agent:
    enabled: true
  helm-agent:
    enabled: true
  # ... other agents

providers:
  default: gemini
  gemini:
    apiKeySecretRef: kagent-gemini
```

### 3.3 Install kagent

```bash
helm install -f helm/kagent/values.yaml kagent \
  oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --namespace kagent
```

### 3.4 Configure LLM provider credentials

Apply the secret and model config for your chosen provider:

**Google Gemini**
```bash
kubectl apply -f helm/kagent/gemini-secret.yaml
```

**AWS Bedrock**
```bash
kubectl apply -f kagent/bedrock-secret.yaml
kubectl apply -f kagent/bedrock-model-config.yaml
```

**OpenAI**
```bash
kubectl apply -f kagent/openai-secret.yaml
kubectl apply -f kagent/openai-model-config.yaml
```

---

### 3.5 Accessing the UI

**Option 1 — localhost only:**

```bash
kubectl -n kagent port-forward service/kagent-ui 8080:8080
```

**Option 2 — accessible from external IP:**

```bash
kubectl port-forward -n kagent svc/kagent-ui --address 0.0.0.0 8080:8080
```

Then open: `http://localhost:8080` or `http://<CLUSTER_EXTERNAL_IP>:8080`

---

### 3.6 Useful Commands

```bash
# List all kagent resources
kubectl -n kagent get agents,modelconfigs,toolservers,memories

# List agents only
kubectl -n kagent get agents

# Stream kagent logs
kubectl -n kagent logs -l app.kubernetes.io/name=kagent -f

# Stream controller logs
kubectl -n kagent logs -l app.kubernetes.io/component=controller -f
```

---

### 3.7 Troubleshooting

| Symptom | Command |
|---|---|
| Pods not starting | `kubectl -n kagent get pods` |
| Unexpected behavior | `kubectl -n kagent get events --sort-by='.lastTimestamp'` |
| Controller errors | `kubectl -n kagent logs -l app.kubernetes.io/component=controller -f` |
| Node not ready | `kubectl describe node <node-name>` |
| Gateway not routing | `kubectl describe gateway agentgateway-proxy -n agentgateway-system` |
