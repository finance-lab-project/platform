# Internal TLS and application ingress preparation — platform #2

Status: **public CA trusted on the Mac/VM and delivered to K3s; four initial
TLS Secrets delivered and validated; temporary pod mounts and internal TLS
validated; ingress resources not applied**. The CA
recovery protection and independent download verification remain pending. This
directory records the references and evidence
from [platform #1 revision 8](https://github.com/finance-lab-project/platform/issues/1)
and its [TLS appendix](https://github.com/finance-lab-project/platform/issues/1#issuecomment-6338942773).
Source files contain no certificate/key payloads and do not bootstrap workloads
or credentials automatically. See the issuance/trust, delivery and temporary
pod validation evidence below.

## Prepared files

- [certificates.yaml](certificates.yaml) is a non-executable inventory of the
  eleven independent leaf keypairs, exact DNS/IP SANs, Secret names and serving
  consumers. It also records the accepted mkcert 1.4.4 Mac ARM64 pin. Do not
  submit this inventory to Kubernetes.
- [ingress.yaml](ingress.yaml) contains one TLSOption, three ServersTransports
  and three IngressRoutes, all in `finance-lab`. It contains references only;
  no Secret or public CA payload, Namespace, Service or workload is included.

The existing [Traefik HelmChartConfig](../k3s/traefik-config.yaml) remains the
controller configuration. It exposes only ClusterIP `10.43.0.80:8443` and the
`websecure` entry point. TLS 1.2 is the explicit minimum for incoming and backend
connections; no maximum excludes TLS 1.3. No new chart, controller, sidecar,
public listener or automatic forwarding is introduced.

## Routes and trust

`app.finance-lab.test` routes to `spa:8443`; `api.finance-lab.test` routes to
`bff:8443`. `auth.finance-lab.test` routes to `keycloak:8443` only under
`/realms/finance-lab/`, `/realms/master/` and `/resources/`. There is no auth
catch-all, `/admin` route, management port 9000, operator UI or Budget route.
The canonical issuer remains
`https://auth.finance-lab.test:8443/realms/finance-lab`.

All routes reference the same-namespace `edge-tls` Secret and TLSOption.
Each backend uses explicit HTTPS with its own ServersTransport, the exact
service FQDN as `serverName`, `insecureSkipVerify: false`, and public root trust
from `local-ca` key `ca.crt`. These transport references require only the public
CA, not backend private keys. The existing Traefik controller RBAC can nevertheless
get/list/watch Secrets cluster-wide; these references are not an API-access
isolation boundary. No controller RBAC change is made in this delivery step.
The [pinned Traefik provider](https://github.com/traefik/traefik/blob/v3.7.13/pkg/provider/kubernetes/crd/kubernetes.go)
and bundled CRD schema support this same-namespace ConfigMap reference.
Cross-namespace references and insecure proxy-header trust remain unnecessary.

The dedicated CA is generated in private Mac storage, outside Git and Lima
mounts. Its private key stays mode 0600 on the Mac; protection of the operator-selected
recovery copy remains unverified below. Never put it in the VM, cluster, CI or an
image. Deliver only its public
certificate out of band to the namespaced `local-ca` ConfigMaps and appropriate
Mac/guest/client trust stores. Keep public Internet roots and K3s native CA and
client credentials intact. Even the public mkcert certificate can contain host
or user information and must stay outside this public repository.

Each leaf entry requires a separate private key. Deliver `kubernetes.io/tls`
Secrets with `tls.crt` and `tls.key`; only the repo-server TLS Secret also needs
`ca.crt`. Mount application leaf files read-only under `/etc/finance-lab/tls`
and public trust at `/etc/finance-lab/trust/ca.crt`, readable by the intended
consumer. Argo uses its native Secret paths. Follow the appendix's PostgreSQL
memory-backed key copy and ownership rules. Use protected files/stdin for later
Secret delivery, without payloads in shell arguments, Git, Helm values/release
metadata or client-side apply annotations. No placeholder Secret is supplied.

## Gates before activation

1. CA/initial leaf issuance and host trust are now verified below; the encrypted
   recovery copy and later consumer-specific leaf issuance remain pending. Check
   SHA-256 before executing mkcert, separate keys, SAN types/names, expiry,
   CA signatures, key/certificate matches and private-file permissions.
2. The namespace baseline is active and validated; actual ClusterIP Services
   and serving workloads remain pending.
   SPA/BFF/Budget native HTTPS 8443 is an owner implementation requirement,
   not a claim about current development servers. Supply compatible probes;
   do not fall back to HTTP or invent TLS sidecars. Create only the TLS material
   required for the services being activated at that stage.
3. The initial CA ConfigMap and four leaf Secrets are delivered and validated
   below. Deliver later consumers separately. Establish
   the accepted workload NetworkPolicies and downstream proxy-header trust
   boundaries before exposing application routes. Keep the existing guest
   firewall and Lima forwarding restrictions. The forwarded source address
   must be observed before configuring downstream trusted proxies.
4. Validate the prepared objects against the live API, then apply only after
   explicit activation authorization. Check controller acceptance, Service
   endpoints and real backend TLS, including wrong-name/untrusted-CA failures.
5. Configure the later Mac application names and explicit loopback-only tunnel;
   the guest/pod names continue to resolve to `10.43.0.80`. Verify native trust
   from the Mac, guest and pods, edge/backend certificate identities and TLS
   versions. Test real auth route behavior, normalized/encoded path cases,
   admin/management exclusions and gateway isolation. These are runtime gates,
   not conclusions from the static PathPrefix expression.

Argo, Redis, Grafana, Prometheus, PostgreSQL and application client wiring remain
with their later deployment steps and the TLS appendix; the inventory does not
install them. Operator UIs use their separate authenticated loopback forwards,
not these application routes. Credential-free internal metrics/health may use
the accepted restricted HTTP exception; this is not an all-traffic-TLS claim.

Renew leaves manually at least 30 days before their actual expiry, using new
keys under the same CA. Update delivery Secrets, restart affected serving or
startup-only trust consumers, and verify served fingerprints and connectivity.
CA replacement requires staged old/new public trust, reissued leaves and
verification before old trust is removed. Preserve the independent K3s CA.

## Static verification — 2026-10-22

Observed offline checks:

| Check | Result |
| --- | --- |
| Safe Ruby/Psych parsing, duplicate keys and aliases, including rendered YAML | PASS |
| Seven objects checked against the exact bundled CRD field schemas: known fields, types, required fields, enums and applicable alternatives | PASS; 111 field nodes checked |
| The encountered RootCA CEL rule forbidding simultaneous Secret/ConfigMap references | PASS by explicit assertion; no general CEL evaluator was run |
| All eleven namespace/Secret identities and DNS/IP SAN sets compared with the accepted TLS appendix | PASS; repo-server extra CA key and backend serverName/SAN matches included |
| Routes, namespaces, TLSOption/Secret/transport references, HTTPS ports, auth prefix allowlist and explicit verification/minimum TLS settings | PASS by parsed assertions; no native Traefik rule-parser execution |
| Both cached chart archives and cached mkcert binary compared with their accepted SHA-256 values | PASS; mkcert was not executed |
| Helm 4.3.1 archive checksum and executable bytes compared with that archive before rendering | PASS |
| Fresh offline Traefik / CRD chart renders for Kubernetes 1.36.5 | PASS: six controller objects / 35 CRDs |
| Rendered image, ingress class, websecure listener, ClusterIP 8443 and absence of host/public/insecure exposure | PASS; existing controller configuration unchanged |
| Existing non-documentation source hashes, local Markdown links/fences and new-file sensitive-payload scan | PASS |
| Repository whitespace check | PASS |

The rendering commands used checksum-verified local artifacts and isolated Helm
configuration/cache/data directories, with `KUBECONFIG` pointing to an absent
scratch file. From the Lima directory, `task_helm`, `task_charts` and `task_check`
refer to the verified executable, retained archives and private scratch space:

```sh
# bundled.yaml is the release HelmChart valuesContent; override.yaml comes
# from the unchanged k3s/traefik-config.yaml valuesContent.
"$task_helm" template traefik "$task_charts/traefik-40.1.5+up40.1.0.tgz" --namespace kube-system --kube-version 1.36.5 -f "$task_check/bundled.yaml" -f "$task_check/override.yaml" --set-string global.systemDefaultRegistry= > "$task_check/traefik-rendered.yaml"
"$task_helm" template traefik-crd "$task_charts/traefik-crd-40.1.5+up40.1.0.tgz" --namespace kube-system --kube-version 1.36.5 > "$task_check/traefik-crd-rendered.yaml"
# Private one-off validation scripts, not repository dependencies:
ruby "$task_check/check.rb"
python3 "$task_check/check.py"
git diff --check
```

The schema checks are a field-tree audit, not Kubernetes admission validation.
No server-side dry run, controller reconciliation, certificate parsing,
handshake, browser trust or path-normalization behavior was tested. The earlier
[upstream CRD chart metadata lint failure](../k3s/README.md#static-verification--2026-10-22)
remains recorded; it was not rerun or reclassified as a pass. An initial local
checker error in Ruby document serialization was corrected before the successful
checks; no manifest correction was needed.

During the static preparation above, no mkcert invocation, tool/trust installation,
hosts edit, tunnel, Kubernetes API call or VM/cluster mutation occurred. The later
authorized issuance/trust step is recorded separately below.


## Dedicated CA and initial host trust — 2026-10-22

Installed the accepted mkcert 1.4.4 Mac ARM64 binary separately at
`$HOME/.local/opt/mkcert/1.4.4/bin/mkcert`; cached and installed SHA-256 values
match [certificates.yaml](certificates.yaml), and `mkcert -version` reported
`v1.4.4`. No global tool symlink or existing default CA was changed.

Created the dedicated `CAROOT` at `$HOME/.local/share/finance-lab/pki/ca` and
four separate leaf pairs below `$HOME/.local/share/finance-lab/pki/leaves`:
`edge-tls`, `spa-tls`, `bff-tls` and `keycloak-tls`. Each leaf directory contains
`tls.crt` and `tls.key`, with exactly the SANs in the inventory. The other seven
inventory entries have not been issued. PKI directories are owner-only 0700;
all PEM/key files are owner-only 0600, with no symlinks. The CA private key has
not been copied to the VM, cluster or repository.

All four leaf certificates expire on **2029-01-22**; the CA expires on
**2036-10-22**. These are observed certificate dates, not renewal guarantees.
The manual renewal threshold above still applies.

Mac trust was installed with native `security add-trusted-cert`, restricted to
**user SSL-server trust** in the existing login keychain. This intentionally
avoids mkcert's system-wide `-install` and automatic NSS/Java changes. Inspection
of the exact CA user trust entry confirmed one `sslServer` policy, trustRoot
semantics and no allowed-error override. Native certificate verification used
the normal keychain search list, without an explicit root override.

Only the public CA was installed as a persistent guest file:
`/usr/local/share/ca-certificates/finance-lab-local-ca.crt`, root-owned 0644.
`update-ca-certificates` added exactly one root while retaining all **121**
previous unique roots. Both native K3s server/client CA fingerprints remained
unchanged; K3s and the firewall remained active. Public leaf certificates were
passed through stdin for verification only; no private key was sent to Ubuntu.

Representative commands actually used, with private paths expressed through
variables rather than personal absolute paths:

```sh
# Issuance ran once per selected row, with its exact inventory SANs and unique
# output paths. Example for the edge leaf (do not rerun to overwrite its files):
task_caroot="$HOME/.local/share/finance-lab/pki/ca"
CAROOT="$task_caroot" TRUST_STORES=system \
  "$HOME/.local/opt/mkcert/1.4.4/bin/mkcert" \
  -cert-file "$HOME/.local/share/finance-lab/pki/leaves/edge-tls/tls.crt" \
  -key-file "$HOME/.local/share/finance-lab/pki/leaves/edge-tls/tls.key" \
  app.finance-lab.test api.finance-lab.test auth.finance-lab.test

# Mac: the public root only; no private-key import.
security add-trusted-cert -r trustRoot -p ssl \
  -k "$HOME/Library/Keychains/login.keychain-db" "$task_caroot/rootCA.pem"
security verify-cert -c "$leaf_cert" -p ssl -n "$dns_name" -L

# Ubuntu: after copying only the public root to the path above.
sudo update-ca-certificates
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt \
  -purpose sslserver -verify_hostname "$dns_name" /dev/stdin
```

The issuance wrapper refused existing outputs and symlinked destination ancestors,
set a private umask, checked hashes before execution and captured tool output
privately. `leaf_cert` and `dns_name` select the issued certificate and each
corresponding inventory SAN for verification. Certificate subjects, PEM data and detailed private logs
are deliberately excluded from Git/public evidence.

| Observed validation | Result |
| --- | --- |
| Exact DNS SANs, CA signatures, matching certificate/key public keys and four distinct leaf keypairs, also distinct from the CA key | PASS |
| Current validity, more than 30 days remaining, CA basic constraint and certificate-signing key usage | PASS |
| Directory/file modes, ownership and absence of symlinks | PASS |
| Mac native default-trust evaluation before / after installation | Expected rejection / PASS |
| All seven declared DNS identities across the four leaves | PASS with Mac native SSL trust and Ubuntu's system CA bundle |
| Wrong hostname | Rejected by both Mac and Ubuntu |
| Unrelated native K3s CA used as the only trust anchor for an ingress leaf | Rejected by Ubuntu; no trust-store mutation |
| Existing Ubuntu roots and independent K3s CA fingerprints | Preserved |
| Encrypted recovery copy | **PENDING**; password-manager secure attachment selected, local package and restore check completed; vault upload and downloaded-copy verification not yet performed |

Native macOS trust checks needed execution outside the command sandbox. An
initial validation assertion incorrectly required CRL-signing usage; it was
removed because the accepted mkcert CA only needs certificate-signing usage.
The exported Mac trust record omits the result field, whose Apple Security SDK
default is trustRoot; the final check accounts for that documented default.
Neither correction changed certificates or weakened verification.

This step did not create Kubernetes namespaces, Secrets, ConfigMaps, routes,
Services or workloads. It did not modify hosts/DNS, open a tunnel, restart the
VM/K3s, or deploy the other certificate consumers. Container trust is not inherited
from the guest trust store: application trust mounts and Secret delivery remain
pending. Certificate-chain/hostname checks passed; no live application TLS
handshake or browser UI was exercised. Changes remain local and uncommitted;
no GitHub check, review or whole-issue completion is claimed.

Before treating CA recovery as ready, store the CA certificate and private key
in the agreed encrypted backup and verify that copy without exposing the key.
Do not send the private key through chat, email, Git or a public artifact.
If removing this CA later, remove only its exact user trust entry with
`security remove-trusted-cert` and its dedicated guest CA file, then regenerate
the guest CA bundle. Do not remove unrelated roots or the K3s CAs.


## Password-manager CA recovery preparation — 2026-10-22

The selected encrypted recovery destination is a **password-manager secure
attachment**. A local `finance-lab-local-ca-recovery-2026-10-22.zip` was prepared
under `$HOME/.local/share/finance-lab/pki/recovery/`, outside Git. The directory
is owner-only 0700; the archive is owner-only 0600. **The local ZIP is not
encrypted**: encryption is provided by the password-manager vault after upload.

The four archive entries are `rootCA.pem`, `rootCA-key.pem`, `SHA256SUMS` and
`README.txt`. Only the dedicated local CA is included, with private recovery
instructions; leaf keys, native K3s CAs and VM data are not included. The
preparation refused existing outputs and symlinked destination ancestors.

Observed local checks:

- Python `ZipFile.testzip()` and an exact four-entry allowlist passed; archive
  bytes matched the source contents, and stored file modes were 0600.
- A temporary owner-only restore reproduced both CA files with identical
  SHA-256 hashes. `openssl x509 -pubkey -noout` and `openssl pkey -pubout`
  returned matching public keys.
- `openssl verify -CAfile "$restored_ca" -check_ss_sig "$restored_ca"` passed.
  `openssl verify -CAfile "$restored_ca" -purpose sslserver "$leaf_cert"`
  passed for each of the four existing leaf certificates.
- All ten original CA/leaf files retained their hashes. Temporary restored
  files were removed; no trust store, VM or Kubernetes resource changed.

At the end of this preparation step, password-manager upload and downloaded-copy
verification were pending. The operator subsequently selected the cloud-synced
copy described below.


## Recovery-copy update — 2026-10-23

The operator reports saving the recovery ZIP in an iCloud-synced Documents
folder and chose to continue foundation work without completing the cloud
protection check. The ZIP is no longer at its original preparation location;
the dedicated source CA files remain on the Mac.

Read-only checks of the locally available Documents/iCloud Drive copy found
4,370 bytes, the exact four expected entries and valid ZIP CRCs. Both archived
CA files match the active source files by SHA-256. No archive or key was
modified or uploaded by these checks. A synchronized local file is not proof
of completed remote upload or an independent cloud download/restore.

Advanced Data Protection status was not confirmed. The ZIP has no archive-level
encryption; iCloud Drive uses encryption in transit and at rest by default,
with end-to-end encryption available through Advanced Data Protection
([Apple's encryption overview](https://support.apple.com/en-us/102651)). No
password-manager attachment, confirmed end-to-end cloud protection or
independent downloaded-copy verification is claimed. This recovery gap remains
open; continuing namespace preparation does not close it or complete issue #2.


## Initial K3s CA and TLS Secret delivery — 2026-10-23

Delivered exactly five resources to the existing `finance-lab` namespace using
an out-of-band local SSH stream:

| Resource | Exact payload keys | Intended later consumer |
| --- | --- | --- |
| ConfigMap `local-ca` | `ca.crt` — public CA certificate only | Application trust mounts and Traefik backend verification |
| TLS Secret `edge-tls` | `tls.crt`, `tls.key` | Traefik edge HTTPS |
| TLS Secret `spa-tls` | `tls.crt`, `tls.key` | SPA native HTTPS |
| TLS Secret `bff-tls` | `tls.crt`, `tls.key` | BFF native HTTPS |
| TLS Secret `keycloak-tls` | `tls.crt`, `tls.key` | Keycloak HTTPS |

All four Secrets have type `kubernetes.io/tls`. The dedicated CA private key was
not read or transferred by this step. The other seven inventory certificates
remain unissued and undelivered; no `argocd` or `monitoring` namespace was created.
Original Mac certificate/key files retained their bytes, ownership and modes.

The private delivery wrapper checked owner-only source files/directories and
absence of symlinks, then streamed an in-memory JSON payload through the existing
non-multiplexed Lima SSH connection. Guest Python fed individual objects through
stdin to native `k3s kubectl create`; no leaf key file was staged on guest disk.
OpenSSL key checks used stdin and public CA checks used an anonymous memory file.
Only sanitized check results and resource names were logged. Certificate subjects,
PEM/base64 data, key material and private fingerprints remain outside Git.

The equivalent native operations actually used were:

```sh
# The five-object JSON List came from protected files in memory, not from Git.
sudo k3s kubectl --request-timeout=15s create --dry-run=server \
  --field-manager=finance-lab-tls --validate=strict -f - -o name
# After all targets were confirmed absent, create each of the five objects.
# create refuses an existing name; no force/overwrite was used.
sudo k3s kubectl --request-timeout=15s create \
  --field-manager=finance-lab-tls --validate=strict -f - -o name
sudo k3s secrets-encrypt status
```

API readback was captured in process memory and compared with the original
bytes; neither `get -o json` output nor private errors were printed. No
`kubectl.kubernetes.io/last-applied-configuration` payload annotation exists.

| Observed validation | Result |
| --- | --- |
| Preflight and admission | Cluster ready, namespace baseline active, all five targets absent; strict server dry run accepts the intended five objects before creation |
| Exact delivery | Retrieved ConfigMap public CA equals both the Mac source and installed guest public CA; all four Secret types, key sets and decoded bytes match their source |
| Cryptographic checks before and after delivery | Four valid, distinct leaf keys, each matching its certificate and distinct from the CA public key; correct issuing CA, server purpose, exact seven DNS identities and more than 30 days validity remaining |
| Negative certificate checks | Wrong hostname and an unrelated native K3s CA reject every leaf; fallback CA directories/stores explicitly disabled for the isolated trust test |
| Encryption at rest | K3s reports encryption Enabled with matching hashes. Read-only SQLite inspection finds AES-CBC envelopes for all four Secret records and their available nonempty historical values; inspected values contain no cleartext PEM/key material |
| Default ServiceAccount | All ten prior sensitive authorization checks still deny access; actual impersonated reads of each delivered Secret return `Forbidden`; no token was minted |
| Preserved state | Namespace NetworkPolicies, default token setting and RBAC unchanged. VM boot ID, node/pod/deployment/Service identities, K3s PID, firewall/configuration/resolver/native CA and encryption configuration hashes unchanged; node Ready, four bundled deployments available, `/readyz` returns `ok` |
| No activation | No application pod, controller, job, Service, ingress, TLSOption, ServersTransport or PVC created. No VM/service restart, trust change or tunnel. All six reserved Mac ports remain available on IPv4/IPv6 loopback |

The authorization checks impersonated `system:serviceaccount:finance-lab:default`
with the `system:serviceaccounts`, `system:serviceaccounts:finance-lab` and
`system:authenticated` groups, as in [namespace validation](../NAMESPACE.md).
Denied permissions covered Secret get/list/watch, pod creation/exec/port-forward,
ServiceAccount token creation, ConfigMap update, NetworkPolicy patch and node
read. Existing cluster administrator and controller access was not restricted:
Traefik's existing ClusterRole still permits Secret get/list/watch. No claim of
per-consumer Secret isolation or admission enforcement follows from these checks.

The SQLite checks used a read-only connection and parameterized lookups of the
four `/registry/secrets/finance-lab/<name>` records, verifying encryption envelopes
and absence of source material without exporting database values. This is evidence
for these records, not a backup/recovery or whole-disk encryption test.

Two validation-only checker corrections preceded successful creation. The first
assertion incorrectly required explicit `CA:FALSE`; the pinned
[mkcert leaf template](https://github.com/FiloSottile/mkcert/blob/v1.4.4/cert.go#L60-L96)
omits that extension. The corrected checks confirm no CA-signing usage and
OpenSSL server/non-CA purposes. The unrelated-CA negative test initially inherited
Ubuntu's installed trust directory; `-no-CApath -no-CAstore` now isolates the
explicit CA. Both failed attempts created zero resources. No certificate was
reissued or changed to satisfy a test; a read-only validation run passed before
the successful delivery.

At delivery completion, mount and live TLS checks had not run. The subsequent
diagnostic-pod checks below validate mounts and direct internal HTTPS using the
delivered material. Real consumer behavior, edge/backend routing, reload, browser
acceptance and Secret restart/restore remain untested. The cloud CA recovery-copy
gap above remains open. This step does not complete platform #2; changes remain
local and uncommitted, with no GitHub Actions or independent review result claimed.


## Temporary pod mounts and internal TLS — 2026-10-23

Validated the existing public CA ConfigMap and all four TLS Secrets with five
temporary, ordinary pods in `finance-lab`: four independent HTTPS servers, each
mounting only its own leaf Secret plus the public CA, and one client mounting
only the public CA. No application workloads, Services, ingress resources or
Mac tunnels were created. The fixtures used the existing default ServiceAccount
with explicit `automountServiceAccountToken: false` and no added RBAC.

The cached pinned images did not contain suitable TLS diagnostic tools. The
temporary fixtures therefore used the official Python image, resolved through
the Docker registry and pinned to its Linux ARM64 manifest:

```text
docker.io/library/python:3.13-alpine@sha256:82dd83b5bb181e36f389c6526af7012943022888047c74bf4cba362bfa07a83c
```

Registry index SHA-256:
`bf12ecc0bea3f171d55145dd02c78b5eeb09f30ee2ac9cc2b4c9e4e8f5e30741`.
Config SHA-256:
`9e136727f903f7d796f2ce264b56b38fac6c6fc2894ef5bb1061ae9e29a33d84`.
Manifest/config bytes matched these digests and declared `linux/arm64`. Each
running container reported the pinned image identity, Python **3.13.17** and
OpenSSL **3.5.9**. The image is a diagnostic dependency, not a new application
runtime pin; its downloaded layers remain in the normal containerd cache.

Each pod had UID/GID/fsGroup 65532, all capabilities dropped,
`allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true` and
RuntimeDefault seccomp. Requests were 10m CPU/32Mi memory, limits 200m/128Mi,
with `restartPolicy: Never`, a 600-second active deadline and two-second
termination grace. No host mount or writable scratch volume was needed.

The mount contract tested inside the actual containers was:

| Source | Read-only mount | File mode and ownership |
| --- | --- | --- |
| `local-ca`, key `ca.crt` | `/etc/finance-lab/trust/ca.crt` | 0444, root:65532 |
| Each server's own TLS Secret, keys `tls.crt` and `tls.key` | `/etc/finance-lab/tls/` | Both 0440, root:65532 |
| Client | Public CA only; no leaf mount | No `/etc/finance-lab/tls` directory |

Secret/ConfigMap volume `items` selected only those keys. Read-only status was
verified from `/proc/self/mountinfo`; opening each mounted file for writing was
denied without writing any data. File bytes matched API-read values by SHA-256
in memory. UID/GID, zero effective capabilities, NoNewPrivs, seccomp and absent
ServiceAccount token were checked in every container. No PEM, private key or
fingerprint was printed or written into repository evidence.

The HTTPS fixtures used Python's standard-library HTTP server and SSL context,
loaded each mounted keypair directly, required TLS 1.2 or newer and returned
only `finance-lab-tls-diagnostic`. Readiness probes performed trusted loopback
TLS handshakes. Client requests used each fixture's pod IP with the exact
inventory DNS name as SNI and verification hostname; application DNS was not
redirected to the fixtures. This tests certificates and internal pod transport,
not Service routing or the prepared Traefik ingress configuration.

Two temporary NetworkPolicies allowed only pods labeled
`app.kubernetes.io/name=platform-tls-diagnostic, role=client` to reach matching
`role=server` pods in the same namespace on TCP 8443. One policy selected the
client for egress, the other selected the servers for ingress. Neither baseline
policy was edited. The temporary rules were removed before the pods, and their
removal was checked against still-Ready servers.

| Observed check | Result |
| --- | --- |
| Native admission | Strict server dry run accepts all five pod specifications and both temporary policies |
| Mounts and process restrictions | All five containers pass the ownership, mode, bytes, read-only, non-root and token/security-context checks above |
| Baseline isolation before allowance | Client cannot connect to any of the four ready TLS servers |
| Trusted HTTPS | Four leaf identities / seven DNS SANs each pass TLS 1.2 and TLS 1.3: 14 HTTP 200 responses with exact fixture body and served certificate fingerprint matching the mounted Secret |
| Certificate rejection | Wrong hostname is rejected for each leaf with certificate verification error 62; a client trust context without the local CA rejects each leaf with an issuer/trust verification error |
| Minimum TLS version | All four servers reject TLS 1.1 with a protocol-version alert; the negative client enabled legacy ciphers only for this test, avoiding a local no-cipher false positive |
| DNS and unrelated paths | Client resolves app/api/auth names to the existing `10.43.0.80`; API Service TCP 443 and direct gateway DNS TCP 53 remain denied while fixture allowances exist |
| Isolation after allowance removal | All four client/server connections are denied again, with all fixtures still Ready |
| Cleanup and continuity | All five pods and both temporary policies deleted. CA/Secret UIDs, resource versions and bytes unchanged; original two policies, ServiceAccount and RBAC unchanged. VM/K3s/native configuration and preexisting resource identities preserved; cluster ready and healthy |
| Mac exposure | All six reserved TCP ports available on both IPv4/IPv6 loopback; no listener or tunnel opened |

Native commands used by the bounded private validation wrapper included:

```sh
sudo k3s kubectl --request-timeout=15s create --dry-run=server \
  --validate=strict -f - -o name
sudo k3s kubectl --request-timeout=15s create --validate=strict -f - -o name
sudo k3s kubectl --request-timeout=15s wait --for=condition=Ready pods \
  -n finance-lab -l app.kubernetes.io/name=platform-tls-diagnostic --timeout=120s
sudo k3s kubectl --request-timeout=15s exec -i -n finance-lab "$task_pod" -- python3 -
sudo k3s kubectl --request-timeout=15s delete networkpolicy "$task_policy" \
  -n finance-lab --ignore-not-found --wait=true --timeout=30s
sudo k3s kubectl --request-timeout=15s delete pod "$task_pod" \
  -n finance-lab --ignore-not-found --wait=true --timeout=30s
```

Manifests and Python checks were streamed through stdin; the wrapper captured
sensitive comparison values only in memory and removed fixtures in its cleanup
handler. All ten runtime check groups passed, followed by the Mac port checks.
No source certificate, delivered Secret, ingress manifest, firewall, trust store
or existing controller was changed or restarted.

**Remaining limits:** this proves the diagnostic mount contract and live TLS
between temporary pods. It does not validate the eventual applications' UIDs,
trust loaders, HTTPS implementations, reload behavior, probes, Service endpoints,
Traefik edge/backend handshakes, browser flow or restart/recovery. Those checks
remain with their real consumers and separately authorized activation. The CA
recovery-copy gap remains open, and platform #2 is not complete. Changes remain
local and uncommitted; no GitHub CI or independent review result is claimed.
