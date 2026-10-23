# Guest firewall — platform #2

Status: **installed, active and enabled; normal restart, controlled reload
and tested node ingress/gateway checks passed** (2026-10-22). The firewall loads
before network setup, preserves SSH through reload and denies the tested inbound
TCP/UDP paths. First-start K3s/pod integration and the subsequent K3s-enabled
normal restart checks pass as summarized below. Coverage limits and earlier
attempts remain stage-specific evidence.
Implements the guest-firewall portion of
[platform #1 revision 8](https://github.com/finance-lab-project/platform/issues/1)
and [platform #2](https://github.com/finance-lab-project/platform/issues/2).
K3s is running; its policy engine was tested with a temporary policy. Final
workload-specific NetworkPolicies remain pending.

## Files and ownership

- [firewall.nft](firewall.nft): native nftables configuration, owning only
  `inet finance_lab`. The installed guest location is `/etc/finance-lab/firewall.nft`.
- [finance-lab-firewall.service](finance-lab-firewall.service): prepared systemd
  persistence unit. The installed location is
  `/etc/systemd/system/finance-lab-firewall.service`.

The configuration adds the owned table if absent, clears only that table's
rules, and defines its INPUT/OUTPUT/FORWARD base chains in one transaction.
It does not flush the whole ruleset, edit Lima/K3s tables, alter NAT/marks,
change forwarding sysctls or replace Kubernetes NetworkPolicy.

All three hooks use priority **10**, after the ordinary iptables filter priority
**0**. A drop remains terminal; an accept in one base chain does not override a
drop elsewhere. Thus the DNS exception cannot bypass a K3s policy rejection.
First-start inspection confirmed K3s filter hooks at priority 0, and a temporary
pod egress-deny policy blocked DNS despite this table's exception.
See the [nftables chain documentation](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains).

INPUT defaults to drop. OUTPUT and FORWARD default to accept so this guest layer
preserves node bootstrap egress and CNI/Service routing; future workload-specific
default-deny/allow rules remain the K3s policy controller's responsibility.
This configuration alone does not enforce application egress, LAN isolation,
production-access restrictions or label-based monitoring permissions.

## Intended traffic handling

The following is a static rule review, **not live packet-test evidence**.

| Traffic | Prepared rule behavior |
| --- | --- |
| Guest loopback INPUT | Accept, including local IPv6 loopback |
| Established/related traffic | Accept in each chain, preserving replies on Mac-initiated SSH; other base chains still apply |
| DHCP requests/renewal | OUTPUT: eth0, UDP source 68 to destination 67, destination `192.168.5.2` or limited broadcast `255.255.255.255` |
| DHCP replies | INPUT: eth0, source `192.168.5.2`, UDP 67 to 68, destination VM `192.168.5.15` or limited broadcast; allowed independently of conntrack state |
| New SSH control connection | INPUT: eth0, source gateway `192.168.5.2`, destination VM `192.168.5.15`, TCP 22 |
| Node and forwarded pod DNS | OUTPUT/FORWARD: destination **`192.168.5.2/32`**, TCP or UDP **destination port 53**, before the gateway denial |
| Other new gateway connections | Drop in OUTPUT/FORWARD, including all Mac operator ports; source port 53 does not grant access. Untracked gateway traffic is also dropped; established traffic above is intentionally retained |
| Future pods to node services | INPUT: only via single-node bridge `cni0`, source `10.42.0.0/16`, TCP 6443/10250/9100; subsequent workload-policy and authentication checks remain required |
| Unsolicited eth0 traffic | INPUT default drop except DHCP/SSH above; FORWARD drops eth0 ingress after established/related traffic. No eth0 allowance for API, kubelet, node-exporter, application ingress or VXLAN 8472, even with a spoofed pod source |
| Other new CNI traffic to node listeners | INPUT default drop; there is no blanket pod-CIDR trust |
| Other IPv6 | No new external INPUT allowance; IPv6/dual-stack guest or pod networking is outside this IPv4 lab plan |

DHCP exceptions precede the invalid-state drop to preserve initial broadcast
exchange. Both DNS protocols precede the gateway drop in OUTPUT and FORWARD.
The forwarded DNS exception does not grant unsolicited eth0 ingress. K3s
NetworkPolicy must still limit which pods may use upstream DNS and node ports.

Verified current assumptions: eth0 uses `192.168.5.15/24`; the DHCP server,
gateway and DNS are `192.168.5.2`; Lima's Mac-initiated SSH arrives from that
gateway. K3s first start now confirms `cni0`, `flannel.1` and the single-node
`10.42.0.0/24` pod route within the declared pod CIDR. Interface-name matching
also permits static validation before the bridge exists.

## Persistence boundaries

The prepared unit loads only this file, before network setup and K3s. Reload
replaces only the owned rules. It intentionally has no `ExecStop`; stopping it
does not remove protection or flush another component's rules. A later rollback
must target only `inet finance_lab`, with operator access verified separately.

Do not enable the distribution's stock `nftables.service` for this design:
the inspected Ubuntu unit has `ExecStop=/usr/sbin/nft flush ruleset`, which would
remove Lima/K3s rules as well. The current stock unit is disabled/inactive.
`ufw status` reports inactive; its enabled, active-exited systemd wrapper does
not mean UFW filtering is enabled. Neither service was changed.

`Before=k3s.service` establishes ordering, not a failure dependency. The installed
K3s drop-in now adds both `Requires` and `After` for this service; the merged unit
was verified before first start. The first activation was rolled back; the approved retry below
left this unit active and enabled. The later normal-restart check below verified
automatic boot loading; the subsequent controlled-reload check also passed.

## Static validation performed

Used the existing pinned Lima **2.2.1** and existing guest packages:
`nftables 1.0.9-1ubuntu0.1` (`nftables v1.0.9`),
`iptables 1.8.10-3ubuntu2` (nf_tables backend). No packages were installed.

From this repository, the native check-only command was equivalent to:

```sh
task_lima_bin="$HOME/.local/opt/lima/2.2.1/bin/limactl"
"$task_lima_bin" shell --workdir=/ finance-lab \
  sudo -n nft --check --file - < environments/local/lima/firewall.nft
```

- Native `nft --check --file -`: **exit 0**, empty stderr. This checks the
  configuration against the running guest kernel without committing it, as
  documented for [nft check mode](https://netfilter.org/projects/nftables/manpage.html).
- Repeated the owned-table declarations twice in one check-only transaction,
  with shared variables defined once: **exit 0**. This checks add/clear/redeclare
  syntax; it is not evidence of two live reloads or persistence across boot.
- `systemd-analyze verify <temporary-directory>/finance-lab-firewall.service`:
  **exit 0**, empty stderr. The temporary unit file was removed. No daemon reload
  or service execution was performed.
- Compared `sudo -n nft --stateless list ruleset` before and after native checks:
  **identical**. The existing Lima NAT table/`LIMADNS` chain remains; the
  `inet finance_lab` table remains absent. Raw output stays outside Git.
- Reviewed rule ordering, interface/source restrictions, exact destination-port
  DNS exceptions, DHCP/SSH continuity, owned-table scope and the traffic cases
  above. This is static review, not a network simulation or enforcement test.

## Activation attempt and recovery — 2026-10-22

Installed the exact prepared files as root-owned mode `0644`:

- `firewall.nft` SHA-256: `b1c77f6004e87e3ce2aefd626609ab1881bfb1e0473a0f7c6505ec3df58411de`.
- `finance-lab-firewall.service` SHA-256: `719f81d62a2090a2db099fe8d4cdc1aff7db509197a605f726911f007f42d114`.

A guest-local, ten-minute `finance-lab-firewall-rollback.timer` was confirmed
active before installation/application. Its root-only Python procedure disables
and stops only `finance-lab-firewall.service`, then deletes only
`table inet finance_lab`. A lock and armed marker guard execution. The first
attempt to launch the script directly failed because `/run` is mounted `noexec`;
no firewall was applied then. Invoking it through `/usr/bin/python3` successfully
armed the timer before the subsequent application.

Native `nft --check --file /etc/finance-lab/firewall.nft` and
`systemd-analyze verify /etc/systemd/system/finance-lab-firewall.service` completed
without an error before `systemctl daemon-reload` and
`systemctl enable --now finance-lab-firewall.service`. Runtime inspection showed
**enabled**, **active (exited)** and `ExecStart` **status 0/SUCCESS**. The host
application command nevertheless timed out after 120 seconds on its existing
SSH transport; this is not recorded as a successful end-to-end application.

A fresh, non-multiplexed SSH connection succeeded under enforcement. However,
the independent SSH heartbeat established before application stopped advancing.
The OUTPUT gateway-drop counter advanced while the old connection was stalled.
This failed the existing-SSH acceptance gate, so further enforcement tests were
not continued and rollback was explicitly triggered before the deadline:

```sh
sudo systemctl start finance-lab-firewall-rollback.service
sudo systemctl stop finance-lab-firewall-rollback.timer
```

Observed rollback result: **`Result=success`, `ExecMainStatus=0`**. The old
heartbeat resumed. After that first rollback, the firewall service was
disabled/inactive, its table was absent and the rollback timer was inactive. A stateless JSON comparison of the
remaining ruleset matched the pre-application Lima rules exactly. The installed
configuration and unit still match the hashes above. The timer's automatic
expiry was not exercised; its rollback action was invoked manually.

After rollback, an established SSH socket lacked a matching conntrack entry.
This supports a first-activation connection-tracking explanation, but packet
tracing during the failure was not performed. A live TCP socket alone does not
prove the firewall has seen both directions of that connection; see the
[nftables connection-state documentation](https://wiki.nftables.org/wiki-nftables/index.php/Matching_connection_tracking_stateful_metainformation).
The prepared persistent rules have not been changed.

**Recovery checks, with the firewall rolled back:**

- Fresh SSH succeeded and the original heartbeat resumed before cleanup.
- `networkctl renew eth0` exited 0; packet capture observed a matching DHCP
  REQUEST/ACK. The lease retained node `192.168.5.15`, gateway/DNS/server
  `192.168.5.2`. The lease file did not need rewriting. Initial capture/check
  attempts failed because outgoing packets were omitted and an unchanged lease
  file's mtime was incorrectly treated as a renewal requirement; the corrected
  check uses the matching exchange and lease values.
- Direct UDP and TCP DNS to `192.168.5.2` resolved both the local application
  name and an external name. System resolution of app/api/auth local names
  returned `10.43.0.80`.
- Outbound HTTPS to Ubuntu images, Ubuntu package metadata and GitHub returned
  200; GHCR `/v2/` returned the expected unauthenticated 401.
- VM boot identity was unchanged; systemd reported running with no failed units;
  K3s and its planned resolver file remained absent.
- Six temporary Mac loopback TCP listeners, one UDP listener and the independent
  SSH heartbeat were closed; all fixture ports were released.

Before application, the seven gateway TCP targets (including Lima SSH) and the
UDP fixture had positive reachability/banner checks. During this first attempt,
the negative enforcement checks were not reached after the SSH failure; that
attempt did not establish gateway isolation. DNS, DHCP and outbound checks above
prove recovery, not behavior under this firewall. No VM restart, K3s installation, stock nftables/UFW changes, host
DNS/trust changes, GitHub updates, commit or push occurred.

## Approved bounded retry — 2026-10-22

The operator explicitly approved the temporary connection-tracking table and
rollback coverage for both owned tables after the initial approval-review block.
The retry kept the prepared persistent configuration and unit byte-for-byte
unchanged. Pinned Lima 2.2.1 and the existing guest tools were used; no packages
were installed.

Before enforcement, the gateway targets had positive TCP/UDP banner checks and
a separate SSH heartbeat was established. A fresh guest-local ten-minute
`finance-lab-firewall-retry-rollback.timer` was confirmed active. Its root-only
procedure disables/stops the custom firewall service and removes only
`inet finance_lab` and `inet finance_lab_bootstrap`; a lock and armed marker
serialize rollback and acceptance. The script is invoked through
`/usr/bin/python3`, respecting the guest's `/run` noexec mount.

The approved temporary table was applied:

```nft
table inet finance_lab_bootstrap {
    chain input {
        type filter hook input priority 10; policy accept;
        ct state established,related counter;
    }
    chain output {
        type filter hook output priority 10; policy accept;
        ct state established,related counter;
    }
}
```

These counter-only hooks activate connection tracking without additional
filtering and cannot override drops in other base chains. A netlink conntrack
read matched both existing SSH sockets with entries marked as having seen both
directions; the heartbeat continued advancing. Then the native configuration
and unit checks passed and the unchanged persistent service was activated:

```sh
sudo nft --check --file /etc/finance-lab/firewall.nft
sudo systemd-analyze verify /etc/systemd/system/finance-lab-firewall.service
sudo systemctl enable --now finance-lab-firewall.service
sudo nft delete table inet finance_lab_bootstrap
```

All completed with exit 0. The temporary table was immediately removed, while
the timed rollback remained armed throughout the following runtime checks.

| Check under enforcement | Observed result |
| --- | --- |
| Existing SSH | Independent heartbeat advanced before/after activation and through validation; Lima control SSH remained usable |
| Fresh SSH | Separate non-multiplexed connections succeeded before and after DHCP and gateway probes |
| DHCP | `networkctl renew eth0` exited 0; raw capture saw a matching REQUEST/ACK; node `.15` and gateway/DNS/server `.2` were retained |
| DNS | Direct UDP and TCP queries to `192.168.5.2` resolved local and external names; app/api/auth system resolution returned `10.43.0.80` |
| Outbound HTTPS | Ubuntu images, Ubuntu package metadata and GitHub returned 200; unauthenticated GHCR `/v2/` returned the expected 401 |
| Gateway TCP denial | Fresh connections to `192.168.5.2` ports 60022, 16443, 8443, 18443, 13000, 19090 and 19443 were blocked |
| Gateway UDP denial | Fresh UDP traffic to `192.168.5.2:19877` was rejected with EPERM |
| Evidence for denial | All targets were reachable before enforcement; Mac fixture listeners remained responsive locally; the OUTPUT gateway-drop counter increased by 22 packets during the completed negative probe set |
| Rule ownership | Stateless JSON comparison showed the existing Lima rules unchanged; no temporary tracking table remained |

The first UDP probe raised EPERM outside its error handler; the harness was
corrected to record immediate rejection as blocked and the gateway checks were
rerun. No firewall change was needed. DHCP acceptance uses the matching exchange
and retained lease values, not whether an unchanged lease file was rewritten.
These checks do not prove every gateway port or any future pod path individually.

After all checks passed, acceptance atomically replaced the armed marker under
the rollback lock and stopped the timer. The retry rollback action never ran;
automatic expiry was not exercised. Final inspection confirmed:

- `finance-lab-firewall.service`: **active (exited), enabled**; installed hashes
  still match the exact prepared files recorded above, root-owned mode `0644`.
- Retry rollback timer/service: **inactive**, accepted marker present, armed and
  rolled-back markers absent; only the persistent firewall table remains.
- Temporary Mac listeners and SSH heartbeat closed; all fixture ports released.
  A final fresh SSH succeeded and Mac TCP 60022 listened only on `127.0.0.1`.
- VM **Running**, boot identity unchanged, no K3s binary or planned K3s resolver
  file. No VM restart, host DNS/trust change, stock firewall service change,
  GitHub write, commit or push occurred.

## Normal restart persistence — 2026-10-22

The maintainer authorized a normal restart and repeated node-level network
checks. Used the existing pinned Lima **2.2.1** and unchanged source/installed
firewall files. No firewall rule was reapplied, no unit was reloaded and no
package or K3s component was installed.

From the Mac:

```sh
task_lima_bin="$HOME/.local/opt/lima/2.2.1/bin/limactl"
"$task_lima_bin" stop --tty=false finance-lab
"$task_lima_bin" start --tty=false finance-lab
```

Both exited **0**. Between them, Lima reported Stopped and all seven reserved
Mac TCP ports were closed and bindable. Start reached READY after an initial
transient SSH readiness retry. The guest boot identifier changed, the root
filesystem UUID remained the same and the instance YAML checksum was unchanged;
private identifiers and raw logs remain outside Git.

The new boot's journal and `systemctl show` timestamps place successful firewall
completion at approximately **1.790 seconds**, before `network-pre.target`
(**2.006 seconds**), networkd startup (**2.017 seconds**) and SSH startup
(**5.630 seconds**). DHCP address acquisition was logged at **2.048 seconds**.
The service was active (exited), enabled, Result=success and ExecMainStatus=0
before the network tests. This establishes automatic boot persistence without
a manual firewall application after restart.

| Post-restart check | Observed result |
| --- | --- |
| Firewall/configuration integrity | Source/installed SHA-256 hashes and instance YAML unchanged; native `nft --check` and `systemd-analyze verify` exited 0 |
| Rule ownership | Exactly `inet finance_lab` and Lima's `ip nat` table; stateless contents and rule ordering within each table match the pre-restart snapshot. Table listing order changed because the firewall now loads earlier; runtime handles/counters were excluded from this comparison |
| VM state | Running, systemd running, zero failed units; 4 vCPUs, 8 GiB configured RAM and 60 GiB disk retained |
| DHCP at boot | Journal records `192.168.5.15/24`, gateway/server `192.168.5.2`, acquired after the firewall loaded |
| DHCP renewal | `sudo networkctl renew eth0` exited 0; a bounded packet capture observed a matching REQUEST/ACK. Address, router, server and DNS retained their expected values |
| DNS | System resolution of app/api/auth returns `10.43.0.80`; direct UDP and TCP queries to `192.168.5.2:53` return the local application record and an external GitHub A record |
| Outbound HTTPS | Certificate-validated requests to Ubuntu images, Ubuntu package metadata and GitHub return 200; GHCR `/v2/` returns the expected unauthenticated 401 |
| Gateway isolation | Fresh TCP connections to gateway ports 60022, 16443, 8443, 18443, 13000, 19090 and 19443 time out; UDP 19877 is rejected with EPERM. The gateway OUTPUT drop counter increases by **22 packets** during these probes |
| Denial evidence | The SSH listener plus six temporary TCP fixtures and one UDP fixture respond on Mac loopback before and after the gateway probes. No firewall exception was added to make a positive guest connection; live fixtures and correlated drop counters support the denial result |
| Mac SSH access | Fresh non-multiplexed SSH succeeds before shutdown and after the completed network checks; all host-agent TCP listeners remain IPv4 loopback. TCP 60022 is bound only to `127.0.0.1`; host self-probes to three assigned non-loopback IPv4 addresses and IPv6 loopback fail |
| Cleanup / boundaries | Temporary fixture process exits 0; all six TCP fixture ports and UDP 19877 are bindable again. VM stays Running; no K3s binary, K3s resolver file, host mounts or forwarded SSH agent appears |

Read-only/check-only commands included:

```sh
systemctl show finance-lab-firewall.service
journalctl -b -u finance-lab-firewall.service -u systemd-networkd.service --no-pager -o short-monotonic
sudo nft --check --file /etc/finance-lab/firewall.nft
sudo systemd-analyze verify /etc/systemd/system/finance-lab-firewall.service
sudo nft --stateless --json list ruleset
sudo nft --json list table inet finance_lab
ip -j -4 address show
ip -j -4 route show
resolvectl dns
getent ahostsv4 app.finance-lab.test
```

The same system-resolution check was run for api/auth and an external name.
Bounded Python socket probes exercised direct UDP/TCP DNS, DHCP capture, Mac
fixture banners and fresh gateway connections. HTTPS checks used `curl` with
certificate validation, HTTPS-only URLs/redirects, a 10-second connection timeout
and 30-second total timeout. Fresh SSH used Lima's generated config with
`ControlMaster=no`, `ControlPath=none`, `BatchMode=yes` and `ConnectTimeout=5`.

One harness assertion initially required the dedicated DHCP INPUT rule's counter
to increase. That counter remained zero; replies can match the earlier
established/related accept rule. The assertion was corrected to use actual DHCP
boot acquisition, the matching renewal exchange and retained lease values.
Previously captured gateway results/counters were revalidated without changing
firewall rules or repeating the packet probes. A fresh SSH check then passed.
The unchanged lease-file mtime is not treated as a failure.

The existing cloud-init result remains **exit 2 / degraded done**, with
`errors: []` and the same generated-configuration deprecations for
`users.0.ssh-authorized-keys` and string-valued `users.0.uid`. No clean cloud-init
exit or fix is claimed. Mac non-loopback evidence is from host self-probes, not
an independent LAN machine. These results cover one graceful stop/start and
the tested node paths; they do not establish crash recovery, every gateway port,
future pod networking or whole-issue completion. No commit, push or GitHub edit
was performed in this validation step.

## Controlled reload and inbound denial — 2026-10-22

Used the existing Lima **2.2.1**, unchanged firewall file/unit and guest tools.
The VM remained Running with the same boot identity; no K3s installation,
package installation, persistent forwarding change or firewall policy edit was
performed.

For reload, captured the active owned table and checked a restoration transaction
that adds/flushes only `inet finance_lab`. Armed a root-owned guest-local
three-minute systemd rollback timer to restore that snapshot. It never disables
the firewall or flushes Lima's table. An independent, non-multiplexed SSH
heartbeat was already running when these commands executed:

```sh
sudo nft --check --file /etc/finance-lab/firewall.nft
sudo systemctl reload finance-lab-firewall.service
```

Both exited **0**. The existing SSH heartbeat continued, fresh SSH succeeded,
local application DNS resolved and certificate-validated HTTPS to GitHub returned
200. Stateless rule contents/order, source/installed hashes, instance YAML and
boot identity were unchanged. Lima's NAT table was unchanged. The service stayed
active/enabled. Acceptance cancelled the timer after these checks; rollback did
not execute and timer expiry was not tested. The heartbeat exited cleanly.

Inbound tests used temporary wildcard guest listeners and a bounded raw-packet
capture on the **actual `eth0` interface**. Each listener responded on guest
loopback before and after the remote probes. TCP probes used Lima's existing
[bounded virtual-network connection endpoint](https://github.com/lima-vm/lima/blob/v2.2.1/pkg/networks/usernet/gvproxy.go#L232).
UDP probes used two temporary Mac-loopback-only forwards through the
[pinned forwarder API](https://github.com/containers/gvisor-tap-vsock/blob/v0.8.9/pkg/client/client.go),
removed in the test driver's cleanup block. Automatic forwarding and the saved
Lima configuration were not changed.

| Inbound path | Observed result |
| --- | --- |
| TCP 22 positive control | Virtual-network connection succeeds (endpoint HTTP 200); fresh direct SSH also succeeds |
| TCP 80, 443 | Ingress-port fixtures remain responsive locally; incoming gateway connections time out (endpoint HTTP 408) |
| TCP 6443, 10250, 9100 | Management-port fixtures remain responsive locally; incoming gateway connections time out (endpoint HTTP 408) |
| TCP 19876 | Arbitrary-port fixture remains responsive locally; incoming gateway connection times out |
| UDP 8472 and 19876 | Synthetic datagrams arrive on eth0, but no reply reaches the Mac and neither datagram reaches the guest listener |
| Packet/fixture evidence | Capture records all nine targeted protocol/port pairs, including SSH, across 15 incoming packets from `192.168.5.2` to `192.168.5.15`. Listener logs contain only the 16 expected loopback positive-control exchanges; no remote probe reaches a listener |

The TCP endpoint timeout was set to two seconds. The UDP forwards temporarily
mapped Mac `127.0.0.1:19878` to guest `192.168.5.15:8472` and Mac
`127.0.0.1:19879` to guest `192.168.5.15:19876`, with two-second receive timeouts.
The listener/capture process had a three-minute ceiling and was explicitly
stopped after the probes. No temporary allow rule or diagnostic firewall table
was added. These are node ingress checks against live synthetic listeners, not
running Kubernetes components.

Final verification confirmed the exact pre-test stateless rules, file hashes,
instance configuration and boot identity, active/enabled firewall, and fresh SSH.
All guest fixture ports were released. Both temporary UDP forwards and test
processes were removed; the forwarder list returned to only
`127.0.0.1:60022` -> `192.168.5.15:22`. Cancelled rollback and fixture files under
`/run` were removed after their results were retained privately. No raw packet
payload, host identity or secret was added to this repository.

The host's first `py_compile` invocation was blocked by its cache-directory
permissions; in-memory `compile(..., 'exec')` validation passed without changing
host permissions. The runtime tests completed successfully. This step records
local evidence only; no commit, push or GitHub edit was performed.

## K3s first-start integration — 2026-10-22

[Detailed evidence and commands](k3s/README.md#first-start-and-bounded-runtime-validation--2026-10-22)
record the authorized first-start checks. The owned table is byte-identical
ignoring counters; no firewall reload or VM restart occurred. Live K3s
iptables-nft chains have priority 0 before the owned priority-10 hooks.

Node and diagnostic-pod UDP/TCP DNS to `192.168.5.2` pass. Both can use cluster
DNS, the private Traefik Service and outbound HTTPS. Node/API and Service/API
paths validate the native CA and return 401 without credentials; pod-to-kubelet
`/pods` returns 401, while pod-to-node SSH is denied. DHCP REQUEST/ACK and fresh
Mac SSH still pass.

Against live Mac-loopback controls, node and pod connections to all seven tested
gateway TCP targets and UDP 19877 fail, with each relevant gateway-drop counter
increasing by 22. A temporary policy selecting only the diagnostic pod blocks
UDP/TCP gateway DNS despite the guest firewall exception, and removal restores
DNS. Temporary resources/listeners were removed; no persistent application
policy or operator tunnel was introduced.

## K3s-enabled restart — 2026-10-22

[Detailed evidence and commands](k3s/README.md#boot-enablement-and-normal-restart--2026-10-22)
record boot enablement and a graceful Lima stop/start. The firewall became active
at 1.792 seconds after boot, before K3s execution at 5.326 seconds. Both services
are enabled and active; the owned stateless table and installed hashes are
unchanged. K3s filter hooks remain at priority 0 before the owned priority 10.

Repeated node/pod DNS, private Service/API routing, outbound HTTPS, gateway
isolation and temporary NetworkPolicy precedence checks pass. Captured DHCP
REQUEST/ACK retains the expected address/gateway/DNS, and fresh non-multiplexed
Mac SSH succeeds. Temporary namespace, policy and echo fixtures were removed;
all reserved Mac API/application/operator TCP ports remain closed. No firewall
reload or persistent workload policy was applied during this stage.

## Live Mac API tunnel — 2026-10-22

The [Mac API-only access checks](MAC-ACCESS.md#validation--2026-10-22) verified the
actual native-TLS listener on `127.0.0.1:16443`. Node and diagnostic-pod connections
to `192.168.5.2:16443` time out while Mac controls succeed before/after; owned
OUTPUT and FORWARD drop counters each increase by 3. Pod DNS still works. The
owned table, VM boot and K3s state are unchanged. Temporary resources and the
foreground tunnel were removed, and reserved Mac ports are closed again.

## Remaining gates

Application/operator tunnels, internal TLS and real credential/workload policies
remain separate stages. The API-only tunnel is validated; the combined-tunnel
maintenance transition is pending. Monitoring,
application-to-node/operator/LAN restrictions and production isolation require
the actual workloads and their policy rules. No spoofed-source, IPv6 or
independent LAN-machine coverage, full storage/recovery acceptance or whole-issue
PASS is claimed. Changes and evidence remain local; no GitHub status was updated.
