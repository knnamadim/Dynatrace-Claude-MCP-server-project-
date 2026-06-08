# Infrastructure Setup Guide

Complete step-by-step setup for the EasyTrade + Dynatrace observability demo environment.

---

## Overview

| Component | Details |
|---|---|
| Cloud | AWS EC2 |
| Instance type | t3.2xlarge (8 vCPU, 32GB RAM) |
| OS | Ubuntu 24.04 LTS |
| Storage | 50 GiB root volume |
| Kubernetes | k3s (single-node) |
| Application | EasyTrade via Helm |
| Observability | Dynatrace Operator + OneAgent |

---

## Step 1 — Provision AWS EC2 Instance

**Instance configuration:**
- Type: `t3.2xlarge` — EasyTrade pulls 15+ container images; anything smaller will struggle
- AMI: Ubuntu Server 24.04 LTS
- Storage: 50 GiB root volume (default 8 GiB is too small for the container images)

**Security group inbound rules:**

| Type | Port | Source | Why |
|---|---|---|---|
| SSH | 22 | Your IP only | Secure shell access |
| HTTP | 80 | 0.0.0.0/0 | EasyTrade frontend |
| Custom TCP | 31424 | 0.0.0.0/0 | EasyTrade NodePort (added later) |
| Custom TCP | 6443 | Your IP | Kubernetes API (optional — only if running kubectl locally) |

> **Important:** Lock port 22 to your IP, not 0.0.0.0/0.

**Assign an Elastic IP** after launching the instance. Dynatrace RUM configuration and browser monitors require a stable IP that does not change on reboot.

---

## Step 2 — Connect and Configure the Instance

```bash
# Set permissions on your .pem key
chmod 400 "Dyna.pem"

# SSH in
ssh -i "Dyna.pem" ubuntu@<your-elastic-ip>

# Update the system
sudo apt-get update && sudo apt-get upgrade -y
```

---

## Step 3 — Install k3s (Kubernetes)

```bash
# Install k3s — disable traefik to avoid port 80 conflicts with EasyTrade
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
```

```bash
# Configure kubectl access
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Make config permanently available across sessions
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bashrc
source ~/.bashrc

# Set file permissions
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

# Verify cluster is running
kubectl get nodes
```

---

## Step 4 — Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## Step 5 — Sign Up for Dynatrace SaaS

1. Go to [dynatrace.com/signup](https://www.dynatrace.com/signup/) and start a free trial
2. Note your tenant ID from the URL: `https://<TENANT_ID>.apps.dynatrace.com`

---

## Step 6 — Install the Dynatrace Operator

In Dynatrace, navigate to **Kubernetes onboarding** and select:
- Distribution: Other
- Monitoring type: Kubernetes platform monitoring + Full Stack Observability
- Enable: Log management, analytics, and telemetry options
- Cluster name: `easytrade-demo`
- Cluster size: < 100 pods (small)

Generate two tokens from the onboarding wizard:
- **Dynatrace Operator token**
- **Data ingestion token**

Download the generated `dynakube.yaml` file.

**Copy dynakube.yaml to your EC2 instance:**
```bash
# Run this from your local machine, not the EC2 instance
scp -i Dyna.pem dynakube.yaml ubuntu@<your-elastic-ip>:~/
```

**Back in your SSH session, deploy the operator:**
```bash
helm install dynatrace-operator oci://public.ecr.aws/dynatrace/dynatrace-operator \
  --create-namespace \
  --namespace dynatrace \
  --atomic

kubectl apply -f dynakube.yaml
```

**Verify OneAgent pods are running (wait ~5 minutes):**
```bash
kubectl get pods -n dynatrace
kubectl get dynakube -n dynatrace
```

You should see pods for:
- `dynatrace-operator`
- `easytrade-demo` (OneAgent DaemonSet — instruments every node)
- `easytrade-demo-agents-activegate` (routing/proxy layer)

---

## Step 7 — Deploy EasyTrade

```bash
helm install easytrade oci://europe-docker.pkg.dev/dynatrace-demoability/helm/easytrade \
  --create-namespace \
  --namespace easytrade
```

**Wait ~5 minutes then verify all pods are running:**
```bash
kubectl get pods -n easytrade -w
```

---

## Step 8 — Expose the Frontend

EasyTrade deploys with a ClusterIP by default. Patch it to NodePort:

```bash
kubectl -n easytrade patch svc easytrade-frontendreverseproxy \
  -p '{"spec": {"type": "NodePort"}}'

# Get the assigned NodePort
kubectl -n easytrade get svc easytrade-frontendreverseproxy
```

Note the `3XXXX` port number in the output.

**Add this port to your AWS security group inbound rules:**
- Type: Custom TCP
- Port: `<your-nodeport>` (e.g. 31424)
- Source: 0.0.0.0/0

**Access EasyTrade in your browser:**
```
http://<your-elastic-ip>:<nodeport>
```

**Demo login credentials:**
- Username: `james_norton`
- Password: `pass_james_123`

---

## Step 9 — Install Monaco CLI (Configuration as Code)

```bash
# Download Monaco
curl -L https://github.com/Dynatrace/dynatrace-configuration-as-code/releases/latest/download/monaco-linux-amd64 \
  -o monaco-linux-amd64

# Verify checksum
curl -L https://github.com/Dynatrace/dynatrace-configuration-as-code/releases/latest/download/monaco-linux-amd64.sha256 \
  -o monaco.sha256
shasum -c monaco.sha256
# Expected output: monaco-linux-amd64: OK

# Install
mv monaco-linux-amd64 monaco
chmod +x monaco
sudo mv monaco /usr/local/bin/
monaco version
```

**Clone the EasyTrade repo (contains Monaco configs):**
```bash
cd ~
git clone https://github.com/Dynatrace/easytrade
cd easytrade/monaco
```

**Create your .env file:**
```bash
cp .env.example .env
nano .env
```

Add your values:
```bash
export TENANT_URL=https://<your-tenant-id>.apps.dynatrace.com
export TENANT_TOKEN=<your-api-token>
export CLIENT_ID=<your-oauth-client-id>
export CLIENT_SECRET=<your-oauth-client-secret>
export DOMAIN=http://<your-elastic-ip>:<nodeport>
```

> **Note:** You need to re-run `source .env` every time you open a new terminal session.

**Load and verify:**
```bash
source .env
echo $TENANT_URL
echo $DOMAIN
```

**Deploy Monaco configurations:**
```bash
monaco deploy manifest.yaml -e dev -p application
monaco deploy manifest.yaml -e dev -p bizevents
monaco deploy manifest.yaml -e dev -p custom-service
```

---

## Step 10 — Configure MCP Server in Claude Desktop

See the [MCP Setup section in README.md](../README.md#mcp-server-setup).

---

## Useful Commands Reference

```bash
# SSH into instance
ssh -i "Dyna.pem" ubuntu@<elastic-ip>

# Fix kubeconfig permissions after reboot
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

# Check cluster nodes
kubectl get nodes

# Check all pods
kubectl get pods -A

# Check Dynatrace components
kubectl get pods -n dynatrace

# Check EasyTrade pods
kubectl get pods -n easytrade

# Restart kubelet if nodes not ready
sudo systemctl restart kubelet

# Port-forward feature flag service for problem injection
kubectl -n easytrade port-forward svc/easytrade-feature-flag-service 8080:8080
```
