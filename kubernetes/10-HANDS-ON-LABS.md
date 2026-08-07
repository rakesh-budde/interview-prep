# Hands-On Labs (30+ Practical Exercises)

> Runnable on kind, minikube, kubeadm, or any managed cluster (EKS/GKE/AKS). Each lab includes setup, steps, and validation.

**Prerequisites:** `kubectl`, `kind` or `minikube`, `helm`, `docker` installed.

---

## Category A: Cluster Setup

### Lab A1: Deploy a Multi-Node Cluster with kind
```bash
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF
kind create cluster --name lab-cluster --config kind-config.yaml
kubectl cluster-info --context kind-lab-cluster
kubectl get nodes -o wide
```
**Validation:** 4 nodes Ready (1 control-plane, 3 workers).

### Lab A2: Set Up HA Control Plane with kubeadm (3 control-plane nodes)
```bash
# On first control-plane node:
kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:6443" \
  --upload-certs --pod-network-cidr=10.244.0.0/16

# On additional control-plane nodes (using the join command with --control-plane
# and --certificate-key printed by the init command above):
kubeadm join LOAD_BALANCER_DNS:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>

# Verify:
kubectl get nodes
kubectl get pods -n kube-system -o wide | grep etcd
```
**Validation:** 3 nodes with `control-plane` role, 3 etcd pods, each on a different node.

### Lab A3: Configure and Test etcd Backup/Restore
```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /tmp/snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

etcdctl snapshot status /tmp/snapshot.db -w table

# Simulate disaster: delete a namespace
kubectl create namespace test-restore
kubectl delete namespace test-restore

# Restore to a NEW data-dir (don't overwrite the live one in this exercise)
etcdctl snapshot restore /tmp/snapshot.db --data-dir /tmp/etcd-restored
```
**Validation:** `etcdctl snapshot status` shows valid hash/revision; restored data-dir contains expected keys via `etcdutl` inspection.

---

## Category B: Workloads

### Lab B1: Deploy a Multi-Tier Microservices App
```bash
kubectl create namespace shop
kubectl -n shop create deployment frontend --image=nginx --replicas=3
kubectl -n shop create deployment backend --image=hashicorp/http-echo --replicas=2 \
  -- -text="backend-ok"
kubectl -n shop expose deployment frontend --port=80
kubectl -n shop expose deployment backend --port=5678
kubectl -n shop run tester --image=busybox --rm -it --restart=Never -- \
  wget -qO- backend:5678
```
**Validation:** tester pod prints "backend-ok", confirming Service DNS + routing works.

### Lab B2: Implement Blue-Green Deployment
```bash
kubectl create namespace bluegreen
kubectl -n bluegreen create deployment app-blue --image=nginx:1.24 --replicas=3
kubectl -n bluegreen label deployment app-blue version=blue --overwrite
kubectl -n bluegreen patch deployment app-blue -p \
  '{"spec":{"template":{"metadata":{"labels":{"version":"blue","app":"myapp"}}}}}'
kubectl -n bluegreen expose deployment app-blue --port=80 --name=myapp-svc \
  --selector=app=myapp,version=blue

# Deploy green
kubectl -n bluegreen create deployment app-green --image=nginx:1.25 --replicas=3
kubectl -n bluegreen patch deployment app-green -p \
  '{"spec":{"template":{"metadata":{"labels":{"version":"green","app":"myapp"}}}}}'

# Cut traffic over (edit Service selector to version=green)
kubectl -n bluegreen patch service myapp-svc -p \
  '{"spec":{"selector":{"app":"myapp","version":"green"}}}'
```
**Validation:** `kubectl -n bluegreen get endpoints myapp-svc -o wide` shows green pod IPs only after cutover.

### Lab B3: Debug and Fix a Stuck StatefulSet Rollout
```bash
kubectl create namespace ss-lab
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: ss-lab
spec:
  clusterIP: None
  selector: {app: web}
  ports: [{port: 80}]
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: ss-lab
spec:
  serviceName: web
  replicas: 3
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - name: web
        image: nginx
        readinessProbe:
          httpGet: {path: /this-path-does-not-exist, port: 80}
          periodSeconds: 5
EOF
kubectl -n ss-lab rollout status statefulset/web --timeout=30s || true
kubectl -n ss-lab get pods
kubectl -n ss-lab describe pod web-0
```
**Task:** Fix the readinessProbe path to `/` and observe the rollout complete. **Validation:** all 3 pods reach Ready.

---

## Category C: Networking

### Lab C1: Deploy and Compare Calico vs Cilium
```bash
# Cluster WITHOUT default CNI (kind supports this via disableDefaultCNI)
cat <<EOF > kind-no-cni.yaml
kind: Cluster
networking:
  disableDefaultCNI: true
nodes:
- role: control-plane
- role: worker
EOF
kind create cluster --name cni-lab --config kind-no-cni.yaml

# Install Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl get pods -n kube-system -l k8s-app=calico-node -w
```
**Validation:** all calico-node pods Running; test pod-to-pod connectivity across nodes with a busybox ping test. Repeat with Cilium's Helm chart for comparison, noting `cilium status` output.

### Lab C2: Debug Pod-to-Pod Connectivity Issues
```bash
kubectl create namespace netdebug
kubectl -n netdebug run pod-a --image=busybox --command -- sleep 3600
kubectl -n netdebug run pod-b --image=busybox --command -- sleep 3600
kubectl -n netdebug wait --for=condition=Ready pod/pod-a pod/pod-b

POD_B_IP=$(kubectl -n netdebug get pod pod-b -o jsonpath='{.status.podIP}')
kubectl -n netdebug exec pod-a -- ping -c3 $POD_B_IP
kubectl -n netdebug exec pod-a -- traceroute $POD_B_IP 2>/dev/null || \
  kubectl -n netdebug exec pod-a -- ping -c1 $POD_B_IP
```
**Task:** apply a NetworkPolicy that blocks this, then debug using `kubectl describe networkpolicy` and fix it. **Validation:** ping succeeds after policy fix.

### Lab C3: Trace Packet Flow with tcpdump and iptables
```bash
kubectl create namespace svc-lab
kubectl -n svc-lab create deployment web --image=nginx --replicas=3
kubectl -n svc-lab expose deployment web --port=80

# On a node (docker exec into kind node):
docker exec -it lab-cluster-worker iptables -t nat -L KUBE-SERVICES -n | head -20
docker exec -it lab-cluster-worker iptables -t nat -L -n | grep KUBE-SVC | head -5

# Capture traffic while curling the service ClusterIP from a test pod
kubectl -n svc-lab run tester --image=busybox --rm -it --restart=Never -- \
  wget -qO- web
```
**Validation:** Identify the DNAT rule matching the Service's ClusterIP and confirm it maps to one of the 3 pod IPs.

### Lab C4: Configure IPVS Mode and Compare
```bash
# Edit kube-proxy ConfigMap (kubeadm cluster)
kubectl -n kube-system edit configmap kube-proxy
# change mode: "" to mode: "ipvs"
kubectl -n kube-system rollout restart daemonset kube-proxy

# Verify
docker exec -it lab-cluster-worker ipvsadm -Ln
```
**Validation:** `ipvsadm -Ln` shows virtual servers for ClusterIPs instead of iptables NAT chains.

### Lab C5: Implement Default-Deny + Explicit Allow NetworkPolicy
```bash
kubectl create namespace secure-ns
kubectl -n secure-ns create deployment frontend --image=nginx
kubectl -n secure-ns create deployment backend --image=hashicorp/http-echo -- -text=ok
kubectl -n secure-ns expose deployment backend --port=5678

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to: [{namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: kube-system}}}]
    ports: [{protocol: UDP, port: 53}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: secure-ns
spec:
  podSelector: {matchLabels: {app: backend}}
  policyTypes: [Ingress]
  ingress:
  - from: [{podSelector: {matchLabels: {app: frontend}}}]
    ports: [{protocol: TCP, port: 5678}]
EOF
```
**Validation:** frontend→backend works, but a new pod without the `frontend` label CANNOT reach backend.

### Lab C6: Debug DNS Issues
```bash
kubectl -n secure-ns run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup backend.secure-ns.svc.cluster.local
kubectl -n secure-ns run dns-test2 --image=busybox --rm -it --restart=Never -- \
  cat /etc/resolv.conf
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50
```
**Task:** scale CoreDNS to 0 replicas temporarily, observe failures, restore, and measure recovery time.

---

## Category D: Storage

### Lab D1: Deploy a CSI Driver and Dynamic Provisioning
```bash
# Using local-path-provisioner (works well with kind)
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
kubectl get storageclass

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources: {requests: {storage: 1Gi}}
EOF
kubectl get pvc test-pvc -w
```
**Validation:** PVC transitions Pending → Bound once referenced by a pod (WaitForFirstConsumer behavior).

### Lab D2: Debug PVC Provisioning Failures
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bad-pvc
spec:
  accessModes: [ReadWriteMany]     # unsupported by local-path!
  storageClassName: local-path
  resources: {requests: {storage: 1Gi}}
EOF
kubectl describe pvc bad-pvc
```
**Validation:** identify the exact error message explaining RWX isn't supported by this provisioner; fix by changing to ReadWriteOnce.

### Lab D3: Implement Volume Snapshots
```bash
# Requires a CSI driver supporting snapshots (e.g. via csi-hostpath driver for lab purposes)
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```
**Task:** create a VolumeSnapshot from an existing PVC, then restore a new PVC from that snapshot and verify data.

---

## Category E: Security

### Lab E1: Harden Cluster with Pod Security Admission
```bash
kubectl create namespace hardened
kubectl label namespace hardened \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# This should be REJECTED:
kubectl -n hardened run bad-pod --image=nginx --privileged 2>&1 || true

# This should SUCCEED:
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  namespace: hardened
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities: {drop: ["ALL"]}
      readOnlyRootFilesystem: true
    volumeMounts:
    - {name: tmp, mountPath: /tmp}
    - {name: cache, mountPath: /var/cache/nginx}
    - {name: run, mountPath: /var/run}
  volumes:
  - {name: tmp, emptyDir: {}}
  - {name: cache, emptyDir: {}}
  - {name: run, emptyDir: {}}
EOF
```
**Validation:** privileged pod rejected with a clear PSA admission error; hardened pod runs successfully.

### Lab E2: Set Up OIDC-Style RBAC and Audit Permissions
```bash
kubectl create namespace team-a
kubectl create serviceaccount dev-user -n team-a
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: pod-reader, namespace: team-a}
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: dev-user-binding, namespace: team-a}
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: team-a
roleRef: {kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io}
EOF

kubectl auth can-i create pods -n team-a --as=system:serviceaccount:team-a:dev-user
kubectl auth can-i get pods -n team-a --as=system:serviceaccount:team-a:dev-user
kubectl auth can-i get pods -n default --as=system:serviceaccount:team-a:dev-user
```
**Validation:** confirm dev-user CAN get pods in team-a, CANNOT create pods, CANNOT get pods in default namespace.

### Lab E3: Configure etcd Encryption at Rest
```bash
cat <<EOF > /etc/kubernetes/enc/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: $(head -c 32 /dev/urandom | base64)
  - identity: {}
EOF
# Add --encryption-provider-config=/etc/kubernetes/enc/enc.yaml to kube-apiserver manifest
kubectl create secret generic test-secret --from-literal=password=supersecret

# Verify it's encrypted in etcd (raw inspection)
ETCDCTL_API=3 etcdctl get /registry/secrets/default/test-secret \
  --endpoints=https://127.0.0.1:2379 --cacert=... --cert=... --key=... | hexdump -C | head
```
**Validation:** raw etcd value begins with `k8s:enc:aescbc:v1:` prefix (not plaintext base64).

---

## Category F: Observability

### Lab F1: Deploy Prometheus Operator Stack
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl -n monitoring get pods
kubectl -n monitoring port-forward svc/kps-grafana 3000:80
```
**Validation:** Grafana reachable on localhost:3000, default dashboards showing cluster metrics.

### Lab F2: Create a Custom ServiceMonitor
```bash
kubectl create namespace demo-app
kubectl -n demo-app create deployment metrics-app --image=quay.io/brancz/prometheus-example-app:v0.5.0
kubectl -n demo-app expose deployment metrics-app --port=8080 --target-port=8080
kubectl -n demo-app label service metrics-app app=metrics-app

cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: metrics-app
  namespace: demo-app
  labels: {release: kps}
spec:
  selector: {matchLabels: {app: metrics-app}}
  endpoints: [{port: "8080", interval: 15s}]
EOF
```
**Validation:** target appears as "up" in Prometheus targets page.

### Lab F3: Deploy Loki + Promtail for Centralized Logging
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack -n monitoring --set promtail.enabled=true
```
**Validation:** query logs in Grafana's Explore view filtered by namespace label.

---

## Category G: Troubleshooting Drills

### Lab G1: Debug Pod Scheduling Failures
```bash
kubectl create namespace sched-lab
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: huge-pod, namespace: sched-lab}
spec:
  containers:
  - name: app
    image: nginx
    resources: {requests: {cpu: "100", memory: "500Gi"}}
EOF
kubectl -n sched-lab describe pod huge-pod
```
**Task:** identify the FailedScheduling reason, fix by reducing requests to realistic values.

### Lab G2: Investigate a CPU-Throttled Pod
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: throttle-lab, namespace: sched-lab}
spec:
  containers:
  - name: stress
    image: polinux/stress
    args: ["--cpu", "4"]
    resources:
      requests: {cpu: "100m"}
      limits: {cpu: "100m"}
EOF
kubectl -n sched-lab top pod throttle-lab
# Compare against container_cpu_cfs_throttled_periods_total in Prometheus
```
**Validation:** confirm the pod is CPU-limited despite node having spare capacity; explain via CFS quota mechanism.

### Lab G3: Simulate and Recover from etcd Member Failure
```bash
# On a 3-node kubeadm cluster, stop etcd on one control-plane node:
docker exec -it <cp-node-2> systemctl stop etcd  # or move static pod manifest out
etcdctl endpoint health --cluster
# Observe: cluster still functions (2 of 3 quorum)
# Restart etcd, confirm rejoin
docker exec -it <cp-node-2> systemctl start etcd
etcdctl endpoint health --cluster
```
**Validation:** API server remains available throughout with 1 member down; full health restored after recovery.

---

## Lab Completion Checklist

```
□ A1-A3: Cluster setup and etcd DR
□ B1-B3: Workload deployment strategies and StatefulSet debugging
□ C1-C6: CNI, Services, NetworkPolicy, DNS
□ D1-D3: CSI storage provisioning and snapshots
□ E1-E3: PSA, RBAC, encryption at rest
□ F1-F3: Prometheus, ServiceMonitor, Loki logging
□ G1-G3: Scheduling, CPU throttling, etcd failure recovery
```

Complete all 30+ labs before your interview — being able to say "I've actually done this hands-on" for troubleshooting questions is a strong differentiator at the senior/staff level.
