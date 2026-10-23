# Application namespace isolation baseline — platform #2

Status: **applied and runtime validated** on 2026-10-23. This is the namespace
prerequisite for the later out-of-band TLS/Secret delivery step. It implements
only the initial application namespace boundary from
[platform #1 section 2.5 and credential appendix](https://github.com/finance-lab-project/platform/issues/1).

[namespace.yaml](namespace.yaml) contains four non-secret resources:

| Resource | Applied behavior |
| --- | --- |
| Namespace `finance-lab` | Dedicated application namespace |
| ServiceAccount `finance-lab/default` | `automountServiceAccountToken: false`; no new RBAC permissions |
| NetworkPolicy `finance-lab/default-deny-all` | Select every ordinary pod; isolate ingress and egress |
| NetworkPolicy `finance-lab/allow-cluster-dns` | Permit only TCP/UDP destination port 53 to CoreDNS-selected pods in `kube-system` |

The DNS peer combines `namespaceSelector` and `podSelector` in one peer, so both
must match. Its observed labels are `kubernetes.io/metadata.name: kube-system`
and `k8s-app: kube-dns`. Application pods use cluster DNS `10.43.0.10`; they
receive no direct egress allowance to the Lima gateway DNS `192.168.5.2`.
CoreDNS's existing upstream path and the guest firewall remain unchanged.

Policies are additive: later explicit application allow rules will extend this
baseline. It intentionally permits no application ingress, database, API,
operator or general Internet flow. It does not select `kube-system` pods for
isolation and supplies no Argo or monitoring resources.

Disabling default token mounting is a default, not an admission constraint.
Each real workload still needs its dedicated ServiceAccount, explicit token
setting and required security context. A pod can override the default; no
Role/RoleBinding, admission policy or claim of complete workload isolation is
introduced here. NetworkPolicy does not replace the guest firewall or protect
against a trusted cluster administrator. Host-network and node-originated
traffic require their separately recorded controls.

## Preparation verification — 2026-10-23

The live cluster reported K3s `v1.36.5+k3s1`, one Ready node, working CoreDNS
labels and enabled Secrets encryption with matching hashes. `finance-lab` did
not exist before or after these preparation checks.

Observed checks:

- Ruby/Psych parsed all four YAML documents. Parsed assertions checked exact
  resource identities, namespace scope, token setting, empty default allows,
  the combined DNS selectors, and only TCP/UDP port 53.
- Native `k3s kubectl create --dry-run=client --validate=strict -f - -o name`
  accepted the original four-object bundle streamed over SSH.
- `k3s kubectl create --dry-run=server --validate=strict -f - -o name`
  accepted the original Namespace. A dry-run Namespace does not persist, so
  the three namespaced objects were server-validated using request-only names
  in the existing `default` namespace. That validates their API schemas and
  current admission there; it is not a full dry run in `finance-lab`.
- A deliberately misspelled `spec.egresz` field was rejected by strict server
  validation. No invalid source was saved to the repository.
- Namespace, NetworkPolicy and ServiceAccount object identities and existing
  token-mount defaults were identical before/after; `/readyz` returned `ok`.

All Kubernetes invocations used the existing non-multiplexed Lima SSH connection
and guest `sudo k3s kubectl`. No Secret/certificate payload, workload, persistent
namespace/policy, tunnel, trust-store change or service restart was introduced.
Live packet and authorization checks were deferred at that stage; their completed
results follow below.

## Application and runtime validation — 2026-10-23

The namespace was still absent at preflight. The four resources were applied
with server-side apply, field manager `finance-lab-platform`, strict validation
and no conflict override. No diagnostic pod was started until both policies and
the default ServiceAccount setting were verified. A second server dry run passed
in the actual `finance-lab` namespace.

Two temporary diagnostic pods used the pinned ARM64 BusyBox image in
[k3s/artifacts.lock.json](k3s/artifacts.lock.json): one in `finance-lab`, one in
an otherwise unisolated temporary control namespace. Both ran as UID/GID 65532,
with read-only root filesystems, dropped capabilities, no privilege escalation,
RuntimeDefault seccomp, resource limits and a 600-second deadline. Their temporary
HTTP listeners had no Service or ingress route. The subject pod omitted its own
automount setting to test inheritance from the default ServiceAccount.

| Observed runtime check | Result |
| --- | --- |
| Applied baseline | Namespace, default ServiceAccount and exactly two policies match the source; native server dry run passes in `finance-lab` |
| Pod DNS configuration | Actual container resolver uses `10.43.0.10` and the `finance-lab.svc.cluster.local` search domain; container `nslookup app.finance-lab.test` returns `10.43.0.80` |
| Cluster DNS over UDP and TCP | `app`, `api` and `auth.finance-lab.test` resolve to `10.43.0.80`; the Kubernetes API name resolves to `10.43.0.1`; external resolution succeeds |
| Direct gateway DNS | Subject pod cannot reach `192.168.5.2:53` over UDP or TCP; node and control pod can still use both protocols |
| Egress isolation | Subject TCP connections fail to the control pod on 18080, API Service on 443, node API on 6443, kubelet on 10250, private Traefik on 8443, CoreDNS metrics on 9153 and external HTTPS on 443. All seven destinations are reachable from the control pod before and after these negative checks |
| Ingress isolation | Control-to-subject TCP 18080 fails while each pod's own loopback listener succeeds |
| Token mounting | Subject pod has no token volume or volume mount, and the token file is absent inside the actual container; default ServiceAccount automount is false |
| Authorization | All ten sensitive permission checks below return `no`; an actual impersonated Secret list returns `Forbidden` |
| Foundation continuity | Same VM boot ID, node UID, K3s PID and bundled deployment/Service identities. Firewall, K3s configuration, resolver and native CA file hashes unchanged. Node Ready, `/readyz` returns `ok`, four bundled deployments available and node certificate-validated HTTPS succeeds |
| Cleanup and access | Both diagnostic pods and their temporary namespace removed. No pod, Secret, Service, IngressRoute or PVC remains in `finance-lab`. All six reserved Mac TCP ports bind successfully on IPv4 and IPv6 loopback without listening; no tunnel remains |

Checks used guest `sudo k3s kubectl` through fresh, non-multiplexed Lima SSH.
The applied four-document YAML was parsed into an equivalent JSON List and
streamed to these native operations:

```sh
sudo k3s kubectl --request-timeout=15s apply --server-side \
  --field-manager=finance-lab-platform --validate=strict -f -
sudo k3s kubectl --request-timeout=15s apply --server-side --dry-run=server \
  --field-manager=finance-lab-platform --validate=strict -f -
```

Detailed UDP/TCP probes reused guest Python through `nsenter -t <sandbox-pid> -n`,
running in each diagnostic pod's network namespace. Actual container `exec`
checks separately verified DNS, the resolver file, UID and token absence.
The authorization matrix used the ServiceAccount identity plus its real groups:

```sh
sudo k3s kubectl --request-timeout=15s \
  --as=system:serviceaccount:finance-lab:default \
  --as-group=system:serviceaccounts \
  --as-group=system:serviceaccounts:finance-lab \
  --as-group=system:authenticated \
  auth can-i get secrets -n finance-lab
```

The same command shape denied `list/watch secrets`, `create pods`,
`create pods/exec`, `create pods/portforward`, `create serviceaccounts/token`,
`update configmaps`, `patch networkpolicies` and `get nodes`. Replacing
`auth can-i get secrets` with `get secrets` produced the observed `Forbidden`.
No token was minted and no Role or RoleBinding was added.

These checks validate this baseline on the current single-node cluster. They
are not application TLS, storage/recovery, restart persistence, host-network
isolation or real workload ServiceAccount acceptance. No application was
deployed, no ingress route or Mac tunnel was opened, and no firewall rule or
service was reloaded. The existing gateway-isolation evidence remains in
[FIREWALL.md](FIREWALL.md).

## Remaining activation gates

The subsequent [initial TLS delivery](tls/README.md#initial-k3s-ca-and-tls-secret-delivery--2026-10-23)
is now complete: the public CA ConfigMap and four TLS Secrets are present.
[Temporary pod mount/TLS checks](tls/README.md#temporary-pod-mounts-and-internal-tls--2026-10-23)
also passed; both fixture-only policies and all diagnostic pods were removed,
restoring this baseline. Application-specific allow policies, real service artifacts and the
[TLS activation gates](tls/README.md#gates-before-activation) still precede
application route/tunnel activation. Do not blanket-apply the TLS inventory.
The cloud recovery-copy protection and independent downloaded-copy check remain
unverified as recorded in the [TLS evidence](tls/README.md). This namespace step
does not complete platform #2.

Semantics: [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
and [ServiceAccount token mounting](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).
