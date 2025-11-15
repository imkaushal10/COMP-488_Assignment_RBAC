# Kubernetes RBAC (Role-Based Access Control) Lab (COMP-488)

A comprehensive hands-on lab for understanding and implementing Kubernetes Role-Based Access Control (RBAC).

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Part 1: Understanding the Basics](#part-1-understanding-the-basics)
  - [Task 1.1: Inspect Default Service Accounts](#task-11-inspect-default-service-accounts)
  - [Task 1.2: Test Current Permissions](#task-12-test-current-permissions)
- [Part 2: Creating Namespace-Scoped Access](#part-2-creating-namespace-scoped-access)
- [Part 3: Cluster-Wide Access](#part-3-cluster-wide-access)
- [Part 4: Troubleshooting RBAC Issues](#part-4-troubleshooting-rbac-issues)
- [Part 5: Real-World Scenario](#part-5-real-world-scenario)
- [Bonus-Challenege: Label Bassed Access Control](#bonus-challenge-label-based-access-control)
- [Project Structure](#project-structure)

---

## Overview

This lab demonstrates Kubernetes RBAC concepts, from basic service account inspection to creating custom roles and bindings, implementing the principle of least privilege for secure cluster management.

## Prerequisites

- Kubernetes cluster (local or cloud-based)
- `kubectl` CLI tool installed and configured
- Basic understanding of Kubernetes concepts (Pods, Namespaces)
- PowerShell or Bash terminal

---

## PART 1: Understanding the Basics

### Task 1.1: Inspect Default Service Accounts

#### Command 1: List all service accounts

```powershell
kubectl get serviceaccounts -n default
```

**Expected Output:**

``` bash
NAME      SECRETS   AGE
default   1         XXd
```

---

#### Command 2: Describe default service account

```powershell
kubectl describe serviceaccount default -n default
```

**Expected Output:**

``` bash
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>
```

---

#### Command 3: Check what default SA can do

```powershell
kubectl auth can-i --list --as=system:serviceaccount:default:default
```

---

### Documentation Answer

**Q: What can the default service account do?**

The default service account has very limited permissions:

- Can access API discovery endpoints (`GET /api`, `/apis`)
- Can access health check endpoints (`/healthz`, `/livez`, `/readyz`)
- Can check its own permissions (`SelfSubjectAccessReview`)
- **CANNOT** list, create, update, or delete any Kubernetes resources

---

**Q: Why doesn't the default service account have many permissions?**

The default service account has minimal permissions due to the **Principle of Least Privilege** - a core security concept in Kubernetes. Here's why:

1. **Security by Default**: If pods had full permissions by default, a compromised pod could access or manipulate other cluster resources, enabling lateral movement attacks.

2. **Explicit Permission Model**: Forces administrators to think about and explicitly grant only the permissions each application needs.

3. **Reduces Attack Surface**: Even if an attacker gains access to a pod, they cannot automatically escalate privileges or access sensitive resources.

4. **Compliance Requirements**: Helps meet security compliance standards by requiring explicit permission grants with audit trails.

5. **Defense in Depth**: Multiple security layers - authentication (who you are) is separate from authorization (what you can do).

**Real-world analogy**: Like giving a new employee a badge that only opens the front door, not every room in the building.

---

### Task 1.2: Test Current Permissions

#### Command 4: Create test pod and try to list pods

```powershell
kubectl run test-pod --image=bitnami/kubectl:latest --rm -it -- bash
```

Inside the pod, run:

```bash
kubectl get pods
```

**Expected Error:**

```
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:default" 
cannot list resource "pods" in API group "" in the namespace "default"
```

Exit the pod:

```bash
exit
```

---

#### Command 5: Verify the denial

```powershell
kubectl auth can-i list pods --as=system:serviceaccount:default:default
```

**Expected Output:**

```
no
```

---

### Documentation Answer

**Q: What error do you see and why?**

**Error Observed:**

``` bash
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:default" 
cannot list resource "pods" in API group "" in the namespace "default"
```

**Why This Error Occurs:**

1. **Authentication SUCCESS**: The pod successfully authenticated as `system:serviceaccount:default:default`

2. **Authorization FAILURE**: The service account has no RoleBinding or ClusterRoleBinding granting permission to list pods

3. **Default Deny**: Kubernetes RBAC denies all actions by default unless explicitly allowed

4. **No Role Bindings**: When Kubernetes checks for permissions:
   - Checks RoleBindings in `default` namespace → None found
   - Checks ClusterRoleBindings → None found
   - Result: DENY (403 Forbidden)

**Security Implication**: This is correct and desired behavior. It prevents unauthorized access and requires explicit permission grants.

---

## PART 2: Creating Namespace-Scoped Access

### Scenario

Create a service account for a monitoring application that needs to:

- Read Pods and Deployments in the monitoring namespace
- Read ConfigMaps in the monitoring namespace
- NOT have permission to delete or modify anything

---

### Solution

#### Step 1: Create the monitoring namespace

```powershell
kubectl create namespace monitoring
```

---

#### Step 2: Create YAML files

**monitoring-rbac.yaml**

```yaml
---
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-reader
  namespace: monitoring
---
# Role with read-only permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: monitoring-read-role
  namespace: monitoring
rules:
  # Permission to read Pods
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  # Permission to read Deployments
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
  # Permission to read ConfigMaps
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
# RoleBinding to connect ServiceAccount to Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-reader-binding
  namespace: monitoring
subjects:
  - kind: ServiceAccount
    name: monitoring-reader
    namespace: monitoring
roleRef:
  kind: Role
  name: monitoring-read-role
  apiGroup: rbac.authorization.k8s.io
```

---

#### Step 3: Apply the configuration

```powershell
kubectl apply -f monitoring-rbac.yaml
```

---

#### Step 4: Verify resources were created

```powershell
kubectl get serviceaccount -n monitoring
kubectl get role -n monitoring
kubectl get rolebinding -n monitoring
```

---

#### Step 5: Create test resources in monitoring namespace

**test-resources.yaml**

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-1
  namespace: monitoring
spec:
  containers:
  - name: nginx
    image: nginx:latest
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: nginx
        image: nginx:latest
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
  namespace: monitoring
data:
  key: value
```

```powershell
kubectl apply -f test-resources.yaml
```

---

#### Step 6: Test the setup

**Test 1: Verify CAN list pods**

```powershell
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:monitoring-reader -n monitoring
```

**Expected Output:** `yes`

---

**Test 2: Verify CAN list deployments**

```powershell
kubectl auth can-i list deployments --as=system:serviceaccount:monitoring:monitoring-reader -n monitoring
```

**Expected Output:** `yes`

---

**Test 3: Verify CAN list configmaps**

```powershell
kubectl auth can-i list configmaps --as=system:serviceaccount:monitoring:monitoring-reader -n monitoring
```

**Expected Output:** `yes`

---

**Test 4: Verify CANNOT delete pods**

```powershell
kubectl auth can-i delete pods --as=system:serviceaccount:monitoring:monitoring-reader -n monitoring
```

**Expected Output:** `no`

---

**Test 5: Verify CANNOT create deployments**

```powershell
kubectl auth can-i create deployments --as=system:serviceaccount:monitoring:monitoring-reader -n monitoring
```

**Expected Output:** `no`

---

#### Step 7: Test with actual pod

**monitoring-test-pod.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: monitoring-test-pod
  namespace: monitoring
spec:
  serviceAccountName: monitoring-reader
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
```

```powershell
kubectl apply -f monitoring-test-pod.yaml
```

**Wait for pod to be ready:**

```powershell
kubectl wait --for=condition=ready pod/monitoring-test-pod -n monitoring --timeout=60s
```

**Test inside the pod:**

```powershell
# Should SUCCEED - list pods
kubectl exec -it monitoring-test-pod -n monitoring -- kubectl get pods

# Should SUCCEED - list deployments
kubectl exec -it monitoring-test-pod -n monitoring -- kubectl get deployments

# Should SUCCEED - list configmaps
kubectl exec -it monitoring-test-pod -n monitoring -- kubectl get configmaps

# Should FAIL - try to delete a pod
kubectl exec -it monitoring-test-pod -n monitoring -- kubectl delete pod test-pod-1
```

**Expected error for delete:**

```
Error from server (Forbidden): pods "test-pod-1" is forbidden: User "system:serviceaccount:monitoring:monitoring-reader" cannot delete resource "pods" in API group "" in the namespace "monitoring"
```

---

#### Cleanup

```powershell
kubectl delete pod monitoring-test-pod -n monitoring
```

---

## PART 3: Cluster-Wide Access

### Scenario

Create a service account for a cluster-wide logging agent that needs to:

- Read Pods in ALL namespaces
- Read Nodes
- NOT have access to Secrets or modify anything

---

### Solution

#### Step 1: Create YAML file

**log-collector-rbac.yaml**

```yaml
---
# Service Account in default namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: log-collector
  namespace: default
---
# ClusterRole with cluster-wide read permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: log-reader
rules:
  # Permission to read Pods in all namespaces
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  # Permission to read Nodes (cluster-scoped resource)
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  # Explicitly NO access to secrets (for clarity)
  # Note: Secrets are not included in rules, so access is denied by default
---
# ClusterRoleBinding to grant cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: log-collector-binding
subjects:
  - kind: ServiceAccount
    name: log-collector
    namespace: default
roleRef:
  kind: ClusterRole
  name: log-reader
  apiGroup: rbac.authorization.k8s.io
```

---

#### Step 2: Apply the configuration

```powershell
kubectl apply -f log-collector-rbac.yaml
```

---

#### Step 3: Verify resources were created

```powershell
kubectl get serviceaccount log-collector -n default
kubectl get clusterrole log-reader
kubectl get clusterrolebinding log-collector-binding
```

---

#### Step 4: Create test namespaces and pods

```powershell
# Create additional namespaces for testing
kubectl create namespace test-ns1
kubectl create namespace test-ns2

# Create test pods in different namespaces
kubectl run test-pod-default --image=nginx -n default
kubectl run test-pod-ns1 --image=nginx -n test-ns1
kubectl run test-pod-ns2 --image=nginx -n test-ns2
```

---

#### Step 5: Test permissions

**Test 1: Can list pods in default namespace**

```powershell
kubectl auth can-i list pods --as=system:serviceaccount:default:log-collector -n default
```

**Expected Output:** `yes`

---

**Test 2: Can list pods in test-ns1**

```powershell
kubectl auth can-i list pods --as=system:serviceaccount:default:log-collector -n test-ns1
```

**Expected Output:** `yes`

---

**Test 3: Can list pods in ALL namespaces**

```powershell
kubectl auth can-i list pods --as=system:serviceaccount:default:log-collector --all-namespaces
```

**Expected Output:** `yes`

---

**Test 4: Can list nodes**

```powershell
kubectl auth can-i list nodes --as=system:serviceaccount:default:log-collector
```

**Expected Output:** `yes`

---

**Test 5: CANNOT list secrets**

```powershell
kubectl auth can-i list secrets --as=system:serviceaccount:default:log-collector -n default
```

**Expected Output:** `no`

---

**Test 6: CANNOT delete pods**

```powershell
kubectl auth can-i delete pods --as=system:serviceaccount:default:log-collector -n default
```

**Expected Output:** `no`

---

#### Step 6: Test with actual pod

**log-collector-test-pod.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-collector-test
  namespace: default
spec:
  serviceAccountName: log-collector
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
```

```powershell
kubectl apply -f log-collector-test-pod.yaml
```

**Wait for pod to be ready:**

```powershell
kubectl wait --for=condition=ready pod/log-collector-test -n default --timeout=60s
```

**Test inside the pod:**

```powershell
# Should SUCCEED - list pods in default namespace
kubectl exec -it log-collector-test -n default -- kubectl get pods -n default

# Should SUCCEED - list pods in test-ns1
kubectl exec -it log-collector-test -n default -- kubectl get pods -n test-ns1

# Should SUCCEED - list pods in ALL namespaces
kubectl exec -it log-collector-test -n default -- kubectl get pods --all-namespaces

# Should SUCCEED - list nodes
kubectl exec -it log-collector-test -n default -- kubectl get nodes

# Should FAIL - list secrets
kubectl exec -it log-collector-test -n default -- kubectl get secrets -n default

# Should FAIL - delete a pod
kubectl exec -it log-collector-test -n default -- kubectl delete pod test-pod-default
```

**Expected error for secrets:**

``` bash
Error from server (Forbidden): secrets is forbidden: User "system:serviceaccount:default:log-collector" cannot list resource "secrets" in API group "" in the namespace "default"
```

---

#### Step 7: Show comprehensive permissions

```powershell
kubectl auth can-i --list --as=system:serviceaccount:default:log-collector
```

---

#### Cleanup

```powershell
kubectl delete pod log-collector-test -n default
kubectl delete pod test-pod-default -n default
kubectl delete pod test-pod-ns1 -n test-ns1
kubectl delete pod test-pod-ns2 -n test-ns2
kubectl delete namespace test-ns1
kubectl delete namespace test-ns2
```

---

## PART 4: Troubleshooting RBAC Issues

### Given Broken Configuration

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployment-manager
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: deployment-manager
  namespace: production
roleRef:
  kind: Role
  name: deployment-role
  apiGroup: rbac.authorization.k8s.io
```

---

### Troubleshooting Analysis

#### Error #1: Wrong API Group for Deployments

**Location:** Role rules section

**Problem:** `apiGroups: [""]` is incorrect

**Explanation:**

- Deployments belong to the `apps` API group, not the core API group (empty string `""`)
- The core API group (`""`) contains resources like Pods, Services, ConfigMaps
- Using the wrong API group means the Role grants no permissions to deployments

**Impact:** Service account cannot perform ANY operations on deployments because the permission doesn't match the actual resource API group.

---

#### Error #2: RoleBinding in Wrong Namespace

**Location:** RoleBinding metadata

**Problem:** RoleBinding is in `namespace: default` but should be in `namespace: production`

**Explanation:**

- The ServiceAccount is in `production` namespace
- The Role is in `production` namespace
- The RoleBinding is in `default` namespace
- RoleBindings only grant permissions within their own namespace
- A RoleBinding in `default` cannot grant permissions to a ServiceAccount in `production`

**Impact:** The binding never takes effect because it's looking for the ServiceAccount in the wrong namespace.

---

#### Error #3: Role References Wrong Namespace

**Location:** RoleBinding roleRef

**Problem:** The RoleBinding tries to reference a Role from a different namespace

**Explanation:**

- RoleBindings can only reference Roles in the SAME namespace
- The RoleBinding is in `default` but tries to bind to a Role in `production`
- This creates a cross-namespace reference that Kubernetes will reject

**Impact:** Kubernetes will not allow this RoleBinding to be created or will ignore it.

---

### Corrected YAML

**deployment-manager-fixed.yaml**

```yaml
---
# Service Account (no changes needed)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployment-manager
  namespace: production
---
# Role with CORRECTED API group
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-role
  namespace: production
rules:
  # FIX #1: Changed apiGroups from [""] to ["apps"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "create", "update", "delete"]
  # Optional: Add deployment status and scale subresources
  - apiGroups: ["apps"]
    resources: ["deployments/status", "deployments/scale"]
    verbs: ["get", "update"]
---
# RoleBinding with CORRECTED namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-binding
  # FIX #2: Changed namespace from "default" to "production"
  namespace: production
subjects:
  - kind: ServiceAccount
    name: deployment-manager
    namespace: production
roleRef:
  kind: Role
  name: deployment-role
  apiGroup: rbac.authorization.k8s.io
```

---

### Testing the Fix

#### Step 1: Create production namespace

```powershell
kubectl create namespace production
```

---

#### Step 2: Apply the BROKEN configuration (to see errors)

```powershell
kubectl apply -f deployment-manager-broken.yaml
```

---

#### Step 3: Test the broken setup

```powershell
# This will return "no" because of the errors
kubectl auth can-i list deployments --as=system:serviceaccount:production:deployment-manager -n production
```

---

#### Step 4: Apply the FIXED configuration

```powershell
kubectl apply -f deployment-manager-fixed.yaml
```

---

#### Step 5: Test the fixed setup

```powershell
# Should now return "yes"
kubectl auth can-i list deployments --as=system:serviceaccount:production:deployment-manager -n production

# Test all verbs
kubectl auth can-i get deployments --as=system:serviceaccount:production:deployment-manager -n production
kubectl auth can-i create deployments --as=system:serviceaccount:production:deployment-manager -n production
kubectl auth can-i update deployments --as=system:serviceaccount:production:deployment-manager -n production
kubectl auth can-i delete deployments --as=system:serviceaccount:production:deployment-manager -n production
```

---

#### Step 6: Test with actual pod

**deployment-manager-test.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: deployment-manager-test
  namespace: production
spec:
  serviceAccountName: deployment-manager
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
```

```powershell
kubectl apply -f deployment-manager-test.yaml
kubectl wait --for=condition=ready pod/deployment-manager-test -n production --timeout=60s
```

**Test operations:**

```powershell
# Should SUCCEED - list deployments
kubectl exec -it deployment-manager-test -n production -- kubectl get deployments

# Should SUCCEED - create a deployment
kubectl exec -it deployment-manager-test -n production -- kubectl create deployment test-deploy --image=nginx

# Should SUCCEED - delete the deployment
kubectl exec -it deployment-manager-test -n production -- kubectl delete deployment test-deploy
```

---

### Summary of Errors and Fixes

| Error | Location | Problem | Fix | Why It Matters |
|-------|----------|---------|-----|----------------|
| #1 | Role rules | `apiGroups: [""]` | `apiGroups: ["apps"]` | Deployments are in apps group, not core |
| #2 | RoleBinding metadata | `namespace: default` | `namespace: production` | RoleBindings only work in their own namespace |
| #3 | RoleBinding reference | Cross-namespace ref | Same namespace | Roles and RoleBindings must be co-located |

---

## PART 5: Real-World Scenario

### Scenario

Three teams with different access needs:

1. **Developers**: Deploy and manage applications in `dev` namespace
2. **QA Team**: Read-only access to `dev` and `staging` namespaces
3. **Ops Team**: Cluster-wide read access + write access to `production` namespace only

---

### Design Document

#### Architecture Overview

```bash
┌────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                  │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ dev namespace│  │staging ns    │  │production ns │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                        │
│  Developers          QA Team            Ops Team       │
│  (Full Access)      (Read Only)    (Read + Prod Write) │
│      │                  │                   │          │
│      ├─ Role            ├─ ClusterRole      ├─ ClusterRole
│      │  (dev)           │  (qa-reader)      │  (ops-reader)
│      │                  │                   │          │
│      └─ RoleBinding     ├─ RoleBinding      ├─ ClusterRoleBinding
│         (dev)           │  (dev)            │  (ops-global)
│                         │                   │          │
│                         └─ RoleBinding      └─ RoleBinding
│                            (staging)           (production)
└────────────────────────────────────────────────────────┘
```

---

### Complete RBAC Design

#### 1. Developers Team

**ServiceAccount:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: developer-sa
  namespace: dev
```

**Role (Namespace-scoped):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev
rules:
  # Full access to application resources
  - apiGroups: ["", "apps", "batch"]
    resources: 
      - pods
      - pods/log
      - pods/exec
      - deployments
      - replicasets
      - services
      - configmaps
      - jobs
      - cronjobs
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  
  # Read-only access to resource quotas and limits
  - apiGroups: [""]
    resources: ["resourcequotas", "limitranges"]
    verbs: ["get", "list"]
  
  # NO access to secrets (handled separately if needed)
```

**RoleBinding:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: developer-sa
    namespace: dev
  # Can also bind to Groups for user authentication
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

**Justification for Role (not ClusterRole):**

- Developers only need access within the dev namespace
- Using a Role limits blast radius if credentials are compromised
- Follows least privilege - no reason to grant cluster-wide permissions
- Easier to audit and manage namespace-specific permissions

---

#### 2. QA Team

**ServiceAccount:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: qa-sa
  namespace: default  # Can be in any namespace
```

**ClusterRole (Reusable read-only role):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: qa-reader-clusterrole
rules:
  # Read-only access to application resources
  - apiGroups: ["", "apps", "batch"]
    resources:
      - pods
      - pods/log
      - deployments
      - replicasets
      - services
      - configmaps
      - jobs
      - cronjobs
    verbs: ["get", "list", "watch"]
  
  # Read events for troubleshooting
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
```

**RoleBinding for dev namespace:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: qa-reader-dev-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: qa-sa
    namespace: default
  - kind: Group
    name: qa-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole  # Binding to a ClusterRole
  name: qa-reader-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

**RoleBinding for staging namespace:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: qa-reader-staging-binding
  namespace: staging
subjects:
  - kind: ServiceAccount
    name: qa-sa
    namespace: default
  - kind: Group
    name: qa-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: qa-reader-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

**Justification for ClusterRole + RoleBindings:**

- **ClusterRole**: Creates a reusable read-only permission template
- **RoleBindings**: Limits the ClusterRole's scope to specific namespaces (dev and staging only)
- **Why not a simple Role?**: ClusterRole can be reused across multiple namespaces with different RoleBindings
- **Why not ClusterRoleBinding?**: That would grant access to ALL namespaces, including production (security risk)
- **Best of both worlds**: Reusability + namespace isolation

---

#### 3. Ops Team

**ServiceAccount:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ops-sa
  namespace: default
```

**ClusterRole for cluster-wide read:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ops-global-reader
rules:
  # Read all resources cluster-wide
  - apiGroups: ["", "apps", "batch", "networking.k8s.io", "storage.k8s.io"]
    resources:
      - pods
      - deployments
      - services
      - nodes
      - namespaces
      - persistentvolumes
      - persistentvolumeclaims
      - ingresses
      - events
    verbs: ["get", "list", "watch"]
  
  # Read cluster-level resources
  - apiGroups: [""]
    resources: ["nodes", "persistentvolumes"]
    verbs: ["get", "list", "watch"]
```

**ClusterRoleBinding for global read:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-global-reader-binding
subjects:
  - kind: ServiceAccount
    name: ops-sa
    namespace: default
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: ops-global-reader
  apiGroup: rbac.authorization.k8s.io
```

**Role for production write access:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ops-production-admin
  namespace: production
rules:
  # Full access to production namespace
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources:
      - pods
      - deployments
      - replicasets
      - services
      - configmaps
      - secrets
      - jobs
      - cronjobs
      - ingresses
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  
  # Access to pod exec and logs for debugging
  - apiGroups: [""]
    resources: ["pods/exec", "pods/log"]
    verbs: ["get", "create"]
```

**RoleBinding for production:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ops-production-admin-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ops-sa
    namespace: default
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: ops-production-admin
  apiGroup: rbac.authorization.k8s.io
```

**Justification for Mixed Approach:**

- **ClusterRole + ClusterRoleBinding**: Needed for cluster-wide read access (nodes, all namespaces overview)
- **Role + RoleBinding**: Limits write access to production namespace only
- **Why both?**: Ops needs visibility across the entire cluster but should only modify production
- **Security**: Prevents accidental changes to dev/staging while troubleshooting
- **Separation of concerns**: Read permissions are global, write permissions are targeted

---

### Summary Table

| Team | ServiceAccount | Role/ClusterRole | Binding Type | Scope | Justification |
|------|----------------|------------------|--------------|-------|---------------|
| Developers | developer-sa (dev ns) | Role: developer-role | RoleBinding (dev) | dev namespace only | Namespace-scoped prevents access to other environments |
| QA Team | qa-sa (default ns) | ClusterRole: qa-reader-clusterrole | 2x RoleBindings (dev, staging) | dev + staging namespaces | ClusterRole for reusability, RoleBindings for namespace isolation |
| Ops Team | ops-sa (default ns) | ClusterRole: ops-global-reader + Role: ops-production-admin | ClusterRoleBinding (read) + RoleBinding (production write) | Cluster-wide read + production write | Mixed approach: global visibility, targeted write access |

---

### Design Decisions Explained

#### Why Role vs ClusterRole?

**Use Role when:**

- Permissions needed in a single namespace only
- Want to limit blast radius
- Following strict least privilege
- **Example**: Developers in dev namespace

**Use ClusterRole when:**

- Permissions needed across multiple namespaces
- Access to cluster-scoped resources (nodes, PVs)
- Creating reusable permission templates
- **Example**: QA team reading multiple namespaces, Ops team cluster-wide read

**Use ClusterRole + RoleBinding when:**

- Want to reuse permission sets but limit to specific namespaces
- Need same permissions in multiple namespaces without duplication
- **Example**: QA team with same read permissions in dev and staging

---

### Visual Relationship Diagram

``` text
┌─────────────────────────────────────────────────────────────────┐
│                       KUBERNETES CLUSTER                         │
└─────────────────────────────────────────────────────────────────┘

DEVELOPERS
──────────
[developer-sa] ──> [RoleBinding] ──> [Role: developer-role]
   (dev ns)         (dev ns)           (dev ns)
                                       │
                                       └──> Full access to dev namespace


QA TEAM
───────
[qa-sa] ──┬──> [RoleBinding] ──> [ClusterRole: qa-reader]
(default)  │     (dev ns)              │
           │                           └──> Read pods, deployments, etc.
           │
           └──> [RoleBinding] ──> [ClusterRole: qa-reader]
                (staging ns)          │
                                      └──> Same read permissions


OPS TEAM
────────
[ops-sa] ──┬──> [ClusterRoleBinding] ──> [ClusterRole: ops-global-reader]
(default)  │                                │
           │                                └──> Read EVERYTHING cluster-wide
           │
           └──> [RoleBinding] ──> [Role: ops-production-admin]
                (production ns)       │
                                      └──> Full access to production only
```

---

### Security Considerations

1. **Principle of Least Privilege**: Each team gets only what they need
2. **Blast Radius Limitation**: Namespace-scoped where possible
3. **Separation of Environments**: Dev ≠ Staging ≠ Production access
4. **No Secret Access by Default**: Secrets require explicit permission
5. **Audit Trail**: Clear RBAC structure makes auditing easier

---

## BONUS CHALLENGE: Label-Based Access Control

### Challenge

Create a service account that can:

- Create and delete Pods
- But ONLY if those pods have the label `managed-by: automation`

---

### Analysis

**Problem:** Standard Kubernetes RBAC cannot filter by labels. RBAC controls access to resource types, not individual resource instances based on their attributes.

**What RBAC CAN do:**

- Allow/deny access to entire resource types (all pods, all deployments)
- Limit by namespace
- Limit by verb (get, create, delete)

**What RBAC CANNOT do:**

- Filter by labels

- Filter by specific field values
- Conditional access based on resource content

---

### Solution Approaches

#### Approach 1: Admission Controllers

Use OPA (Open Policy Agent) Gatekeeper or Kyverno to enforce label requirements.

**How it works:**

1. RBAC grants permission to create/delete pods
2. Admission controller validates that pods have required label
3. If label is missing, admission controller rejects the request

---

### Implementation with Kyverno

#### Step 1: Install Kyverno

```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml
```

---

#### Step 2: Create ServiceAccount with pod permissions

**automation-sa.yaml**

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: automation-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: automation-role
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "delete", "get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: automation-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: automation-sa
    namespace: default
roleRef:
  kind: Role
  name: automation-role
  apiGroup: rbac.authorization.k8s.io
```

---

#### Step 3: Create Kyverno Policy

**enforce-automation-label.yaml**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-automation-label
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: check-automation-label
      match:
        any:
        - resources:
            kinds:
              - Pod
            subjects:
              - kind: ServiceAccount
                name: automation-sa
                namespace: default
      validate:
        message: "Pods created by automation-sa must have label 'managed-by: automation'"
        pattern:
          metadata:
            labels:
              managed-by: "automation"
```

**Apply the configuration:**

```bash
kubectl apply -f automation-sa.yaml
kubectl apply -f enforce-automation-label.yaml
```

---

#### Step 4: Test the policy

**Test 1: Create pod WITHOUT required label (should FAIL)**

**test-pod-no-label.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-no-label
  namespace: default
spec:
  serviceAccountName: automation-sa
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f test-pod-no-label.yaml --as=system:serviceaccount:default:automation-sa
```

**Error:**

```
Error from server: error when creating "test-pod-no-label.yaml": admission webhook "validate.kyverno.svc" denied the request: 

policy Pod/default/test-no-label for resource violation:

require-automation-label:
  check-automation-label: 'validation error: Pods created by automation-sa must have label 'managed-by: automation'. rule check-automation-label failed at path /metadata/labels/managed-by/'
```

---

**Test 2: Create pod WITH required label (should SUCCEED)**

**test-pod-with-label.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-with-label
  namespace: default
  labels:
    managed-by: automation
spec:
  serviceAccountName: automation-sa
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f test-pod-with-label.yaml --as=system:serviceaccount:default:automation-sa
```

**Expected:** Success

```bash
kubectl get pod test-with-label
```

---

**Test 3: Try to delete the labeled pod (should SUCCEED)**

```bash
kubectl delete pod test-with-label --as=system:serviceaccount:default:automation-sa
```

**Expected:** Success

---

#### Architecture

```
User Request (automation-sa)
    ↓
[1] API Server Authentication
    ↓
[2] RBAC Authorization ← "Can this SA create pods?" → YES
    ↓
[3] Admission Controllers ← "Does pod have required label?" → CHECK
    ↓
    ├─ YES → Pod Created
    └─ NO  → Request Rejected
```

---

### Testing Summary

| Test Case | Label Present | Expected Result | Actual Result |
|-----------|---------------|-----------------|---------------|
| Create pod without label | No | FAIL (rejected) | Rejected by Kyverno |
| Create pod with label | Yes | SUCCESS | Created |
| Delete labeled pod | Yes | SUCCESS | Deleted |
| Create pod with wrong label |  Wrong value | FAIL (rejected) | Rejected |

---

## Key Learnings

1. **RBAC is namespace-aware** - RoleBindings and Roles must be co-located
2. **API groups matter** - Deployments are in "apps", not core ("")
3. **Least privilege is crucial** - Start with minimal permissions
4. **ClusterRole + RoleBinding** - Powerful pattern for reusable permissions with namespace isolation
5. **RBAC limitations** - Cannot filter by labels; need admission controllers
6. **Testing is essential** - Use `kubectl auth can-i` to verify permissions
7. **Security by default** - Default SA has almost no permissions

---

## Testing Methodology

For each part:

1. Created YAML configurations
2. Applied to cluster
3. Tested positive cases (should succeed)
4. Tested negative cases (should fail)
5. Used `kubectl auth can-i` for verification
6. Created test pods with service accounts
7. Executed commands inside pods
8. Captured screenshots of all outputs

---

## Project Structure

- `documentation/` - PDF documentation for Parts 4, 5, and Bonus
- `screenshots/` - Visual evidence organized by assignment part
- `yaml-files/` - Kubernetes YAML configurations organized by part

``` bash
COMP-488_Assignment_RBAC/
├── README.md
├── documentation/
│   ├── Bonus_Challenge_Documentation.pdf
│   ├── Part4_Troubleshooting_Documentation.pdf
│   └── Part5_Design_Documentation.pdf
├── screenshots/
│   ├── part1/
│   ├── part2/
│   ├── part3/
│   ├── part4/
│   ├── part5/
│   └── bonus/
└── yaml-files/
    ├── part2/
    ├── part3/
    ├── part4/
    ├── part5/
    └── bonus/
```

---

## License

Author: [Kaushal Bhattarai]
This educational content is provided as-is for learning purposes.
