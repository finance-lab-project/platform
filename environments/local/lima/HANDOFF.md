# Local platform checkpoint — 2026-10-23

Start here in a fresh chat, then read the selected issue and only its relevant
evidence/configuration files. This is a checkpoint of observed work, not a fresh
runtime health check. Dated sections in the linked documents retain their
historical stage boundaries, including earlier pending/uncommitted states.

## Completed and observed

- Lima 2.2.1 / Ubuntu ARM64: one `finance-lab` VM, 4 vCPUs, 8 GiB RAM,
  60 GiB disk, no host mounts or automatic port forwarding. SSH is Mac loopback.
- Guest firewall active/enabled; DHCP, UDP/TCP DNS, SSH, outbound HTTPS,
  gateway isolation, controlled reload and normal restart checks passed.
- K3s `v1.36.5+k3s1` active/enabled with SQLite Secret encryption; one Ready
  node, four ready bundled deployments. Pinned artifacts, DNS/networking and
  firewall-before-K3s restart persistence passed.
- Private Traefik ClusterIP `10.43.0.80:8443`; VM/pod app/api/auth names point
  there. Gateway/upstream DNS is `192.168.5.2`. API-only Mac loopback access
  passed native TLS/authentication and isolation checks; the tunnel was closed.
- `finance-lab` namespace: default-deny ingress/egress, CoreDNS-only allowance,
  default ServiceAccount token automount disabled; live isolation and denied
  sensitive permissions passed.
- Dedicated local CA trusted on Mac/Ubuntu. Public `local-ca` ConfigMap and
  four TLS Secrets (`edge-tls`, `spa-tls`, `bff-tls`, `keycloak-tls`) delivered
  out of band. Exact readback, cryptographic checks, encrypted SQLite records
  and denied workload Secret reads passed. No private CA key entered the VM.
- Temporary non-root pods validated read-only CA/Secret mounts: seven SANs
  over TLS 1.2/1.3 (14 successful HTTPS checks), plus 12 negative checks for
  wrong names, missing trust and TLS 1.1. Isolation passed before/after the
  temporary allowances. All five pods and two temporary policies were removed.

## Current boundaries and limitations

No application workloads, application Services, ingress routes or operator
stack have been deployed. `tls/ingress.yaml` is prepared only; certificate
inventory YAML is not a Kubernetes manifest. Seven later leaf pairs remain
unissued. No application/operator tunnel remains open; all six reserved Mac
ports were available on IPv4/IPv6 at the last check.

Real consumer TLS, permissions/probes/reload behavior, Secret persistence after
restart, ingress routing and maintenance transitions remain untested. The
reported cloud CA ZIP is not archive-encrypted; protection and an independently
downloaded restore are unverified. Traefik retains bundled cluster-wide Secret
read/watch RBAC. The upstream CRD chart's static metadata lint failure remains
documented despite successful installation. No CI or independent review PASS
is claimed. This checkpoint makes no runtime changes.

## Remaining work, one issue per chat

| Issue | Bounded outcome | Start condition |
| --- | --- | --- |
| [#8](https://github.com/finance-lab-project/platform/issues/8) | Protected CA copy and independent recovery check | Operator selects/confirms protected destination |
| [#9](https://github.com/finance-lab-project/platform/issues/9) | Real consumer TLS, credentials, isolation and Secret restart persistence | Relevant service artifacts/runtime contracts exist |
| [#10](https://github.com/finance-lab-project/platform/issues/10) | Private ingress, Mac access and maintenance-transition acceptance | Relevant real consumers ready; explicit activation request |

[Parent #2](https://github.com/finance-lab-project/platform/issues/2) stays open
for compound acceptance and maintainer sign-off. PostgreSQL (#3), identity (#4),
GitOps/frontend (#5), monitoring (#6) and full stack recovery (#7) keep their
existing ownership; do not duplicate them here or wait for #2 to close before
starting prerequisites that use its delivered foundation.

## Read only what the next step needs

- [VM pins, evidence and shell/SSH commands](README.md)
- [Firewall evidence](FIREWALL.md)
- [K3s configuration and runtime evidence](k3s/README.md), [artifact lock](k3s/artifacts.lock.json)
- [Mac API tunnel and pinned client](MAC-ACCESS.md)
- [Namespace isolation and ServiceAccount checks](NAMESPACE.md)
- [TLS issuance, delivery, mount tests and remaining gates](tls/README.md)

Private source credentials, kubeconfigs and raw validation scratch files stay
outside Git. Public evidence records commands and sanitized results, not secret
values. Recheck live prerequisites when executing a new step; do not rerun the
entire historical validation sequence merely to recover context.

Suggested next-chat request: "Read environments/local/lima/HANDOFF.md and
platform #8. Work only on the protected CA recovery follow-up, step by step."
