# Mac Kubernetes API access — platform #2

Use the dedicated `finance-lab-local` context and an explicit foreground SSH
forward. The Mac endpoint is `https://127.0.0.1:16443`, carried to guest
`127.0.0.1:6443`. K3s's native CA and client certificate authenticate the
connection; no mkcert trust or application ingress is involved in this stage.

## Pinned client and credential location

| Item | Local selection |
| --- | --- |
| Client | kubectl `v1.36.5`, `darwin/arm64`, matching K3s `v1.36.5+k3s1` |
| Binary | `$HOME/.local/opt/kubectl/1.36.5/bin/kubectl` |
| SHA-256 | `c5850a4a6b9469b26cce47c1ab456285f58e308283dcdf72b4ddda3db49b0ec0` |
| Kubeconfig | `$HOME/.kube/finance-lab/config`, owner-only mode 0600 in directory mode 0700 |
| Context, cluster and user names | `finance-lab-local` |
| Existing default client/config | Unchanged; no PATH replacement, default-context switch or merge |

The accepted client pin is recorded in [artifacts.lock.json](k3s/artifacts.lock.json).
Verify the [download](https://dl.k8s.io/release/v1.36.5/bin/darwin/arm64/kubectl)
against both this SHA-256 and the
[publisher checksum](https://dl.k8s.io/release/v1.36.5/bin/darwin/arm64/kubectl.sha256)
before installing or executing it. Installation is alongside the existing Mac
client, using the explicit path above.

This kubeconfig carries the local cluster administrator identity. Keep it outside
Git, shared mounts, applications and CI. It contains no production credentials.
Do not print `kubectl config view --raw` output or attach it to issue/PR evidence.

## Open the API-only tunnel

With the VM already running, keep this command in its own terminal:

```sh
ssh -F "$HOME/.lima/finance-lab/ssh.config" \
  -o ControlMaster=no -o ControlPath=none \
  -o ExitOnForwardFailure=yes -o GatewayPorts=no \
  -o ForwardAgent=no -o BatchMode=yes \
  -N -L 127.0.0.1:16443:127.0.0.1:6443 \
  lima-finance-lab
```

An occupied port is an error; do not choose another address/port or share a
multiplexed connection. There is no background process, autostart job, wildcard
bind or application/operator forward in this command. Port 8443 stays closed.

In a second terminal:

```sh
"$HOME/.local/opt/kubectl/1.36.5/bin/kubectl" \
  --kubeconfig "$HOME/.kube/finance-lab/config" \
  --context finance-lab-local --request-timeout=10s get nodes

"$HOME/.local/opt/kubectl/1.36.5/bin/kubectl" \
  --kubeconfig "$HOME/.kube/finance-lab/config" \
  --context finance-lab-local --request-timeout=10s \
  -n kube-system get deployments
```

Press **Ctrl-C in the tunnel terminal** after use. Check that its listener is gone:

```sh
lsof -nP -iTCP:16443 -sTCP:LISTEN
```

No listener output is expected after shutdown. API commands then fail until the
foreground tunnel is opened again; the VM and K3s continue running.

## Export or refresh the protected kubeconfig

The initial export reads K3s's actual CA/client credentials over Lima SSH,
changes only the endpoint and local entry names, and writes the dedicated Mac
file. Existing default configuration is not read or modified by this procedure.
The following command refreshes only this lab's file atomically; it never prints
credential contents:

```sh
python3 - <<'PY'
import json, os, subprocess, tempfile
from pathlib import Path

os.umask(0o077)
home = Path.home()
ssh = ['ssh', '-F', str(home / '.lima/finance-lab/ssh.config'),
       '-o', 'ControlMaster=no', '-o', 'ControlPath=none',
       '-o', 'BatchMode=yes', 'lima-finance-lab']
r = subprocess.run(ssh + ['sudo k3s kubectl config view --raw --minify -o json'],
                   check=True, capture_output=True, text=True, timeout=30)
source = json.loads(r.stdout)
assert len(source['clusters']) == len(source['users']) == len(source['contexts']) == 1
cluster = source['clusters'][0]['cluster']
user = source['users'][0]['user']
assert set(cluster) == {'server', 'certificate-authority-data'}
assert cluster['server'] in {'https://127.0.0.1:6443', 'https://0.0.0.0:6443'}
assert set(user) == {'client-certificate-data', 'client-key-data'}
name = 'finance-lab-local'
config = {
    'apiVersion': 'v1', 'kind': 'Config',
    'clusters': [{'name': name, 'cluster': {
        'server': 'https://127.0.0.1:16443',
        'certificate-authority-data': cluster['certificate-authority-data']}}],
    'users': [{'name': name, 'user': user}],
    'contexts': [{'name': name, 'context': {'cluster': name, 'user': name}}],
    'current-context': name,
}
directory = home / '.kube/finance-lab'
assert not directory.is_symlink()
directory.mkdir(mode=0o700, parents=True, exist_ok=True)
directory.chmod(0o700)
fd, temporary = tempfile.mkstemp(prefix='.config-', dir=directory)
try:
    with os.fdopen(fd, 'w') as output:
        json.dump(config, output, indent=2)
        output.write('\n')
    os.replace(temporary, directory / 'config')
finally:
    if os.path.exists(temporary):
        os.unlink(temporary)
print('Protected finance-lab-local kubeconfig refreshed.')
PY
```

K3s can renew the inline client certificate at startup; an exported copy does not
update itself. Refresh it after credential/CA changes and before certificate
expiry, then rerun the authenticated API checks. This is the manual refresh
procedure described by [K3s cluster access documentation](https://docs.k3s.io/cluster-access).
No rotation, new cluster identity or secret manager is introduced here.

## Validation — 2026-10-22

The accepted client download, private kubeconfig export and foreground API-only
forward were completed. The tunnel was closed after verification; opening the
command above is an explicit operator action. No VM/service restart, firewall
reload, cluster configuration change, application route or host DNS/trust change
was made.

| Observed check | Result |
| --- | --- |
| Client integrity | Complete download and installed bytes match the accepted SHA-256 and publisher checksum; executed client reports `v1.36.5`, `darwin/arm64`; API server reports `v1.36.5+k3s1` |
| Credential separation | Dedicated context/cluster/user and loopback endpoint; original CA/client credentials preserved. Directory 0700, file 0600, owned by the Mac operator; no exec/auth-provider, proxy URL or insecure TLS flag |
| Native TLS | K3s CA validates the server certificate, validity and IP SAN `127.0.0.1`; missing CA trust and a deliberately wrong server name are rejected |
| Authentication | Unauthenticated `GET /api/v1/nodes` receives HTTP 401; dedicated client credentials access `/readyz`, nodes and kube-system deployments |
| Cluster identity/health | Same node UID, Ready, four bundled deployments available; guest boot identity, K3s state and guest kubeconfig fingerprint unchanged |
| Listener exposure | `lsof` shows only `127.0.0.1:16443`; connections to three current Mac non-loopback IPv4 addresses and IPv6 loopback fail. No independent LAN-host test was run |
| Port collision | A second identical forward exits nonzero with a bind failure; the first tunnel continues to serve the API. No fallback port is selected |
| Live gateway boundary | Node and a restricted diagnostic pod cannot connect to `192.168.5.2:16443` while Mac native-TLS controls succeed before/after. Owned OUTPUT/FORWARD gateway-drop counters each increase by 3 |
| DNS and RBAC | Diagnostic pod resolves cluster API and private app names. The default ServiceAccount in its temporary namespace is denied `create pods/portforward`, `create pods/exec` and `get secrets`; operator authorization checks succeed |
| Cleanup/fail-closed behavior | Temporary namespace/pod removed. Ctrl-C closes the foreground tunnel; API requests report connection refused. All six reserved Mac TCP ports pass IPv4/IPv6 loopback bind checks |
| Existing state | Default Mac client binary/target and default kubeconfig hashes unchanged. Owned stateless firewall unchanged; fresh SSH and guest API readiness still pass |

API checks used the pinned client, explicit kubeconfig/context above and:

```sh
# Arguments appended to the explicit kubectl invocation above:
version -o json
get --raw=/readyz
get nodes -o json
-n kube-system get deployments -o json
auth can-i get pods -A
auth can-i create pods/portforward -n kube-system
--tls-server-name=not-finance-lab.invalid get --raw=/readyz
```

The last command is a deliberate negative check: TLS validation must fail.
Python's verified SSL context used only the native public CA to test the IP SAN
and unauthenticated HTTP status. Socket tests covered the actual Mac addresses;
node and pod gateway probes used a three-second connection timeout and compared
owned firewall counters. The pod used the locked ARM64 BusyBox digest, non-root
UID, dropped capabilities, read-only root filesystem, resource limits, no token
mount and no host mounts. Pod network probes used `nsenter` from the node; DNS
queries used `kubectl exec` in the diagnostic pod. RBAC checks used
`--as=system:serviceaccount:platform-validation:default` in its temporary namespace.
These checks do not establish final application ServiceAccount policies.

The first download hit its 120-second timeout; the remaining four byte ranges
were fetched and the complete artifact passed both checksum comparisons before
installation. An export guard initially expected a loopback guest server field;
the actual K3s-generated field was `https://0.0.0.0:6443`, matching the configured
guest bind. Only the exported endpoint was changed to `127.0.0.1:16443`.
Temporary probe quoting and closed-tunnel error-wording assertions were corrected
before their checks passed; no runtime configuration change was needed.

Application TLS, the combined API/application tunnel and its maintenance
transition, backup recovery and whole-issue completion remain separate gates.
The API-only endpoint is validated; no application/operator UI forward or
persistent workload policy has been installed. Changes/evidence remain local;
no commit, push or GitHub status update was performed in this stage.
