# Section 11: Services & Ingress

Services and Ingress are the primary mechanisms for exposing workloads within and outside the cluster. While Section 8 (Networking) covered the packet-level mechanics of how kube-proxy implements Services, this section focuses on the API semantics, behavioral differences between Service types, Ingress controller internals, and production patterns for exposing applications securely and reliably.

## Subtopic Index

- [ClusterIP](#clusterip)
- [NodePort](#nodeport)
- [LoadBalancer](#loadbalancer)
- [ExternalName](#externalname)
- [Headless Service](#headless-service)
- [Session Affinity](#session-affinity)
- [EndpointSlice](#endpointslice)
- [Service Topology and externalTrafficPolicy](#service-topology-and-externaltrafficpolicy)
- [Ingress](#ingress)
- [NGINX Ingress Controller](#nginx-ingress-controller)
- [Traefik](#traefik)
- [AGIC — Application Gateway Ingress Controller](#agic--application-gateway-ingress-controller)
- [Gateway API](#gateway-api)

---

## ClusterIP

A ClusterIP Service exposes a stable virtual IP reachable only from within the cluster. It is the default Service type and the foundation for all in-cluster service discovery. The ClusterIP is allocated from the Service CIDR at creation and never changes, even as backing pods come and go.

kube-proxy watches Services and EndpointSlices. For each ClusterIP Service, it programs iptables or IPVS rules that DNAT packets destined for the ClusterIP to one of the ready backend pod IPs. The virtual IP does not exist on any interface — it only exists as a matching rule in iptables/IPVS. If you `ping` a ClusterIP, the ICMP packet matches no rule (ICMP isn't a listed port) and is dropped.

The DNS name for a ClusterIP Service is `<service>.<namespace>.svc.cluster.local`. CoreDNS resolves this to the ClusterIP. The application connects to the ClusterIP; kube-proxy handles load balancing transparently.

**Port fields**: `spec.port` (the port clients connect to on the ClusterIP), `spec.targetPort` (the port on the pod), `spec.protocol` (TCP/UDP/SCTP). Named ports allow `targetPort` to reference a named port in the pod's container spec, which enables changing the container's actual port without updating the Service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-api
  namespace: production
spec:
  type: ClusterIP        # default, can be omitted
  selector:
    app: payments
  ports:
  - name: http
    port: 80             # ClusterIP port (what clients call)
    targetPort: 8080     # pod port (what the app listens on)
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: metrics  # named port — matches container.ports[].name
```

### Key commands
```bash
# Get ClusterIP
kubectl get service payments-api -o jsonpath='{.spec.clusterIP}'

# Test from within cluster
kubectl run curl --image=curlimages/curl --restart=Never --rm -it -- \
  curl http://payments-api.production.svc.cluster.local/health

# See all Services and their ClusterIPs
kubectl get service -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,TYPE:.spec.type,CLUSTER-IP:.spec.clusterIP,PORT:.spec.ports[*].port

# Verify iptables rules for a ClusterIP
SVC_IP=$(kubectl get svc payments-api -o jsonpath='{.spec.clusterIP}')
iptables-save | grep $SVC_IP
```

---

## NodePort

NodePort extends ClusterIP by additionally exposing the Service on a static port (30000–32767) on every node's IP. Traffic reaching `<any-node-ip>:30080` is forwarded to the Service's backend pods.

kube-proxy programs an additional iptables rule in the PREROUTING and INPUT chains matching on `dport=30080`. This rule jumps to the same `KUBE-SVC-<hash>` chain used for ClusterIP access, so load balancing behavior is identical.

NodePort is often used as the underlying mechanism for cloud LoadBalancer Services and Ingress controllers. The cloud load balancer routes external traffic to one of the cluster nodes on the NodePort, and kube-proxy handles the rest.

The NodePort range (`30000-32767`) is configurable via apiserver flag `--service-node-port-range`. Ports below 30000 can be requested with `spec.ports[].nodePort` if the apiserver allows it (requires `--service-node-port-range` adjustment).

**Anti-pattern**: exposing Services directly via NodePort for production traffic. Use LoadBalancer or Ingress for external traffic. NodePorts bypass many production features: no TLS termination, no L7 routing, no health-check-aware load balancing, and every node exposes the port (increasing attack surface).

### Key commands
```bash
# Create a NodePort Service
kubectl expose deployment my-app --type=NodePort --port=80

# Get the assigned NodePort
kubectl get service my-app -o jsonpath='{.spec.ports[*].nodePort}'

# Access via any node
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
NODE_PORT=$(kubectl get service my-app -o jsonpath='{.spec.ports[0].nodePort}')
curl http://$NODE_IP:$NODE_PORT/health

# Check iptables NodePort rules
iptables-save | grep NODEPORTS | head -10
```

---

## LoadBalancer

A LoadBalancer Service provisions an external cloud load balancer that routes traffic to the cluster nodes. The cloud-controller-manager (or a dedicated controller like AWS Load Balancer Controller) watches for Services of type LoadBalancer and calls the cloud API to provision the LB.

The allocated external IP/hostname appears in `status.loadBalancer.ingress[*]`. For AWS NLB/ALB: a DNS hostname; for GCP/Azure: an IP address. The LB routes traffic to all cluster nodes on the NodePort assigned to the Service, and kube-proxy handles the rest inbound.

**Annotations drive LB behavior**. Each cloud provider has its own annotation set:

```yaml
# AWS NLB via AWS Load Balancer Controller
service.beta.kubernetes.io/aws-load-balancer-type: external
service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip    # bypass NodePort, route directly to pod IPs
service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

# Azure
service.beta.kubernetes.io/azure-load-balancer-internal: "true"     # internal LB (private IP)

# GCP
cloud.google.com/load-balancer-type: Internal                        # internal LB
```

`spec.loadBalancerSourceRanges` restricts which IP CIDRs can access the LB — the cloud LB enforces this with security group rules. `spec.loadBalancerIP` requests a specific IP if the provider supports it (GCP does; AWS NLB with EIP does; Azure does with static IP resources).

### Key commands
```bash
# Create a LoadBalancer Service
kubectl expose deployment my-app --type=LoadBalancer --port=80

# Watch for external IP assignment
kubectl get service my-app -w

# Get the external IP/hostname
kubectl get service my-app -o jsonpath='{.status.loadBalancer.ingress[*].ip}'
kubectl get service my-app -o jsonpath='{.status.loadBalancer.ingress[*].hostname}'

# Check if cloud controller is provisioning (watch logs)
kubectl -n kube-system logs -l app=aws-load-balancer-controller --tail=30 -f
```

---

## ExternalName

An ExternalName Service creates a DNS CNAME alias for an external hostname. It has no ClusterIP, no kube-proxy rules, and no backend pods. When a pod resolves `my-db.namespace.svc.cluster.local`, CoreDNS returns a CNAME record pointing to the external hostname (e.g., `db.example.com`).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  type: ExternalName
  externalName: prod-db.cluster.us-east-1.rds.amazonaws.com
  # No selector, no ports, no ClusterIP
```

Use cases: migrating from an external database to an internal one (change `externalName` to point to the new internal Service, no application changes needed), abstracting external services behind a stable in-cluster DNS name.

**Caveats**: ExternalName does not work for all protocols. TLS/SNI issues arise because the application connects to the alias (`my-db.namespace.svc.cluster.local`) but the TLS certificate is for `prod-db.cluster.us-east-1.rds.amazonaws.com`. HTTP redirects can also cause issues if the external host returns a redirect containing its hostname. Use Interface Endpoints (VPC endpoints, Private Link) for internal-only access to cloud services where possible.

### Key commands
```bash
# Verify ExternalName resolution
kubectl run dns-test --image=busybox --restart=Never --rm -it -- \
  nslookup external-db.production.svc.cluster.local
# Should return CNAME → prod-db.cluster.us-east-1.rds.amazonaws.com

# Check ExternalName
kubectl get service external-db -o jsonpath='{.spec.externalName}'
```

---

## Headless Service

A Headless Service (`spec.clusterIP: None`) has no virtual IP. DNS for a Headless Service returns the IP addresses of individual pods directly (A records for each ready pod), rather than a single ClusterIP.

For StatefulSets, the Headless Service provides stable per-pod DNS: `pod-0.<svc>.<ns>.svc.cluster.local → <pod-0-IP>`. This is how Cassandra seeds, Kafka brokers, and ZooKeeper nodes find each other.

For Deployments, a Headless Service returns all ready pod IPs in DNS. Applications can perform client-side load balancing by resolving the hostname and connecting to one of the returned IPs — used by gRPC (which opens persistent connections and needs all backend IPs for load balancing).

kube-proxy does not create rules for Headless Services — there's nothing to DNAT since there's no ClusterIP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mydb-headless
spec:
  clusterIP: None    # headless
  selector:
    app: mydb
  ports:
  - port: 5432
    targetPort: 5432
```

### Key commands
```bash
# Verify headless DNS returns individual pod IPs
kubectl exec <pod> -- nslookup mydb-headless.default.svc.cluster.local
# Returns: multiple A records (one per ready pod)

# Compare with regular ClusterIP Service
kubectl exec <pod> -- nslookup my-regular-service.default.svc.cluster.local
# Returns: single A record (the ClusterIP)

# Check pod-specific DNS for StatefulSet pods
kubectl exec <pod> -- nslookup mydb-0.mydb-headless.default.svc.cluster.local
```

---

## Session Affinity

Session affinity (sticky sessions) routes all requests from the same source IP to the same backend pod. This is useful for applications with in-memory session state that hasn't been externalized.

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800   # 3 hours
```

kube-proxy implements this with an iptables `recent` module that records the ClusterIP→pod mapping for each source IP. For `timeoutSeconds` duration, all connections from the same source IP go to the same pod. After the timeout, the next connection is load-balanced again.

**Limitations**: session affinity by source IP breaks when clients are behind a NAT or proxy (multiple users appear as the same source IP). It also creates uneven load distribution over time as long-lived sessions keep hitting the same pods. The preferred production pattern is stateless applications with externalized session storage (Redis, database), not session affinity.

Session affinity is not the same as hash-based consistent routing (which can be done in Ingress controllers with cookies or in service meshes with consistent hash load balancing).

---

## EndpointSlice

EndpointSlice (GA in 1.21) replaced the older Endpoints resource for tracking ready pod IPs behind a Service. The Endpoints resource had a fundamental scalability problem: one large object per Service, updated atomically on every pod add/remove. A Service with 1000 pods produces a 100KB Endpoints object, and every pod restart rewrites the entire object.

EndpointSlices shard a Service's endpoints into chunks of up to 100 endpoints per slice. A Service with 1000 pods has 10 EndpointSlice objects. Adding or removing one pod updates one slice — 10x less write traffic to etcd and 10x less watch traffic to kube-proxy.

Each endpoint in an EndpointSlice has: `addresses` (pod IPs), `conditions` (ready, serving, terminating), `hostname`, `nodeName`, `zone`, and `targetRef` (the Pod object). The `serving` condition is set to true when the pod is serving traffic even during graceful termination — used by kube-proxy to keep the endpoint in rotation until the pod actually stops responding.

The EndpointSlice controller watches Services and Pods. For topology-aware routing, it annotates EndpointSlice endpoints with `topology.kubernetes.io/zone` hints so kube-proxy can prefer endpoints in the same zone as the requesting client.

### Key commands
```bash
# List EndpointSlices for a Service
kubectl get endpointslice -l kubernetes.io/service-name=payments-api

# Inspect endpoints including readiness/terminating state
kubectl get endpointslice -l kubernetes.io/service-name=payments-api -o yaml | \
  grep -A10 'endpoints:'

# Watch endpoint changes during a rolling deploy
kubectl get endpointslice -l kubernetes.io/service-name=payments-api -w

# Compare with old Endpoints resource
kubectl get endpoints payments-api
```

---

## Service Topology and externalTrafficPolicy

`externalTrafficPolicy` controls how external traffic (NodePort/LoadBalancer) is handled at the node level.

`Cluster` (default): the receiving node may forward traffic to a pod on any node. Return traffic is routed back through the same node via SNAT. The pod sees the node IP, not the real client IP. Distribution is even across all pods.

`Local`: the receiving node only forwards to pods running on that same node. No SNAT — the pod sees the real client IP. If no local pod exists, the connection is dropped. Cloud LBs use the per-node health check endpoint (`/healthz/ready` at `spec.healthCheckNodePort`) to detect which nodes have ready local pods and route only to those.

**When to use Local**: when you need real client IPs for rate limiting, geo-routing, logging, or security. When you need to avoid cross-node traffic costs. Be aware that pod distribution across nodes becomes critical — uneven distribution causes uneven load on remaining nodes.

**Topology-aware routing** (k8s 1.23+, `spec.internalTrafficPolicy: Local` and EndpointSlice hints): for ClusterIP Services, routes traffic to endpoints in the same zone as the calling pod when possible, reducing cross-zone data transfer costs. The EndpointSlice controller adds zone hints based on the zone distribution of endpoints.

### Key commands
```bash
# Check externalTrafficPolicy
kubectl get service my-service -o jsonpath='{.spec.externalTrafficPolicy}'

# Check healthCheckNodePort (for Local policy)
kubectl get service my-service -o jsonpath='{.spec.healthCheckNodePort}'
curl http://<node-ip>:<healthCheckNodePort>/healthz/ready

# Check internalTrafficPolicy
kubectl get service my-service -o jsonpath='{.spec.internalTrafficPolicy}'

# Monitor zone distribution of traffic (using EndpointSlice hints)
kubectl get endpointslice -l kubernetes.io/service-name=my-service -o yaml | grep hints -A5
```

---

## Ingress

An Ingress resource defines L7 HTTP/HTTPS routing rules: which hostname and URL path maps to which backend Service. Ingress is the standard Kubernetes way to expose multiple Services through a single external IP/LB using virtual hosting and path-based routing.

The Ingress API itself does nothing — it requires an Ingress controller to read and implement the rules. The `spec.ingressClassName` (or `kubernetes.io/ingress.class` annotation for older clusters) selects which controller handles the Ingress.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /     # controller-specific
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts: [payments.example.com]
    secretRef:
      name: payments-tls         # Secret with tls.crt and tls.key
  rules:
  - host: payments.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: payments-api
            port: {number: 80}
      - path: /admin
        pathType: Exact
        backend:
          service:
            name: payments-admin
            port: {number: 8080}
```

`pathType` matters: `Exact` matches exactly `/admin` (not `/admin/`). `Prefix` matches `/api`, `/api/v1`, `/api/anything`. `ImplementationSpecific` is controller-defined.

TLS: the Ingress controller reads the specified Secret (type `kubernetes.io/tls`) for the certificate and private key, and serves HTTPS. cert-manager can automatically issue and renew certificates by watching Ingress objects with appropriate annotations.

**Limitations**: Ingress only handles HTTP/HTTPS. It has no native TCP/UDP routing. Annotations are implementation-specific (not portable between controllers). These limitations drove Gateway API.

### Key commands
```bash
# Create an Ingress
kubectl apply -f ingress.yaml

# Check Ingress status (shows assigned load balancer)
kubectl describe ingress prod-ingress
kubectl get ingress prod-ingress -o jsonpath='{.status.loadBalancer.ingress[*].ip}'

# List IngressClasses
kubectl get ingressclass

# Test routing (from outside cluster)
curl -H "Host: payments.example.com" https://<ingress-ip>/api/health -k

# Check TLS certificate
kubectl get secret payments-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout | grep -E 'Subject:|DNS:'
```

---

## NGINX Ingress Controller

NGINX Ingress Controller (nginx.ingress.kubernetes.io) is the most widely deployed Ingress controller. It runs as a Deployment exposed via a LoadBalancer Service, watches Ingress objects, and dynamically configures NGINX.

**Architecture**: the controller watches Ingress, Service, EndpointSlice, and Secret objects. When any changes, it generates a new NGINX config and either reloads NGINX (`nginx -s reload`) or uses the NGINX Plus dynamic API for zero-reload updates. The generated config maps Ingress rules to NGINX `server` and `location` blocks.

**TLS termination**: the controller reads the referenced Secret and writes the cert/key to a temp file that NGINX reads. For wildcard certs or cert-manager integration, it watches Certificate objects and triggers reloads when certs rotate.

**Annotations** extend NGINX configuration at the Ingress level:
- `nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"` — NGINX upstream connect timeout
- `nginx.ingress.kubernetes.io/proxy-read-timeout: "60"` — upstream read timeout
- `nginx.ingress.kubernetes.io/rate-limit: "100"` — per-IP rate limiting (uses `limit_req_zone`)
- `nginx.ingress.kubernetes.io/auth-url` — external authentication
- `nginx.ingress.kubernetes.io/canary: "true"` + `canary-weight: "20"` — traffic weighting for canary

**Performance tuning**: `worker_processes` should equal CPU cores. `worker_connections` should account for concurrent connections. `keepalive` to backends reduces connection overhead. For high-throughput, the controller can be deployed with multiple replicas behind a LoadBalancer with session stickiness.

### Key commands
```bash
# Check NGINX controller pods and version
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx exec -it <pod> -- nginx -v

# View generated NGINX config (shows how Ingress rules are translated)
kubectl -n ingress-nginx exec -it <pod> -- cat /etc/nginx/nginx.conf | grep -A20 'payments.example.com'

# Check NGINX error/access logs
kubectl -n ingress-nginx logs <pod> --tail=50

# Check NGINX metrics (Prometheus endpoint)
kubectl -n ingress-nginx exec <pod> -- curl -s localhost:10254/metrics | grep nginx_ingress

# Test that a specific Ingress rule routes correctly
kubectl -n ingress-nginx exec <pod> -- curl -H "Host: payments.example.com" localhost/api/health
```

---

## Traefik

Traefik is an edge router and Ingress controller that auto-discovers configuration from Kubernetes objects. It supports Ingress, IngressRoute CRDs (its own extended API), and Gateway API.

Traefik's key differentiator is automatic certificate management (built-in Let's Encrypt ACME client) and dynamic configuration without restarts. When an Ingress or IngressRoute is updated, Traefik applies the change immediately — no config reload needed.

Traefik uses **providers** as configuration sources: Kubernetes Ingress, Kubernetes CRD (IngressRoute), file, Docker, etc. All providers are watched simultaneously. Its `IngressRoute` CRD supports features Ingress doesn't: TCP routing, middleware chaining (rate limiting, auth, header manipulation, circuit breaker), service mirroring, and weighted traffic splitting.

```yaml
# Traefik IngressRoute (more expressive than standard Ingress)
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: payments-route
spec:
  entryPoints: [websecure]
  routes:
  - match: Host(`payments.example.com`) && PathPrefix(`/api`)
    kind: Rule
    services:
    - name: payments-api
      port: 80
      weight: 90
    - name: payments-api-v2
      port: 80
      weight: 10      # 10% canary traffic
    middlewares:
    - name: rate-limit
  tls:
    certResolver: letsencrypt
```

### Key commands
```bash
# Traefik dashboard (if enabled)
kubectl port-forward -n traefik svc/traefik 9000:9000 &
# open http://localhost:9000/dashboard/

# Check Traefik routing rules
kubectl -n traefik exec <pod> -- traefik healthcheck

# List IngressRoutes
kubectl get ingressroute -A

# Check Traefik logs for routing issues
kubectl -n traefik logs <pod> | grep -E 'error|Error' | tail -20
```

---

## AGIC — Application Gateway Ingress Controller

AGIC (Application Gateway Ingress Controller) is Azure-specific. It translates Kubernetes Ingress resources into Azure Application Gateway configuration. The Application Gateway (AGW) is an Azure L7 load balancer with WAF capabilities, deployed outside the cluster.

AGIC runs as a pod inside the cluster and uses the Azure Resource Manager (ARM) API to configure the Application Gateway. Unlike in-cluster controllers (NGINX, Traefik), AGIC doesn't run a proxy — the AGW itself handles TLS termination, routing, and WAF.

**Trade-offs vs NGINX**:
- AGW is a cloud managed service — Microsoft maintains HA, patching, scaling. No pod failures due to controller issues.
- AGW has a limited feature set vs NGINX annotations — complex rewrites, custom auth, etc. require AGW Policy.
- Changes to AGW take 30–90 seconds to propagate (ARM API round-trip), vs seconds for in-cluster controllers.
- Cost: AGW is billed by the hour plus data processed; NGINX controller adds pod resource cost only.

```yaml
# Ingress with AGIC
metadata:
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/backend-path-prefix: "/api"
    appgw.ingress.kubernetes.io/request-timeout: "30"
    appgw.ingress.kubernetes.io/waf-policy-for-path: "/subscriptions/.../wafPolicies/myPolicy"
```

### Key commands
```bash
# Check AGIC pod
kubectl -n kube-system get pods -l app=ingress-appgw

# AGIC logs (shows ARM API calls and sync status)
kubectl -n kube-system logs -l app=ingress-appgw --tail=50

# Verify Application Gateway backend health in Azure
az network application-gateway show-backend-health --name <agw-name> --resource-group <rg>

# Check AGW probe status
az network application-gateway probe list --gateway-name <agw-name> --resource-group <rg>
```

---

## Gateway API

Gateway API is the evolution beyond Ingress. It's a first-class Kubernetes API (not just an annotation hack) that supports TCP, UDP, TLS, gRPC, and HTTP routing with a role-based model where infrastructure teams and application teams have separate resources.

The four core resources: **GatewayClass** (infrastructure provider defines the controller), **Gateway** (cluster operator provisions the listener — port, protocol, TLS), **HTTPRoute / TCPRoute / TLSRoute / GRPCRoute** (application team defines routing rules in their namespace).

```yaml
# Cluster operator: provision the gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: infra
spec:
  gatewayClassName: nginx-gateway-fabric
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: wildcard-cert
        namespace: infra
    allowedRoutes:
      namespaces:
        from: Selector         # only allow routes from labeled namespaces
        selector:
          matchLabels:
            gateway-access: "true"
---
# Application team: define routing in their namespace
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: payments-route
  namespace: payments   # different namespace from Gateway
spec:
  parentRefs:
  - name: prod-gateway
    namespace: infra
    sectionName: https
  hostnames: ["payments.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: X-Gateway-Source
          value: "prod-gateway"
    backendRefs:
    - name: payments-api
      port: 80
      weight: 95
    - name: payments-api-canary
      port: 80
      weight: 5
```

Policy attachment extends Gateway API: `HTTPRoutePolicy`, `BackendLBPolicy`, `BackendTLSPolicy` attach behavior to routes or backends without modifying the route itself — used for timeouts, retry policies, and mTLS to backends.

**Gateway API vs Ingress**: Ingress requires annotations for features like traffic weighting, header manipulation, and retry — all implementation-specific. Gateway API has first-class fields for these. Ingress doesn't support TCP/UDP; Gateway API has TCPRoute and UDPRoute. Ingress has no multi-tenancy model; Gateway API's role separation is built in.

### Key commands
```bash
# List Gateway API resources
kubectl get gatewayclasses
kubectl get gateways -A
kubectl get httproutes -A
kubectl get tcproutes -A

# Check Gateway status
kubectl describe gateway prod-gateway -n infra | grep -A20 Status

# Check HTTPRoute conditions
kubectl describe httproute payments-route -n payments | grep -A20 Status

# Test weighted routing (run 100 requests, check distribution)
for i in $(seq 100); do curl -s https://payments.example.com/api/version | grep version; done | sort | uniq -c

# Check if route is attached to gateway
kubectl get httproute payments-route -n payments -o jsonpath='{.status.parents}'
```

---

## Interview Questions

### Conceptual / Internals (8 questions)

**1. What is the difference between a ClusterIP, NodePort, and LoadBalancer Service at the kube-proxy implementation level?**

ClusterIP: kube-proxy adds iptables rules to KUBE-SERVICES chain matching on `dst=ClusterIP:port`, jumping to KUBE-SVC chain which selects a pod via probability-weighted KUBE-SEP rules and applies DNAT. Traffic is only reachable from within the cluster. NodePort: extends ClusterIP by additionally adding iptables rules in PREROUTING and INPUT chains matching on `dpt=<nodeport>` across all interfaces (`0.0.0.0/0`), jumping to the same KUBE-SVC chain. Any traffic reaching any node IP on that port is processed identically to ClusterIP traffic. LoadBalancer: identical to NodePort in kube-proxy terms, but additionally a cloud controller (CCM or AWS LBC) creates an external load balancer resource pointing to the nodes' NodePort. The LB handles external TLS termination, health checking of nodes, and cross-region routing. kube-proxy's role is the same as NodePort — it doesn't know about the external LB.

**2. Why does a Headless Service with `clusterIP: None` not require kube-proxy rules, and how does DNS resolution differ?**

A ClusterIP Service has a virtual IP that needs iptables/IPVS rules to translate from the VIP to a real pod IP. The VIP doesn't exist on any interface — kube-proxy makes it work by intercepting packets before routing decisions. A Headless Service has no VIP, so there's nothing to intercept. DNS for the Headless Service returns actual pod IPs directly. Clients get real IPs from DNS and connect directly to pods — no DNAT, no iptables traversal, no kube-proxy involvement. This is more efficient (fewer hops) but shifts load balancing to the client: the client must implement its own selection from the multiple returned IPs. gRPC clients do this naturally (they resolve all IPs and maintain per-backend connections). Regular HTTP clients that only use the first returned IP are effectively pinned to one pod.

**3. Explain how `externalTrafficPolicy: Local` preserves client IPs but introduces availability risks.**

With `Cluster`: incoming external traffic at `nodeIP:nodePort` is SNAT-ed before forwarding to a pod on a different node. The source IP is rewritten to the receiving node's IP, so the destination pod sees the node IP as client. With `Local`: incoming traffic is forwarded only to pods on the same node. No SNAT — the original source IP is preserved in the packet forwarded to the local pod. If no local pod exists, the connection is dropped. The availability risk: the cloud LB doesn't inherently know which nodes have local pods. If relying on NodePort with `Local`, a client connecting to a node with no local pod gets a dropped connection. The fix is the per-node health check endpoint (`spec.healthCheckNodePort`): the LB performs health checks on this port, and kube-proxy responds 200 only when a local pod is ready, 503 when not. The LB then only routes to nodes that pass the health check.

**4. How does the NGINX Ingress Controller apply configuration changes, and what is the difference between a reload-based update and a socket update?**

When an Ingress object changes, the controller updates its in-memory model and generates a new NGINX configuration file. For a reload-based update: `nginx -s reload` sends SIGHUP to the NGINX master process, which forks new worker processes with the new config and gracefully drains existing workers. This causes a brief period where some workers have old config and new workers have new config — not an outage but a small inconsistency window. For NGINX Plus (or open-source NGINX with Lua), the dynamic config API allows updating upstreams (backend pod IPs) without a reload: a Lua module hooks into upstream updates and calls the NGINX Plus API to add/remove servers. This provides truly zero-reload endpoint updates. kube-proxy changes (pod added/removed) can be applied without NGINX reload; only routing rule changes (new Ingress, path change) require reload.

**5. What is the EndpointSlice's `terminating` condition and how does kube-proxy use it?**

When a pod is terminating (deletionTimestamp is set but containers haven't exited), the EndpointSlice controller sets `conditions.terminating: true` on that endpoint. It also maintains `conditions.serving: true` until the pod actually stops passing its readiness probe or exits. kube-proxy (in IPVS mode with `--feature-gates=EndpointSliceTerminatingCondition=true`) can keep a terminating endpoint in the load balancing rotation as long as `serving: true`, even though `ready: false`. This allows a pod to finish processing in-flight requests during its terminationGracePeriodSeconds while not receiving new connections. When the pod actually stops serving (readiness fails or containers exit), `serving` becomes false and kube-proxy removes it from rotation. In iptables mode, kube-proxy uses `ready` only by default, so a pod is immediately removed from rotation when readiness fails, even if it's still processing requests.

**6. Explain how cert-manager integrates with Ingress to automate TLS certificate issuance and renewal.**

cert-manager watches Ingress objects for a `cert-manager.io/cluster-issuer` (or `issuer`) annotation. When found, it reads the Ingress's `spec.tls[*].secretName` and the hostnames. It creates a `Certificate` object targeting those hostnames and the specified Secret. The Certificate object triggers cert-manager's controllers to: (1) create an `Order` with the ACME provider, (2) create a `Challenge` (HTTP-01 or DNS-01), (3) for HTTP-01: temporarily expose a well-known path on the Ingress by creating a temporary Ingress rule, wait for ACME validation, then clean up. (4) Store the issued certificate in the specified Secret (type `kubernetes.io/tls`). The Ingress controller reads this Secret for TLS. Before expiry (usually 30 days before), cert-manager renews by creating a new Order and updating the Secret. The Ingress controller detects the Secret update and reloads NGINX with the new certificate.

**7. How does Gateway API's role separation improve on Ingress for multi-tenant clusters?**

In Ingress, both the infrastructure team (who manages the load balancer) and application teams (who define routing rules) use the same resource type. There's no API-level separation — an application team can modify Ingress to change the load balancer configuration, causing security or operational issues. Gateway API separates: the cluster operator creates `Gateway` (listener configuration, TLS, allowed route namespaces). Application teams create `HTTPRoute` in their namespaces. The Gateway's `allowedRoutes.namespaces` restricts which namespaces can attach routes — enforced by the API server, not by human trust. An application team cannot route traffic for a hostname they don't own without the cluster operator's explicit permission (Gateway must be configured to allow routes from their namespace, and the route must reference a hostname the Gateway listener accepts). This enables true multi-tenancy: teams manage their routes independently without cluster-admin access.

**8. What happens to in-flight requests when a Service's EndpointSlice is updated to remove a pod during a rolling deploy?**

The EndpointSlice controller sets the pod's endpoint to `ready: false` and optionally `serving: false` when the pod becomes unready. kube-proxy propagates this change to its iptables/IPVS rules and removes the pod from the load-balancing set. New connections no longer go to this pod. In-flight connections (TCP connections already established before the rule update) continue using the existing conntrack entry — conntrack maintains the NAT mapping independently of iptables rules. The conntrack entry remains valid until the connection closes or times out. So in-flight long-lived connections (HTTP keep-alive, WebSocket, gRPC streams) are NOT dropped when the endpoint is removed — they continue until the pod closes them or the client reconnects. Only new connections are prevented from reaching the removed pod. This is why preStop sleep + graceful shutdown (server.Shutdown with timeout) is the correct pattern: the pod finishes in-flight requests (which continue via conntrack) and then exits cleanly.

---

### Scenario / Troubleshooting (6 questions)

**9. A Service's external LoadBalancer is routing traffic, but 15% of requests return connection refused. Diagnose.**

With `externalTrafficPolicy: Cluster` this shouldn't happen — all pods receive traffic. With `Local`, nodes without ready local pods return connection refused if the LB doesn't respect healthCheckNodePort. Check: `kubectl get service -o jsonpath='{.spec.externalTrafficPolicy}'`. If Local: `kubectl get service -o jsonpath='{.spec.healthCheckNodePort}'`. Test the health endpoint on each node: `curl http://<node-ip>:<healthCheckNodePort>/healthz/ready`. Nodes returning 503 have no local pods but may still receive traffic if the cloud LB doesn't use the health check. Fix: ensure the LB is configured to use the healthCheckNodePort, or switch to `externalTrafficPolicy: Cluster`.

**10. An application uses a ClusterIP Service to connect to a database. Connections are refused intermittently. The database pod is running and healthy. Diagnose.**

Check EndpointSlices: `kubectl get endpointslice -l kubernetes.io/service-name=<db-svc>` — confirm the pod IP is in the endpoints with `ready: true`. If the pod is failing its readiness probe intermittently, it's removed from endpoints intermittently — causing connection failures. Check: `kubectl describe pod <db-pod>` for probe failures. Next: check conntrack table — if entries are maxed, new connections fail: `conntrack -C` vs `cat /proc/sys/net/netfilter/nf_conntrack_max`. Check kube-proxy: `kubectl -n kube-system logs -l k8s-app=kube-proxy | grep error`. Also check: does the application reuse connections (connection pool)? If it creates new connections per request and the pod IP changes (pod restart), conntrack entries referencing the old pod IP are stale. Ensure the application uses Service DNS, not pod IPs directly.

**11. An Ingress routes traffic for `payments.example.com` but returns 404 for `/api/v1`. Debug the Ingress routing.**

Check the Ingress spec: `kubectl describe ingress prod-ingress` — read the `Rules` section. Verify the path is `/api` (Prefix) not `/api/v1` (Exact). Check pathType: `Exact` matches exactly `/api/v1`; `Prefix` matches `/api/v1` and all sub-paths. Check NGINX rewrite annotations — `nginx.ingress.kubernetes.io/rewrite-target: /` rewrites the path to `/` on the backend. If the upstream expects `/api/v1` but the rewrite sends `/`, the backend returns 404. Use `$request_uri` in logs to see what path NGINX sends to the backend: `kubectl -n ingress-nginx exec <pod> -- grep "payments.example.com" /var/log/nginx/access.log | tail -5`. Test directly: `kubectl -n ingress-nginx exec <pod> -- curl -H "Host: payments.example.com" localhost/api/v1`.

**12. A Gateway API HTTPRoute is configured with weights 90/10 but traffic shows 100% going to the primary backend. What's wrong?**

Check if the Gateway API controller is installed and running: `kubectl get gatewayclasses`. If the GatewayClass doesn't have a registered controller, the Gateway and HTTPRoute exist in etcd but nothing processes them. Check HTTPRoute status: `kubectl describe httproute payments-route -n payments | grep -A20 Status`. If `Status.Parents[*].Conditions` shows `Accepted: False` or `ResolvedRefs: False`, the route isn't attached. Check: (1) the Gateway's `allowedRoutes` doesn't include the route's namespace; (2) the `parentRefs` GatewayClass name is wrong; (3) the backend Services don't exist. For NGINX Gateway Fabric: check `kubectl -n nginx-gateway get pods` and logs. Some controllers fall back to equal distribution if weighted routing isn't supported for the specific route configuration.

**13. After deploying cert-manager and adding the issuer annotation to an Ingress, the TLS secret is never created. Debug.**

Check cert-manager Certificate object: `kubectl get certificate -n <namespace>`. If missing, cert-manager may not be watching the namespace or the annotation is wrong. Check annotation: `kubectl describe ingress <name> | grep Annotations` — should show `cert-manager.io/cluster-issuer` or `cert-manager.io/issuer`. Check ClusterIssuer/Issuer: `kubectl describe clusterissuer letsencrypt-prod` — look for Ready condition. For ACME HTTP-01: check the ACME challenge: `kubectl get challenge -A`. If the challenge is stuck in `pending`, the ACME server can't reach the HTTP-01 path — check if the temporary Ingress rule is created: `kubectl get ingress -A | grep cm-acme`. If the temp Ingress exists but ACME fails, the external DNS for the hostname doesn't point to the Ingress IP. Check: `nslookup payments.example.com` from outside.

**14. During a Deployment rolling update, some users experience 502 errors for about 5 seconds per pod replacement. The preStop sleep is already set to 5 seconds. What's still wrong?**

The preStop sleep delays SIGTERM but may not be long enough. The full sequence: (1) deletionTimestamp set, (2) EndpointSlice controller removes pod from endpoints, (3) kube-proxy propagates to iptables (1–3s), (4) preStop runs (5s), (5) SIGTERM sent, (6) app shuts down. If kube-proxy propagation takes 3s and preStop is only 5s, the SIGTERM arrives at ~5s from deletion, and the app starts shutting down at 5s. But some iptables rules may still be in mid-update on some nodes, routing traffic to the pod which is now closing connections — causing 502. Solutions: (1) increase preStop sleep to 10–15s for a safety margin; (2) implement graceful shutdown in the app (accept SIGTERM, stop accepting new connections, finish in-flight requests, then exit); (3) ensure `terminationGracePeriodSeconds` is large enough to accommodate preStop + actual shutdown time.

---

### FAANG-Level Deep Dive (6 questions)

**15. How would you design a zero-downtime blue-green deployment using only Kubernetes Services, without Argo Rollouts or Flagger?**

Maintain two Deployments: `payments-blue` (active, label `version: blue`) and `payments-green` (inactive, 0 replicas). The active Service selects `version: blue`. For a deploy: (1) Scale `payments-green` to full replicas with the new image. (2) Wait for green pods to pass readiness. (3) Switch the Service selector from `version: blue` to `version: green` — atomic, near-instantaneous at the API level; kube-proxy propagates within seconds. (4) Monitor error rates for a bake window. (5) If good: scale `payments-blue` to 0. If bad: switch Service selector back to `version: blue` — instant rollback. The switch is a single `kubectl patch service` — no rollout, no gradual. Limitation: during the selector switch, some requests may be in-flight to blue pods (conntrack). These finish naturally. The green pods start receiving new requests immediately. Cost: double resource usage during the bake window. For large Deployments, this can be expensive. Use readiness gates (AWS ALB Controller target health) to ensure green pods are fully registered with the LB before switching.

**16. Explain how NGINX Ingress Controller prevents reload-induced traffic loss when upstream endpoints change frequently (pods scaling up/down).**

Upstream changes (pod IP adds/removes) and routing changes (Ingress path/host modifications) have different update paths. For upstream changes: the controller uses Lua (via `lua-resty-balancer`) to dynamically update the NGINX upstream server list without triggering `nginx -s reload`. The Lua module maintains an in-memory table of backends per Service. When EndpointSlice changes, the controller calls the Lua socket API to update the balancer — no worker restart, no connection drain. For routing changes (new Ingress, host/path modification): a full reload is required because NGINX's routing is configured in the static config file. The reload triggers graceful drain of old workers — connections are maintained. New workers start with new config. The window where both old and new workers exist is short (seconds). Persistent connections (HTTP/2, WebSocket) on old workers are maintained until they close or the worker's drain timeout expires. Only then are connections terminated.

**17. How does the Gateway API's `allowedRoutes` enforce multi-tenancy at the API server level, not just by convention?**

`Gateway.spec.listeners[*].allowedRoutes.namespaces` configures which namespaces can attach routes to this listener. When a Gateway controller sees a new HTTPRoute referencing this Gateway, it checks: does the HTTPRoute's namespace match the `allowedRoutes` configuration? If `from: Same`, only routes in the Gateway's namespace are allowed. If `from: Selector`, only routes in namespaces matching the label selector are allowed. This check is NOT performed by kube-proxy or the controller alone — it's enforced by the gateway controller before setting the route's `status.parents[*].conditions.Accepted: True`. An HTTPRoute that references a Gateway but isn't allowed will have `Accepted: False` and will not be implemented by the controller. Since the Gateway spec can only be modified by users with RBAC access to Gateway resources in the `infra` namespace, application teams in their own namespaces cannot modify the allowedRoutes — they're gated by the cluster operator.

**18. At large scale (1000+ Services, 10000+ pods), what are the bottlenecks in the EndpointSlice controller and kube-proxy update pipeline?**

EndpointSlice controller bottlenecks: the controller watches all pods and services. Pod readiness changes trigger EndpointSlice updates. At 10,000 pods with frequent readiness transitions (rolling deploy of 100 services), the controller's work queue fills. Each update requires reading the current EndpointSlice, computing the diff, patching affected slices. The apiserver write rate becomes the bottleneck. Mitigation: EndpointSlice controller batches updates with a short debounce window (configurable via `--endpoint-slice-max-staleness`). kube-proxy bottlenecks: each node's kube-proxy watches all EndpointSlices (for all Services). At 10,000 EndpointSlices, the initial LIST on kube-proxy restart is large. With iptables mode, each Service update rewrites the iptables ruleset — at 1000 Services, `iptables-restore` takes seconds. IPVS mode is incremental (O(1) per update). Cilium eBPF maps are also incremental. The recommendation at large scale: IPVS or Cilium to eliminate iptables bottleneck; reduce unnecessary endpoint churn (stable deployment strategy, PodDisruptionBudgets); use topology-aware routing to reduce the number of endpoints each kube-proxy must track.

**19. How would you implement per-tenant traffic isolation using Gateway API in a SaaS Kubernetes cluster?**

Architecture: one Gateway per environment (prod, staging) in an `infra` namespace managed by platform team. Each tenant gets a namespace labeled `tenant-id: <id>` and `gateway-access: prod`. The Gateway's `allowedRoutes.namespaces.selector` matches `gateway-access: prod`. Each tenant creates HTTPRoutes in their namespace for their subdomain (e.g., `tenant-a.saas.example.com`). The Gateway listener uses wildcard TLS (`*.saas.example.com`) and validates hostnames — only hostnames matching the wildcard are accepted from routes. A ValidatingAdmissionPolicy prevents tenants from creating HTTPRoutes for hostnames outside their allocated subdomain pattern. NetworkPolicies prevent cross-tenant pod communication. ResourceQuotas limit each tenant namespace. The Gateway controller enforces route isolation at the routing level; NetworkPolicy enforces at the network level; RBAC restricts who can create routes. Audit logging captures all HTTPRoute changes for compliance.

**20. Explain how kube-proxy implements session affinity by source IP at the iptables level, including what exactly is stored and for how long.**

When `spec.sessionAffinity: ClientIP` is set, kube-proxy adds two iptables rules to the KUBE-SVC chain. First, a "read" rule using `xt_recent` module: `-m recent --rcheck --name KUBE-SVC-<hash> --seconds 10800 --reap -j KUBE-SEP-<hash1>`. The `xt_recent` module maintains an in-kernel table (`/proc/net/xt_recent/`) keyed by source IP. If the source IP is in the table and the entry is within `timeoutSeconds` (10800 = 3h by default), the packet jumps directly to the previously selected backend's KUBE-SEP chain. Second, "write" rules after selection: `-m recent --set --name KUBE-SVC-<hash>` records the source IP in the `xt_recent` table after the backend is selected. The table stores: source IP → timestamp of last access. The `--reap` flag removes expired entries. The selected backend is NOT stored in `xt_recent` — the table only stores the source IP. The same KUBE-SEP rule order is preserved, so the same source IP always evaluates the same rule with the same probability until the backend changes. This is NOT perfect sticky routing (backend IP isn't stored) but achieves stickiness because the probability selection is deterministic given the same source IP and the same KUBE-SVC rule set.

---

## Hands-On Labs

### Lab 1: Service Types Comparison

**Objective:** Experience all four main Service types and understand their differences.

**Tasks:**
1. Deploy a simple web app. Create a ClusterIP Service. Test connectivity from within and outside cluster.
2. Change to NodePort. Get the assigned port and test from the node IP.
3. Change to LoadBalancer (cloud cluster). Wait for external IP. Test from outside.
4. Create an ExternalName Service pointing to `httpbin.org`. Resolve the DNS from a pod.
5. Create a Headless Service. Resolve the DNS and compare the result to a ClusterIP Service.

### Lab 2: NGINX Ingress Setup and TLS

**Objective:** Configure NGINX Ingress with TLS and observe cert-manager automation.

**Tasks:**
1. Install NGINX Ingress Controller and cert-manager.
2. Create a ClusterIssuer for Let's Encrypt staging (or use a self-signed issuer).
3. Deploy two apps and create an Ingress routing `app1.example.com/api` to app1 and `app1.example.com/admin` to app2.
4. Add the `cert-manager.io/cluster-issuer` annotation and watch certificate issuance: `kubectl get certificate -w`.
5. Test TLS: `curl https://app1.example.com/api/health`.
6. Watch NGINX config update: `kubectl exec <nginx-pod> -- cat /etc/nginx/nginx.conf | grep app1`.

### Lab 3: Gateway API with Traffic Weighting

**Objective:** Use Gateway API for canary traffic splitting.

**Tasks:**
1. Install NGINX Gateway Fabric or another Gateway API controller.
2. Create a GatewayClass and Gateway.
3. Deploy two versions of an app.
4. Create an HTTPRoute with `weight: 90` and `weight: 10` across two backends.
5. Send 100 requests and verify ~10% go to the canary: `for i in $(seq 100); do curl -s http://app.example.com/ | grep version; done | sort | uniq -c`.
6. Gradually shift to 50/50 and then 100% new version.

---

## Production Incidents

### Incident 1: Session Affinity Causes Uneven Load After Pod Restart

**Symptom:** After a Deployment rollout, CPU utilization is extremely uneven: pod-1 is at 85% CPU, pod-2 at 90%, pod-3 at 15%. All pods report the same request rate in application metrics. The Service has `sessionAffinity: ClientIP`.

**Investigation:** Session affinity binds source IPs to backends. After the rollout, new pods have new pod IPs. kube-proxy's session affinity table still has entries for the old pod IPs (which are now gone). When a cached source IP maps to a no-longer-existing KUBE-SEP rule, kube-proxy falls back to probability-based selection. Over time, most source IPs have been redistributed. However, large NAT gateways with few exit IPs cause many users to appear as the same source IP. If 1000 users share 3 exit IPs, those 3 IPs are "sticky" to 3 pods — each pod receives approximately 333 users' traffic. If one of those IPs routes to a high-traffic pod, that pod is overloaded.

**Root cause:** Session affinity by source IP is incompatible with NAT/proxy architectures. The 3 exit IPs were all hashed to pod-1 and pod-2 by the existing session affinity rules.

**Recovery:** Disable session affinity: `kubectl patch service payments-api -p '{"spec":{"sessionAffinity":"None"}}'`. Load redistributes within seconds as new connections are randomly balanced.

**Prevention:** Never use source-IP session affinity for internet-facing services. Use cookie-based session affinity at the Ingress controller level (NGINX `nginx.ingress.kubernetes.io/affinity: cookie`) which creates a per-user stable cookie, avoiding the NAT problem. Better: make the application stateless and use externalized session storage.

### Incident 2: Ingress Controller Reload Storm During Mass Deployment

**Symptom:** A deployment pipeline deploys 200 services simultaneously during a maintenance window. For 8 minutes, the NGINX Ingress controller triggers hundreds of NGINX reloads per minute. During each reload, some requests receive 502 errors (~0.1% error rate). Total: ~5000 502 errors during the deploy window.

**Investigation:** Each Ingress update triggers a controller reconcile and NGINX reload. 200 simultaneous deploys = 200 Ingress objects updated = 200 reloads attempted. The controller's debounce mechanism groups changes within a 250ms window — but 200 Ingress updates spread over 30 seconds means each arrives in a different window. Each reload causes a 100–200ms window where old workers are draining and new workers are starting — during this window, some connections hit old workers that are shutting down.

**Root cause:** Mass deployment without rate limiting Ingress updates.

**Recovery:** Reloads complete after all deploys finish. 5000 502 errors are logged.

**Prevention:** Use batch deployment strategies that update services in waves (10 per minute). The NGINX controller has a `--sync-period` flag that batches Ingress reconciles. For zero-reload endpoint updates, use NGINX Plus or enable the Lua dynamic upstream update feature. Use Gateway API which separates routing config (HTTPRoute) from endpoint discovery — routing changes (rare) cause reloads; endpoint changes (frequent) don't.
