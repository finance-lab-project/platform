# Guest firewall — platform #2

Status: **installed, active and enabled; requested runtime checks passed**
(2026-10-22). The approved retry preserved existing SSH, removed its temporary
tracking table and cancelled rollback after validation. Restart persistence and
K3s/pod integration remain untested. The initial failed attempt is retained below.
Implements the guest-firewall portion of
[platform #1 revision 8](https://github.com/finance-lab-project/platform/issues/1)
and [platform #2 revision 3](https://github.com/finance-lab-project/platform/issues/2).
K3s and NetworkPolicies remain pending; this is guest-level enforcement only.

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
Inspect the actual K3s chains and hook priorities after its separate installation;
current validation cannot prove their future integration.
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
gateway. `cni0` and its pod range are future single-node Flannel assumptions,
not an observed CNI. Interface-name matching allows syntax validation before
that bridge exists. Do not broaden permissions if installed CNI behavior differs;
review the observed path against the accepted plan first.

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

`Before=k3s.service` establishes ordering, not a failure dependency. When K3s is
later installed, its unit must require and start after this firewall service so
a failed firewall start also prevents K3s startup. No K3s unit/drop-in is created
in this step. The first activation was rolled back; the approved retry below
left this unit active and enabled. Restart persistence remains untested.

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

## Remaining gates

Normal restart persistence, owned-table reload and broader inbound-denial
coverage remain pending. No restart was performed in this step. After K3s and
the permitted workload policies exist, verify pod gateway isolation, actual hook
ordering, API/Service routing and allowed monitoring paths alongside denied
application-to-node/operator/LAN flows. No whole-issue PASS is claimed.
