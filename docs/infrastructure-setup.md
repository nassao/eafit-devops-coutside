# CourtSide infrastructure preparation and recovery guide

**Operational guide — 11 November 2026.** This guide covers infrastructure, not CourtSide application implementation. It records the reference environment and identifies checks that must pass when reproducing it. Commands marked **Mac** run on the workstation; commands marked **VPS** run on the Linux server. Replace placeholders before running anything.

**Current integration status:** the reference environment passed all three jobs in [pipeline 3052919118](https://gitlab.com/nassao/courtside/-/pipelines/3052919118), commit `8bea63b`: test database, rootless image build/push, and restricted Kubernetes deployment/rollout/cleanup (61 seconds). The test workload was removed and PostgreSQL remained healthy. The generalized commands in this guide were reviewed and syntax-checked, but the entire guide has not been replayed on a second clean VPS. No CourtSide application code has been written.

## 1. Choose your route

- **Instructor reference:** an independent Contabo VPS, controlled from a Mac. PostgreSQL runs in rootful Podman; GitLab jobs run through a separate rootless Podman account; K3s uses its own containerd runtime.
- **Participant teams:** use your own authorized, preinstalled Kubernetes service, working Podman installation, GitLab account/project, registry and compatible runner. You do not need to purchase a VPS per team. Adapt networking and namespace administration with the environment owner. Never apply the instructor's host configuration blindly to corporate or shared infrastructure.

Each pair or trio gets its own fork and resources. The instructor server does not host all participant environments. A single-node VPS is a teaching environment, not high availability or a production reference architecture.

## 2. Inventory, prerequisites and placeholders

| Item | Reference checked in this environment |
| --- | --- |
| Server | Ubuntu 24.04.5 LTS, amd64, 6 vCPU, 12 GB nominal RAM, 100 GB nominal NVMe |
| Host Podman | 4.9.3 from Ubuntu; cgroups v2 |
| PostgreSQL | 17.11, official image pinned below |
| Rootless network helpers | Netavark and Aardvark DNS 1.14.0, isolated from Ubuntu packages |
| Kubernetes | K3s `v1.36.6+k3s1`, containerd 2.3.7 |
| Runner | GitLab Runner 19.5.3, Docker executor using the rootless Podman socket |
| Builder image | Podman 6.1.3, official image pinned below |
| Mac database client | Existing psql 18.3; no Node/nvm changes were made |

These are the versions used, not a recommendation to install the latest available version. Check availability, advisories and compatibility before repeating the exercise later. Stop if a package or release differs unexpectedly.

Define your own values, in your own shell; never assign to HOME or other system variables:

```sh
# Mac — examples contain placeholders, not working account details.
COURTSIDE_VPS_IP='<VPS_IP>'
COURTSIDE_SSH_USER='courtside'
COURTSIDE_PRIVATE_DIR="$HOME/.config/courtside"
COURTSIDE_SERVER_KEY="$HOME/.ssh/id_ed25519_courtside"
COURTSIDE_GIT_KEY="$HOME/.ssh/id_ed25519_courtside_gitlab"
COURTSIDE_GITLAB_SSH='git@gitlab.com:<YOUR_NAMESPACE>/courtside.git'
COURTSIDE_GITHUB_SSH='git@github.com:<YOUR_NAMESPACE>/<YOUR_REPOSITORY>.git'
mkdir -p "$COURTSIDE_PRIVATE_DIR"
chmod 700 "$COURTSIDE_PRIVATE_DIR"
```

You need an IDE, Git, OpenSSH, kubectl compatible with the server, a PostgreSQL client, project administration rights and access to the server console. Inventory existing tools before installing replacements. Preserve an existing Node/nvm setup. Check subnet overlap before choosing the reference networks: Podman DB `10.89.0.0/24`, K3s pod/service CIDRs and any VPN ranges must not conflict.

## 3. Establish SSH and ordinary sudo

1. Purchase/provision the server yourself and record its cancellation conditions. Keep provider-console recovery access available.
2. Create a dedicated Ed25519 SSH key on the Mac. Use a passphrase and your SSH agent. Upload only the `.pub` file through the provider's authorized setup route; never send a private key or password to a chat.
3. Obtain the server host-key fingerprint through the trusted console/provider channel. Compare it during the first SSH connection. Do not use `StrictHostKeyChecking=no` or accept an unexplained changed key.
4. On the server, create `courtside`, install its authorized public key and set `.ssh` to700 and `authorized_keys` to600, owned by that user. Add it to the sudo group. Set its password interactively and verify ordinary password-based sudo in a second session.
5. Keep the original working administration session open while testing. Do not remove recovery access before testing the replacement. The reference retained root access by key; it did not use `NOPASSWD: ALL`.
6. After checking the effective configuration and running `sshd -t`, disable SSH password and keyboard-interactive authentication. Reference settings: `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin prohibit-password`. Reload and test a fresh connection.

UFW must initially permit SSH before it is enabled. Keep public PostgreSQL, Kubernetes API and kubelet ports closed. Do not expose VXLAN to the Internet. Confirm both IPv4 and IPv6 behavior.

**Provider-specific exception:** the reference cloud-init bootcmd attempted an early sshd reload and failed. Only that module was excluded after inspecting the provider configuration; the remaining cloud-init modules and networking were preserved. Do not copy this workaround unless the same failure is verified. Do not disable cloud-init globally or clear its state as a routine setup step.

Acceptance: a fresh key-based login works, ordinary sudo works, DNS works, `systemctl --failed` is empty, and `cloud-init status --long` reports completion. Perform a controlled reboot only after saving work and confirming console recovery.

## 4. PostgreSQL outside Kubernetes

Install Podman from the authorized Ubuntu repository after inspecting package candidates. The reference kept Podman4.9.3 and used rootful system Quadlet units for PostgreSQL. Do not run `podman system reset` or volume-pruning commands.

Use the pinned official image:

```text
docker.io/library/postgres@sha256:665dc7b37ac52a17e298511414c21d23ffa4e6a6ef6225b3e5d59c7afb35e994
```

The reproducible configuration must provide:

- `/etc/containers/systemd/courtside-postgres.container`, generating `courtside-postgres.service`, attached to `multi-user.target`, with restart behavior and limits of2GiB/2CPU.
- Named volume `courtside-postgres-data` mounted at `/var/lib/postgresql/data`; preserve it across container recreation.
- Host publication **only** `127.0.0.1:5432:5432`. PostgreSQL itself runs as the image's postgres user.
- Private credentials under `/etc/courtside/postgres/`, directory700 and files600. The administrative password is a separate read-only file, not plaintext in a systemd unit or repository.
- Three databases and three distinct owner roles: `courtside_dev`, `courtside_test`, `courtside_demo`; different generated passwords, no superuser/CREATEDB/CREATEROLE, connection limit20. Each role owns only its database and may create its own schema for migrations.
- PUBLIC access revoked for those databases and for postgres/template1. Do not accidentally remove the administrator's recovery access.

Use Appendix A for the new-instance credential/role bootstrap, adapted from the procedure executed on the reference server. Review it before running; its generalized form has not been independently replayed. Use the sanitized Quadlets now supplied in `infra/admin/`. Never reinitialize an existing populated volume.

After loading the reviewed units:

```sh
# VPS
sudo systemctl daemon-reload
sudo systemctl start courtside-postgres.service
sudo systemctl is-active courtside-postgres.service
sudo podman ps
```

Quadlet generates services; its `[Install]` configuration establishes boot dependencies. Do not assume `systemctl enable` behaves like a hand-written service. See the [Podman4.9.3 Quadlet manual](https://docs.podman.io/en/v4.9.3/markdown/podman-systemd.unit.5.html).

## 5. Private Mac database access

Create a private SSH configuration in `$COURTSIDE_PRIVATE_DIR/ssh.conf`. Replace every placeholder; use the verified host key:

```sshconfig
Host courtside-db
  HostName <VPS_IP>
  User courtside
  IdentityFile <ABSOLUTE_PATH_TO_DEDICATED_KEY>
  IdentitiesOnly yes
  StrictHostKeyChecking yes
  ExitOnForwardFailure yes
  ServerAliveInterval 30
  ServerAliveCountMax 3
  ControlMaster auto
  ControlPersist 10m
  ControlPath <PRIVATE_DIRECTORY>/ssh-db-control
  LocalForward 127.0.0.1:15432 127.0.0.1:5432
```

```sh
# Mac
chmod 600 "$COURTSIDE_PRIVATE_DIR/ssh.conf"
ssh -F "$COURTSIDE_PRIVATE_DIR/ssh.conf" -fNT courtside-db
ssh -F "$COURTSIDE_PRIVATE_DIR/ssh.conf" -O check courtside-db
```

Prepare private `dev.env`, `test.env` and `demo.env` with PGHOST=127.0.0.1, PGPORT=15432 and the corresponding database/user/password plus DATABASE_URL. Use correctly URL-encoded passwords in URLs. Files must be600, directory700. Never commit them, print their contents or enable `set -x` while loading them. Connect from your own terminal using the existing psql client.

Acceptance: each role reads/writes its own database; all six cross-database combinations fail; wrong passwords fail; port5432 is inaccessible from outside the VPS. Create temporary test tables, restart the PostgreSQL container, verify the data remains and remove only those test tables. Record both container IDs to distinguish recreation from an unchanged container.

## 6. Back up and test restoration before network changes

1. Make consistent dumps of all three databases and a separate roles/global configuration dump, plus the Quadlet configuration. Store them under a root-private directory such as `/var/backups/courtside/pre-k3s`.
2. Transfer an encrypted-in-transit copy to the private Mac backup directory. Role dumps may contain password hashes: treat all backup files as secrets.
3. Restore each database dump into a separate temporary database using an administrator connection. Check tables/data and ownership expectations. Delete only the temporary restore databases after checking their names.
4. Verify backup file permissions and retain the originals before changing networking.

A volume is not an independent backup. The reference restored all three dumps successfully and removed the temporary databases. Exact dump/restore commands depend on the reviewed credential-bootstrap files; never place administrative passwords on a command line.

## 7. Private database network and K3s

Create the reviewed `courtside-db.network` Quadlet under `/etc/containers/systemd/`, network name `courtside-db`, subnet10.89.0.0/24. Link PostgreSQL through its `.network` reference and assign10.89.0.2. This creates the service dependency. Keep the loopback host-port publication unchanged. Confirm the network gateway is10.89.0.1 and that the range does not overlap existing routes. Reload/start the units and repeat the three Mac database checks before continuing.

Install K3s using the official installer, pinned to `v1.36.6+k3s1`, after downloading and reviewing the installer. Reference configuration: server, no Traefik or ServiceLB, Secrets encryption enabled, API certificate SAN covering the private bridge address10.89.0.1 used by CI. Do not use `insecure-skip-tls-verify`. Keep local-path, CoreDNS and metrics-server. The reviewed `infra/admin/k3s-config.yaml` now supplies these settings; the [official configuration reference](https://docs.k3s.io/installation/configuration) explains persistence and version selection.

UFW must permit required internal cluster and DB traffic without opening public6443/10250/5432 or UDP8472. Match actual pod/service CIDRs, interfaces and bridge routes; do not substitute a broad public allow rule when a pod cannot connect. Verify actual connectivity, not just the printed firewall configuration.

```sh
# VPS
sudo systemctl is-active k3s
sudo k3s kubectl get nodes
sudo k3s kubectl -n kube-system get pods
sudo k3s secrets-encrypt status
sudo systemctl --failed
```

Copy the administrative kubeconfig privately to the Mac, file600, and change its server endpoint for the SSH tunnel. Never publish the admin kubeconfig or supply it to CI. Add a separate SSH Host `courtside-k8s` with its own ControlPath and `LocalForward 127.0.0.1:16443 127.0.0.1:6443`; otherwise reuse the database Host's identity and connection settings. Confirm certificate validation for the tunnel endpoint and adjust only the reviewed TLS server-name setting if needed.

```sh
# Mac
ssh -F "$COURTSIDE_PRIVATE_DIR/ssh.conf" -fNT courtside-k8s
kubectl --kubeconfig "$COURTSIDE_PRIVATE_DIR/kubeconfig-admin" get nodes
```

## 8. Apply the namespace security baseline

Use the reviewed public `infra/admin/kubernetes-base.json` from the workshop repository. This file is supplied in `infra/admin/`; verify it exists at your reviewed commit before proceeding. It must contain no Secret values, tokens or administrative kubeconfig. Review its contents and apply it with the administrator identity. Do not give the CI identity permission to maintain its own security boundary.

Required baseline:

| Resource | Reference configuration |
| --- | --- |
| Namespaces | courtside-demo and courtside-ci |
| PSA | Demo enforce/audit/warn restricted, versionv1.36 |
| Network | Default-deny ingress/egress; DNS to actual CoreDNS selectors; DB5432 only for `courtside.io/database-access=true` |
| External DB service | `postgres` and internal endpoint10.89.0.2:5432; no public service |
| Quota | requestsCPU2/memory2Gi; limitsCPU3/memory3Gi; pods8/services6/PVC2/storage1Gi; zero NodePort/LoadBalancer |
| Deployment identity | SA in courtside-ci, RoleBinding into courtside-demo |
| Role | Deployment/configmap CRUD; replicaSet/pod/event/log observation; no direct secrets/RBAC/exec/nodes/other namespaces |

Create the demo DB Secret separately from private files. Never add it to the public baseline. A deployer able to create workloads can mount demo Secrets indirectly even without `get secrets`; it is trusted for demo data. Pod labels are not a security boundary against that deployer. Keep dev/test/admin credentials out of the demo namespace.

Verify with the actual deployment token, not only an administrator: authorized deployment dry-run succeeds; privileged Pod is rejected; secrets/RBAC/exec/node/other-namespace requests fail. Test DB by Service and IP with an authorized pod and a pod lacking the label; allow only the former. Confirm DNS works, Internet egress is blocked, and demo credentials cannot connect to dev/test. Test PVC persistence across two disposable pods, then remove the test objects/PVC. A passing server dry-run does not create or run the workload.

## 9. Rootless Podman for the runner

Install GitLab Runner19.5.3 from the [official GitLab package repository](https://docs.gitlab.com/runner/install/linux-repository/); inspect the repository script before running it. Check candidate versions and install the corresponding helper-images package version when required. Do not silently fall back to a newer release.

Use a dedicated `gitlab-runner` account without sudo. Do not assume UID999 on another server:

```sh
# VPS
id gitlab-runner
stat -fc %T /sys/fs/cgroup
sudo loginctl enable-linger gitlab-runner
```

Check `/etc/subuid` and `/etc/subgid`: allocate a sufficient **unused** subordinate range to this account, without overlapping another account. Do not copy a fixed numeric range blindly. Preserve its home directory ownership. To run user services from an administrator session, compute the UID dynamically and pass the matching runtime directory and D-Bus socket:

```sh
# VPS
COURTSIDE_RUNNER_UID=$(id -u gitlab-runner)
sudo -u gitlab-runner env \
  XDG_RUNTIME_DIR="/run/user/$COURTSIDE_RUNNER_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$COURTSIDE_RUNNER_UID/bus" \
  systemctl --user enable --now podman.socket
```

Confirm the user manager exists before this command. Socket path: `/run/user/<ACTUAL_UID>/podman/podman.sock`. Check rootless Podman info, cgroupsv2, storage ownership and socket availability after logout/reboot. Never grant sudo to resolve a rootless setup error.

### Isolated networking helpers

Ubuntu's reference packages were aardvark-dns1.4.0-5 and netavark1.4.0-4. They remained unchanged for PostgreSQL. GitLab documents intermittent DNS failures with Aardvark≤1.10.0.

Download **official** Netavark/Aardvark1.14.0 amd64 release assets and their published SHA256 files from the upstream release API/pages. Verify checksums before decompressing/executing. Keep compressed-asset hashes distinct from decompressed-binary hashes. Install only the verified binaries into `/opt/courtside/podman-helpers/1.14.0`, owned by root and not writable by jobs. Record the release URLs and hashes in your installation notes.

Configure `[engine].helper_binaries_dir` only in the runner user's `~/.config/containers/containers.conf`, with the isolated directory first and existing helper search directories retained. Do not change `/etc/containers/containers.conf`, replace `/usr/lib/podman` helpers, or install overrides in a globally searched location. Confirm the runner uses1.14.0 while rootful Podman still uses the Ubuntu helpers.

Create disposable rootless networks/containers, check internal aliases and external DNS repeatedly, then clean up those resources. Forty DNS queries passed in the reference environment. See [GitLab's Podman executor requirements](https://docs.gitlab.com/runner/executors/docker/#use-podman-to-run-docker-commands).

## 10. Prove a build before registering success

Use the official builder image pinned by digest:

```text
quay.io/podman/stable@sha256:7b5e95b4934dde2d367b949c47a68f3705ca21e077fb27ed9a5b4fb8b01b9008
```

The checked image contains Podman6.1.3. Run it through the dedicated rootless account with internal user `podman`, VFS storage and `podman build --isolation=chroot`. Do not use privileged mode, host network, a mounted host Podman socket, administrative credentials or shared PostgreSQL storage. Use disposable build storage and the reviewed `infra/smoke/Containerfile`.

The manual reference build passed actual RUN steps using network access to GitLab, adduser and chown. A COPY-only build is insufficient. Repeat within the real runner: user namespaces, storage, workspace ownership and security profiles may differ. Do not disable AppArmor/seccomp globally or add SYS_ADMIN automatically when a nested build fails; diagnose the exact denial first.

## 11. Create the project and register a constrained runner

1. Create an empty public GitLab project for synthetic workshop code. Keep internal course material, private files and credentials out of it. Use the actual project URL, not a project-creation page.
2. Configure origin to your GitLab SSH URL; optionally add a GitHub remote. A configured remote does not imply a remote repository exists or has received a push.
3. Generate a separate passphrase-protected Ed25519 Git key. Add its public key as a **write-enabled project deploy key** for the authorized publishing machine; keep the private key on that machine. Verify GitLab's SSH host fingerprint through its official documentation. Check protected-main permissions allow the intended key/user to push before attempting it.
4. Create a project runner using GitLab's current authentication-token workflow. Enter/store the token privately; never paste it into chat, tracked files or examples. Set the runner to protected, project-specific, explicit tag `courtside` and no untagged jobs. Protected main is the initial execution route; arbitrary forks/unprotected MRs are not supported by this trust model.
5. Register using the Docker executor and host `unix:///run/user/<UID>/podman/podman.sock`. This is the runner's control connection, **not** a socket mounted into jobs. Docker Engine is not required.
6. Configure global concurrency1, runner limit1, CPU2, memory3g, cache disabled, privileged=false, `FF_NETWORK_PER_BUILD` and `FF_DISABLE_UMASK_FOR_DOCKER_EXECUTOR`. Do not mount host directories or credential files into all jobs. Use tag `courtside` consistently in config and pipeline; the reference runner name is `courtside-vps`.
7. Run runner verification and check it appears online in the project. This proves registration, not the entire pipeline.

## 12. Scope deployment and database credentials to jobs

Create a deployment token for the reviewed SA with a requested duration of120h. Kubernetes may cap the requested duration: read its actual expiry locally without printing the JWT. Keep the token in a600 file. Generate a **compact JSON kubeconfig** containing the deployment identity, CA and private CI-reachable API endpoint, not the admin identity. Compact JSON avoids whitespace restrictions when storing a masked File variable. Do not bypass TLS verification.

| GitLab variable | Type | Environment scope | Settings |
| --- | --- | --- | --- |
| KUBE_CONFIG | File | demo | Protected, masked/hidden, expansion disabled |
| TEST_DATABASE_URL | Variable | test | Protected, masked/hidden, expansion disabled |

The pipeline jobs must declare exactly the corresponding GitLab environment names; otherwise scoped variables will not be available. The File variable's value at runtime is a temporary **path**, not the kubeconfig text. TEST_DATABASE_URL uses only the test role and a job-reachable DB endpoint, not the Mac loopback tunnel. Check its routing from the runner container.

Masking is not protection against malicious pipeline code. Only reviewed code on trusted protected refs should receive these credentials. Do not put credentials in artifacts, caches, debug traces or job logs. Record expiry and renewal owner. Rotate/revoke credentials after the course; never reuse the instructor's token.

## 13. Run and diagnose the infrastructure pipeline

Review `.gitlab-ci.yml`, `infra/smoke/Containerfile` and `infra/smoke/deployment.yaml` at the intended commit. The three jobs check test DB, build/publish an image, and deploy/check/clean up an infrastructure workload. No CourtSide application is implemented by this smoke pipeline.

Use GitLab CI Lint plus server-side Kubernetes dry-run. After pushing to protected main, inspect each job's trace and status. Treat success as complete only when database access, image push, deployment rollout and cleanup have all passed in GitLab. Check that the image tag/digest corresponds to the commit.

At this checkpoint pipeline3052878956 was blocked after commitd4c0397. GitLab reported that the user was not verified; `yamlErrors=false` and no jobs existed. The owner must complete [GitLab identity verification](https://gitlab.com/-/identity_verification) personally, then rerun the pipeline. This is not evidence of an infrastructure failure or a failure caused by the minimum role allowed to supply pipeline variables. After the owner completed verification, jobs ran. The psql invocation and non-root temporary-file permissions were then corrected as documented in Appendix E; pipeline3052919118 passed all three jobs.

| Symptom | Check first |
| --- | --- |
| Pipeline fails with no jobs and user-not-verified reason | Complete GitLab identity verification as the account owner; do not change runner privileges |
| Job stuck | Runner online, matching tags, protected ref and project assignment |
| Variable absent | Protected branch and exact environment scope; File-variable path semantics |
| Prepare/build permission error | Effective job user, workspace ownership, UID maps and umask feature flag |
| DNS intermittent | Actual helper versions for the rootless service, per-build network and alias tests |
| Nested build EPERM | User namespaces, AppArmor/seccomp denial, subordinate IDs; VFS/chroot do not bypass all restrictions |
| DB timeout | Container route, policy label, DB network endpoint and UFW; localhost is the job container |
| DB authentication error | Correct test/demo role, database, password encoding and scope; never print the URL |
| Kubernetes unauthorized | Actual token expiry, kubeconfig identity/context and RBAC |
| Certificate error | CA, endpoint and TLS SAN/name; do not disable verification |
| Deployment forbidden | Restricted securityContext, requests/limits, quotas, service account and image user |
| Public port reachable | Stop dependent work and correct network exposure before continuing |

## 14. Reboot and recovery acceptance

Save work, verify backups/console access, then reboot deliberately. On the VPS confirm k3s, courtside-postgres, gitlab-runner and the runner user manager are active, the socket exists, no services failed and the node/components are Ready/Running. Reopen both Mac tunnels and verify kubectl plus authenticated access to all three databases. These checks passed on the instructor server; they must be repeated for a new environment.

Do not conflate a tunnel process existing with a usable destination. If a stale control socket remains, inspect the owning process before removing only that stale socket. Never delete arbitrary files under the private directory as a troubleshooting shortcut.

## 15. Cleanup and billing

**Destructive operations require an explicit decision and checked backups.** Routine smoke cleanup removes only the named disposable workload/test PVC. It must not delete the namespace, PostgreSQL volume, real schemas or backup directory.

At course end:

1. Stop accepting jobs, remove/unregister the project runner and revoke its authentication token.
2. Revoke/delete CI variables and deployment credentials; remove the write deploy key if no longer needed. Remove temporary workload resources and close SSH tunnels with `ssh -F <SSH_CONFIG> -O exit <HOST_ALIAS>`.
3. Export required backups and verify restoration before deleting databases or the VPS. Decide explicitly whether public repositories should remain.
4. If decommissioning the server, use the provider's cancellation workflow and confirm the effective cancellation date, renewal status and any separately billed storage/IP/backup resources. **Stopping containers, shutting down the VPS or uninstalling K3s does not establish that billing stopped.** Retain the cancellation confirmation.
5. Delete private local credentials/backups only according to the agreed retention plan. Do not remove unrelated SSH keys or files.

## Review gates before participant distribution

- Review supplied Quadlets/K3s configuration and Appendix A against the target server; generalized bootstrap examples have not been replayed.
- Verify the published commit includes `infra/admin/kubernetes-base.json` and reviewed host configuration with no Secrets, private identity or tokens.
- Reference integration passed pipeline3052919118. Re-run all three jobs on each team’s own environment before marking that environment ready.
- Replace instructor-specific pipeline/commit evidence with a stable release reference for participants, confirm runner tags and exercise times, and rehearse from a clean authorized environment.


## Appendix A. New-instance PostgreSQL bootstrap

Run only on a **new, dedicated** VPS after checking routes and installing Podman. Do not run against an existing database volume. The following adapts the reference procedure: dynamically discover the image's postgres UID/GID, create private files once, then bootstrap fixed role names. Any partial failure requires inspection before another attempt, not deletion of existing state. No credentials appear in the script or normal output.

First pull the pinned image and inspect its metadata. Put the two reviewed Quadlets from `infra/admin/` into `/etc/containers/systemd/` only after reviewing the subnet/IP. In an administrator-controlled working directory on the VPS:

```sh
sudo podman pull docker.io/library/postgres@sha256:665dc7b37ac52a17e298511414c21d23ffa4e6a6ef6225b3e5d59c7afb35e994
sudo install -d -m 755 /etc/containers/systemd
sudo install -m 644 infra/admin/courtside-db.network /etc/containers/systemd/
sudo install -m 644 infra/admin/courtside-postgres.container /etc/containers/systemd/
```

Generate credentials without printing them:

```sh
sudo python3 - <<'PYBOOT'
from pathlib import Path
import os, secrets, subprocess
image = 'docker.io/library/postgres@sha256:665dc7b37ac52a17e298511414c21d23ffa4e6a6ef6225b3e5d59c7afb35e994'
config = Path('/etc/courtside/postgres')
assert not config.exists(), 'Existing credentials: stop'
assert subprocess.run(['podman','volume','exists','courtside-postgres-data']).returncode != 0, 'Existing volume: stop'
def image_id(flag):
    return int(subprocess.check_output(['podman','run','--rm','--network=none','--entrypoint=id',image,flag,'postgres'], text=True).strip())
uid, gid = image_id('-u'), image_id('-g')
config.parent.mkdir(mode=0o700, exist_ok=True)
config.mkdir(mode=0o700)
os.umask(0o077)
admin = config/'admin-password'
admin.write_text(secrets.token_hex(32)+'\n')
admin.chmod(0o600)
os.chown(admin,uid,gid)
for stage in ('dev','test','demo'):
    role = 'courtside_'+stage
    password = secrets.token_hex(32)
    target = config/(stage+'.env')
    target.write_text(f'PGHOST=127.0.0.1\nPGPORT=5432\nPGDATABASE={role}\nPGUSER={role}\nPGPASSWORD={password}\nDATABASE_URL=postgresql://{role}:{password}@127.0.0.1:5432/{role}\n')
    target.chmod(0o600)
subprocess.run(['podman','volume','create','courtside-postgres-data'],check=True)
print('Private credentials and empty named volume prepared.')
PYBOOT
sudo systemctl daemon-reload
sudo systemctl start courtside-postgres.service
for attempt in $(seq 1 30); do
  sudo podman exec courtside-postgres pg_isready -U postgres && break
  sleep 1
done
sudo podman exec courtside-postgres pg_isready -U postgres
```

Wait for readiness before bootstrapping roles; the hex-only generated passwords make the following constrained SQL construction safe. Do not adapt it to arbitrary unescaped user input:

```sh
sudo python3 - <<'PYROLES'
from pathlib import Path
import re, subprocess
base=['podman','exec','-i','courtside-postgres','psql','-X','-v','ON_ERROR_STOP=1','-U','postgres','-d','postgres']
existing=subprocess.check_output(base+['-Atc',"SELECT rolname FROM pg_roles WHERE rolname LIKE 'courtside_%'"],text=True).strip()
assert not existing, 'Existing roles: stop; never regenerate their passwords'
sql=['REVOKE CONNECT ON DATABASE postgres FROM PUBLIC;','REVOKE CONNECT ON DATABASE template1 FROM PUBLIC;']
for stage in ('dev','test','demo'):
    role='courtside_'+stage
    values=dict(line.split('=',1) for line in Path('/etc/courtside/postgres/'+stage+'.env').read_text().splitlines())
    password=values['PGPASSWORD']
    assert values['PGUSER']==role and re.fullmatch('[0-9a-f]{64}',password)
    sql += [f"CREATE ROLE {role} LOGIN PASSWORD '{password}' NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION CONNECTION LIMIT 20;",
            f'CREATE DATABASE {role} OWNER {role};',
            f'REVOKE ALL ON DATABASE {role} FROM PUBLIC;',
            f'GRANT CONNECT, TEMPORARY ON DATABASE {role} TO {role};']
result=subprocess.run(base,input='\n'.join(sql),text=True,capture_output=True)
if result.returncode:
    raise RuntimeError('Bootstrap failed; inspect privately. Output withheld to protect credentials.')
print('Three databases and isolated owner roles created.')
PYROLES
```

Do not paste error output into chat: SQL errors can echo a password-bearing statement. Inspect securely and rotate affected credentials if exposure occurs. Verify ownership and all positive/negative database tests from section5. Transfer environment files only through authenticated SSH to private local files; change local PGHOST/PGPORT to the Mac tunnel and preserve corresponding URL encoding. CI's test URL must use the private DB address instead.

## Appendix B. Reviewed K3s configuration and internal firewall

After subnet review and DB backup/restore checks, copy `infra/admin/k3s-config.yaml` into `/etc/rancher/k3s/config.yaml` mode600. Its reference TLS SANs are127.0.0.1 and10.89.0.1; pod CIDR10.42.0.0/16 and service CIDR10.43.0.0/16. Adapt all related configuration together if these ranges conflict.

Download the official installer to a local file, inspect it, then run the pinned version:

```sh
curl -fsSL https://get.k3s.io -o /tmp/courtside-install-k3s.sh
# Review /tmp/courtside-install-k3s.sh before execution.
sudo env INSTALL_K3S_VERSION=v1.36.6+k3s1 sh /tmp/courtside-install-k3s.sh
```

Reference UFW additions, only for the matching interfaces/CIDRs:

```sh
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow in on cni0 from 10.42.0.0/16 to any port 6443 proto tcp
sudo ufw allow in on cni0 from 10.42.0.0/16 to any port 10250 proto tcp
sudo ufw route allow in on cni0 from 10.42.0.0/16 to 10.89.0.2 port 5432 proto tcp
sudo ufw status verbose
```

These additions assume the already-reviewed default-deny incoming/forwarding baseline and allowed outbound traffic. They are not a complete replacement firewall for another server. Test both pod-originated DB connections and public-port rejection. Avoid exposing6443 simply to make kubectl work: the Mac uses the SSH tunnel, while CI uses the private bridge address with valid TLS.

## Appendix C. Apply the public baseline and prepare the scoped kubeconfig

Clone the public workshop repository into a new directory and inspect the intended commit. Do not run `git init` over an unrelated directory. From the checkout on the Mac, after opening the API tunnel:

```sh
kubectl --kubeconfig "$COURTSIDE_PRIVATE_DIR/kubeconfig-admin" apply \
  --dry-run=server -f infra/admin/kubernetes-base.json
kubectl --kubeconfig "$COURTSIDE_PRIVATE_DIR/kubeconfig-admin" apply \
  -f infra/admin/kubernetes-base.json
umask 077
kubectl --kubeconfig "$COURTSIDE_PRIVATE_DIR/kubeconfig-admin" \
  -n courtside-ci create token courtside-deployer --duration=120h \
  > "$COURTSIDE_PRIVATE_DIR/deployer-token"
```

The following creates a dedicated compact JSON configuration for CI without copying administrative credentials. Its API endpoint is private and reachable from the VPS runner; it is not the Mac tunnel endpoint. Run on the Mac, with private-directory defaults matching section2:

```sh
python3 - <<'PYCONFIG'
from pathlib import Path
import json, os, subprocess
private = Path.home()/'.config/courtside'
admin = private/'kubeconfig-admin'
raw = json.loads(subprocess.check_output(['kubectl','--kubeconfig',str(admin),'config','view','--raw','--flatten','-o','json']))
context = next(x['context'] for x in raw['contexts'] if x['name']==raw['current-context'])
cluster = next(x['cluster'] for x in raw['clusters'] if x['name']==context['cluster'])
ca = cluster['certificate-authority-data']
token = (private/'deployer-token').read_text().strip()
assert token and ca
output = {'apiVersion':'v1','kind':'Config',
 'clusters':[{'name':'courtside','cluster':{'server':'https://10.89.0.1:6443','certificate-authority-data':ca}}],
 'users':[{'name':'deployer','user':{'token':token}}],
 'contexts':[{'name':'demo','context':{'cluster':'courtside','user':'deployer','namespace':'courtside-demo'}}],
 'current-context':'demo'}
os.umask(0o077)
path = private/'ci-kubeconfig'
path.write_text(json.dumps(output,separators=(',',':')))
path.chmod(0o600)
print('Private scoped kubeconfig created; value not printed.')
PYCONFIG
```

Upload the private file contents into the protected File variable using GitLab's secure UI. Do not paste them into a chat or terminal command. If masking validation rejects a value, review its format rather than silently making the credential visible. Prepare the test connection URL similarly in a600 file using the test role, private DB address and port5432; do not reuse the development or administrative role.

For a deployment dry-run from the Mac, create a separate private test copy of the deployer kubeconfig with `server=https://127.0.0.1:16443`, preserving its deployer identity and CA. Do not overwrite the CI file's private-bridge endpoint. From the runner environment use the original CI file. Verify denied operations with that identity and actual server responses before issuing production-like changes.

The instructor's local sanitized evidence record `.build/courtside-infra/validation.json` records the checks reported above and records the successful final CI pipeline separately from the initial account-verification failure. It contains no credentials and is internal evidence, not a file participants must possess.

## Appendix D. Shell prompts and private runner-token entry

The Mac normally uses **zsh**; the VPS examples assume **Bash** unless a Python block is shown. Their `read` syntax differs. A prompt is not a place to insert a token into the command itself.

```sh
# Mac zsh: run this line literally, then paste the value at the prompt.
read -rs 'COURTSIDE_RUNNER_TOKEN?Runner authentication token: '
printf '\n'
```

```bash
# VPS Bash: run this line literally, then paste the value at the prompt.
read -rs -p 'Runner authentication token: ' COURTSIDE_RUNNER_TOKEN
printf '\n'
```

Input is intentionally invisible. Paste only the token when prompted, with no added colon or spaces, then press Enter. Never replace the quoted prompt text with the secret, and never type a secret as a standalone shell command. Do not enable shell tracing. Immediately write the captured value to the intended private600 file in a700 directory, then unset the variable. On the server, perform this in a root-controlled session so the file never becomes job-readable.

For registration, load the root-private file into the runner registration process through `CI_SERVER_TOKEN` rather than putting a token on its command line. Use the reviewed registration procedure with the Docker executor and rootless socket. Capture/redact registration responses before sharing diagnostics; do not assume tool output excludes tokens. Remove temporary token copies when the durable root-private runner configuration is verified.

Keep both dedicated SSH private keys mode600. If SSH reports an unprotected private key, correct the key's permissions and ownership; do not disable SSH checks or make the key readable by everyone.

## Appendix E. Command checklist for a new reference VPS

Use this appendix together with sections 3–12 and A–D. This is a sequence of
reviewable steps, not a script to paste wholesale. Run the VPS blocks from a
root administration shell (`sudo -i` after authenticating), unless marked Mac.
The private files contain credentials: keep shell tracing off. Do not repeat
initialization blocks on an existing environment.

### E1. Keys, checkout and host packages

```zsh
# Mac; choose a passphrase when prompted and load the key into your SSH agent.
ssh-keygen -t ed25519 -f "$COURTSIDE_SERVER_KEY" -C courtside-workshop
ssh-keygen -t ed25519 -f "$COURTSIDE_GIT_KEY" -C courtside-gitlab-workshop
chmod 600 "$COURTSIDE_SERVER_KEY" "$COURTSIDE_GIT_KEY"
# Copy only the public server key into the provider console / authorized_keys.
pbcopy < "$COURTSIDE_SERVER_KEY.pub"
# Verify the server fingerprint independently before accepting the first login.
ssh -i "$COURTSIDE_SERVER_KEY" root@"$COURTSIDE_VPS_IP"
```

```bash
# VPS root shell, new host only. Inspect apt candidates before installation.
apt-get update
apt-cache policy podman uidmap slirp4netns
apt-get install -y podman uidmap slirp4netns curl ca-certificates git ufw
adduser courtside
usermod -aG sudo courtside
install -d -m 700 -o courtside -g courtside /home/courtside/.ssh
# If root's authorized_keys contains ONLY the workshop key just installed:
install -m 600 -o courtside -g courtside /root/.ssh/authorized_keys /home/courtside/.ssh/authorized_keys
# Otherwise install only the intended public key, not every root key.
ufw allow 22/tcp
ufw default deny incoming
ufw default allow outgoing
ufw default deny routed
ufw enable
```

Test the separate courtside login and ordinary sudo before applying SSH settings
from section 3. Validate with `sshd -t` and `sshd -T`, reload `ssh`, then test a new
session. These commands do not replace fingerprint verification or provider
recovery access.

Clone the reference into a new directory on the VPS to obtain the reviewed
configuration files (or copy your reviewed fork there). Keep credentials outside it:

```bash
git clone https://gitlab.com/nassao/courtside.git /opt/courtside-reference
cd /opt/courtside-reference
git log -1 --oneline
```

Run Appendix A from this checkout. Its two Quadlets configure the final private
DB network from the start. Verify PostgreSQL readiness before proceeding.
Install the K3s configuration **before** running Appendix B's installer:

```bash
install -d -m 700 /etc/rancher/k3s
install -m 600 infra/admin/k3s-config.yaml /etc/rancher/k3s/config.yaml
```

### E2. Backups and restoration

```bash
# VPS root; stop the block on any error. Unique backup directory.
set -e
umask 077
COURTSIDE_BACKUP="/var/backups/courtside/$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "$COURTSIDE_BACKUP"
for stage in dev test demo; do
  podman exec courtside-postgres pg_dump -U postgres -Fc "courtside_$stage" > "$COURTSIDE_BACKUP/courtside_$stage.dump"
  podman exec -i courtside-postgres pg_restore --list < "$COURTSIDE_BACKUP/courtside_$stage.dump" >/dev/null
done
podman exec courtside-postgres pg_dumpall -U postgres --globals-only > "$COURTSIDE_BACKUP/globals.sql"
cp /etc/containers/systemd/courtside-postgres.container /etc/containers/systemd/courtside-db.network "$COURTSIDE_BACKUP/"
# Choose a new disposable name; createdb fails if it already exists.
for stage in dev test demo; do
  podman exec courtside-postgres createdb -U postgres courtside_restore_probe
  podman exec -i courtside-postgres pg_restore -U postgres --no-owner --no-privileges -d courtside_restore_probe < "$COURTSIDE_BACKUP/courtside_$stage.dump"
  podman exec courtside-postgres dropdb -U postgres courtside_restore_probe
done
```

Run with `set -e` in a reviewed script, or stop manually at the first failure.
This verifies readable/restorable dumps, not a complete disaster recovery drill
with role ownership restored. Copy the directory through authenticated SSH to a
private Mac directory and compare SHA256 hashes. Do not publish globals.sql.

### E3. Runner package and rootless account

```bash
curl -fsSL https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh -o /root/courtside-runner-repo.sh
# Inspect this official script before executing it.
bash /root/courtside-runner-repo.sh
apt-cache policy gitlab-runner gitlab-runner-helper-images
apt-get install -y gitlab-runner=19.5.3-1 gitlab-runner-helper-images=19.5.3-1
id gitlab-runner
cat /etc/subuid /etc/subgid
```

Select a free range of at least 65,536 subordinate IDs for both files. Example
`231072-296607` was unused in the reference host; it is **not automatically free**
on another host. After checking, set `COURTSIDE_SUBID_RANGE` to your selected range:

```bash
COURTSIDE_SUBID_RANGE='<CHECKED_START>-<CHECKED_END>'
usermod --add-subuids "$COURTSIDE_SUBID_RANGE" --add-subgids "$COURTSIDE_SUBID_RANGE" gitlab-runner
COURTSIDE_RUNNER_UID=$(id -u gitlab-runner)
COURTSIDE_RUNNER_HOME=$(getent passwd gitlab-runner | cut -d: -f6)
install -d -m 700 -o gitlab-runner -g gitlab-runner "$COURTSIDE_RUNNER_HOME/.config"
install -d -m 700 -o gitlab-runner -g gitlab-runner "$COURTSIDE_RUNNER_HOME/.config/containers"
loginctl enable-linger gitlab-runner
systemctl start "user@$COURTSIDE_RUNNER_UID.service"
```

The `.config` parent must belong to the runner too. A root-owned parent caused
`systemctl --user enable` to fail in the initial setup. Also change out of `/root`
before running a command as this user; the account cannot traverse that directory.

### E4. Verified per-user network helpers

```bash
install -d -m 755 /opt/courtside/podman-helpers/1.14.0
for helper in aardvark-dns netavark; do
  work=$(mktemp -d)
  curl -fsSL "https://github.com/containers/$helper/releases/download/v1.14.0/$helper.gz" -o "$work/$helper.gz"
  curl -fsSL "https://github.com/containers/$helper/releases/download/v1.14.0/sha256sum" -o "$work/sha256sum"
  (cd "$work" && sha256sum --check --ignore-missing sha256sum) || exit 1
  gzip -dc "$work/$helper.gz" > "/opt/courtside/podman-helpers/1.14.0/$helper"
  chmod 755 "/opt/courtside/podman-helpers/1.14.0/$helper"
  "/opt/courtside/podman-helpers/1.14.0/$helper" --version
  # Keep the checked downloads/checksums as installation evidence if desired.
done
cat > "$COURTSIDE_RUNNER_HOME/.config/containers/containers.conf" <<'CONF'
[engine]
helper_binaries_dir = ["/opt/courtside/podman-helpers/1.14.0", "/usr/local/libexec/podman", "/usr/local/lib/podman", "/usr/libexec/podman", "/usr/lib/podman"]
CONF
chown gitlab-runner:gitlab-runner "$COURTSIDE_RUNNER_HOME/.config/containers/containers.conf"
cd "$COURTSIDE_RUNNER_HOME"
runuser -u gitlab-runner -- env HOME="$COURTSIDE_RUNNER_HOME" XDG_RUNTIME_DIR="/run/user/$COURTSIDE_RUNNER_UID" DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$COURTSIDE_RUNNER_UID/bus" systemctl --user enable --now podman.socket
runuser -u gitlab-runner -- env HOME="$COURTSIDE_RUNNER_HOME" XDG_RUNTIME_DIR="/run/user/$COURTSIDE_RUNNER_UID" DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$COURTSIDE_RUNNER_UID/bus" podman info
```

Check `rootless: true`, cgroups v2 and both helper paths/versions. Rootful
`podman info` must still report the original Ubuntu helper paths. Do not infer
nested-build support from these checks: section 10 and the pipeline test it.

### E5. Register without secrets in command history

Create the protected, tagged project runner in GitLab first. In the VPS **root
Bash** session, capture its authentication token interactively:

```bash
umask 077
install -d -m 700 /root/courtside-private
read -rs -p 'Paste runner token, then Enter: ' COURTSIDE_RUNNER_TOKEN
printf '\n'
printf '%s' "$COURTSIDE_RUNNER_TOKEN" > /root/courtside-private/runner.token
unset COURTSIDE_RUNNER_TOKEN
python3 - <<'PYREGISTER'
from pathlib import Path
import os, pwd, subprocess
p=Path('/root/courtside-private/runner.token')
env=os.environ.copy(); env['CI_SERVER_TOKEN']=p.read_text().strip()
assert env['CI_SERVER_TOKEN'].startswith('glrt-')
uid=pwd.getpwnam('gitlab-runner').pw_uid
args=['gitlab-runner','register','--non-interactive','--url','https://gitlab.com',
 '--name','courtside-vps','--executor','docker',
 '--docker-host',f'unix:///run/user/{uid}/podman/podman.sock',
 '--docker-image','docker.io/library/alpine:3.22','--docker-memory','3g',
 '--docker-cpus','2','--limit','1','--env','FF_NETWORK_PER_BUILD=1',
 '--env','FF_DISABLE_UMASK_FOR_DOCKER_EXECUTOR=true','--debug-trace-disabled']
r=subprocess.run(args,env=env,text=True,capture_output=True)
if r.returncode: raise RuntimeError('Registration failed; inspect privately, do not publish raw output')
p.unlink()
print('Runner registered; temporary token file removed.')
PYREGISTER
```

Review `/etc/gitlab-runner/config.toml` privately. Preserve its generated token;
set `concurrent = 1` globally, `limit = 1` for this runner, and under
`[runners.docker]` set `privileged = false`, `disable_cache = true`, `volumes = []`,
`memory = "3g"`, `cpus = "2"`. Both feature flags above belong in the runner's
`environment` list. No host socket or credential path belongs in `volumes`.

```bash
install -d /etc/systemd/system/gitlab-runner.service.d
printf '[Unit]\nWants=user@%s.service\nAfter=user@%s.service network-online.target\n' "$COURTSIDE_RUNNER_UID" "$COURTSIDE_RUNNER_UID" > /etc/systemd/system/gitlab-runner.service.d/courtside.conf
systemctl daemon-reload
systemctl restart gitlab-runner
# Verification output can contain a short runner identifier; never share config.toml.
gitlab-runner verify
```

### E6. Demo Secret and workstation credential files

After applying the public Kubernetes baseline, create the demo Secret from the
private server files. This outputs only its resource name, not its contents:

```bash
python3 - <<'PYSECRET'
from pathlib import Path
import json, subprocess
env=dict(line.split('=',1) for line in Path('/etc/courtside/postgres/demo.env').read_text().splitlines())
env.update(PGHOST='postgres.courtside-demo.svc.cluster.local',PGPORT='5432')
env.pop('DATABASE_URL',None)
secret={'apiVersion':'v1','kind':'Secret','metadata':{'name':'courtside-demo-db','namespace':'courtside-demo'},'type':'Opaque','stringData':env}
subprocess.run(['kubectl','apply','-f','-'],input=json.dumps(secret),text=True,check=True,stdout=subprocess.DEVNULL)
print('Demo Secret configured.')
PYSECRET
```

Transfer private files via your authenticated SSH session, never through the
repository. For example, from Mac zsh with the section 2 variables set:

```zsh
umask 077
for stage in dev test demo; do
  ssh -i "$COURTSIDE_SERVER_KEY" root@"$COURTSIDE_VPS_IP" "cat /etc/courtside/postgres/$stage.env" > "$COURTSIDE_PRIVATE_DIR/$stage.env"
done
ssh -i "$COURTSIDE_SERVER_KEY" root@"$COURTSIDE_VPS_IP" 'kubectl config view --raw -o json' > "$COURTSIDE_PRIVATE_DIR/kubeconfig-admin"
```

If root SSH is disabled in your approved environment, use its approved
administrative transfer method instead. These commands write secrets to files;
do not run them without the redirection. On the Mac, adjust endpoints privately:

```sh
python3 - <<'PYLOCAL'
from pathlib import Path
import json
p=Path.home()/'.config/courtside'
for stage in ('dev','test','demo'):
    f=p/(stage+'.env')
    values=dict(line.split('=',1) for line in f.read_text().splitlines())
    if stage=='test':
        out=p/'ci-test-database-url'
        out.write_text(values['DATABASE_URL'].replace('127.0.0.1','10.89.0.2'));out.chmod(0o600)
    values['PGPORT']='15432'
    values['DATABASE_URL']=values['DATABASE_URL'].replace(':5432/',':15432/')
    f.write_text(''.join(f'{k}={v}\n' for k,v in values.items()));f.chmod(0o600)
f=p/'kubeconfig-admin';c=json.loads(f.read_text())
c['clusters'][0]['cluster']['server']='https://127.0.0.1:16443'
f.write_text(json.dumps(c));f.chmod(0o600)
PYLOCAL
```

Run Appendix C to create the scoped deployment kubeconfig. In GitLab's
**Settings → CI/CD → Variables**, use the exact types/scopes in section 12.
Copy each value only when ready to paste it into the correct Value field:

```zsh
pbcopy < "$COURTSIDE_PRIVATE_DIR/ci-kubeconfig"
# Paste into KUBE_CONFIG: File, demo, Protected, Masked and hidden, expansion off.
pbcopy < "$COURTSIDE_PRIVATE_DIR/ci-test-database-url"
# Paste into TEST_DATABASE_URL: Variable, test, Protected, Masked and hidden, expansion off.
```

For an interactive Mac database check, load only your trusted generated env file:

```zsh
set -a
source "$COURTSIDE_PRIVATE_DIR/test.env"
set +a
psql -X -v ON_ERROR_STOP=1 -c 'SELECT current_database(), current_user;'
unset PGPASSWORD DATABASE_URL
```

The CI job passes its URL explicitly as `psql --dbname="$TEST_DATABASE_URL"`.
Do not assume exporting a URL as PGDATABASE makes this client invocation expand
it as connection parameters; the initial smoke job attempted a local socket with
that approach. Keep CI debug tracing disabled and the URL masked.

### E7. Non-root builder temporary files

The build job overrides `FF_DISABLE_UMASK_FOR_DOCKER_EXECUTOR` to `"false"`, restoring GitLab Runner's standard temporary-workspace umask behavior for that job only. With the global flag enabled and `image.docker.user: "1000:1000"`, its `CI_SERVER_TLS_CA_FILE` was not writable at script initialization. This adjustment concerns disposable files inside the job volume; it does not enable privileged containers, host mounts or host sockets. See the [official feature-flag description](https://docs.gitlab.com/runner/configuration/feature-flags/). The adjusted build passed in pipeline3052919118. Revalidate before applying this setting to another runner version.
