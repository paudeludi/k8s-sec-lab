# Kubernetes Network Policies and Security Enforcement Using Kyverno

## Scope

This lab implements a two-tier application without adding unrelated platform components. A frontend workload calls a backend HTTP service internally via ClusterIP Services. Kubernetes NetworkPolicies restrict packet flow at the network layer. Kyverno validates Pod objects at admission time and rejects Pods that do not meet the required metadata and resource standards.

## Prerequisites and Cluster Verification

Create the Lab Cluster (Only If It Does Not Already Exist)

```
minikube start \
  -p k8s-sec-lab \
  --driver=docker \
  --cni=calico \
  --nodes=2 \
  --cpus=2 \
  --memory=2048
```

Select and Inspect the Cluster

```
minikube profile k8s-security-lab
kubectl config use-context k8s-security-lab
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
```

## Create the Workspace and Namespace

```
mkdir -p ~/k8s-sec-lab/{applications,network-policies,kyverno,tests,evidence}
cd ~/k8s-sec-lab

kubectl create namespace security-lab
kubectl get namespace security-lab
```

The security-lab namespace acts as the isolation boundary for the application workloads and network policies.

## Deploy the Two Applications and Their Services

Frontend Deployment and Service

```
cat > applications/frontend.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: security-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        # The frontend application serves HTTP on port 80.
        - name: frontend-web
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
        # This sidecar is the in-Pod curl client used for the lab tests.
        - name: curl
          image: curlimages/curl:8.12.1
          command: ["sh", "-c", "sleep 360000"]
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: security-lab
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
EOF

kubectl apply -f applications/frontend.yaml
```

*Port detail:* frontend-service selects the frontend Pod and forwards TCP 80 to the frontend-web Nginx container on TCP 80\. The curl sidecar shares the Pod network namespace, so its requests are subject to the same Pod-level NetworkPolicies.

Backend Deployment and Service

```
cat > applications/backend.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: security-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: security-lab
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
EOF

kubectl apply -f applications/backend.yaml
```

Wait and Verify Rollout

```
kubectl rollout status deployment/frontend -n security-lab
kubectl rollout status deployment/backend -n security-lab

kubectl get deployments -n security-lab
kubectl get pods -n security-lab -o wide
kubectl get services -n security-lab
kubectl get endpointslices -n security-lab
```

*Expected:* Both Deployments show 1/1 available; both Services show TYPE ClusterIP; both frontend-service and backend-service have active endpoint addresses.

Verify Baseline Service Discovery and Connectivity (Before Policies)

```
FRONTEND_POD=$(kubectl get pod -n security-lab -l app=frontend -o jsonpath='{.items[0].metadata.name}')

# Verify frontend-service resolves and serves HTTP on port 80
kubectl exec -n security-lab "$FRONTEND_POD" -c curl -- \
  curl --fail --show-error --silent http://frontend-service

# Verify frontend-to-backend HTTP connectivity over ClusterIP DNS
kubectl exec -n security-lab "$FRONTEND_POD" -c curl -- \
  curl --fail --show-error --silent http://backend-service
```

*Expected:* Both commands print the default Nginx welcome page HTML and exit with status 0.

## Prepare Controlled Network Test Clients

Create two test fixtures before applying NetworkPolicies. They exist solely to prove policy enforcement.

Unauthorized Client Pod

```
cat > tests/unauthorized-client.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: unauthorized-client
  namespace: security-lab
  labels:
    app: unauthorized-client
spec:
  containers:
    - name: curl
      image: curlimages/curl:8.12.1
      command: ["sh", "-c", "sleep 360000"]
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 64Mi
EOF

kubectl apply -f tests/unauthorized-client.yaml
kubectl wait --for=condition=Ready pod/unauthorized-client -n security-lab --timeout=120s
```

Egress Destination Pod and Service

```
cat > tests/egress-target.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: egress-target
  namespace: security-lab
  labels:
    app: egress-target
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - name: http
          containerPort: 80
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: egress-target-service
  namespace: security-lab
spec:
  type: ClusterIP
  selector:
    app: egress-target
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF

kubectl apply -f tests/egress-target.yaml
kubectl wait --for=condition=Ready pod/egress-target -n security-lab --timeout=120s
```

Prove Both Test Paths Are Open Before Policies

```
kubectl exec -n security-lab "$FRONTEND_POD" -c curl -- \
  curl --fail --silent --show-error http://egress-target-service

kubectl exec -n security-lab unauthorized-client -- \
  curl --fail --silent --show-error http://backend-service
```

*Expected:* Both commands print Nginx HTML. No NetworkPolicy is isolating these Pods yet, so all ingress and egress traffic is allowed by default.

## Apply NetworkPolicies

Backend Ingress: Allow Only Frontend Pods on TCP 80

```
cat > network-policies/backend-allow-frontend-ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend-ingress
  namespace: security-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
EOF

kubectl apply -f network-policies/backend-allow-frontend-ingress.yaml
kubectl describe networkpolicy backend-allow-frontend-ingress -n security-lab
```

Test Backend Ingress Policy

```
# Allowed: frontend -> backend
kubectl exec -n security-lab "$FRONTEND_POD" -c curl -- \
  curl --fail --silent --show-error http://backend-service

# Blocked: unauthorized-client -> backend
kubectl exec -n security-lab unauthorized-client -- \
  curl --connect-timeout 3 --max-time 5 --verbose http://backend-service
```

*Expected:* The frontend curl succeeds (exit 0). The unauthorized-client curl fails or times out (non-zero exit code).

Frontend Egress: Allow Backend HTTP and CoreDNS Only

Verify CoreDNS Pod and namespace labels first:

```
kubectl get pods -n kube-system -l k8s-app=kube-dns --show-labels
kubectl get namespace kube-system --show-labels
```

*Precondition:* At least one CoreDNS Pod with k8s-app=kube-dns must exist in kube-system (labelled kubernetes.io/metadata.name=kube-system).

```
cat > network-policies/frontend-restrict-egress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-restrict-egress
  namespace: security-lab
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    # Frontend may connect to backend Pods over HTTP.
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80

    # Frontend may query CoreDNS for Kubernetes Service discovery.
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
EOF

kubectl apply -f network-policies/frontend-restrict-egress.yaml
kubectl describe networkpolicy frontend-restrict-egress -n security-lab
```

*Selector semantics:* namespaceSelector and podSelector within the same to list item are combined with logical **AND**.

Test Frontend Egress Policy

```
# Allowed: frontend -> backend (DNS lookup + TCP 80)
kubectl exec -n security-lab "$FRONTEND_POD" -c curl -- \
  curl --fail --silent --show-error http://backend-service

# Blocked: frontend -> egress-target (not in allowed egress rules)
kubectl exec -n security-lab "$FRONTEND_POD" -c curl -- \
  curl --connect-timeout 3 --max-time 5 --verbose http://egress-target-service
```

*Expected:* The backend-service request succeeds. The egress-target-service request resolves DNS (proving CoreDNS access works) and then fails or times out on TCP connection.

Confirm Policy Objects and Selected Pods

```
kubectl get networkpolicy -n security-lab
kubectl describe networkpolicy -n security-lab
kubectl get pods -n security-lab --show-labels
```

## Install Kyverno Using Helm

Keep the Helm installation procedure, provided chart version `3.10.0` is the intended chart release for your lab.

```
# Add the official Kyverno Helm repository
helm repo add kyverno https://kyverno.github.io/kyverno/

# Update Helm repositories
helm repo update

# Confirm available chart versions
helm search repo kyverno/kyverno --versions | head -n 10

# Install Kyverno chart version 3.10.0
helm install kyverno kyverno/kyverno \
  --version 3.10.0 \
  --namespace kyverno \
  --create-namespace \
  --wait \
  --timeout 5m
```

Verify Kyverno Installation

```
helm list -n kyverno

kubectl get deployments,pods,services -n kyverno

kubectl get crds | grep kyverno.io

kubectl get crds | grep policies.kyverno.io

kubectl get validatingwebhookconfigurations | grep kyverno
```

**Expected results:**

* Helm shows the `kyverno` release as deployed.  
* Kyverno controller Pods are Running and Ready.  
* Kyverno CRDs are registered, including the CEL-based policy CRDs.  
* Kyverno validating webhooks are registered.

**Important:** Do not create policies until the Kyverno admission controller is Running and Ready.

## Create Kyverno ValidatingPolicies

We will create two cluster-scoped `ValidatingPolicy` resources.

Unlike the legacy `ClusterPolicy`, the new `ValidatingPolicy` uses **Common Expression Language (CEL)** for validation.

Both policies will:

1. Apply to Pods across Kubernetes namespaces.  
2. Enforce validation during CREATE and UPDATE operations.  
3. Exclude system namespaces:  
   * `kube-system`  
   * `kyverno`  
   * `kube-public`  
   * `kube-node-lease`  
4. Reject non-compliant resources using `validationActions: Deny`.  
5. Enable background evaluation.

### ValidatingPolicy: Every Application Pod Must Have a Non-Empty app Label

Create the policy file:

```
cat > kyverno/require-app-label.yaml <<'EOF'
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: require-app-label
spec:
  evaluation:
    admission:
      enabled: true
    background:
      enabled: true

  matchConstraints:
    resourceRules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        operations:
          - CREATE
          - UPDATE
        resources:
          - pods

  validationActions:
    - Deny

  validations:
    - expression: >-
        object.metadata.namespace in [
          'kube-system',
          'kyverno',
          'kube-public',
          'kube-node-lease'
        ] ||
        (
          has(object.metadata.labels) &&
          'app' in object.metadata.labels &&
          object.metadata.labels['app'] != ''
        )
      message: >-
        Every application Pod must have a non-empty app label.
EOF
```

Apply the policy:

```
kubectl apply -f kyverno/require-app-label.yaml
```

Verify:

```
kubectl get validatingpolicy require-app-label
```

**CEL expression explanation:**

```
has(object.metadata.labels) &&
'app' in object.metadata.labels &&
object.metadata.labels['app'] != ''
```

This expression checks three conditions:

1. The Pod has a labels object.  
2. The `app` label exists.  
3. The `app` label value is not an empty string.

The policy allows Pods in the four excluded system namespaces without requiring the `app` label.

### ValidatingPolicy: Require CPU and Memory Requests and Limits

Create the policy file:

```
cat > kyverno/require-cpu-memory-resources.yaml <<'EOF'
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: require-cpu-memory-resources
spec:
  evaluation:
    admission:
      enabled: true
    background:
      enabled: true

  matchConstraints:
    resourceRules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        operations:
          - CREATE
          - UPDATE
        resources:
          - pods

  validationActions:
    - Deny

  validations:
    - expression: >-
        object.metadata.namespace in [
          'kube-system',
          'kyverno',
          'kube-public',
          'kube-node-lease'
        ] ||
        (
          (
            object.spec.containers +
            object.spec.?initContainers.orValue([])
          ).all(container,
            has(container.resources) &&
            has(container.resources.requests) &&
            'cpu' in container.resources.requests &&
            'memory' in container.resources.requests &&
            has(container.resources.limits) &&
            'cpu' in container.resources.limits &&
            'memory' in container.resources.limits
          )
        )
      message: >-
        Every container and initContainer must define CPU and memory
        requests and limits.
EOF
```

Apply the policy:

```
kubectl apply -f kyverno/require-cpu-memory-resources.yaml
```

Verify:

```
kubectl get validatingpolicy require-cpu-memory-resources
```

**CEL expression explanation:**

The expression checks both regular containers and initContainers.

Every container must define:

| Resource configuration | Required |
| ----- | ----- |
| CPU request | Yes |
| Memory request | Yes |
| CPU limit | Yes |
| Memory limit | Yes |

The expression:

```
object.spec.containers +
object.spec.?initContainers.orValue([])
```

Combines regular containers and initContainers into one list.

The `.all()` function checks that every container satisfies all four resource requirements.

The optional expression `.orValue([])` handles Pods that do not define initContainers.

Pods in the four excluded system namespaces bypass this validation.

---

### Verify Both ValidatingPolicies Are Ready

Check both policies:

```
kubectl get validatingpolicies
```

Describe the individual policies:

```
kubectl describe validatingpolicy require-app-label
kubectl describe validatingpolicy require-cpu-memory-resources
```

**Expected results:**

* Both policies should be registered.  
* Both policies should report `READY: True`.  
* Both policies should report `ADMISSION: true`.

Do not proceed to testing until both policies are ready and admission enforcement is active.

## Test Admission Rejection and Successful Admission

Run each test separately to isolate the rule evaluated by the admission controller.

### Negative Test A: Missing or Empty app Label

```
# Test A1: Missing app label
cat > tests/pod-missing-app-label.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-missing-app-label
  namespace: security-lab
spec:
  containers:
    - name: test
      image: nginx:1.27-alpine
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 64Mi
EOF

kubectl apply -f tests/pod-missing-app-label.yaml

# Test A2: Empty app label ("")
cat > tests/pod-empty-app-label.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-empty-app-label
  namespace: security-lab
  labels:
    app: ""
spec:
  containers:
    - name: test
      image: nginx:1.27-alpine
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 64Mi
EOF

kubectl apply -f tests/pod-empty-app-label.yaml
```

*Expected:* Both API requests are denied with a Kyverno policy violation referencing require-app-label. Neither Pod is created.

### Negative Test B: Missing All or Partial CPU/Memory Requests and Limits

```
# Test B1: Completely missing resources block
cat > tests/pod-missing-resources.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-missing-resources
  namespace: security-lab
  labels:
    app: policy-test
spec:
  containers:
    - name: test
      image: nginx:1.27-alpine
EOF

kubectl apply -f tests/pod-missing-resources.yaml

# Test B2: Partial resources (requests present, limits omitted)
cat > tests/pod-partial-resources.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-partial-resources
  namespace: security-lab
  labels:
    app: policy-test
spec:
  containers:
    - name: test
      image: nginx:1.27-alpine
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
EOF

kubectl apply -f tests/pod-partial-resources.yaml
```

*Expected:* Both API requests are denied with a Kyverno policy violation referencing require-cpu-memory-resources.

### Positive Test: Compliant Pod

bash

```
cat > tests/pod-compliant.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-compliant
  namespace: security-lab
  labels:
    app: policy-test
spec:
  containers:
    - name: test
      image: nginx:1.27-alpine
      ports:
        - name: http
          containerPort: 80
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 64Mi
EOF

kubectl apply -f tests/pod-compliant.yaml
kubectl wait --for=condition=Ready pod/pod-compliant -n security-lab --timeout=120s
kubectl get pod pod-compliant -n security-lab -o wide
```

*Expected:* Pod creation succeeds and pod-compliant reaches Ready status.

### Positive Test (Optional): Compliant Pod with initContainers

```
cat > tests/pod-with-init-container.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-init-container
  namespace: security-lab
  labels:
    app: policy-test
spec:
  initContainers:
    - name: init
      image: busybox:1.36
      command: ["sh", "-c", "echo init-complete"]
      resources:
        requests:
          cpu: 10m
          memory: 8Mi
        limits:
          cpu: 50m
          memory: 16Mi
  containers:
    - name: app
      image: nginx:1.27-alpine
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 64Mi
EOF

kubectl apply -f tests/pod-with-init-container.yaml
kubectl wait --for=condition=Ready pod/pod-with-init-container -n security-lab --timeout=120s
```

## Export Demonstration Evidence to file.

```
{
  echo "=== NODES & CNI ==="
  kubectl get nodes -o wide
  kubectl get pods -n kube-system | grep -iE 'calico|coredns'

  echo -e "\n=== WORKLOADS, SERVICES & ENDPOINTS ==="
  kubectl get deployments,services,endpointslices,pods -n security-lab -o wide --show-labels

  echo -e "\n=== NETWORK POLICIES ==="
  kubectl describe networkpolicy -n security-lab

  echo -e "\n=== KYVERNO RELEASE & POLICIES ==="
  helm list -n kyverno
  kubectl get pods -n kyverno
  kubectl get clusterpolicies
  kubectl describe clusterpolicy require-app-label require-cpu-memory-resources
} > evidence/lab-verification-report.txt

cat evidence/lab-verification-report.txt
```
