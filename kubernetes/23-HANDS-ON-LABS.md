# Section 23: Hands-On Labs

Practical labs for every level — from local development clusters to production cloud environments. Each lab builds on the concepts in previous sections and includes troubleshooting guidance for common mistakes.

## Subtopic Index

- [Environment Setup](#environment-setup)
- [Beginner Labs — Local (kind/minikube)](#beginner-labs--local-kindminikube)
- [Intermediate Labs — Cloud (AKS/EKS/GKE)](#intermediate-labs--cloud-akseksgke)
- [Advanced Labs — Production Patterns](#advanced-labs--production-patterns)
- [Expert Labs — Internals and Debugging](#expert-labs--internals-and-debugging)

---

## Environment Setup

### Local: kind (Kubernetes in Docker)

```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x kind && mv kind /usr/local/bin/

# Multi-node cluster with ingress support
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
EOF

kubectl cluster-info --context kind-kind
```

### Cloud: EKS (AWS)

```bash
# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl /usr/local/bin

# Create cluster with Karpenter-ready node group
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --version 1.30 \
  --nodegroup-name standard-workers \
  --node-type m5.xlarge \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10 \
  --managed
```

### Cloud: AKS (Azure)

```bash
# Create AKS cluster
az aks create \
  --resource-group myRG \
  --name myAKS \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-managed-identity \
  --network-plugin azure \
  --network-policy calico \
  --enable-oidc-issuer \
  --enable-workload-identity

az aks get-credentials --resource-group myRG --name myAKS
```

---

## Beginner Labs — Local (kind/minikube)

### Lab B1: Pod Lifecycle Deep Dive

**Objective**: Observe every phase of pod creation, probes, and graceful shutdown.

**Setup**: kind cluster.

**Tasks**:
```bash
# 1. Create a pod with all probe types
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    startupProbe:
      httpGet: {path: /, port: 80}
      failureThreshold: 30
      periodSeconds: 2
    readinessProbe:
      httpGet: {path: /, port: 80}
      initialDelaySeconds: 5
      periodSeconds: 5
    livenessProbe:
      httpGet: {path: /, port: 80}
      periodSeconds: 10
      failureThreshold: 3
    lifecycle:
      preStop:
        exec:
          command: ["sleep", "5"]
  terminationGracePeriodSeconds: 30
EOF

# 2. Watch pod transitions
kubectl get pod probe-demo -w &

# 3. Force a liveness failure
kubectl exec probe-demo -- nginx -s stop
# Observe pod restart

# 4. Watch graceful shutdown
kubectl delete pod probe-demo
# Note: takes ~5s due to preStop hook
```

**Expected outcome**: You observe Pending → Running → readiness passing → traffic routing → probe failure → restart → graceful 5s shutdown.

**Troubleshooting**: If pod stays Pending, check `kubectl describe pod probe-demo` for Events.

---

### Lab B2: RBAC — Least Privilege

**Objective**: Understand what permissions a ServiceAccount needs and nothing more.

**Tasks**:
```bash
# 1. Create a restricted ServiceAccount
kubectl create serviceaccount app-reader
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods
kubectl create rolebinding app-reader-binding \
  --role=pod-reader \
  --serviceaccount=default:app-reader

# 2. Test what it can and cannot do
kubectl auth can-i get pods --as=system:serviceaccount:default:app-reader
kubectl auth can-i delete pods --as=system:serviceaccount:default:app-reader  # should be "no"
kubectl auth can-i get secrets --as=system:serviceaccount:default:app-reader  # should be "no"

# 3. Run a pod with this SA and try to list secrets
kubectl run test --image=bitnami/kubectl \
  --serviceaccount=app-reader \
  --restart=Never -- sh -c "kubectl get secrets"
kubectl logs test   # should show "Forbidden"

# 4. Decode the SA token
kubectl create token app-reader --duration=10m | \
  cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
```

**Expected outcome**: Full understanding of token claims, RBAC scope, and least-privilege verification.

---

### Lab B3: NetworkPolicy Default-Deny

**Objective**: Implement and verify zero-trust namespace networking.

**Tasks**:
```bash
# 1. Setup: two namespaces, two services
kubectl create ns frontend
kubectl create ns backend

kubectl run frontend-app -n frontend --image=nginx --port=80 --expose
kubectl run backend-app -n backend --image=nginx --port=80 --expose

# 2. Verify traffic flows without policy
kubectl exec -n frontend frontend-app -- curl -m3 backend-app.backend.svc.cluster.local
# Should succeed

# 3. Apply default-deny to backend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: backend
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF

# 4. Verify all traffic is blocked (including DNS!)
kubectl exec -n frontend frontend-app -- curl -m3 backend-app.backend.svc.cluster.local
# Should fail

# 5. Allow DNS egress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: backend
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
EOF

# 6. Allow frontend to reach backend on port 80
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: backend
spec:
  podSelector: {matchLabels: {run: backend-app}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
    ports:
    - port: 80
EOF

# 7. Verify targeted access works
kubectl exec -n frontend frontend-app -- curl -m3 backend-app.backend.svc.cluster.local
# Should succeed
```

---

### Lab B4: Rolling Deploy and Rollback

**Objective**: Observe a rolling deploy, simulate failure, and rollback.

**Tasks**:
```bash
# 1. Deploy initial version
kubectl create deployment web --image=nginx:1.24 --replicas=4
kubectl rollout status deployment/web

# 2. Update to new version
kubectl set image deployment/web nginx=nginx:1.25
kubectl rollout status deployment/web -w
# Watch pods roll one at a time

# 3. Deploy a broken image
kubectl set image deployment/web nginx=nginx:9.99.99
kubectl rollout status deployment/web
# Observe rollout stalls — new pods can't pull image

# 4. Rollback
kubectl rollout undo deployment/web
kubectl rollout status deployment/web

# 5. Check history
kubectl rollout history deployment/web
kubectl rollout history deployment/web --revision=2
```

---

## Intermediate Labs — Cloud (AKS/EKS/GKE)

### Lab I1: EKS with IRSA — AWS API Access from Pod

**Objective**: Give a pod access to AWS S3 using IRSA (no static credentials).

**Setup**: EKS cluster with OIDC enabled.

**Tasks**:
```bash
# 1. Create an S3 bucket
BUCKET=my-k8s-irsa-test-$(date +%s)
aws s3 mb s3://$BUCKET --region us-east-1

# 2. Create IAM role with OIDC trust
CLUSTER_NAME=my-cluster
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_PROVIDER=$(aws eks describe-cluster --name $CLUSTER_NAME \
  --query "cluster.identity.oidc.issuer" --output text | sed -e "s/^https:\/\///")

cat <<EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::$ACCOUNT_ID:oidc-provider/$OIDC_PROVIDER"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "$OIDC_PROVIDER:sub": "system:serviceaccount:default:s3-accessor",
        "$OIDC_PROVIDER:aud": "sts.amazonaws.com"
      }
    }
  }]
}
EOF

ROLE_ARN=$(aws iam create-role \
  --role-name eks-s3-accessor \
  --assume-role-policy-document file://trust-policy.json \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name eks-s3-accessor \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 3. Create annotated ServiceAccount
kubectl create serviceaccount s3-accessor
kubectl annotate serviceaccount s3-accessor \
  eks.amazonaws.com/role-arn=$ROLE_ARN

# 4. Run a pod with the SA and access S3
kubectl run s3test \
  --image=amazon/aws-cli:latest \
  --serviceaccount=s3-accessor \
  --restart=Never \
  --command -- aws s3 ls s3://$BUCKET --region us-east-1

kubectl logs s3test
# Should list bucket contents (empty) without Access Denied

# 5. Verify identity
kubectl run whoami \
  --image=amazon/aws-cli:latest \
  --serviceaccount=s3-accessor \
  --restart=Never \
  --command -- aws sts get-caller-identity
kubectl logs whoami
# Should show the eks-s3-accessor role ARN
```

**Expected outcome**: Pod calls AWS API using short-lived credentials from IRSA, no `AWS_ACCESS_KEY_ID` env vars.

---

### Lab I2: ArgoCD GitOps Pipeline

**Objective**: Deploy an application via Git and observe sync, drift detection, and self-heal.

**Setup**: EKS cluster, ArgoCD installed.

**Tasks**:
```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get pods -w

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# 2. Create a Git repo with a Deployment manifest (push to GitHub)
# In your repo: manifests/deployment.yaml with nginx:1.24

# 3. Create ArgoCD Application
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-ORG/YOUR-REPO
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# 4. Observe sync
kubectl -n argocd get application my-app -w

# 5. Simulate drift: manually change the image
kubectl set image deployment/my-app nginx=nginx:1.25

# 6. Observe ArgoCD self-heal: reverts to nginx:1.24
watch kubectl get deployment my-app -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

### Lab I3: Karpenter JIT Autoscaling

**Objective**: Observe Karpenter provision nodes on-demand and consolidate.

**Setup**: EKS with Karpenter installed.

**Tasks**:
```bash
# 1. Create a NodePool
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: [spot, on-demand]
      - key: karpenter.k8s.aws/instance-category
        operator: In
        values: [m, c]
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
  limits:
    cpu: 100
EOF

# 2. Deploy a workload that needs new nodes
kubectl create deployment scale-test --image=nginx \
  --replicas=0

# 3. Scale up rapidly
kubectl scale deployment scale-test --replicas=20

# 4. Watch Karpenter launch nodes
kubectl get nodeclaims -w
kubectl -n kube-system logs -l app.kubernetes.io/name=karpenter \
  | grep launched

# 5. Scale down and watch consolidation
kubectl scale deployment scale-test --replicas=2
# After 30s, watch Karpenter consolidate
kubectl get nodeclaims -w
```

---

### Lab I4: Prometheus + SLO Dashboard

**Objective**: Set up monitoring, create an SLO, and build a burn-rate alert.

**Setup**: EKS/AKS with kube-prometheus-stack.

**Tasks**:
```bash
# 1. Install kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prom prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# 2. Deploy a demo app with Prometheus metrics
kubectl apply -f https://raw.githubusercontent.com/prometheus/client_python/master/examples/metrics.py

# 3. Create a ServiceMonitor
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
  labels:
    release: prom
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 15s
EOF

# 4. Create SLO recording rules
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-slo
  namespace: monitoring
  labels:
    release: prom
spec:
  groups:
  - name: slo
    interval: 30s
    rules:
    - record: job:http_requests:rate5m
      expr: sum(rate(http_requests_total[5m])) by (job)
    - record: job:http_errors:rate5m
      expr: sum(rate(http_requests_total{code=~"5.."}[5m])) by (job)
    - alert: HighErrorBudgetBurn
      expr: |
        (job:http_errors:rate5m / job:http_requests:rate5m) > (14.4 * 0.001)
      for: 5m
      labels:
        severity: page
      annotations:
        summary: "Error budget burning too fast"
EOF

# 5. Port-forward to Grafana and import dashboard
kubectl port-forward -n monitoring svc/prom-grafana 3000:80
# Open http://localhost:3000 (admin/prom-operator)
```

---

## Advanced Labs — Production Patterns

### Lab A1: Zero-Downtime Blue-Green Deployment

**Objective**: Implement a blue-green deploy with instant rollback.

**Tasks**:
```bash
# 1. Deploy blue version
kubectl create deployment payments-blue \
  --image=nginx:1.24 --replicas=5
kubectl expose deployment payments-blue --port=80 --name=payments-blue

# 2. Create the public Service pointing to blue
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: payments
spec:
  selector:
    app: payments-blue    # points to blue
  ports:
  - port: 80
EOF

# 3. Deploy green version (inactive)
kubectl create deployment payments-green \
  --image=nginx:1.25 --replicas=5
kubectl rollout status deployment/payments-green

# 4. Test green internally
kubectl expose deployment payments-green --port=80 --name=payments-green
kubectl run test --image=curlimages/curl --restart=Never --rm -it -- \
  curl payments-green/version

# 5. Switch traffic to green (atomic)
kubectl patch service payments \
  -p '{"spec":{"selector":{"app":"payments-green"}}}'

# 6. Verify
kubectl run test2 --image=curlimages/curl --restart=Never --rm -it -- \
  curl payments/version

# 7. Instant rollback if needed
kubectl patch service payments \
  -p '{"spec":{"selector":{"app":"payments-blue"}}}'
```

---

### Lab A2: etcd Backup and Restore

**Objective**: Practice the full backup and restore procedure.

**Setup**: kubeadm cluster (or kind with etcd accessible).

**Tasks**:
```bash
# 1. Set etcd client environment
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# 2. Create test resources
kubectl create deployment pre-backup --image=nginx --replicas=2
kubectl create configmap important-config --from-literal=key=value

# 3. Take a snapshot
etcdctl snapshot save /tmp/backup-$(date +%Y%m%d-%H%M%S).db
etcdctl snapshot status /tmp/backup-*.db --write-out=table

# 4. Simulate disaster: delete resources
kubectl delete deployment pre-backup
kubectl delete configmap important-config

# 5. Restore from snapshot (on each control plane node):
# Stop apiserver first
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

BACKUP_FILE=$(ls /tmp/backup-*.db | tail -1)
etcdctl snapshot restore $BACKUP_FILE \
  --name=default \
  --initial-cluster=default=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380 \
  --data-dir=/var/lib/etcd-restored

# 6. Swap data directories
sudo mv /var/lib/etcd /var/lib/etcd-old
sudo mv /var/lib/etcd-restored /var/lib/etcd

# 7. Restart apiserver
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 8. Verify restoration
kubectl get deployment pre-backup   # should exist again
kubectl get configmap important-config   # should exist again
```

---

## Expert Labs — Internals and Debugging

### Lab E1: PLEG Internals — Container Count vs Relist Time

**Objective**: Observe PLEG health degrading with high container count.

**Tasks**:
```bash
# 1. Baseline PLEG relist duration (before adding pods)
kubectl get --raw='/metrics' 2>/dev/null | \
  grep pleg_relist_duration_seconds | grep le=\"5\" | awk '{print $2}' | \
  xargs -I{} echo "Baseline cumulative count at 5s: {}"

# 2. Create 200 pods to simulate high container count
for i in $(seq 1 200); do
  kubectl run busybox-$i --image=busybox --restart=Never -- sleep 3600
done
kubectl wait --for=condition=ready pod -l run --timeout=120s

# 3. Check PLEG relist duration again
kubectl get --raw='/metrics' 2>/dev/null | \
  grep pleg_relist_duration_seconds

# 4. On the node: watch the relist in real time
crictl ps -a | wc -l    # count all containers including stopped
journalctl -u kubelet | grep "relist" | tail -5

# 5. Clean up
kubectl delete pods -l run
```

---

### Lab E2: iptables vs IPVS — Rule Count at Scale

**Objective**: Compare iptables and IPVS rule counts.

**Tasks**:
```bash
# 1. Check current kube-proxy mode
kubectl -n kube-system get configmap kube-proxy -o yaml | grep mode

# 2. Create 100 Services
for i in $(seq 1 100); do
  kubectl create service clusterip svc-$i --tcp=80:80
done

# 3. Count iptables rules in KUBE-SERVICES
iptables-save | grep -c KUBE-SVC
iptables-save | grep -c KUBE-SEP

# 4. Switch to IPVS
kubectl -n kube-system edit configmap kube-proxy
# Change: mode: "ipvs"
kubectl -n kube-system rollout restart daemonset kube-proxy
sleep 30

# 5. Compare IPVS vs iptables rule count
ipvsadm -Ln | grep -c TCP
iptables-save | grep KUBE | wc -l

# Clean up
for i in $(seq 1 100); do kubectl delete svc svc-$i; done
```

---

### Lab E3: Writing a Simple Kubernetes Controller

**Objective**: Write a controller that creates a ConfigMap when a new Namespace is created.

**Setup**: Go environment, controller-runtime.

```go
// main.go
package main

import (
    "context"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
)

type NamespaceReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *NamespaceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)
    
    var ns corev1.Namespace
    if err := r.Get(ctx, req.NamespacedName, &ns); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "namespace-info",
            Namespace: ns.Name,
        },
        Data: map[string]string{
            "created-at": ns.CreationTimestamp.String(),
            "labels":     fmt.Sprintf("%v", ns.Labels),
        },
    }
    
    // Server-side apply — idempotent
    if err := r.Patch(ctx, cm, client.Apply, client.FieldOwner("ns-controller")); err != nil {
        log.Error(err, "failed to create ConfigMap")
        return ctrl.Result{}, err
    }
    
    log.Info("Reconciled namespace", "name", ns.Name)
    return ctrl.Result{}, nil
}

func main() {
    ctrl.SetLogger(zap.New())
    
    mgr, _ := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{})
    
    ctrl.NewControllerManagedBy(mgr).
        For(&corev1.Namespace{}).
        Complete(&NamespaceReconciler{
            Client: mgr.GetClient(),
            Scheme: mgr.GetScheme(),
        })
    
    mgr.Start(ctrl.SetupSignalHandler())
}
```

**Tasks**:
1. Build and run the controller: `go run main.go`.
2. Create a namespace: `kubectl create ns test-ns`.
3. Verify the ConfigMap was created: `kubectl get cm namespace-info -n test-ns`.
4. Delete and recreate — verify idempotency.
5. Add a watch for ConfigMap changes that triggers reconcile.

---

### Lab E4: eBPF Observability with Hubble

**Objective**: Use Cilium + Hubble to trace network flows at L7.

**Setup**: Cluster with Cilium installed.

```bash
# 1. Enable Hubble
cilium hubble enable --ui

# 2. Deploy two services
kubectl run server --image=nginx --port=80 --expose
kubectl run client --image=curlimages/curl --restart=Never -- \
  sh -c 'while true; do curl -s http://server/; sleep 1; done'

# 3. Observe L4 flows
hubble observe --from-pod default/client --to-pod default/server --follow

# 4. Apply an L7 NetworkPolicy (Cilium-specific)
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-get-only
spec:
  endpointSelector:
    matchLabels: {run: server}
  ingress:
  - fromEndpoints:
    - matchLabels: {run: client}
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: GET
EOF

# 5. Test: GET should succeed, POST should fail
kubectl exec client -- curl -X GET http://server/      # works
kubectl exec client -- curl -X POST http://server/api  # blocked

# 6. Watch drops in Hubble
hubble observe --verdict DROPPED --follow
```

---

## Lab Troubleshooting Guide

### Common Lab Failures

**"No API server connection"**
```bash
kubectl cluster-info
# If failed: check kubeconfig context
kubectl config get-contexts
kubectl config use-context <correct-context>
```

**"Image pull failed"**
```bash
kubectl describe pod <pod> | grep -A10 Events
# Check registry access; use public images for labs
```

**"NetworkPolicy lab: traffic still allowed after apply"**
```bash
# Verify CNI supports NetworkPolicy
kubectl -n kube-system get pods | grep -E 'calico|cilium|weave'
# If using basic Flannel: NetworkPolicy is not enforced
# Switch to Calico or use kind with Cilium
```

**"etcd restore: apiserver won't start"**
```bash
# Check apiserver manifest is back in place
ls /etc/kubernetes/manifests/
# Check etcd data dir permissions
ls -la /var/lib/etcd/
# Check kubelet logs
journalctl -u kubelet | tail -30
```

**"Controller lab: reconciler not triggered"**
```bash
# Verify RBAC for the controller SA
kubectl auth can-i get namespaces --as=system:serviceaccount:default:default
# Check manager is started and cache synced
```
