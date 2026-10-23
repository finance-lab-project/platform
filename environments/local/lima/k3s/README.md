# K3s configuration and runtime foundation — platform #2

Prepared against [platform #1 revision 8](https://github.com/finance-lab-project/platform/issues/1)
and [platform #2](https://github.com/finance-lab-project/platform/issues/2).
The pinned binary, service and five configuration files are installed in the
guest. **K3s is active/running with one Ready node and enabled at boot.**
First-start and normal VM restart checks passed below. Generated credentials
remain in root-protected guest files; a separate owner-only Mac kubeconfig now
supports [validated API-only access](../MAC-ACCESS.md). Its foreground tunnel is
closed after use. [Internal TLS and ingress configuration](../tls/README.md) is
prepared separately. The dedicated CA and four initial leaf pairs are issued,
with public CA trust verified on the Mac/VM. The public CA ConfigMap and four
initial TLS Secrets are delivered and validated. Temporary pods passed the
read-only mount and internal TLS checks, with all fixtures and temporary policies
removed. Encrypted CA recovery verification, real consumer wiring, ingress
activation and application/operator access remain pending.
The [namespace baseline](../NAMESPACE.md) is applied, with runtime DNS, isolation
and default ServiceAccount restriction checks passed and diagnostic pods removed.

## Inputs and intended placement

| Source | Guest destination | Ownership / mode |
| --- | --- | --- |
| [config.yaml](config.yaml) | `/etc/rancher/k3s/config.yaml` | root:root / 0600 |
| [resolv.conf](resolv.conf) | `/etc/rancher/k3s/resolv.conf` | root:root / 0644 |
| [traefik-config.yaml](traefik-config.yaml) | `/var/lib/rancher/k3s/server/manifests/traefik-config.yaml` | root:root / 0644 |
| [coredns-custom.yaml](coredns-custom.yaml) | `/var/lib/rancher/k3s/server/manifests/coredns-custom.yaml` | root:root / 0644 |
| [10-finance-lab-firewall.conf](10-finance-lab-firewall.conf) | `/etc/systemd/system/k3s.service.d/10-finance-lab-firewall.conf` | root:root / 0644 |

[artifacts.lock.json](artifacts.lock.json) records the exact K3s ARM64 release
`v1.36.5+k3s1`, binary/archive checksums, the two bundled chart archives, eight
expected image identities, the temporary static-rendering tool and the accepted
Mac kubectl client pin. It is an
inventory, not a file consumed by K3s. K3s version selection is an installation
input; there is deliberately no unsupported `version` key in `config.yaml`.
The installer source checksum was computed from the exact release-tag source
during this preparation; no publisher signature verification is claimed.

## Configuration decisions

- One server named `finance-lab`, default SQLite, bundled containerd, Flannel
  VXLAN on `eth0`, and the K3s NetworkPolicy controller retained. No embedded-etcd
  initialization, external datastore/runtime, additional node or application.
- Node/advertise IP `192.168.5.15`; pod CIDR `10.42.0.0/16`; Service CIDR
  `10.43.0.0/16`; DNS `10.43.0.10`; domain `cluster.local`. IPv4 cluster only.
- API listener `0.0.0.0:6443` **inside the guest**, allowing both the later SSH
  loopback target and in-cluster clients. This is not a Mac wildcard forward.
  The existing [guest firewall](../FIREWALL.md) and disabled Lima automatic
  forwarding remain required. `127.0.0.1` is an explicit serving-certificate SAN
  for the Mac endpoint `https://127.0.0.1:16443`; TLS verification stays on.
- Secrets encryption enabled from first creation; administrator kubeconfig mode
  0600. K3s retains its native CA. No token, key or kubeconfig payload is in Git.
  On a fresh single server K3s may generate its token; protect that token and the
  encryption/recovery material outside Git after bootstrap. This file does not
  generate or recover application credentials.
- The dedicated resolver points only to Lima DNS `192.168.5.2`, matching the
  firewall's ordered UDP/TCP 53 exception. CoreDNS imports the separate
  `finance-lab.test` zone from `coredns-custom`, mapping only app/api/auth to
  `10.43.0.80`. No root-block replacement, upstream fallthrough, operator name or
  loopback application answer is introduced. Node and diagnostic-pod DNS checks passed at first start below.
- Disable **only ServiceLB**. Keep K3s ownership of Traefik/CRDs, CoreDNS,
  metrics-server and local-path storage. The HelmChartConfig overrides Traefik
  to IPv4 ClusterIP `10.43.0.80`, with **one Service port, TCP 8443**. No published
  HTTP, UDP/HTTP3, hostPort, hostNetwork or dashboard route; proxy-header trust
  remains disabled with empty trusted-IP lists. No application route or TLS
  Secret is created: trusted browser/backend TLS is a later stage.
- The prepared systemd drop-in requires the existing firewall service and orders
  K3s after its successful activation. It does not replace the vendor unit,
  change firewall rules or continuously monitor rules.

## Image-lock boundaries

The pause image and Helm-controller job image are directly digest-qualified in
the server config. `helm-controller-arg: default-job-image=...` uses the pinned
release's supported controller option, avoiding the deprecated `helm-job-image`
alias. Traefik's digest is passed through the bundled chart's `image.digest`
field; the rendered reference matches the selected ARM64 manifest.

K3s's bundled CoreDNS, metrics-server, local-path-provisioner and its BusyBox
helper manifests still contain upstream **tags**. Their expected ARM64 digests
are recorded, but this preparation does not claim that those manifests are
digest-enforced. Do not edit K3s's generated manifests or install duplicate
components to hide this distinction. The ServiceLB image is inventory only;
ServiceLB is disabled. At bootstrap, verify all actual images, including
chart-install Jobs, sandbox and storage-helper images, against the lock; any
unexpected identity stops acceptance. Compare a runtime manifest/image identity,
not an unrelated container ID or image configuration hash. If the optional
checksum-locked air-gap archive is chosen, inspect its contents against the same
inventory before use; that archive has not been downloaded in this step.

## Static verification — 2026-10-22

Observed during the earlier preparation, before installation:

| Check | Result |
| --- | --- |
| Ruby/Psych safe parsing, duplicate-key and alias checks, including nested `valuesContent` | PASS |
| All 22 server keys and value types checked against exact-release CLI definitions; controller image option wiring inspected | PASS |
| Fixed addresses, nonoverlapping declared CIDRs, DNS exception/zone alignment, Secret-encryption and kubeconfig settings | PASS; no live route/port recheck |
| Live publisher checksum text and eight-image release list compared with the accepted lock | PASS; K3s binary/layers not downloaded |
| Both retained chart archives SHA-256 compared with P1.1; temporary Helm 4.3.1 archive SHA-256 checked before execution | PASS |
| `helm lint --strict` for Traefik with bundled values then the HelmChartConfig values | PASS, exit 0 |
| `helm template` with Kubernetes version 1.36.5 for Traefik and its CRD chart | PASS: six Traefik objects and 35 CRDs |
| Render inspection: exact Traefik image, single ClusterIP/8443 Service, IPv4, no host ports/network, dashboard route, Secret, extra container or insecure proxy trust | PASS |
| CoreDNS custom import/mount and Default DNS wiring compared with pinned source | PASS; no CoreDNS process or DNS query run |
| Firewall dependency drop-in syntax/references; pinned installer `sh -n` | PASS; native systemd merged-unit verification deferred |
| `helm lint --strict` for the unchanged CRD chart | **FAIL**, exit 1: upstream chart metadata declares `apiVersion: v1` together with `type: application` |

The CRD archive checksum matches the accepted pin. Its metadata remains
nonconforming, so the complete static check set is **not all green**. The
compatibility follow-up below identifies the failing rule as a lint-only check
in the inspected Helm implementation; the client-side install path accepts the
unchanged archive. No archive was repackaged and no accepted pin was changed.
The later first-start section records actual Kubernetes installation separately
from this static check.

Reproducible parser/render commands, from this directory, using checksum-verified
local artifacts. Set `task_helm` to the temporary Helm 4.3.1 executable,
`task_charts` to the directory containing both pinned archives, `task_k3s_source`
to the exact-release source directory, and `task_check` to private scratch space.
These commands only parse/render; they do not contact a Kubernetes API:

```sh
ruby -rpsych -e 'ARGV.each { |p| Psych.safe_load(File.read(p)); puts "#{p}: YAML OK" }' config.yaml traefik-config.yaml coredns-custom.yaml
ruby -rjson -e 'JSON.parse(File.read(ARGV[0])); puts "JSON OK"' artifacts.lock.json
"$task_helm" version --short
ruby -rpsych -e 'd=Psych.load_stream(File.read(ARGV[0])).find { |x| x.dig("metadata", "name")=="traefik" }; puts d.fetch("spec").fetch("valuesContent")' "$task_k3s_source/manifests/traefik.yaml" > "$task_check/bundled-values.yaml"
ruby -rpsych -e 'puts Psych.safe_load(File.read(ARGV[0])).fetch("spec").fetch("valuesContent")' traefik-config.yaml > "$task_check/override-values.yaml"
"$task_helm" lint "$task_charts/traefik-40.1.5+up40.1.0.tgz" --strict --kube-version 1.36.5 -f "$task_check/bundled-values.yaml" -f "$task_check/override-values.yaml" --set-string global.systemDefaultRegistry=
"$task_helm" template traefik "$task_charts/traefik-40.1.5+up40.1.0.tgz" --namespace kube-system --kube-version 1.36.5 -f "$task_check/bundled-values.yaml" -f "$task_check/override-values.yaml" --set-string global.systemDefaultRegistry= > "$task_check/traefik-rendered.yaml"
"$task_helm" template traefik-crd "$task_charts/traefik-crd-40.1.5+up40.1.0.tgz" --namespace kube-system --kube-version 1.36.5 > "$task_check/traefik-crd-rendered.yaml"
# Known upstream metadata failure; do not interpret it as a passing lint check.
"$task_helm" lint "$task_charts/traefik-crd-40.1.5+up40.1.0.tgz" --strict --kube-version 1.36.5
```

The inspection additionally checked parsed render contents, YAML AST duplicate
keys/aliases, source flag definitions, hashes and the exact settings above.
Those preparation checks did not execute native K3s or establish runtime
acceptance. The later installation and first-start sections record the native
checks and their limits separately.

## CRD metadata compatibility follow-up — 2026-10-22

The digest-verified ARM64 manifest and configuration of the pinned klipper-helm
image identify source revision `09c52422fd0654369e3c24a2a971289d1bffd5cd`.
That revision's [Dockerfile](https://github.com/k3s-io/klipper-helm/blob/09c52422fd0654369e3c24a2a971289d1bffd5cd/Dockerfile)
builds Helm **4.1.4**, commit `05fa37973dc9e42b76e1d2883494c87174b6074f`;
the separately pinned Mac renderer remains **4.3.1**. Source files fetched by
tag matched the image revision and Helm commit. No image layers were downloaded
or executed.

The [controller](https://github.com/k3s-io/helm-controller/blob/v0.17.9/pkg/controllers/chart/chart.go)
and [job entry script](https://github.com/k3s-io/klipper-helm/blob/09c52422fd0654369e3c24a2a971289d1bffd5cd/entry)
invoke Helm install/upgrade, not lint. Helm 4.1.4's
[lint rule](https://github.com/helm/helm/blob/05fa37973dc9e42b76e1d2883494c87174b6074f/pkg/chart/v2/lint/rules/chartfile.go)
rejects `type` in a v1 chart. Its
[metadata validation](https://github.com/helm/helm/blob/05fa37973dc9e42b76e1d2883494c87174b6074f/pkg/chart/v2/metadata.go)
and [installability check](https://github.com/helm/helm/blob/05fa37973dc9e42b76e1d2883494c87174b6074f/pkg/cmd/install.go)
accept `type: application` without that cross-field restriction.

A temporary official [Helm 4.1.4 darwin/arm64 archive](https://get.helm.sh/helm-v4.1.4-darwin-arm64.tar.gz)
was checked against its [publisher checksum](https://get.helm.sh/helm-v4.1.4-darwin-arm64.tar.gz.sha256sum):
`7c2eca678e8001fa863cdf8cbf6ac1b3799f9404a89eb55c08260ef5732e658d`.
It reported `v4.1.4+g05fa379`. It was used only in private scratch space with
isolated Helm directories and an empty kubeconfig; no global tool was installed.

| Follow-up check with the unchanged CRD archive | Observed result |
| --- | --- |
| Helm 4.1.4 `lint --strict --kube-version 1.36.5` | **FAIL**, exit 1, same metadata error |
| Helm 4.1.4 `install --dry-run=client --server-side=true --force-conflicts=true` | PASS, exit 0, `Dry run complete` |
| Helm 4.1.4 `template --kube-version 1.36.5` | PASS, exit 0 |
| Parsed dry-run manifest compared with explicit-version template | Equal: 35 uniquely named `apiextensions.k8s.io/v1` CRDs |

Reproduction: set `task_job_helm` to that checksum-verified temporary 4.1.4
executable, use the chart/scratch variables above, and supply an empty kubeconfig
with no clusters, contexts or users. Run with `KUBECONFIG` pointing to that file
and `HELM_CONFIG_HOME`, `HELM_CACHE_HOME`, `HELM_DATA_HOME` pointing to separate
scratch directories, with other inherited Helm overrides removed:

```sh
"$task_job_helm" version --short
# Still expected to exit 1 on upstream chart metadata.
"$task_job_helm" lint "$task_charts/traefik-crd-40.1.5+up40.1.0.tgz" --strict --kube-version 1.36.5
"$task_job_helm" install traefik-crd "$task_charts/traefik-crd-40.1.5+up40.1.0.tgz" --namespace kube-system --dry-run=client --server-side=true --force-conflicts=true --output json > "$task_check/crd-client-dry-run.json"
"$task_job_helm" template traefik-crd "$task_charts/traefik-crd-40.1.5+up40.1.0.tgz" --namespace kube-system --kube-version 1.36.5 > "$task_check/crd-template.yaml"
```

Conclusion: this metadata defect does not block the tested client-side install
path. Retain the original chart and document the lint failure. This is not a
successful K3s Job or server-side CRD admission test: client dry run uses mock
Kubernetes clients, default capabilities and in-memory release storage. The
explicit-version template comparison is not API schema validation. The Mac test
binary is the upstream release, not the Linux image's custom build with module
overrides. Actual image execution, CRD establishment and Traefik readiness were still
pending at this compatibility stage; the first-start results are recorded below.

## Installation without starting — 2026-10-22

The maintainer authorized continuing with installation while leaving service
enablement and startup disabled. Preflight found no existing K3s binary, unit,
configuration, datastore, runtime process or environment-file overrides. The VM
was running with 4 CPUs, approximately 7.4 GiB available guest memory and 56 GiB
free disk; the owned firewall was active/enabled and its source hashes matched.

The **71,434,402-byte** ARM64 binary SHA-256 matched both the accepted lock and
the exact release's publisher checksum. A slow first transfer timed out; it was
resumed from the same release URL and the complete file was hashed before any
guest copy or execution. The saved installer matched the lock and passed
`sh -n`. The transferred binary, installer, procedure and configuration files
passed `sha256sum --check` again inside the guest.

Installed `/usr/local/bin/k3s` as root:root, mode 0755, and staged the five files
at the locations/modes in the table above. The configuration and K3s data/server
directories are root-owned mode 0700; manifests and systemd drop-in directories
are 0755. The pinned installer then ran from the saved local file in a clean
environment, with the exact invocation below (`task_stage` denotes the private
guest transfer directory):

```sh
sudo env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  INSTALL_K3S_VERSION=v1.36.5+k3s1 \
  INSTALL_K3S_SKIP_DOWNLOAD=true \
  INSTALL_K3S_SKIP_ENABLE=true \
  INSTALL_K3S_SKIP_START=true \
  INSTALL_K3S_SKIP_SELINUX_RPM=true \
  /bin/sh "$task_stage/install.sh" server
sudo systemctl daemon-reload
sudo systemd-analyze verify /etc/systemd/system/k3s.service
sudo env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin /usr/local/bin/k3s server --help
```

The [installer flags](https://docs.k3s.io/reference/env-variables) suppressed
further downloads, service enablement and service startup. The installer's
standard `kubectl`, `crictl` and `ctr` symlinks, uninstall/killall scripts, service
and empty environment file were created. Neither maintenance script was run.
Native CLI execution unpacked bundled userspace files under
`/var/lib/rancher/k3s/data`; this is installed software, not a cluster datastore.
No package installation, service start, VM restart or firewall reload was run.

| Observed verification | Result |
| --- | --- |
| `/usr/local/bin/k3s --version` | PASS: `v1.36.5+k3s1 (3dd98cc5)`, built with Go 1.26.8 |
| Installed binary SHA-256; five installed file hashes versus repository sources | PASS; configuration values and accepted pins unchanged |
| `stat -c '%u:%g %a %s %n'` for binary, configuration and environment files | PASS: root ownership, table modes, empty mode-0600 service environment |
| `systemd-analyze verify /etc/systemd/system/k3s.service` | PASS, exit 0, no diagnostics |
| `k3s server --help` in a clean environment | PASS, exit 0, empty stderr; CLI-only check, not full startup validation |
| `systemctl show` / `systemctl cat k3s.service` | Loaded, disabled, inactive/dead, MainPID 0, no execution timestamp, no pending daemon reload |
| Merged unit configuration | ExecStart is only `/usr/local/bin/k3s server`; the single prepared drop-in adds both Requires/After for `finance-lab-firewall.service` |
| Configuration/environment precedence | No `config.yaml.d`, `/etc/default/k3s` or `/etc/sysconfig/k3s`; generated service environment is empty |
| State/process inspection | No server database, TLS directory, token, node-token, admin kubeconfig, agent state or `/run/k3s`; no persistent K3s/containerd/kubelet process |
| Before/after `nft --stateless list ruleset`, `ss -H -lntu` and private boot-ID comparison | Identical; firewall remains active/enabled and VM was not restarted |
| Fresh non-multiplexed Mac SSH; `getent ahostsv4 github.com`; outbound HTTPS with curl | PASS; HTTPS 200 |
| Mac listener checks on 16443, 8443, 18443, 13000, 19090 and 19443 | No API/application/operator listeners; no tunnel opened |

The native help path does not prove effective runtime values, Secrets encryption,
Kubernetes admission or chart installation. The CRD lint limitation above is
unchanged. At the end of this install-only step, guest writer/credential state
was still uninitialized. This stage did not complete #2 or publish its GitHub
progress checklist.

## First start and bounded runtime validation — 2026-10-22

The maintainer authorized first start and runtime validation. Rechecked the
binary and all seven installed configuration/firewall file hashes, merged unit,
absence of prior cluster state, guest addresses/DNS/resources and six closed Mac
reserved ports. Then ran **`sudo systemctl start --no-block k3s.service`**.
No service enablement, VM restart, firewall reload or operator tunnel was run.
K3s initialized SQLite, generated its native credentials/certificates and pulled
its components. Source values, image/chart pins and the owned firewall were not
changed.

| Observed check | Result |
| --- | --- |
| Node, API and service | `finance-lab` Ready on `v1.36.5+k3s1`, ARM64, containerd `2.3.4-k3s1.36`; `/readyz` returns `ok`; service active/running with zero service restarts, still disabled at boot |
| Bundled components | CoreDNS, local-path-provisioner, metrics-server and Traefik each have one ready/available replica; both Helm jobs Complete with exit 0 |
| CRD installation | All 35 chart CRDs Established (39 total including four K3s/Helm CRDs); pinned chart archive hashes match the lock |
| Actual image identities | Seven downloaded images match locked Linux ARM64 manifests; checked containerd content hashes, index-to-ARM64 references where applicable, manifest-to-config digests and runtime image IDs. Disabled ServiceLB image absent |
| Secrets and SQLite | `k3s secrets-encrypt status`: Enabled, rotation stage `start`, hashes match, AES-CBC; read-only SQLite query confirms the synthetic Secret record begins with the encryption envelope and does not contain its cleartext sample |
| Credential files | Kubeconfig, server token and encryption config are root-owned mode 0600; values were not printed or copied to the Mac/Git. SQLite file is mode 0644 inside the root-only data/server path |
| Resolver wiring | Actual CoreDNS sandbox DNS server is `192.168.5.2`; diagnostic pod is ClusterFirst with `10.43.0.10` and cluster search domains |
| Node and pod DNS | Direct UDP/TCP queries to both `.2` and `10.43.0.10` return `10.43.0.80` for app/api/auth and resolve external names; cluster DNS resolves `kubernetes.default.svc.cluster.local` to `10.43.0.1`; guest system resolver also returns the three private application answers |
| Private routing | Node and pod reach `10.43.0.80:8443` and receive expected HTTP 404 without an application route. Traefik is IPv4 ClusterIP with only TCP 8443; no external IP, host port/network or ServiceLB. Ingress TLS trust was deliberately not validated in this transport-only probe |
| API and node paths | Node/pod API requests through node and Service IPs pass native-CA validation, then receive HTTP 401 without credentials; pod-to-kubelet `/pods` returns 401; pod-to-node SSH times out |
| Outbound traffic and DHCP | Node/pod HTTPS to GitHub returns 200 with certificate verification. DHCP renewal capture observes a matching REQUEST/ACK, retaining node `.15`, gateway/DNS `.2`; lease-file rewrite timestamp is unchanged |
| Gateway isolation | Node and pod fresh TCP connections to gateway ports 60022, 16443, 8443, 18443, 13000, 19090 and 19443 fail; UDP 19877 fails. Live Mac-loopback echo controls pass before/after; owned OUTPUT and FORWARD gateway-drop counters each increase by 22 |
| Firewall/CNI coexistence | Owned table remains byte-identical ignoring counters; VM boot identity unchanged. `cni0`, `flannel.1` and node pod route `10.42.0.0/24` exist. K3s iptables-nft filter hooks use priority 0 before owned hooks at 10 |
| NetworkPolicy precedence | Temporary diagnostic-pod egress-deny policy blocks both UDP/TCP DNS to `.2` despite the guest exception; deleting the policy restores both. This tests the policy engine, not final application policies |
| Final access and health | Fresh non-multiplexed Mac SSH passes; Lima SSH remains `127.0.0.1:60022`. All six reserved Mac TCP ports are closed. Four deployments remain ready; `kubectl top node` works |
| Cleanup and resource snapshot | Temporary namespace, pod, synthetic Secret, policy and Mac echo fixtures removed. No PVC or persistent NetworkPolicy exists. Guest reports 6,755 MiB available memory and 54 GiB free disk; this is a snapshot, not workload-capacity acceptance |

Read-only command examples used inside the guest:

```sh
sudo k3s kubectl wait --for=condition=Ready node/finance-lab --timeout=45s
sudo k3s kubectl get --raw=/readyz
sudo k3s kubectl -n kube-system get deployments,jobs,pods,services
sudo k3s kubectl get crds -o json
sudo k3s crictl images -o json
sudo k3s secrets-encrypt status
sudo k3s kubectl top node
sudo nft --stateless list table inet finance_lab
sudo nft -j list ruleset
sudo iptables-save
```

A disposable non-root, resource-limited BusyBox pod used the locked ARM64 digest,
no service-account token and no host mounts. Bounded Python UDP/TCP socket probes
ran on the node and with `nsenter -t "$task_pod_pid" -n python3 -` in that pod's
network namespace; this uses the pod network path, not its filesystem or Python
image. Its actual `/etc/resolv.conf` was also inspected with `kubectl exec`.
Image checks used `k3s ctr -n k8s.io content get` on locked digests, and a read-only
SQLite connection checked only the synthetic record's encrypted representation.
No application credential, database, PVC or user data was introduced.

The CRD chart's static metadata lint failure remains recorded; actual server-side
installation now succeeds without repackaging. The Traefik installer container
restarted once before completing; previous-container logs were unavailable, so
no cause is asserted. At the end of first-start validation, all four long-running
workloads had zero restarts.
An initial DNS probe incorrectly required NXDOMAIN for an absent local name;
the configured hosts block returns SERVFAIL with no answer. The harness was
corrected to check the negative response, without changing DNS configuration.
The kubelet probe was corrected from a quoting error and then an undefined root
path (404) to the protected `/pods` path (401); no cluster change was needed.

## Boot enablement and normal restart — 2026-10-22

The maintainer authorized boot enablement and a normal VM restart. Before the
restart, recorded cluster object identities, private credential fingerprints,
installed file hashes and the stateless owned firewall table. Created a temporary
namespace containing the same restricted diagnostic pod described above and a
synthetic Secret; neither contained application/user data. Then ran in the guest:

```sh
sudo systemctl enable k3s.service
```

K3s was already running; no manual service start was needed. From the Mac:

```sh
"$HOME/.local/opt/lima/2.2.1/bin/limactl" stop finance-lab
"$HOME/.local/opt/lima/2.2.1/bin/limactl" start finance-lab --tty=false --timeout=120s
```

Both commands exited 0. Graceful stop took 30.61 seconds and reached Stopped;
start took 16.86 seconds and reached Running. The guest boot identity changed.
No force-stop, VM recreation, firewall reload, credential rotation or manual
K3s start after boot was performed. Configuration and artifact pins are unchanged.

| Observed post-restart check | Result |
| --- | --- |
| Automatic startup and ordering | Firewall and K3s enabled/active; firewall became active at 1.792 seconds after boot, before K3s execution at 5.326 seconds. K3s became active/running at 12.471 seconds; both units report zero automatic service restarts |
| Cluster continuity | Node, kube-system namespace and Kubernetes API Service UIDs unchanged; native CA, server token, encryption config and kubeconfig fingerprints unchanged. Fingerprints and credential payloads are not published |
| Secret continuity and encryption | Synthetic Secret and diagnostic pod UIDs unchanged; Secret still decrypts to its original sample through the API. Read-only SQLite check still finds an encrypted envelope without the sample plaintext; encryption Enabled, hashes match, rotation stage `start` |
| Components and pinned inputs | Node Ready, API `/readyz` returns `ok`, four deployments ready, both Helm jobs Complete, all 39 CRDs Established. Seven downloaded ARM64 image identities and installed file hashes match their pins; disabled ServiceLB remains absent |
| DNS and routing | Repeated node/pod UDP/TCP gateway and cluster DNS checks, private app/api/auth answers, external resolution, cluster API name, node system resolver and actual CoreDNS sandbox resolver all pass. Private ingress remains ClusterIP `10.43.0.80:8443`, returning expected 404 without an application route |
| API and outbound paths | Native-CA-validated API requests return 401 without credentials through node and Service addresses; pod-to-kubelet `/pods` returns 401, pod-to-node SSH is denied, and node/pod external HTTPS returns 200 with certificate validation |
| Firewall and gateway isolation | Owned stateless table unchanged; K3s filter hooks at priority 0 precede owned hooks at 10. Node/pod connections to all seven prior gateway TCP targets and UDP 19877 fail against live Mac-loopback controls; OUTPUT/FORWARD drop counters each increase by 22 |
| Policy precedence and DHCP | Temporary diagnostic-pod egress denial blocks UDP/TCP gateway DNS; policy removal restores both. `networkctl renew eth0` succeeds with matching captured DHCP REQUEST/ACK, retaining node `.15` and gateway/DNS `.2` |
| Mac access | Fresh non-multiplexed SSH succeeds; Lima SSH listens only on `127.0.0.1:60022`. All six reserved Mac API/application/operator TCP ports remain closed after temporary echo fixtures are removed |
| Final cleanup and resources | Temporary namespace, pod, Secret and policy removed; no PVC or persistent NetworkPolicy exists. Four bundled deployments remain ready and metrics work. Guest reports 6,853 MiB available memory and 55 GiB free disk; this is a snapshot, not workload-capacity acceptance |

The four long-running component pods each show one container restart after the
VM reboot; the K3s service itself reports `NRestarts=0`. Checks reuse the bounded
first-start probes and read-only commands above. Ingress certificate trust was
again deliberately outside the transport test; internal TLS is still pending.
This verifies graceful reboot persistence, not crash recovery or backup/restore.
The upstream CRD chart's static metadata lint limitation remains unchanged.

## Mac API-only access — 2026-10-22

[Connection/refresh procedure and validation](../MAC-ACCESS.md) record the pinned
Mac client, dedicated protected kubeconfig and native-TLS API-only tunnel checks.
The live endpoint remained unreachable through the gateway from node/pod paths.
The foreground tunnel was closed, leaving all reserved Mac ports available;
cluster identity, guest credentials and firewall state were unchanged.

## Remaining authorized-stage gates — NOT RUN

1. Application ingress still needs internal TLS/routes and the planned access
   checks. The combined API/application tunnel and its API-only maintenance
   transition remain pending; the standalone API-only path is now validated.
2. Extend the validated namespace baseline with application-specific allow policies;
   validate real workload ServiceAccounts, storage/recovery and credential
   consumers in their own stages. BusyBox's diagnostic execution does not test
   local-path volume provisioning or its storage-helper execution.

Initial certificate delivery is complete; live application TLS/routes, later
credentials, service-specific NetworkPolicies, Retain storage policy, GitOps,
monitoring and backup/recovery integration remain later scoped work.
No user/application PVC is created; bundled local-path defaults are not a claim
that the planned Retain policy already exists. Earlier firewall evidence is retained;
the updated firewall notes distinguish these passing pod checks from remaining
workload-policy and isolation gates. This step does not complete #2.

## Source references

- [Pinned server flags](https://github.com/k3s-io/k3s/blob/v1.36.5%2Bk3s1/pkg/cli/cmds/server.go), [agent flags](https://github.com/k3s-io/k3s/blob/v1.36.5%2Bk3s1/pkg/cli/cmds/agent.go), [controller option wiring](https://github.com/k3s-io/k3s/blob/v1.36.5%2Bk3s1/pkg/cli/server/server.go).
- [Pinned Traefik HelmCharts](https://github.com/k3s-io/k3s/blob/v1.36.5%2Bk3s1/manifests/traefik.yaml), [CoreDNS](https://github.com/k3s-io/k3s/blob/v1.36.5%2Bk3s1/manifests/coredns.yaml), [local storage/helper](https://github.com/k3s-io/k3s/blob/v1.36.5%2Bk3s1/manifests/local-storage.yaml), [metrics-server](https://github.com/k3s-io/k3s/blob/v1.36.5%2Bk3s1/manifests/metrics-server/metrics-server-deployment.yaml).
- [Configuration-file precedence](https://docs.k3s.io/installation/configuration), [pinned installer](https://github.com/k3s-io/k3s/blob/v1.36.5%2Bk3s1/install.sh). Chart URLs/checksums are in the adjacent lock file.
