# Local Lima foundation — platform #2

This directory contains the standalone VM configuration for
[platform #2](https://github.com/finance-lab-project/platform/issues/2),
using [platform #1 revision 8](https://github.com/finance-lab-project/platform/issues/1).
The instance name is `finance-lab`, matching the YAML filename.

Completed scope: configuration preparation, pinned host-tool installation,
image checksum verification, static Mac preflight, VM creation, first-boot
validation and a graceful stop/start with resource, DNS, network and Mac access
checks. Lima completed its normal guest bootstrap. The prepared guest firewall
is now active/enabled after an approved retry with temporary connection tracking
and timed rollback. Existing/fresh SSH, DHCP, DNS, outbound HTTPS and guest gateway
isolation passed runtime checks; temporary rules/listeners were removed and
rollback was cancelled. See [FIREWALL.md](FIREWALL.md) for the evidence and initial
failed attempt. Restart persistence, K3s installation, Kubernetes resources,
application credentials, certificates and deployment remain pending.

## Connect from the Mac

Use the pinned Lima 2.2.1 binary to enter the existing, running VM:

```sh
"$HOME/.local/opt/lima/2.2.1/bin/limactl" shell --workdir=/ finance-lab
```

The explicit working directory avoids depending on a Mac directory being present
in the guest; no host directory is mounted. Run `exit` to return to the Mac.
Leaving the shell keeps the VM running.

For a direct SSH session using Lima's generated identity and loopback settings:

```sh
ssh -F "$HOME/.lima/finance-lab/ssh.config" lima-finance-lab
```

These commands connect to an already running instance; they do not start it,
restart it or create application/API tunnels. Keep the generated SSH files and
private keys outside Git.

## Pinned inputs

| Input | Selection |
| --- | --- |
| Host tool | Lima **2.2.1**, macOS ARM64 |
| Backend / guest | VZ / Ubuntu Server 24.04 LTS, ARM64 |
| Ubuntu build | `release-20261017`, exact URL and SHA-256 in [finance-lab.yaml](finance-lab.yaml) |
| Allocation | 4 vCPUs, 8 GiB RAM, 60 GiB disk |
| Mounts / runtime | No host mounts; both Lima containerd modes disabled |
| Network | Default user-mode network only; no additional attachment |
| SSH control | Mac loopback port 60022; no agent forwarding or extra host SSH keys |
| Application DNS | The three `app/api/auth.finance-lab.test` names resolve to planned ingress `10.43.0.80` inside the guest |
| Guest DNS upstream | DHCP-supplied `192.168.5.2`; also the Lima host gateway, so the active guest firewall includes a narrow TCP/UDP 53 exception |
| Forwarding | Ignore automatic forwarding for all guest ports/protocols; explicit application/API tunnels are a later step |

The pinned Lima archive is
[lima-2.2.1-Darwin-arm64.tar.gz](https://github.com/lima-vm/lima/releases/download/v2.2.1/lima-2.2.1-Darwin-arm64.tar.gz),
SHA-256 `9e9eacce88f37e185c346bad73aa6136f738d8cdf8c3bb23cd42b071824bc66e`.
The downloaded archive's SHA-256 matches this pin and its entry in the release's
[SHA256SUMS](https://github.com/lima-vm/lima/releases/download/v2.2.1/SHA256SUMS).
The checksum manifest also matches the hash published in the
[release notes](https://github.com/lima-vm/lima/releases/tag/v2.2.1).
The pinned Ubuntu image has also been downloaded and verified: **616,522,752
bytes**, SHA-256
`50e24e1fb79e89116e939d90aa7738f7af48c442fa1bc2510131e0b39c50348e`,
matching both [finance-lab.yaml](finance-lab.yaml) and the dated build's
[publisher checksum](https://cloud-images.ubuntu.com/releases/noble/release-20261017/SHA256SUMS).
The separately verified download is retained outside Git in a private temporary
directory. During creation, Lima downloaded the same pinned image into its cache
and prepared the instance disk; the cached bytes were independently rehashed
and matched the pin. First boot is now observed below. No publisher signature
verification is claimed. A later download must still pass the pinned digest.

The official release bundle is installed at `$HOME/.local/opt/lima/2.2.1`,
using Lima's supported [binary archive installation](https://lima-vm.io/docs/installation/).
All 200 installed regular files match the checksum-verified archive, including
`limactl` and the native ARM64 guest agent. No additional guest-agent archive
is needed for the selected ARM64 guest.
The existing Homebrew Lima 2.2.0 and shell PATH are unchanged. Select the pinned
binary explicitly below; an unqualified `limactl` still resolves to 2.2.0 on
this host.

`minimumLimaVersion: "2.2.1"` rejects older tools. It is a lower bound,
not an exact-version lock: use exactly 2.2.1 for the accepted baseline.

## Correction found during static validation

The planning snippet's top-level `ipv6: false` is unsupported. This file uses
`hostResolver.ipv6: false`, as defined in the
[pinned Lima types](https://github.com/lima-vm/lima/blob/v2.2.1/pkg/limatype/lima_yaml.go)
and [configuration reference](https://github.com/lima-vm/lima/blob/v2.2.1/templates/default.yaml).
This controls the host resolver's IPv6 answers; it does not disable the guest's
IPv6 kernel stack. The agreed lab names retain IPv4 addresses only.
The issue text was not changed in this implementation step.

An empty `base`/mount list does not suppress global Lima configuration.
Before any later creation, inspect the effective configuration for additions
from `_config/base.yaml`, `default.yaml` and `override.yaml` in the selected
Lima home. Stop if they introduce mounts, extra runtimes, forwarding,
provisioning hooks or conflicting values. Do not change global settings silently.

## Static validation

Run from the repository root. These commands do not create or start a VM:

```sh
task_lima_bin="$HOME/.local/opt/lima/2.2.1/bin/limactl"
ruby -rpsych -e 'Psych.safe_load(File.read(ARGV[0])); puts "YAML syntax: OK"' environments/local/lima/finance-lab.yaml
"$task_lima_bin" --version
```

Using that explicit **Lima 2.2.1** binary, check the exact version and perform
native validation:

```sh
test "$("$task_lima_bin" --version)" = "limactl version 2.2.1" &&
  "$task_lima_bin" validate environments/local/lima/finance-lab.yaml
```

For private inspection of the merged configuration, with that same version:

```sh
task_lima_check_dir="$(mktemp -d)"
"$task_lima_bin" validate --fill environments/local/lima/finance-lab.yaml > "$task_lima_check_dir/merged.yaml"
```

Inspect the merged values; do not commit the generated output, which can contain
host-specific user names and paths. Confirm the dated image/digest, architecture,
allocation, zero mounts/additional networks/provisioning hooks, both containerd
flags false, exact SSH/DNS settings and the single catch-all ignore rule.
The [pinned validator](https://github.com/lima-vm/lima/blob/v2.2.1/cmd/limactl/template.go)
merges global defaults/overrides; successful YAML parsing alone does not prove
the effective configuration has these boundaries.

## Observed results — 2026-10-22

Configuration preparation checks, completed before installation:

- YAML parsing and duplicate-key/alias checks: **PASS** with the existing Ruby/Psych.
- Compared the file with P1.1's fragment (applying only the documented IPv6 field
  correction), Ubuntu URL/digest and resource/tool pins: **PASS**.
- Checked declared top-level/nested field names against Lima v2.2.1's source
  types: **PASS**.

Pinned installation and validation checks, completed before VM creation:

- `shasum -a 256` on the downloaded release archive: **PASS**, exact pinned hash
  above. Compared it with the publisher's checksum entry: **PASS**.
- `"$task_lima_bin" --version`: **limactl version 2.2.1**. The installed executable
  is Mach-O ARM64; all installed regular files match the verified archive.
- `"$task_lima_bin" validate environments/local/lima/finance-lab.yaml`: exit **0**,
  **OK**. The source YAML and minimum-version guard are unchanged.
- `"$task_lima_bin" validate --fill environments/local/lima/finance-lab.yaml`:
  exit **0**, **OK**; generated output retained only in a private temporary file.
- Ruby/Psych assertions on the merged output: **PASS** for the pinned image,
  minimum version, VZ/Linux/ARM64, allocation, disabled package upgrades,
  no mounts/additional networks/disks/hooks/copy-to-host entries, disabled
  containerd, exact SSH/DNS values and one ignored automatic forwarding rule
  with loopback host address. X11 forwarding is also disabled.
- The selected Lima home's entries and global configuration are unchanged.
  Its three global configuration files and the `finance-lab` instance directory
  remain absent.

### Ubuntu image and Mac preflight

The following checks were performed after the pinned Lima installation and
before VM creation. Raw
host routes, interface addresses, process details and paths remain outside Git;
only the relevant results are recorded here.

| Check | Observed result |
| --- | --- |
| Ubuntu image bytes | **PASS**: complete download matches the accepted SHA-256 and publisher entry |
| Architecture / virtualization | **PASS**: ARM64, macOS 27.0.1, `kern.hv_support=1` |
| CPU allocation | **PASS** for initial allocation: 12 host CPUs versus 4 proposed vCPUs; sampled CPU idle approximately 85%, load averages below 4 |
| Disk allocation | **PASS**: approximately 535 GiB available on the VM-home filesystem before download; approximately 534.6 GiB afterward, exceeding the planned 60 GiB disk |
| Memory headroom | **PASS on recheck**: 24 GiB physical RAM; memory-pressure level `1` (normal) in both readings, approximately 3.83 GiB swap used and no additional swap-out pages. The earlier warning-level hold is cleared for this snapshot; whole-VM capacity remains untested |
| Specific route/interface overlap | **PASS on recheck**: none overlapping `192.168.5.0/24`, `10.42.0.0/16` or `10.43.0.0/16`; all IPv4 route rows parsed |
| VPN routing | **Conditional**: lookups for planned VM/pod/Service addresses select an active tunnel through default/split-default routes; guest behavior and VPN filtering remain untested |
| Reserved Mac TCP ports | **PASS on recheck**: `60022`, `16443`, `8443`, `18443`, `13000`, `19090`, `19443`; no observed IPv4/IPv6 listeners, and all seven explicit `127.0.0.1` bind probes succeeded and immediately closed without listening |
| Existing Lima state | **PASS**: selected Lima-home entries/global configuration and source YAML unchanged; `finance-lab` instance remains absent |
| Live ingress Service-IP availability | **NOT RUN**: no cluster exists; verify `10.43.0.80` is unused before creating the Traefik Service |

**Recheck — 2026-10-22, 12:27 UTC:** after the maintainer closed unnecessary
applications, repeated the memory, route/interface, VPN and seven-port checks
below. Two memory samples approximately 15 seconds apart were normal, with
zero new swap-out pages. Existing swap use alone was not treated as ongoing
memory pressure. No specific route/interface overlap was found, and all seven
route lookups still used a tunnel. All seven TCP bind probes passed, with no
reserved listeners observed. Lima state and the source YAML are unchanged;
the VM remains absent. The earlier image-checksum, CPU and disk results were
retained, not rerun in this recheck.

Commands/checks actually run:

- Downloaded the exact YAML image URL and the same dated directory's
  `SHA256SUMS` with `curl --fail --silent --show-error --location`, HTTPS-only
  redirect handling and bounded timeouts. Compared the exact manifest filename
  entry and streamed the entire image through Python `hashlib.sha256`; both
  comparisons passed before accepting the file.
- `uname -m`, `sw_vers -productVersion`, `sysctl -n hw.logicalcpu`,
  `sysctl -n hw.physicalcpu`, `sysctl -n hw.memsize`,
  `sysctl -n kern.hv_support`, `sysctl vm.loadavg`, `top -l 1 -n 0`;
  Python `os.statvfs` on the existing ancestor of the selected Lima home.
- Two readings of `sysctl -n kern.memorystatus_vm_pressure_level`,
  `sysctl vm.swapusage`, `memory_pressure -Q` and `vm_stat`.
  No memory stress test or large allocation was performed.
- `netstat -rn -f inet`, `ifconfig`, `scutil --nc list`; Python `ipaddress`
  comparisons of all specific IPv4 routes and assigned interface subnets
  against all three planned CIDRs. Default and split-default catch-all routes
  were recorded separately, not classified as dedicated subnet collisions.
- `route -n get` for `192.168.5.15`, `192.168.5.3`, `10.42.0.1`,
  `10.42.255.254`, `10.43.0.10`, `10.43.0.80` and `10.43.255.254`.
  These are route-table queries, not reachability tests. The Mac does not need
  direct routes to these guest addresses under the agreed SSH-tunnel design.
- `lsof -nP -iTCP -sTCP:LISTEN -FpnT` filtered to the seven reserved ports,
  plus Python `socket(AF_INET, SOCK_STREAM).bind(('127.0.0.1', port))` for each
  port, without `SO_REUSEADDR` or `listen()`, closing each socket immediately.

Apple's [sysctl implementation](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/kern/kern_memorystatus_notify.c)
returns dispatch pressure flags; the [warning flag is 2](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/sys/event_private.h).
The `memory_pressure -Q` percentage is not a guarantee that another 8 GiB can
be assigned comfortably. The earlier warning-level hold is cleared by the
normal readings above. Keep the agreed 8 GiB allocation unchanged and measure
actual host/guest headroom again during later workload validation.
No applications were closed or VPN settings changed by these checks.
A snapshot of free ports and routes is not a reservation; recheck after network,
VPN or workload changes and immediately before creation/start. Tunnel routing
alone does not prove a subnet collision or successful guest networking.

### VM creation without boot — 2026-10-22, 12:36 UTC

Immediately before creation, the exact tool version, absent target instance,
absent global base/default/override files, normal host memory pressure and all
seven free reserved TCP ports were rechecked. Source native validation and
merged-configuration assertions passed. The source YAML was not edited.

From the repository root, using the explicit pinned binary:

```sh
task_lima_bin="$HOME/.local/opt/lima/2.2.1/bin/limactl"
"$task_lima_bin" create --tty=false --name=finance-lab environments/local/lima/finance-lab.yaml
```

The command exited **0**. Lima downloaded the pinned Ubuntu image, converted
it to raw sparse format and prepared a **60 GiB** instance disk. No start
command was run.

Post-create verification:

- `"$task_lima_bin" list --json finance-lab`: **Stopped**, VZ/aarch64,
  4 vCPUs, 8 GiB RAM, 60 GiB disk, no inspection errors.
- Native `validate` and `validate --fill` on the instance's saved
  `lima.yaml`: exit **0**. Ruby/Psych comparisons confirmed the pinned image,
  resource allocation and all previously validated isolation/runtime/SSH/DNS/
  forwarding settings match the source's effective configuration.
- Lima's cached image: **616,522,752 bytes** and the pinned SHA-256 above.
  Instance disk logical size: **64,424,509,440 bytes**; sparse allocation does
  not mean that much physical disk space is already consumed.
- Repository YAML checksum unchanged; no host-agent or VM-driver PID files.
  All seven reserved ports still had no observed listeners and passed loopback
  bind probes.
- Lima's generated disk, cloud-init configuration and instance metadata remain
  outside Git. Raw inspection output and logs remain private.

### First boot, resources and basic networking — 2026-10-22, 12:47 UTC

Rechecked the stopped instance, exact Lima version and effective configuration,
absent global overrides, normal memory pressure, specific route/interface
overlaps and all seven reserved ports before starting. No overlaps were found;
all ports were free. Source and instance YAML checksums remained unchanged.

```sh
task_lima_bin="$HOME/.local/opt/lima/2.2.1/bin/limactl"
"$task_lima_bin" start --tty=false --timeout=10m finance-lab
```

Start exited **0** and Lima reported **READY**. Read-only guest probes ran through
`"$task_lima_bin" shell --workdir=/ finance-lab python3 -`; they did not install
packages, change guest configuration or deploy a workload.

| Check | Observed result |
| --- | --- |
| Boot / services | **PASS with cloud-init warning below**: Lima Running with no inspection errors; systemd running, zero failed units; SSH and Lima guest agent active |
| Guest identity | Ubuntu **24.04.5 LTS**, aarch64, kernel **6.8.0-147-generic** from the pinned image |
| CPU / RAM | **PASS**: 4 online CPUs; 8 GiB configured, approximately 7.74 GiB usable and 7.37 GiB available in the guest |
| Disk / filesystem | **PASS**: guest vda exactly 64,424,509,440 bytes (60 GiB); expanded ext4 root filesystem approximately 57.08 GiB, 55.32 GiB available |
| Clock | **PASS**: `NTPSynchronized=yes` |
| Mounts / runtime | **PASS**: no virtiofs, 9p or SSHFS host mounts; system/user containerd inactive with no containerd executable; no K3s executable or unit; SSH agent not forwarded |
| VM address / route | **PASS**: `192.168.5.15/24`, default gateway `192.168.5.2`, no additional network interface |
| Guest DNS | **PASS; planning correction recorded below**: all three app/api/auth names resolve to `10.43.0.80`; external GitHub resolution succeeds |
| Outbound HTTPS | **PASS** with certificate validation: Ubuntu cloud-image index, Ubuntu ARM64 repository Release metadata and GitHub return 200; GHCR v2 returns its expected unauthenticated 401 |
| Mac access | **PASS for current SSH path**: control listener only on `127.0.0.1:60022`; other six reserved ports have no listeners and pass bind probes; Lima's auxiliary listening endpoints are loopback |
| Host memory | **PASS for this light-load sample**: normal pressure before and after boot, approximately 3.82 GiB existing swap, zero new swap-out pages |

Commands/checks actually run in the guest:

- `cloud-init status --wait --long`; `systemctl is-system-running`;
  `systemctl --failed --no-legend --plain --no-pager`;
  `systemctl is-active ssh` and `systemctl is-active lima-guestagent`.
- `nproc`, `free -b`, `lsblk -b -J -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS`,
  `df -B1 /`, and `timedatectl show --property=NTPSynchronized --property=Timezone`.
- `findmnt -rn -t virtiofs,9p,fuse.sshfs -o TARGET,FSTYPE`;
  system/user `systemctl is-active containerd`; `systemctl list-unit-files k3s* --no-legend --no-pager`;
  Python executable-path checks for containerd/K3s and an SSH-agent-presence check.
- `ip -j -4 address show`, `ip -j -4 route show`, `resolvectl dns`;
  `getent ahostsv4` for each lab name and github.com.
- Bounded `curl` HTTPS requests with normal certificate validation, HTTPS-only
  redirects, 10-second connection and 30-second total timeouts. Only public
  endpoint/metadata reads were performed; no registry authentication or image pull.
- Direct Python socket DNS A queries to both proposed upstream addresses,
  over UDP and TCP, with 3-second timeouts. The functioning address returned
  the expected lab record; UDP external lookup also succeeded.

**Cloud-init warning:** `cloud-init status` exited **2**, reporting
`degraded done`, `errors: []` and only deprecation warnings for Lima-generated
`users.0.ssh-authorized-keys` and string-valued `users.0.uid`. Inspection of the
generated configuration confirmed those fields without exposing key contents.
The guest completed initialization and systemd has no failed units. This is
recorded as a warning, not a clean cloud-init exit or a failed boot. Generated
configuration and version pins were not changed.

**DNS planning correction recorded — 2026-10-22:** the prior first-boot
checks showed DHCP-supplied DNS **192.168.5.2** answering over UDP/TCP, while
**192.168.5.3** timed out. [P1.1 revision 8](https://github.com/finance-lab-project/platform/issues/1)
and [P1.2 revision 3](https://github.com/finance-lab-project/platform/issues/2)
now use **192.168.5.2** for the VM and future K3s upstream.

The planned guest file `/etc/rancher/k3s/resolv.conf` contains
`nameserver 192.168.5.2`, selected with
`resolv-conf: /etc/rancher/k3s/resolv.conf`. The planned firewall permits
destination `192.168.5.2/32`, TCP/UDP destination port 53, in **both OUTPUT and
FORWARD before** denying other NEW gateway connections. Established/related
traffic, Lima DHCP renewal/replies and established Mac-initiated SSH are
preserved. The DNS exception does not permit other Mac services or bypass pod
NetworkPolicy. After implementation, node and permitted-pod UDP/TCP DNS success
must be tested alongside continued Mac operator-port denial.

Both issue bodies were statically checked, updated and reread; saved text
matched the prepared updates, with issue metadata/state unchanged. Artifact
pins, application/operator addresses, credential/recovery appendices and
unchecked acceptance criteria were preserved. The source Lima YAML is
unchanged; its existing DHCP/host-resolver setup already supplies the working
upstream. No guest DNS test was rerun by this documentation step, and no K3s
resolver file or firewall rule was created or applied.

The VM was left **Running** after first boot; this planning correction performed
no VM operation. K3s was not installed. No host DNS entries, firewall rules or
trust stores were changed. Raw logs and host-specific evidence remain outside
Git. Only issues #1/#2 and these local notes were updated; no commit, push, PR or
issue closure is included.

### Normal restart validation — 2026-10-22, 14:14–14:24 UTC

Used the pinned **Lima 2.2.1** binary for a graceful stop and start of the existing
instance, without force, recreation or configuration changes:

```sh
task_lima_bin="$HOME/.local/opt/lima/2.2.1/bin/limactl"
"$task_lima_bin" stop --tty=false finance-lab
"$task_lima_bin" start --tty=false --timeout=10m finance-lab
```

Both commands exited **0**. Between them, Lima reported **Stopped**, the SSH
control connection failed as expected and all seven reserved Mac TCP ports were
free. Start reported **READY**. The guest boot identifier changed and the root
filesystem UUID remained the same; their values are retained only in private
evidence. This validates a normal shutdown/start cycle, not crash recovery.

| Check | Observed result |
| --- | --- |
| Configuration integrity | **PASS**: exact tool version and merged pinned/isolation settings unchanged; source and instance YAML checksums unchanged after restart |
| Boot / services | **PASS with existing cloud-init warning**: Running, no inspection errors, systemd running, zero failed units; SSH and Lima guest agent active; clock synchronized |
| Guest / resources | **PASS**: Ubuntu 24.04.5 LTS, aarch64, kernel 6.8.0-147-generic; 4 CPUs, 8 GiB configured (7.74 GiB usable, 7.41 GiB available), 60 GiB disk; existing ext4 root retained with approximately 55.29 GiB available |
| Runtime boundaries | **PASS**: no virtiofs/9p/SSHFS host mounts, forwarded SSH agent, K3s binary/unit or containerd binary; system/user containerd inactive; planned K3s resolver file absent |
| VM network | **PASS**: only loopback and eth0 IPv4 interfaces; `192.168.5.15/24`, gateway and DHCP DNS `192.168.5.2` retained |
| DNS | **PASS**: app/api/auth names resolve to `10.43.0.80`; direct UDP and TCP queries to `192.168.5.2:53` return the lab A record and an external GitHub A record |
| Outbound HTTPS | **PASS** with certificate validation: Ubuntu cloud-image index, Ubuntu ARM64 Release metadata and GitHub return 200; GHCR v2 returns expected unauthenticated 401 |
| Mac SSH access | **PASS for current operator path**: SSH banner on `127.0.0.1:60022`; all host-agent TCP listeners IPv4 loopback; no listener on the other six reserved ports, all six bind probes pass |
| Mac non-loopback checks | **PASS for host self-probes**: TCP 60022 connections to all three currently assigned non-loopback IPv4 addresses fail; IPv6 loopback connection also fails |
| Automatic forwarding | **PASS for the tested TCP listener**: a temporary guest listener on `0.0.0.0:19876` responds inside the guest, while Mac TCP 19876 has no listener, accepts a bind probe and refuses connection after 10 seconds of discovery time; the guest process then exits 0 and both sides are checked unused |
| Host memory | **WARN**: pressure level 2 before restart, while stopped and after restart; approximately 3.71 GiB swap used at the final sample, with zero new swap-out pages across the checks. Host workload headroom is not a PASS |

Before stopping, all 8,676 IPv4 route rows and assigned interface subnets were
checked against the three planned CIDRs, with no specific overlap. Existing
default/split-default VPN routing was retained. No VPN settings were changed.

Read-only guest checks repeated the commands listed under first boot, including
`cloud-init status --wait --long`, service/resource/mount/runtime checks,
`ip -j -4 address show`, `ip -j -4 route show`, `resolvectl dns`, `getent ahostsv4`,
direct Python socket DNS queries and bounded public HTTPS requests.
`findmnt -n -o UUID /` and `/proc/sys/kernel/random/boot_id` supplied the private
before/after identity comparison. `cloud-init status` still exits **2** with
`degraded done`, `errors: []` and the same two generated-configuration
deprecations documented above; no clean cloud-init exit is claimed.

Mac checks used `limactl list --json finance-lab`,
`lsof -nP -iTCP -sTCP:LISTEN -FpnT`, `ifconfig`, Python socket bind/connect/banner
probes, and PID-filtered `lsof` for the Lima host agent. The forwarding check used
a bounded Python socket listener through `limactl shell --workdir=/ finance-lab`,
with an explicit stop message and confirmed cleanup. No guest file, package or
service was created for it. `sysctl -n kern.memorystatus_vm_pressure_level`,
`sysctl vm.swapusage` and `vm_stat` supplied the host memory samples.

**Limits and pending work:** no second LAN device was used for an external
connection test; Mac-only access evidence is the listener bindings and local
negative connection probes. UDP forwarding was not exercised at runtime; its
catch-all ignore configuration remains validated. Guest firewall and guest-to-Mac
isolation, K3s/pod DNS/networking, ingress Service-IP availability, TLS, workload
capacity and recovery remain separate steps. No application serves the planned
ingress address yet.

The VM is left **Running**. K3s and platform firewall rules were not installed or
applied. No host DNS, trust-store or VM configuration changes were made. This
restart step updates only these local notes; raw evidence remains private and
no issue edits, commit, push, PR or issue closure were performed.

### Guest firewall preparation — 2026-10-22

Prepared [firewall.nft](firewall.nft) and
[finance-lab-firewall.service](finance-lab-firewall.service) locally. Native
nftables check-only validation, a repeated-declaration check-only transaction
and systemd unit verification all exited **0**. The live guest ruleset was
identical before and after, with no `finance_lab` table installed. No rule,
service, K3s component or resolver file was installed or activated.

[FIREWALL.md](FIREWALL.md) records the DNS-before-deny rules for `192.168.5.2`
in OUTPUT/FORWARD, DHCP/SSH and future CNI paths, owned-table persistence,
observed checks and limitations. Enforcement, reboot persistence and later
K3s/NetworkPolicy integration remain untested and require separate execution.


### Guest firewall activation attempt — 2026-10-22

Installed and activated the exact prepared rules and unit with a ten-minute
rollback timer armed. Fresh SSH worked, but the pre-existing heartbeat stalled;
explicit rollback succeeded before the deadline. That attempt left the VM Running
with unchanged boot identity, the original Lima rules and the custom firewall
**disabled/inactive**. DNS, DHCP renewal, fresh SSH and outbound HTTPS passed
recovery checks after rollback. Temporary test listeners were removed.

[FIREWALL.md](FIREWALL.md#activation-attempt-and-recovery--2026-10-22) records the
observed failure and recovery. The temporary connection-tracking retry was not
applied during that first attempt; the subsequently approved execution is
recorded below. No VM restart or K3s installation occurred in either attempt.


### Approved guest firewall retry — 2026-10-22

The unchanged prepared firewall is **active/enabled**. A ten-minute rollback
covered both the persistent and temporary tables. Connection tracking was
confirmed for both existing SSH sessions before activation; the temporary table
was immediately removed. Existing/fresh SSH, actual DHCP REQUEST/ACK, local and
external UDP/TCP DNS, outbound HTTPS and fresh gateway-denial probes all passed.
The deny counter increased by 22 packets while the Mac fixtures remained live.

Rollback was disarmed only after validation. Temporary listeners were removed;
Lima's original rules and VM boot identity are unchanged. See the
[retry evidence](FIREWALL.md#approved-bounded-retry--2026-10-22). Restart persistence,
K3s and pod-policy integration remain pending; no VM restart or K3s installation
was performed.


### Publication checks — 2026-10-22

Before the initial repository publication, the pinned binary reported
`limactl version 2.2.1`. Both commands below exited 0:

```sh
"$HOME/.local/opt/lima/2.2.1/bin/limactl" validate environments/local/lima/finance-lab.yaml
"$HOME/.local/opt/lima/2.2.1/bin/limactl" validate --fill environments/local/lima/finance-lab.yaml
```

The filled output was captured privately. Assertions confirmed the pinned
image digest, VZ/ARM64 allocation, empty mounts/networks/provisioning, disabled
containerd, SSH restrictions, DNS overrides and catch-all forwarding ignore.
Ruby/Psych YAML parsing, local Markdown links, balanced code fences and
whitespace checks passed for the five source files. A scan found no personal
host paths, personal email addresses or common private-key/token markers.

The documented Lima shell and a fresh non-multiplexed direct SSH connection
both succeeded. In the guest, these read-only/check-only commands exited 0:

```sh
systemctl is-active finance-lab-firewall.service
systemctl is-enabled finance-lab-firewall.service
sudo nft --check --file /etc/finance-lab/firewall.nft
sudo systemd-analyze verify /etc/systemd/system/finance-lab-firewall.service
sha256sum /etc/finance-lab/firewall.nft /etc/systemd/system/finance-lab-firewall.service
sudo nft list tables
```

The service remained active/enabled; both installed hashes matched the source
pins recorded in FIREWALL.md. Only the original Lima NAT table and the persistent
`inet finance_lab` table were listed. The rollback timer was inactive; K3s and
its planned resolver file remained absent. The VM stayed Running with unchanged
boot identity. No firewall reload, VM restart or K3s installation was performed.
Earlier DHCP, DNS, HTTPS and gateway-isolation evidence was not rerun in this
publication step. No GitHub Actions or independent review result is claimed.
