Perfect! Now let's create the documentation. I'll start with **GitHub** first, then **Medium**.

---

# **PART 1: GITHUB README.md**

This is the technical documentation that goes in your GitHub repository.

---

```markdown
# Kubernetes Admission Controller Security Lab

A hands-on project demonstrating policy-as-code security enforcement in Kubernetes using Kyverno admission controllers and a custom external validation webhook.

## 🎯 Project Overview

This project implements a multi-layered security enforcement system for Kubernetes that:

- Blocks insecure pod configurations using Kyverno policies
- Validates container images against an external allowlist via a Python webhook
- Demonstrates shift-left security practices in cloud-native environments
- Runs entirely on a local development machine using `kind` (Kubernetes in Docker)

## 🏗️ Architecture

```
Developer → kubectl → Kubernetes API Server
                              ↓
                        Kyverno Admission Controller
                              ↓
                    ┌─────────┴─────────┐
                    ↓                   ↓
            Built-in Policies    External Webhook Policy
                    ↓                   ↓
            ✓ No :latest tags    Python Flask API
            ✓ No root containers      ↓
            ✓ No hostNetwork    Image Allowlist Check
                    ↓                   ↓
                    └─────────┬─────────┘
                              ↓
                    ALLOW or DENY pod creation
```

## 🛡️ Security Policies Implemented

### 1. **Disallow Latest Tag Policy**
- **Purpose:** Prevents deployment of containers using `:latest` or untagged images
- **Rationale:** Ensures version pinning for reproducible deployments and rollback capability
- **Enforcement:** Blocks pods with images lacking specific version tags

### 2. **Disallow Root Containers Policy**
- **Purpose:** Requires all containers to run as non-root users
- **Rationale:** Implements principle of least privilege; limits blast radius of container compromise
- **Enforcement:** Requires `securityContext.runAsNonRoot: true` on all containers

### 3. **Disallow Host Network Policy**
- **Purpose:** Prevents containers from sharing the host's network namespace
- **Rationale:** Maintains network isolation; prevents containers from sniffing host traffic
- **Enforcement:** Blocks pods with `hostNetwork: true`

### 4. **External Image Allowlist Policy**
- **Purpose:** Validates container images against an approved list maintained by security team
- **Rationale:** Centralized governance; enables dynamic policy updates without redeploying Kyverno
- **Enforcement:** Calls Python webhook API for image approval

## 🧰 Tech Stack

- **Kubernetes:** v1.35.0 (via kind)
- **Kyverno:** v1.11.0 (admission controller)
- **Python:** 3.9 (webhook server)
- **Flask:** Lightweight web framework for webhook API
- **Docker:** Container runtime and image builder
- **kind:** Kubernetes in Docker (local cluster)

## 📋 Prerequisites

- macOS (or Linux/Windows with adjustments)
- Docker Desktop installed and running
- `kind` CLI tool
- `kubectl` CLI tool
- Basic understanding of Kubernetes concepts

## 🚀 Setup Instructions

### Step 1: Create Local Kubernetes Cluster

```bash
# Create cluster configuration
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# Create cluster
kind create cluster --config kind-config.yaml --name security-lab

# Verify cluster
kubectl cluster-info --context kind-security-lab
```

### Step 2: Install Kyverno

```bash
# Install Kyverno admission controller
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.11.0/install.yaml

# Wait for Kyverno pods to be ready
kubectl wait --for=condition=ready pod --namespace kyverno --all --timeout=300s

# Verify installation
kubectl get pods -n kyverno
```

### Step 3: Deploy Built-in Security Policies

**Disallow Latest Tag:**
```bash
kubectl apply -f disallow-latest-tag.yaml
```

**Disallow Root Containers:**
```bash
kubectl apply -f disallow-root-container.yaml
```

**Disallow Host Network:**
```bash
kubectl apply -f disallow-host-network.yaml
```

### Step 4: Build and Deploy External Webhook

```bash
# Navigate to webhook directory
cd webhook

# Build Docker image
docker build -t image-allowlist-webhook:v1.1 .

# Load image into kind cluster
kind load docker-image image-allowlist-webhook:v1.1 --name security-lab

# Return to project root
cd ..

# Deploy webhook
kubectl apply -f webhook-deployment.yaml

# Verify webhook is running
kubectl get pods -l app=allowlist-webhook
```

### Step 5: Apply External Allowlist Policy

```bash
kubectl apply -f external-allowlist-policy.yaml
```

## 🧪 Testing

### Test 1: Block :latest Tag

```bash
kubectl run bad-pod --image nginx:latest
```

**Expected:** ❌ Blocked with message: "Image tag 'latest' is not allowed. Use a specific version tag."

### Test 2: Block Root Container

```bash
kubectl run root-test --image nginx:1.25
```

**Expected:** ❌ Blocked with message: "Running as root is not allowed. Set securityContext.runAsNonRoot to true"

### Test 3: Block Host Network

```bash
kubectl run hostnet-test --image nginx:1.25 --overrides='
{
  "spec": {
    "hostNetwork": true,
    "containers": [{
      "name": "hostnet-test",
      "image": "nginx:1.25",
      "securityContext": {
        "runAsNonRoot": true,
        "runAsUser": 1000
      }
    }]
  }
}'
```

**Expected:** ❌ Blocked with message: "Using hostNetwork is not allowed for security reasons"

### Test 4: Block Unapproved Image (External Webhook)

```bash
kubectl run blocked-test --image nginx:1.26 --overrides='
{
  "spec": {
    "containers": [{
      "name": "blocked-test",
      "image": "nginx:1.26",
      "securityContext": {
        "runAsNonRoot": true,
        "runAsUser": 1000
      }
    }]
  }
}'
```

**Expected:** ❌ Blocked with message: "Image 'nginx:1.26' is not in the approved allowlist. Allowed: nginx:1.25, alpine:3.18, redis:7.0"

### Test 5: Allow Approved Image (External Webhook)

```bash
kubectl run approved-test --image alpine:3.18 --overrides='
{
  "spec": {
    "containers": [{
      "name": "approved-test",
      "image": "alpine:3.18",
      "command": ["sleep", "3600"],
      "securityContext": {
        "runAsNonRoot": true,
        "runAsUser": 1000
      }
    }]
  }
}'
```

**Expected:** ✅ Pod created successfully

## 📁 Project Structure

```
.
├── kind-config.yaml                    # Kubernetes cluster configuration
├── disallow-latest-tag.yaml           # Policy: Block :latest tags
├── disallow-root-container.yaml       # Policy: Require non-root
├── disallow-host-network.yaml         # Policy: Block hostNetwork
├── external-allowlist-policy.yaml     # Policy: External webhook validation
├── webhook-deployment.yaml            # Webhook deployment & service
└── webhook/
    ├── server.py                      # Flask webhook server
    ├── requirements.txt               # Python dependencies
    └── Dockerfile                     # Container image definition
```

## 🔧 Customizing the Allowlist

To modify approved images, edit `webhook/server.py`:

```python
ALLOWED_IMAGES = [
    "nginx:1.25", 
    "alpine:3.18", 
    "redis:7.0",
    # Add your approved images here
]
```

Then rebuild and redeploy:

```bash
cd webhook
docker build -t image-allowlist-webhook:v1.2 .
kind load docker-image image-allowlist-webhook:v1.2 --name security-lab
cd ..

# Update webhook-deployment.yaml to use v1.2, then:
kubectl apply -f webhook-deployment.yaml
```

## 🧹 Cleanup

```bash
# Delete the cluster
kind delete cluster --name security-lab

# Stop the local webhook container (if running)
docker stop webhook
docker rm webhook
```

## 🎓 Key Learnings

### Security Concepts Demonstrated

1. **Admission Control:** Pre-deployment validation catches issues before they reach production
2. **Policy-as-Code:** Security rules version-controlled and auditable
3. **Defense in Depth:** Multiple policy layers (built-in + external validation)
4. **Shift-Left Security:** Developers get immediate feedback on security violations
5. **Principle of Least Privilege:** Non-root execution and network isolation by default

### Technical Skills Applied

- Kubernetes admission webhooks and admission controllers
- Kyverno policy engine and pattern matching
- Python Flask API development
- Docker containerization and multi-stage builds
- Kubernetes networking and service discovery
- YAML configuration and policy definition

### Challenges Encountered

**Challenge 1: Kyverno apiCall Content-Type Headers**

The initial webhook implementation failed because Kyverno's `apiCall` context feature doesn't automatically set `Content-Type: application/json` headers. Flask's strict content-type checking rejected these requests.

**Solution:** Modified the webhook to accept requests without strict content-type validation:

```python
if request.is_json:
    body = request.json
else:
    body = json.loads(request.data.decode('utf-8'))
```

**Challenge 2: Policy Blocking Its Own Webhook**

When updating the webhook deployment to v1.1, the external allowlist policy blocked the update because it tried to validate itself, creating a circular dependency.

**Solution:** Temporarily deleted the external policy, updated the webhook, then re-applied the policy:

```bash
kubectl delete clusterpolicy external-allowlist-check
kubectl apply -f webhook-deployment.yaml
kubectl apply -f external-allowlist-policy.yaml
```

## 🔗 References

- [Kyverno Documentation](https://kyverno.io/docs/)
- [Kubernetes Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [kind Documentation](https://kind.sigs.k8s.io/)
- [MITRE ATT&CK - Container Techniques](https://attack.mitre.org/matrices/enterprise/containers/)

## 📝 License

This project is for educational purposes. Feel free to use and modify for your learning.

## 👤 Author

**Zeteo**  
[zeteosec.com](https://zeteosec.com) | [LinkedIn](www.linkedin.com/in/paulayegbusi) | [GitHub](#)

---

**Built with ☕ for hands-on cloud security learning**
```

---

## **Save this as `README.md` in your GitHub repo**

Now let me create the **Medium article** (portfolio version with narrative). Ready?
