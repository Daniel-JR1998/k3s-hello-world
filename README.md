# k3s Hello World – SRE Coding Challenge

A complete solution demonstrating automated k3s installation, Nginx "Hello World" deployment, and GitHub Actions CI/CD.

## Repository Structure

```
k3s-hello-world/
├── ansible/
│   ├── install-k3s.yml      # Ansible playbook to install k3s
│   ├── inventory.ini        # Target server inventory
│   └── requirements.yml     # Ansible Galaxy collections
├── k8s/
│   ├── 00-namespace.yaml    # Kubernetes namespace
│   ├── 01-deployment.yaml   # Nginx deployment (2 replicas)
│   └── 02-service.yaml      # NodePort service (port 30080)
├── nginx/
│   ├── Dockerfile           # Custom Nginx image (non-root, port 8080)
│   ├── nginx.conf           # Nginx configuration
│   └── index.html           # Hello World HTML page
└── .github/
    └── workflows/
        └── deploy.yml       # GitHub Actions CI/CD pipeline
```

---

## Part 1 – Install k3s with Ansible

### Prerequisites

- Ansible 2.12+ installed locally
- SSH access to a Linux server (Ubuntu 20.04/22.04 or RHEL/CentOS 8+)
- Python 3 on the target server

### Steps

```bash
# 1. Install required Ansible collections
cd ansible/
ansible-galaxy collection install -r requirements.yml

# 2. Edit inventory with your server details
vim inventory.ini   # set ansible_host, ansible_user, key path

# 3. Run the playbook
ansible-playbook -i inventory.ini install-k3s.yml

# 4. Verify the cluster (on the server)
kubectl get nodes
kubectl cluster-info
```

### What the playbook does

1. Installs dependencies (`curl`, `wget`, `git`, etc.)
2. Disables swap (required by Kubernetes)
3. Loads required kernel modules (`overlay`, `br_netfilter`)
4. Configures sysctl networking parameters
5. Downloads and runs the official k3s install script (latest stable)
6. Enables and starts the `k3s` systemd service
7. Waits for the node to reach `Ready` state
8. Copies kubeconfig to `~/.kube/config`

---

## Part 2 – Kubernetes Manifests

The manifests in `k8s/` deploy Nginx with production-grade settings:

| Feature | Detail |
|---|---|
| Replicas | 2 (rolling update strategy) |
| Image | Custom non-root Nginx on port 8080 |
| Liveness probe | `/healthz` endpoint |
| Readiness probe | `/healthz` endpoint |
| Resource limits | CPU: 200m, RAM: 128Mi |
| Security | Non-root user, capabilities dropped |
| Access | NodePort 30080 → `http://<NODE_IP>:30080` |

### Manual deployment (without CI/CD)

```bash
export DOCKER_IMAGE=yourdockerhub/hello-world-nginx
export IMAGE_TAG=latest

# Apply manifests (envsubst fills in image variables)
for f in k8s/*.yaml; do
  envsubst < "$f" | kubectl apply -f -
done

# Check rollout
kubectl rollout status deployment/hello-world-nginx -n hello-world
kubectl get pods -n hello-world
```

---

## Part 3 – GitHub Actions CI/CD Pipeline

### Required Repository Secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret | Description |
|---|---|
| `DOCKER_USERNAME` | Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub access token |
| `K3S_HOST` | Public IP or hostname of your k3s node |
| `K3S_SSH_USER` | SSH username (e.g. `ubuntu`) |
| `K3S_KUBECONFIG` | Contents of `/etc/rancher/k3s/k3s.yaml` on the server |

### Pipeline Overview

```
Push to main
    │
    ▼
┌─────────┐     ┌──────────────────┐     ┌────────────────┐
│  Lint   │────▶│  Build & Push    │────▶│  Deploy to k3s │
│ kubeval │     │  Docker → Hub    │     │  kubectl apply │
└─────────┘     └──────────────────┘     └────────────────┘
                                              │
                                              ▼
                                         Smoke test
                                      HTTP 200 on :30080
```

### Trigger

The pipeline fires automatically when files in `nginx/`, `k8s/`, or the workflow itself are pushed to `main`.

### Getting the kubeconfig secret

```bash
# On your k3s server:
sudo cat /etc/rancher/k3s/k3s.yaml
# Copy the output and store as K3S_KUBECONFIG secret
```

---

## Accessing the Application

Once deployed, open a browser and navigate to:

```
http://<YOUR_K3S_NODE_IP>:30080
```

You should see the styled "Hello World" page.

---

## Security Notes

- The kubeconfig is stored as a GitHub encrypted secret and never logged
- The Docker image runs as a non-root user (UID 1001)
- Container capabilities are fully dropped
- The Nginx version string is hidden (`server_tokens off`)
- Secrets are never embedded in manifests or source code
