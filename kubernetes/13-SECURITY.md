# Section 13: Security

Kubernetes security is defense-in-depth: every layer — identity, authorization, admission, pod configuration, runtime, network, and supply chain — needs hardening. A breach at any layer without compensating controls can compromise the entire cluster or the data it handles.

## Subtopic Index

- [Authentication](#authentication)
- [Authorization and RBAC](#authorization-and-rbac)
- [Service Accounts](#service-accounts)
- [OIDC Integration](#oidc-integration)
- [Security Context](#security-context)
- [Pod Security Standards](#pod-security-standards)
- [Secrets Management](#secrets-management)
- [etcd Encryption at Rest](#etcd-encryption-at-rest)
- [Network Policies](#network-policies)
- [Admission Controllers and Policy Engines](#admission-controllers-and-policy-engines)
- [mTLS and Service Mesh Security](#mtls-and-service-mesh-security)
- [Supply Chain Security](#supply-chain-security)
- [Audit Logging](#audit-logging)
- [Runtime Security](#runtime-security)

---

## Authentication

Kubernetes authentication identifies *who* is making an API request. The apiserver supports multiple authenticators in a chain:

**X.509 Client Certificates**: the most common for control-plane components. The certificate's CN becomes the username; O fields become groups. `system:masters` group (O=system:masters) bypasses all RBAC. Kubelet certificates are `system:node:<nodename>` with group `system:nodes`. Admin kubeconfig uses a certificate in the `system:masters` group — protect it like a root credential.

**Bearer Tokens (ServiceAccount JWTs)**: pods receive projected ServiceAccount tokens — short-lived JWTs (1h by default) bound to the pod's lifetime, signed by the cluster's key. The apiserver validates them via the `--service-account-issuer` key. Legacy tokens (Secrets-based) are long-lived and should be disabled.

**OIDC**: external identity providers (Okta, Azure AD, Google) issue JWTs. The apiserver validates via JWKS endpoint, checking `iss`, `aud`, `exp`, and maps claims to username/groups. Used for human operator authentication.

**Webhook TokenReview**: delegates authentication to an external service. The apiserver sends a TokenReview request; the webhook returns username/groups/extra.

### Key commands
```bash
# Who am I?
kubectl auth whoami

# Inspect a ServiceAccount token
kubectl create token default --duration=10m
TOKEN=$(kubectl create token default)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# Check certificate info in kubeconfig
kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}' | \
  base64 -d | openssl x509 -text -noout | grep -E 'Subject:|Issuer:|Not After'

# Check projected SA token in pod
kubectl exec <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d. -f2 | base64 -d | python3 -m json.tool
```

---

## Authorization and RBAC

After authentication, authorization decides whether the identity may perform the requested action. RBAC (Role-Based Access Control) is the standard: it evaluates rules across all RoleBindings and ClusterRoleBindings for the authenticated user/group.

RBAC is **additive only** — you can only grant permissions, never deny specific permissions. If you need to deny, use admission webhooks or remove overbroad grants.

**Least-privilege principles**:
- Never bind `cluster-admin` outside break-glass scenarios.
- Use namespace-scoped Roles instead of ClusterRoles where possible.
- Grant only the specific verbs, resources, and resource names needed.
- Audit regularly with `kubectl auth can-i --list`.

```yaml
# Minimal role: read-only on Deployments in one namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deploy-reader
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-deploy-reader
  namespace: production
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deploy-reader
  apiGroup: rbac.authorization.k8s.io
```

### Key commands
```bash
# Check what a user/SA can do
kubectl auth can-i list pods -n production --as=alice
kubectl auth can-i --list -n production --as=system:serviceaccount:production:my-sa

# Find all cluster-admin bindings (high risk)
kubectl get clusterrolebinding -o json | \
  python3 -c "import json,sys; [print(b['metadata']['name'], [s.get('name') for s in b.get('subjects',[])])  for b in json.load(sys.stdin)['items'] if b['roleRef']['name']=='cluster-admin']"

# Audit a ServiceAccount's permissions
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa -n default

# Check RBAC aggregation labels
kubectl get clusterrole edit -o yaml | grep aggregationRule -A10
```

---

## Service Accounts

Service Accounts (SAs) are Kubernetes identities for pods. A pod runs as the `default` SA in its namespace unless overridden. Every SA gets a projected token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`.

**Minimize SA permissions**: create per-workload SAs with only needed permissions. Never reuse SAs across workloads with different access needs.

**Disable auto-mount where not needed**: most application pods don't call the Kubernetes API. Set `automountServiceAccountToken: false` at the SA or pod level.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-app
  namespace: production
automountServiceAccountToken: false   # default: disable token mount
---
# Enable only in pods that need it
spec:
  serviceAccountName: payments-app
  automountServiceAccountToken: true   # explicit opt-in
```

**Bound/projected tokens** (since 1.21 default): tokens are audience-bound, time-limited (1h), and automatically rotated by the kubelet. The old Secret-based tokens (perpetual) should be disabled cluster-wide via `--service-account-extend-token-expiration=false` or simply by not creating them.

---

## OIDC Integration

OIDC connects Kubernetes to your corporate identity provider. Users authenticate via OIDC, receive a JWT, and present it to `kubectl` as a bearer token (via `kubectl oidc-login` or the `exec` credential plugin).

**Configuration on the apiserver**:
```
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=my-k8s-client
--oidc-username-claim=email
--oidc-groups-claim=groups
```

The apiserver fetches the JWKS from `<issuer-url>/.well-known/openid-configuration`, validates the JWT signature, checks `iss` and `aud`, and maps the `email` claim to the username and `groups` claim to groups.

RBAC RoleBindings then grant permissions to these OIDC-derived usernames or groups — enabling corporate SSO-to-Kubernetes-RBAC integration without managing Kubernetes users manually.

For workloads, **IRSA (IAM Roles for Service Accounts) on EKS** and **Workload Identity on GKE/AKS** use OIDC federation: the cluster is an OIDC provider; cloud IAM trusts the cluster's OIDC tokens; pods exchange their SA token for cloud credentials.

---

## Security Context

Security context controls Linux-level security settings for pods and containers: user/group IDs, capabilities, seccomp profiles, read-only filesystems.

```yaml
spec:
  securityContext:
    runAsNonRoot: true          # pod-level: all containers must run non-root
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001              # GID for mounted volumes
    seccompProfile:
      type: RuntimeDefault      # apply default seccomp profile
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false   # prevent setuid/sudo escalation
      readOnlyRootFilesystem: true      # no writes to container FS
      capabilities:
        drop: ["ALL"]                   # drop all Linux capabilities
        add: ["NET_BIND_SERVICE"]       # re-add only what's needed (port 80)
```

**Capabilities**: Linux capabilities split root privileges into granular units. `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, `CAP_SYS_PTRACE` are particularly dangerous. Drop ALL and add back only what's needed.

**readOnlyRootFilesystem**: forces the container to write only to explicitly mounted volumes. Prevents attackers from modifying binaries or config files in the container filesystem.

**Seccomp**: syscall filtering. `RuntimeDefault` applies the container runtime's default profile, blocking ~40% of syscalls that are almost never needed by applications. `Localhost` allows custom profiles.

---

## Pod Security Standards

Pod Security Standards (PSS) replaced PodSecurityPolicy in Kubernetes 1.25. Three levels enforced by the PodSecurity admission controller:

- **Privileged**: no restrictions. For system-level workloads only.
- **Baseline**: blocks known privilege escalation vectors (host namespaces, dangerous capabilities, hostPath mounts).
- **Restricted**: follows current hardening best practices (non-root, read-only root, seccomp, dropped capabilities, no privilege escalation).

Applied per namespace with three modes:
- `enforce`: reject non-compliant pods.
- `audit`: log violations but allow.
- `warn`: return warning header but allow.

```bash
# Apply Restricted policy to a namespace (enforce + warn + audit)
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# Test what would be rejected before enforcing
kubectl label namespace test pod-security.kubernetes.io/warn=restricted
kubectl apply -f my-pod.yaml   # see warnings in output

# Check namespace labels
kubectl get ns production -o yaml | grep pod-security
```

---

## Secrets Management

Kubernetes `Secret` objects base64-encode values (not encrypted by default). Secrets are stored in etcd as plaintext unless etcd encryption at rest is configured.

**Problems with native Secrets**:
- Anyone with `get secret` RBAC can read them.
- etcd backup contains all secrets.
- No rotation support.
- No audit trail per-secret-access.

**Better approaches**:

*External Secrets Operator (ESO)*: pulls secrets from external vaults (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault) into Kubernetes Secrets. Secrets exist briefly in Kubernetes and are refreshed on a schedule.

*Secrets Store CSI Driver*: mounts secrets directly from external vaults into pod volumes as files — never stored in Kubernetes Secrets at all. No risk of `kubectl get secret` exposure.

*Vault Agent Injector*: a MutatingAdmissionWebhook that injects a Vault agent sidecar into pods. The sidecar authenticates to Vault using the pod's SA token and writes secrets to a shared memory volume.

For production: never store database passwords or API keys in Kubernetes Secrets without etcd encryption. Prefer Secrets Store CSI or ESO with short-lived dynamically generated secrets.

---

## etcd Encryption at Rest

By default, etcd stores all Kubernetes objects including Secrets as plaintext. etcd encryption at rest encrypts the data before it's written to disk.

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets", "configmaps"]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}      # fallback for existing unencrypted data during migration
```

Apply via `--encryption-provider-config` on the apiserver. After applying, existing Secrets must be re-written to encrypt them: `kubectl get secrets -A -o json | kubectl replace -f -`.

**KMS provider**: for production, use `kms` provider with a cloud KMS key (AWS KMS, Azure Key Vault). The envelope encryption pattern: a Data Encryption Key (DEK) encrypts the Secret; the DEK is encrypted by the KMS Key Encryption Key (KEK). The KEK never leaves KMS; only the DEK traverses the network.

### Key commands
```bash
# Verify encryption is active (value should NOT be base64 of plaintext)
etcdctl get /registry/secrets/default/my-secret | xxd | head -5
# Should start with "k8s:enc:aescbc:v1:" prefix

# Check apiserver for encryption config
kubectl -n kube-system get pod kube-apiserver-<node> -o yaml | grep encryption

# Rotate keys: add new key first, re-write all secrets, then remove old key
```

---

## Network Policies

Covered in depth in Section 8 (Networking). Key security points:

- Default: all traffic allowed. Apply default-deny-ingress and default-deny-egress to all production namespaces.
- Always allow DNS egress (UDP/TCP 53 to CoreDNS).
- Use Cilium for L7 (HTTP path, gRPC method) NetworkPolicy enforcement beyond L3/L4.
- Test policies: `kubectl exec source -- nc -zv dest-ip port` before and after.

**Zero-trust posture**: every Service requires an explicit inbound allow from exactly the Services that should call it. No catch-all allows.

---

## Admission Controllers and Policy Engines

**OPA/Gatekeeper**: uses `ConstraintTemplate` (Rego) + `Constraint` CRD. Validates at admission. Supports audit mode (scan existing objects). Large policy library available (policy-library repo). Rego is powerful but has a learning curve.

**Kyverno**: YAML-native policy engine. Rules are `validate`, `mutate`, `generate`, or `verify-image`. Easier authoring than Rego. Also supports audit and background scan. Growing library of policies.

**ValidatingAdmissionPolicy** (GA in 1.30): CEL expressions evaluated in-process without a webhook. Fastest and most reliable (no external dependency). Best for simple, local validation rules (require labels, resource limits, etc.).

```yaml
# Kyverno: require all pods have resource limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-memory-limit
    match:
      resources:
        kinds: ["Pod"]
    validate:
      message: "All containers must have memory limits"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
```

### Key commands
```bash
kubectl get constraints -A            # Gatekeeper violations
kubectl get cpol,pol -A               # Kyverno policies
kubectl get policyreport -A           # Kyverno audit results
kubectl get validatingadmissionpolicies
kubectl apply --dry-run=server -f pod.yaml  # test admission
```

---

## mTLS and Service Mesh Security

Mutual TLS (mTLS) authenticates both sides of a connection — the client proves its identity in addition to the server. In Kubernetes, service meshes (Istio, Linkerd) implement mTLS transparently: sidecar proxies intercept all traffic and establish mTLS between services.

**SPIFFE/SPIRE identity**: each workload gets a SPIFFE SVID (Secure Verifiable IDentity Document) — an X.509 certificate with a SPIFFE URI: `spiffe://<trust-domain>/ns/<namespace>/sa/<serviceaccount>`. Istio's Citadel (now istiod) issues these short-lived certs (24h default) and rotates them automatically.

**AuthorizationPolicy**: fine-grained L4/L7 access control based on SPIFFE identity:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-payments
  namespace: production
spec:
  selector:
    matchLabels: {app: payments}
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/v1/charge"]
```

This denies all traffic to `payments` except POST /api/v1/charge from the `frontend` ServiceAccount — enforced at the mTLS layer, not at the application.

---

## Supply Chain Security

**Image signing (cosign/Sigstore)**: sign container images at build time. Signatures are stored in the registry or in a transparency log (Rekor). Admission webhooks (Kyverno `verifyImages`, Sigstore `policy-controller`) reject pods that reference unsigned or incorrectly signed images.

```yaml
# Kyverno: require signed images
spec:
  rules:
  - name: verify-image-signature
    match: {resources: {kinds: ["Pod"]}}
    verifyImages:
    - imageReferences: ["registry.example.com/*"]
      attestors:
      - entries:
        - keyless:
            issuer: "https://token.actions.githubusercontent.com"
            subject: "https://github.com/my-org/my-repo/.github/workflows/build.yml@refs/heads/main"
```

**SBOMs (Software Bill of Materials)**: a list of all components and dependencies in an image. Generated at build time (Syft, Trivy), attached to the image as an attestation. Used for vulnerability tracking and compliance.

**Image scanning**: scan images before deployment (Trivy in CI, ECR scan-on-push) and continuously in the registry. Block images with CRITICAL vulnerabilities via admission policy.

---

## Audit Logging

Kubernetes audit logging records every API request: who, what, when, and what the response was. Essential for incident response and compliance.

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all pod exec/attach at RequestResponse level
- level: RequestResponse
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach"]
# Log Secret access
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
# Log all other requests at Metadata level
- level: Metadata
  omitStages: ["RequestReceived"]
```

Levels: `None` (don't log), `Metadata` (log headers/auth only), `Request` (include request body), `RequestResponse` (include both). High-verbosity levels increase audit log volume significantly.

Ship audit logs to a SIEM (Elastic, Splunk, Chronicle) for alerting on: `pods/exec` to production, Secret reads outside business hours, cluster-admin bindings created.

---

## Runtime Security

Runtime security detects and alerts on suspicious behavior AFTER a container starts — in case a vulnerability is exploited.

**Falco**: uses eBPF kprobes to detect system calls matching suspicious patterns: shell spawned in a container, network connection to unusual IPs, sensitive file reads, privilege escalation. Rules are written in Falco's YAML DSL.

**Tetragon** (Cilium): eBPF-based security observability at the kernel level. Can enforce (kill process) in addition to observing. More powerful but more complex than Falco.

Common Falco rules to enable:
- `Terminal shell in container` — attacker spawned a shell
- `Write below binary dir` — modified /usr or /bin
- `Contact K8S API Server From Container` — exfiltration via API
- `Netcat Remote Code Execution` — suspicious network tool

```bash
# Install Falco
helm install falco falcosecurity/falco -n falco --set driver.kind=ebpf

# Watch Falco alerts
kubectl logs -n falco -l app=falco -f | grep -E 'Warning|Error|Critical'

# Test with a shell exec
kubectl exec <prod-pod> -- sh  # should trigger Falco alert
```

---

## Interview Questions

### Conceptual / Internals (8 questions)

**1. Explain Kubernetes RBAC evaluation logic including how denial works.**
RBAC is additive — permissions are unioned across all matching RoleBindings/ClusterRoleBindings. There is no explicit deny. If any binding grants the permission, it's allowed. To restrict access: ensure no binding grants the permission. If a user has `cluster-admin` via ClusterRoleBinding, you cannot RBAC-deny them specific operations — you must remove the binding or use an admission webhook. The evaluation: for each attribute (verb, resource, namespace, name), check all relevant bindings. If any rule matches → allow. If none match → deny (implicit).

**2. What's the difference between running as non-root (runAsNonRoot) and a user namespace?**
`runAsNonRoot: true` tells the kubelet to refuse to start the container if the image's entrypoint runs as UID 0. It's a check, not an enforcement — if the container image has a non-root default user, it passes. The container still runs with UID X on the host kernel. A user namespace maps UID 0 inside the container to a high unprivileged UID (e.g., 65534) on the host. Even if the process runs as UID 0 inside the namespace, it has no host privileges. User namespaces provide stronger isolation: a container escape gives only the mapped unprivileged UID on the host. Kubernetes 1.25+ supports user namespaces via feature gate `UserNamespacesStatelessPodsSupport`.

**3. How does etcd envelope encryption protect Secrets?**
A Data Encryption Key (DEK) is generated per Secret, used to encrypt the Secret value with AES-CBC or AES-GCM. The DEK is then encrypted by the Key Encryption Key (KEK) stored in a cloud KMS. The encrypted DEK and the encrypted Secret value are stored in etcd together. To decrypt: the apiserver calls the KMS to decrypt the DEK, then uses the DEK to decrypt the value — the KEK never leaves KMS. An etcd backup contains only encrypted values — useless without KMS access. Rotating the KEK: generate a new KEK, re-encrypt all DEKs with the new KEK, update the EncryptionConfiguration, optionally re-write all Secrets (so any old DEKs encrypted with the old KEK are replaced).

**4. Why is `cluster-admin` ClusterRoleBinding so dangerous and what are safer alternatives?**
`cluster-admin` has `*/*/* verbs` on all resources — full control including deleting namespaces, modifying RBAC, reading all Secrets, executing into pods, and modifying the apiserver. A compromised identity with this binding can escalate to full cluster and data access. Safer alternatives: `edit` ClusterRole (create/update/delete most resources but not RBAC or Secrets-by-name), `view` (read-only), or custom ClusterRoles with exactly the required permissions. For break-glass access: create a highly-audited, MFA-gated process for binding `cluster-admin` only when needed, with time-limited bindings and immediate revocation.

**5. Explain how Kyverno image verification works and what it checks.**
Kyverno's `verifyImages` rule intercepts pod creation. For each container image, it: (1) fetches the image's cosign signature from the registry or Rekor transparency log; (2) verifies the signature against the configured public key or keyless certificate (verifying the OIDC identity used for keyless signing — GitHub Actions workflow identity, etc.); (3) validates attestations (SBOM, vulnerability scan results) if configured; (4) optionally mutates the image reference to include the verified digest (pinning). If verification fails, the pod is rejected. This ensures only images built by your trusted CI pipeline (signed with a known identity) run in production.

**6. What does `allowPrivilegeEscalation: false` actually prevent at the kernel level?**
It sets `PR_SET_NO_NEW_PRIVS` via `prctl(2)` on the container process. This flag prevents the process from gaining new privileges through execve — specifically it prevents setuid/setgid binaries from granting elevated privileges and prevents file capabilities from being honored. Without this, a container running as UID 1000 that executes a setuid-root binary (e.g., `sudo`) could escalate to root. With `PR_SET_NO_NEW_PRIVS`, the setuid bit is ignored by the kernel.

**7. How does Pod Security Admission enforcement differ from Gatekeeper?**
PSA is built into the apiserver, evaluates instantly with no webhook dependency. It checks pods against the three fixed policy levels (Privileged/Baseline/Restricted). No custom rules, no gradual rollout per field, no audit scanning of existing objects. Gatekeeper is a validating webhook: it sends AdmissionReview to an external service running Rego evaluation. More flexible (custom policies, parameterized constraints), but adds an external dependency to the admission path. PSA is better for "enforce the standard levels without operational complexity." Gatekeeper is better for "enforce custom organization policies across all workloads."

**8. Explain the SPIFFE identity model used by Istio for mTLS.**
SPIFFE (Secure Production Identity Framework For Everyone) defines a standard identity format: a URI like `spiffe://trust-domain/path`. In Istio, each workload gets an X.509 SVID (SVID = SPIFFE Verifiable Identity Document): a certificate with the SPIFFE URI as the SAN. For `payments` SA in `production` namespace: `spiffe://cluster.local/ns/production/sa/payments`. Istiod issues these certs (24h lifetime), using a mesh-internal CA. When two Envoy sidecars connect via mTLS, each validates the other's SVID against the mesh CA. AuthorizationPolicy rules match on the SPIFFE URI's SA/namespace portion — so policies are identity-based, not IP-based, and survive pod restarts with IP changes.

### Scenario Questions (6 questions)

**9. A pod is running as root. How do you detect and fix this without breaking the application?**
Detect: `kubectl exec <pod> -- id` → `uid=0(root)`. Or: `kubectl get pod <pod> -o jsonpath='{.spec.securityContext.runAsUser}'` (empty = default). Fix: (1) Check if the application requires root — run `strace -p <pid>` or inspect Dockerfile. (2) Add `runAsNonRoot: true` to the pod spec, set `runAsUser: 10001` (or whatever the app supports). (3) Add `allowPrivilegeEscalation: false`. (4) Drop capabilities: `capabilities.drop: ["ALL"]`, add back only what the app needs. (5) Test readiness/liveness probes still pass. If the application breaks on non-root: it may be writing to `/` (fix: mount emptyDir at write path), listening on port < 1024 (fix: add `CAP_NET_BIND_SERVICE` or change to port > 1024), or reading privileged files (fix: security concern worth addressing).

**10. After applying PSA `Restricted` to a namespace, several Pods fail admission. How do you handle the migration?**
Don't jump straight to `enforce`. First apply `warn` + `audit`:
```bash
kubectl label namespace production pod-security.kubernetes.io/warn=restricted pod-security.kubernetes.io/audit=restricted
```
Deploy or restart pods and read warnings. Check audit events: `kubectl get events -n production | grep PodSecurity`. Fix non-compliant pods: add `securityContext`, `runAsNonRoot`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem`. For pods that genuinely need exceptions (legacy apps, privileged infrastructure), move them to a separate namespace with `Baseline` or `Privileged` and apply compensating controls. Once all pods are compliant under `warn`, switch to `enforce`.

**11. An attacker executes a shell in a production pod. What controls detect and contain this?**
Detection: Falco rule `Terminal shell in container` fires → alert in SIEM/PagerDuty. Audit log records `pods/exec` create with the user identity. Network policies prevent the shell from making unexpected outbound connections. Containment: the pod's RBAC-limited ServiceAccount prevents API access. `readOnlyRootFilesystem` prevents writing malware. `capabilities.drop: ALL` limits kernel calls. mTLS (Istio) prevents traffic to other services without a valid SVID. Alert triggers: isolate the pod (add a `quarantine: true` taint), capture forensics (runtime memory dump via ephemeral container), and terminate. Post-incident: investigate how the attacker got execution (CVE, misconfigured service, stolen credential).

**12. A developer needs to debug a production pod but you can't give them `exec` access. What alternatives exist?**
(1) **Ephemeral containers**: `kubectl debug -it <pod> --image=busybox --target=<container>` — adds a temporary debug container sharing process namespace. Requires only `pods/ephemeralcontainers` subresource. Can grant this separately from `pods/exec`. (2) **kubectl debug node**: creates a pod on the node with host PID/network. Requires `pods/exec` on the debug pod, not the original pod. (3) **Logs and metrics**: most debugging doesn't need exec — `kubectl logs`, `kubectl describe`, `kubectl top`, custom metrics. (4) **Port-forward to connect a local debugger**: only requires `pods/portforward` subresource. (5) **Sidecar with metrics/pprof endpoint**: bake observability into the container image at build time. Grant access to the pprof HTTP endpoint via internal Service.

### FAANG Deep Dive (6 questions)

**13. How does Falco's eBPF driver detect a privileged container syscall without modifying the application?**
Falco's eBPF driver attaches kprobe programs to kernel syscall entry and exit points (e.g., `__x64_sys_execve`, `__x64_sys_connect`, `__x64_sys_open`). When a syscall is invoked by any process, the kprobe fires in kernel context, reads the syscall arguments and the calling process's namespace/container metadata from kernel data structures, and writes a structured event to a per-CPU ring buffer. A Falco userspace thread reads from the ring buffer, evaluates the event against the loaded rules (compiled to a JIT-ed decision tree), and generates an alert if a rule matches. The overhead is ~2% CPU for typical workloads. The probe doesn't need to be loaded into the application process — it's kernel-wide, so all processes on the node are monitored.

**14. Describe how cosign keyless signing works for a GitHub Actions CI pipeline.**
Keyless signing uses short-lived OIDC identity as the signing key rather than a long-lived private key. In GitHub Actions: (1) The workflow requests an OIDC token from GitHub's OIDC provider — a JWT proving the workflow identity (repo, branch, workflow). (2) cosign receives this token and requests a short-lived certificate from Sigstore's Fulcio CA, which verifies the OIDC token and issues an X.509 certificate with the identity embedded. (3) cosign signs the image digest using the ephemeral private key (generated in-memory), uploads the signature to the OCI registry or Rekor transparency log, and the certificate is included in the signature. (4) At admission time, Kyverno/policy-controller fetches the signature, verifies it against Sigstore's CA, checks the certificate's embedded identity matches the expected GitHub org/repo/workflow, and verifies the signature over the image digest. No private key to manage or rotate.

**15. How would you implement a data-perimeter in Kubernetes to prevent exfiltration of sensitive data?**
Layered approach: (1) **Network egress restriction**: NetworkPolicy default-deny-egress with explicit allow-lists for known destinations. Cilium L7 policy to restrict HTTP calls to specific domains/paths. (2) **Service Account restriction**: disable auto-mount; pods handling sensitive data use dedicated SAs with minimal permissions and no `list secrets` access. (3) **Secrets Store CSI**: secrets never stored in etcd; mounted read-only at runtime from external vault. (4) **Admission policy**: Kyverno/Gatekeeper prevents pods in sensitive namespaces from using `hostNetwork` or `hostPID` (which bypass NetworkPolicy). (5) **Runtime**: Falco alerts on `connect` to unexpected IPs from sensitive pods. (6) **Audit**: log all `get secret` and `pods/exec` operations to SIEM; alert on access outside business hours.

**16. How does the Node Authorizer work and why can a kubelet only access its own node's objects?**
The Node Authorizer is a specialized authorization plugin that runs alongside RBAC. It activates when the authenticated user is a member of `system:nodes` group AND has a username matching `system:node:<nodename>`. For a kubelet authenticated as `system:node:worker-1`, the Node Authorizer: (1) allows get/list/watch of Pods assigned to `worker-1` only; (2) allows get of Secrets/ConfigMaps referenced by Pods on `worker-1` only; (3) allows update of Pod/Node status for `worker-1` only. This prevents a compromised kubelet from reading Secrets for pods on other nodes, escalating privileges by modifying other nodes' objects, or learning about workloads it doesn't host. Without the Node Authorizer, a single compromised kubelet could read all cluster Secrets.

---

## Hands-On Labs

### Lab 1: RBAC Least Privilege
Create a ServiceAccount with minimal permissions for a specific task. Verify it can only perform allowed operations. Attempt to escalate and confirm failure.

### Lab 2: Pod Security Standards Enforcement
Apply Baseline policy in warn mode to a namespace. Deploy a privileged pod and observe warnings. Fix the pod spec. Switch to enforce mode.

### Lab 3: Runtime Security with Falco
Install Falco. Run `kubectl exec <pod> -- sh` in a production namespace. Observe the Falco alert. Write a custom rule to alert on any `curl` or `wget` from containers.

---

## Production Incidents

### Incident 1: Service Account Token Exfiltration
A compromised pod exfiltrated its auto-mounted SA token. The token had cluster-admin rights (misconfigured). Attacker used the token to list all Secrets. **Prevention**: disable auto-mount on all SAs; use IRSA/Workload Identity for cloud access; rotate compromised tokens immediately.

### Incident 2: Unsigend Image Deployed in Production
A supply chain attack injected malicious code into a base image in the public registry. No image signing or scanning was in place. **Prevention**: require cosign signatures in Kyverno; scan all images for CVEs; pin all images to digests; use private registry mirrors.
