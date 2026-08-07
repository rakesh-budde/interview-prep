# Security Hardening (15% of Interview Weight)

> Authentication, RBAC internals, Pod Security Standards, secrets management, runtime security

---

## Table of Contents
- [4.1 Authentication & Authorization](#41-authentication--authorization)
- [4.2 Pod Security](#42-pod-security)
- [4.3 Secrets Management](#43-secrets-management)

---

## 4.1 Authentication & Authorization

### Authentication Methods

```
1. X.509 CLIENT CERTIFICATES
   Every core component (kubelet, scheduler, controller-manager,
   admin users) authenticates via a client cert signed by the
   cluster CA. Identity is derived from the certificate's Subject:
     CN (Common Name)         → username
     O  (Organization)         → group membership (can list multiple)
   Example: CN=alice,O=dev-team,O=system:masters
   → username=alice, groups=[dev-team, system:masters]
   ⚠️ system:masters group bypasses RBAC entirely (superuser) — never
      issue certs with this group to regular users.

2. SERVICE ACCOUNT TOKENS (JWT, "bound" tokens since 1.21+)
   Modern tokens are:
   ├─ Time-bound (default 1hr expiry, auto-refreshed by kubelet via
   │  TokenRequest API — "projected" volume, replaces old
   │  long-lived static Secret-based SA tokens which never expired)
   ├─ Audience-bound (aud claim restricts which API server/service
   │  the token is valid for — prevents token replay against a
   │  different cluster/service)
   └─ Object-bound (bound to the specific Pod object via UID — if
      the pod is deleted, the token is immediately invalid, even
      before its time expiry)
   
   kubectl create token my-serviceaccount --duration=1h  # manual test

3. OIDC (OpenID Connect) — for human users via external IdP
   API server configured with:
     --oidc-issuer-url=https://login.microsoftonline.com/<tenant>/v2.0
     --oidc-client-id=<app-id>
     --oidc-username-claim=email
     --oidc-groups-claim=groups
   Flow: user authenticates with IdP (Okta/Azure AD/Google) → gets
   ID token (JWT) → kubectl sends it as Bearer token → API server
   validates JWT signature against IdP's public keys (JWKS endpoint),
   extracts username/groups from claims — API server NEVER talks
   directly to the IdP for this (stateless validation).

4. WEBHOOK TOKEN AUTHENTICATION
   API server calls out to an external HTTP service with the token,
   service returns yes/no + user info — used for custom auth systems
   not fitting OIDC/certs (e.g. internal SSO not exposing OIDC).
```

### RBAC Evaluation Logic

```
RBAC OBJECTS:
├─ Role: namespaced, list of rules (apiGroups + resources + verbs)
├─ ClusterRole: cluster-scoped (or can be used namespaced too),
│  additionally covers non-namespaced resources (nodes, PVs) and
│  aggregation
├─ RoleBinding: grants a Role (or ClusterRole!) to subjects, scoped
│  to ONE namespace
└─ ClusterRoleBinding: grants a ClusterRole to subjects, cluster-wide

KEY SUBTLETY: A RoleBinding CAN reference a ClusterRole (not just a
Role) — this is how you reuse a common ClusterRole (e.g. "view",
"edit", "admin" built-ins) but scope its grant to just one namespace.

EVALUATION ALGORITHM (for "can user X do verb V on resource R in
namespace N?"):
1. Gather ALL RoleBindings in namespace N + ALL ClusterRoleBindings
   (cluster-wide) that reference user X (directly OR via a group
   X belongs to, OR via the ServiceAccount X uses)
2. For each binding, resolve its Role/ClusterRole to the actual rule set
3. Union all rules from all applicable bindings
4. If ANY rule matches (apiGroup+resource+verb, and namespace scope
   correct) → ALLOW
5. If no rule matches → implicit DENY (there is NO explicit deny in
   native Kubernetes RBAC — only additive allow)

AGGREGATED CLUSTERROLES (aggregationRule):
Built-in "admin", "edit", "view" ClusterRoles are aggregations —
they automatically include rules from any ClusterRole labeled with
their aggregation label, e.g.:
  rbac.authorization.k8s.io/aggregate-to-admin: "true"
This lets you EXTEND the built-in admin role (e.g. for a CRD you
add) without editing the built-in object directly — just label your
new ClusterRole and it merges in automatically.
```

```yaml
# Example: namespaced Role + RoleBinding (least privilege)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-dev-team
  namespace: production
subjects:
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
---
# Aggregated ClusterRole extending built-in "view"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: crd-viewer
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["mycompany.io"]
  resources: ["widgets"]
  verbs: ["get", "list", "watch"]
```

```bash
# Auditing RBAC permissions
kubectl auth can-i create pods --as alice --namespace production
kubectl auth can-i '*' '*' --as system:serviceaccount:default:my-sa
kubectl auth can-i --list --as alice -n production

# Find who can do a dangerous action cluster-wide
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'
```

### ServiceAccount Token Projection

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-sa
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600     # short-lived, auto-rotated by kubelet
          audience: vault             # audience-bound (e.g. for Vault auth)
```

### Interview Questions — AuthN/AuthZ

**Q1: Design RBAC for a multi-tenant cluster.**
> Namespace-per-tenant isolation. Per-tenant Role (not ClusterRole) scoped to their namespace granting CRUD on workload resources they need, bound via RoleBinding to that tenant's group. Use ResourceQuota + LimitRange per namespace to prevent noisy-neighbor resource exhaustion. NetworkPolicy default-deny cross-namespace traffic. Avoid ClusterRoleBindings for tenant users entirely (they'd get cluster-wide access) — the only cluster-scoped access tenants might need (e.g. viewing their own CRDs if cluster-scoped) should go through a tightly-scoped aggregated ClusterRole, not `cluster-admin`.

**Q2: User can't create pods — debug permissions.**
> `kubectl auth can-i create pods --as <user> -n <namespace>` to confirm the denial (and rule out non-RBAC causes like ResourceQuota or PodSecurity Admission rejecting first). Then `kubectl get rolebindings,clusterrolebindings -A -o json | jq` filtered for that user/group to see what's actually bound. Common root causes: binding references wrong namespace, typo in subject name/kind (Group vs User mismatch), RoleBinding referencing a Role that doesn't have the `pods create` verb, or the identity's group claim from OIDC doesn't match what the binding expects (case sensitivity, or IdP not sending the groups claim at all — check via `kubectl get --raw /apis/authorization.k8s.io -v=8` style debugging or decode the JWT).

**Q3: Implement break-glass access for emergencies.**
> Pre-provision a `ClusterRoleBinding` to `cluster-admin` for a small set of dedicated "break-glass" ServiceAccounts/users, but keep their credentials sealed (e.g. in a physical safe, or a Vault path requiring 2-person approval to unseal), NOT normally usable. Use short-lived certificates (a few hours) generated on-demand rather than standing credentials. Alert loudly (audit log webhook → PagerDuty) the instant a break-glass identity is used, and require a mandatory postmortem for every use. Combine with OPA/Kyverno policy that only allows break-glass identities to act during an active declared incident (checked against an external system).

**Q4: Explain ServiceAccount token projection and rotation.**
> Since 1.21+ (previously opt-in via TokenRequestProjection, GA now), pods get a `projected` volume with a `serviceAccountToken` source instead of a long-lived Secret mount. Kubelet requests short-lived, audience-bound tokens (default 1hr) from the API server's TokenRequest API and auto-refreshes them (rewriting the file) before expiry — WITHOUT restarting the pod. This drastically reduces blast radius if a token leaks (limited validity window) compared to the old model where the SA token Secret never expired.

**Q5: What's the Node authorizer and why does it exist separately from RBAC?**
> It's a purpose-built authorizer specifically restricting kubelet identities (`system:node:<nodename>`) to only read/write objects related to their OWN node — their own Pods, Node object, and Secrets/ConfigMaps referenced by pods scheduled to them. Without it, a compromised kubelet (which needs SOME broad read access to function) could otherwise read secrets belonging to pods on OTHER nodes if given equivalent RBAC — Node authorizer closes this gap with node-specific scoping logic that plain RBAC rules can't express.

**Q6: Explain the difference between `Role` used via `RoleBinding` vs `ClusterRole` used via `RoleBinding`.**
> Both grant NAMESPACE-scoped permissions (the RoleBinding itself is namespaced, limiting the effective grant to that namespace) — the difference is REUSABILITY. A `ClusterRole` can be bound via RoleBinding in MANY different namespaces (e.g., the built-in "edit" ClusterRole is commonly bound per-namespace for different teams) without duplicating rule definitions, whereas a plain `Role` must be redefined per namespace if you want the same rules elsewhere.

---

## 4.2 Pod Security

### Pod Security Standards (PSS) & Admission (PSA)

```
THREE BUILT-IN PROFILES (replacing the deprecated PodSecurityPolicy):

PRIVILEGED: unrestricted (no security policy applied) — for
  infrastructure/system components needing full host access
  (CNI plugins, CSI node drivers, monitoring agents needing host
  metrics access)

BASELINE: prevents KNOWN privilege escalations, but stays broadly
  compatible with common container images:
  ├─ Disallows privileged containers
  ├─ Disallows most hostPath, hostNetwork/hostPID/hostIPC
  ├─ Disallows dangerous Linux capabilities beyond default set
  └─ Allows running as root (no enforcement of non-root)

RESTRICTED: heavily hardened, enforces current pod security best
  practice:
  ├─ Everything in Baseline, PLUS:
  ├─ Must run as non-root (runAsNonRoot: true)
  ├─ Must NOT allow privilege escalation (allowPrivilegeEscalation: false)
  ├─ Must drop ALL capabilities, add back only NET_BIND_SERVICE if needed
  ├─ Must use RuntimeDefault (or localhost) seccomp profile
  └─ Volumes restricted to specific safe types

PSA is enforced via NAMESPACE LABELS (no separate policy object,
unlike the old PodSecurityPolicy which needed RBAC binding to apply):

  kubectl label namespace production \
    pod-security.kubernetes.io/enforce=restricted \
    pod-security.kubernetes.io/enforce-version=v1.28 \
    pod-security.kubernetes.io/audit=restricted \
    pod-security.kubernetes.io/warn=restricted

Three MODES per label:
├─ enforce: reject non-compliant pods at admission time
├─ audit: allow, but log a warning in audit log
└─ warn: allow, but return a warning message to kubectl user
(You typically run audit+warn at "restricted" while enforce is still
at "baseline" during migration, to see what WOULD break before
actually enforcing the stricter policy — a safe rollout pattern.)
```

### Runtime Security Hardening

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10000
    runAsGroup: 10000
    fsGroup: 10000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]   # only if binding to port <1024
    volumeMounts:
    - name: tmp
      mountPath: /tmp             # writable tmpfs since rootfs is read-only
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

```
SECCOMP (Secure Computing Mode): restricts which SYSCALLS a container
can make. RuntimeDefault uses the container runtime's built-in
allow-list (blocks ~44 dangerous syscalls like `reboot`, `mount`,
`ptrace`-for-others out of ~300+ total). Custom profiles (JSON files
on the node, referenced via `type: Localhost, localhostProfile: ...`)
allow even tighter allow-lists tailored to exactly what your app needs
(generated via tools like `strace` during testing, or `oci-seccomp-bpf-hook`).

APPARMOR / SELINUX: Mandatory Access Control (MAC) at the file/resource
level (vs seccomp's syscall level) — e.g. AppArmor profile can say
"this container may only read /etc/myapp/*, never write to /etc" even
if the syscall (open) itself is allowed. Applied via annotation
(older) or securityContext.appArmorProfile (1.30+) / seContext for
SELinux (`seLinuxOptions`).

CONTAINER ESCAPE PREVENTION CHECKLIST:
├─ No privileged: true, no hostPID/hostNetwork/hostIPC
├─ Drop ALL capabilities, add back minimally
├─ readOnlyRootFilesystem: true
├─ runAsNonRoot + specific non-zero UID (not just "not root" — an
│  attacker-controlled UID 0-in-name-only trick is still blocked)
├─ No hostPath volume mounts to sensitive paths (/, /etc, /var/run/docker.sock)
├─ seccomp RuntimeDefault minimum, custom profile for high-security workloads
└─ Falco or similar runtime detection for anomalous syscalls/behavior
   IN CASE all preventive controls are bypassed (defense in depth)
```

### Interview Questions — Pod Security

**Q1: Implement a container security hardening checklist.**
> See checklist above — non-root, read-only rootfs, drop all capabilities, no privilege escalation, seccomp RuntimeDefault, no host namespaces, minimal/scratch/distroless base images (smaller attack surface, fewer CVEs), image scanning in CI (Trivy/Grype) blocking known-critical CVEs before deployment, and signed images (cosign/Notary) verified via admission webhook (e.g., Kyverno/Connaisseur) to prevent unauthorized image injection.

**Q2: Detect and prevent container escape.**
> Prevention: PSA restricted profile (no privileged, drop capabilities, non-root), avoid hostPath mounts to `/var/run/docker.sock` (classic root-equivalent escape vector), avoid `CAP_SYS_ADMIN`/`CAP_SYS_PTRACE`. Detection: Falco monitoring for suspicious syscalls (e.g., a process trying to `setns()` into another container's namespace, unexpected `mount` calls, writes to `/proc/sys`), plus node-level auditd rules. Combine with runtime sandboxing (gVisor or Kata Containers) for untrusted multi-tenant workloads — these provide a stronger isolation boundary than standard runc containers by intercepting syscalls in userspace (gVisor) or running each pod in a lightweight VM (Kata).

**Q3: Design security policy for financial services (regulated industry).**
> PSA `restricted` enforced cluster-wide (no exceptions without documented waiver + compensating control). Mandatory image scanning + signing verified at admission. Full audit logging with immutable, offsite log shipping (tamper-evidence for compliance). Encryption at rest for etcd (see Secrets section) and in-transit mTLS everywhere (service mesh). Network segmentation (NetworkPolicy default-deny + explicit allow-lists, documented as compliance evidence). Regular CIS Kubernetes Benchmark scans (kube-bench) and remediation SLAs. RBAC least-privilege with periodic access reviews (who has cluster-admin, why, revoke if unused).

**Q4: Why is `readOnlyRootFilesystem: true` important, and how do apps that need to write handle it?**
> Prevents an attacker (or compromised dependency) from modifying application binaries, dropping malware, or persisting changes within the running container's filesystem — a common post-exploitation step is disabled entirely. Apps needing scratch space (logs, temp files, cache) mount explicit `emptyDir` volumes at just those specific writable paths (e.g. `/tmp`, `/app/cache`), keeping everything else immutable.

---

## 4.3 Secrets Management

### Built-in Secrets & Encryption at Rest

```
BY DEFAULT: Secrets are stored in etcd as base64 (NOT ENCRYPTED —
base64 is encoding, not encryption; anyone with etcd access or a
backup snapshot can trivially decode them).

ENCRYPTION AT REST configuration (EncryptionConfiguration, referenced
via kube-apiserver --encryption-provider-config flag):

apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:                      # or 'kms' for cloud KMS integration (preferred)
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}                 # fallback for reading unencrypted-legacy objects

KMS PROVIDER (recommended for production — integrates with cloud
HSM-backed key management, e.g. AWS KMS, Azure Key Vault, GCP KMS):
providers:
- kms:
    name: azure-kms
    endpoint: unix:///var/run/kmsplugin/socket.sock
    cachesize: 1000
    timeout: 3s

WHY KMS OVER RAW aescbc KEYS: with aescbc, the encryption key ITSELF
sits in a file on the control plane node (still a secret you must
protect and rotate manually). KMS delegates actual key material to a
cloud HSM — kube-apiserver only holds a reference/short-lived
data-encryption-key, and rotation/revocation can happen centrally in
the KMS without needing to re-encrypt every existing Secret in etcd
immediately (envelope encryption pattern).

⚠️ IMPORTANT: enabling encryption-at-rest AFTER secrets already exist
does NOT retroactively encrypt them — you must re-write each Secret
(e.g. `kubectl get secrets -A -o json | kubectl replace -f -`) to
force re-encryption under the new provider.
```

### External Secrets Management

```
WHY EXTERNAL SECRETS (Vault/AWS Secrets Manager/Azure Key Vault)?
├─ Centralized secret lifecycle across MULTIPLE clusters/clouds
│  (not tied to one cluster's etcd)
├─ Fine-grained, dynamic secrets (Vault can generate short-lived DB
│  credentials PER REQUEST, auto-expiring — vs a static Secret that
│  lives forever until manually rotated)
├─ Detailed audit trail of every secret access (who/what/when)
└─ Automatic secret rotation without requiring a pod restart (with
   the right integration pattern)

EXTERNAL SECRETS OPERATOR (ESO) PATTERN:
1. Define a SecretStore/ClusterSecretStore (credentials to talk to
   Vault/AWS SM/Azure KV)
2. Define an ExternalSecret object referencing a path in the external
   store + target Kubernetes Secret name to create/sync
3. ESO controller polls the external store on a refreshInterval,
   writes/updates a native Kubernetes Secret object
4. Your pod consumes the native Secret NORMALLY (env var or volume
   mount) — application code doesn't need any Vault-specific SDK

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials-k8s-secret
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/production/db
      property: password

CSI SECRETS STORE DRIVER (alternative pattern, avoids creating a
native K8s Secret at all — mounts directly from Vault/Key Vault
as a volume, reducing exposure in etcd entirely):
volumes:
- name: secrets-store
  csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: "azure-kv-provider"
```

### Secrets Rotation Without Pod Restart

```
CHALLENGE: rotating a Secret's VALUE doesn't automatically restart
pods using it — env-var-injected secrets are FROZEN at container
start (env vars can't be updated live), but VOLUME-MOUNTED secrets
DO get updated on disk automatically by kubelet (via periodic sync,
~1 minute, using the kubelet's secret/configmap manager cache).

IMPLICATIONS:
├─ Always prefer VOLUME mounts over env vars for values that may
│  need rotation (databases passwords, API keys, TLS certs)
├─ Application must WATCH the mounted file for changes (inotify) and
│  reload in-memory config — this requires application code support,
│  it's not automatic just because the file updates
└─ For cases where the app truly can't hot-reload: combine with a
   "Reloader" style controller (e.g. Stakater Reloader) that watches
   Secret/ConfigMap changes and triggers a rolling restart of
   Deployments annotated to opt in — automating what would otherwise
   be a manual `kubectl rollout restart`
```

### Interview Questions — Secrets

**Q1: Secrets are visible in etcd. How do you encrypt them?**
> Configure `EncryptionConfiguration` with a KMS provider (preferred, delegates to cloud HSM) or at minimum `aescbc` with a locally-managed key, passed to kube-apiserver via `--encryption-provider-config`. Remember this only encrypts NEW writes — existing Secrets need to be re-written (`kubectl get secrets -A -o json | kubectl replace -f -` or similar re-apply) to actually get encrypted under the new config. Combine with RBAC restricting who can `get`/`list` Secret objects at all (encryption at rest protects against etcd/backup theft, not against an authorized API user simply reading the decrypted value via kubectl).

**Q2: Design secrets management for 100 microservices.**
> Centralize in Vault (or cloud-native equivalent) as source of truth, use External Secrets Operator to sync only what's needed into each namespace as native Secrets (least-privilege — service A's namespace shouldn't need read access to service B's Vault paths). Prefer dynamic, short-lived credentials (Vault database secrets engine) for database access over static passwords. Use volume-mount injection (not env vars) so rotation can flow through without requiring code changes for basic file-reload support, and use a Reloader-style controller for services that can't hot-reload. Enable full audit logging on the Vault side for compliance/incident investigation.

**Q3: Implement automatic secrets rotation.**
> At the source (Vault/cloud secrets manager): configure automatic rotation policies (e.g. rotate DB password every 30 days) or dynamic secrets (fresh credential per lease, auto-expiring — best option, no "rotation" needed since nothing is long-lived). Sync layer (ESO) picks up new value on its `refreshInterval` and updates the native K8s Secret. For volume-mounted secrets, kubelet propagates the file update within ~1 minute automatically; ensure the application watches for file changes (inotify) or falls back to a periodic re-read. For env-var-based configs that can't hot reload, use a Reloader controller to trigger a rolling restart automatically when the Secret changes, keeping the process transparent to on-call engineers.

**Q4: What's the security weakness of injecting Secrets as environment variables, and why is it still commonly done anyway?**
> Env vars are visible via `/proc/<pid>/environ` to anything with sufficient access on the host/container, get dumped in crash reports/core dumps, are often accidentally logged (e.g., a debug log statement printing all env vars), and CANNOT be updated live (frozen at container start, requiring a pod restart to pick up rotation). Still common because it's the simplest integration — zero application code changes required (just read `os.Getenv(...)`), whereas volume-mount-based rotation requires the app to actively watch for file changes to benefit from live updates.
