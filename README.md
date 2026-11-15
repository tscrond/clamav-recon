# ClamAV on a Recon-controlled Docker host

This example mirrors the live Compose project named `clamav` on the
`edge-server-1` testbed. It keeps the same services, container names, ports,
health checks, resource limits, read-only filesystem, user IDs, named volumes,
external `proxy` network, and Traefik labels.

The live stack kept the Traefik Basic Auth entry inline in `compose.yaml`. Here
that value comes from a SOPS-encrypted dotenv file, so the repository contains
ciphertext while the host keeps the private age key.

```
recon.yaml                     Recon desired state
compose/clamav/compose.yaml    ClamAV Compose stack
secrets/clamav.sops.env        encrypted Basic Auth value
secrets/clamav.env.example     plaintext template
.sops.yaml                     permitted encryption recipients
.gitignore                     keeps unencrypted secret copies out of Git
```

The images are pinned to the digests observed on `edge-server-1`:

```
ajilaag/clamav-rest@sha256:bdf9af38492ac5232068050bc1866feb7a3b96942dfc7880e97b441d7f627aad
sergeymakinen/clamav_exporter:v2.1.12@sha256:66bc4f313d4c1f3baa73a75cb57a5c55c1095fb8687c6eb6cd66954eed084911
```

## 1. Put the example in the infrastructure repository

These files belong at the root of the repository Recon deploys from, not in
Recon's own source tree. For a new infrastructure repository, copy the
directory contents, including the dotfiles:

```sh
cp -a /path/to/recon/examples/clamav/. /path/to/infra-repository/
cd /path/to/infra-repository
$EDITOR recon.yaml
```

Change `repository.url` and `repository.ref` to the repository and branch that
will contain these files. Commit and push them before pointing a host at it.

For an existing infrastructure repository, copy only the new stack and secret
files, then merge the relevant entries into the existing `recon.yaml`,
`.sops.yaml`, and `.gitignore`:

```sh
install -d compose/clamav secrets
cp /path/to/recon/examples/clamav/compose/clamav/compose.yaml compose/clamav/
cp /path/to/recon/examples/clamav/secrets/clamav.sops.env secrets/
cp /path/to/recon/examples/clamav/secrets/clamav.env.example secrets/
```

Do not overwrite an existing `.sops.yaml`. Reuse a rule that matches
`secrets/*.sops.env` and add both ClamAV recipients; otherwise merge in the
example's `creation_rules` entry.

If the host already manages other stacks, merge the `clamav` entry from this
`recon.yaml` into the existing `stacks:` block. Replacing the whole document
would stop Recon from managing any stacks it no longer names. It would not
delete them, but it is not the intended migration.

The live project depends on the pre-existing external network `proxy`. Check it
before reconciling:

```sh
docker network inspect proxy
```

## 2. Prepare the host key and authoring key

Recon calls SOPS; it does not install it. The host needs `sops` and `age`, plus
an age private key readable by the user that runs `recon.service`.

The example `.sops.yaml` names two public recipients:

- `age186...` is the existing `/etc/recon/age.txt` identity on `edge-server-1`.
- `age12c...` is the authoring identity on the workstation used to edit the
  encrypted file.

The edge host already has its key. On a new host, create one:

```sh
sudo install -d -m 0750 /etc/recon
sudo age-keygen -o /etc/recon/age.txt
sudo chmod 0600 /etc/recon/age.txt
sudo sed -n 's/^# public key: //p' /etc/recon/age.txt
```

Put a new host's public key in `.sops.yaml`. Then re-wrap the existing file so
both the old and new recipients can decrypt its data key:

```sh
sops updatekeys -y secrets/clamav.sops.env
```

Do not commit or copy the private key out of `/etc/recon/age.txt`. The public
recipients are embedded in the encrypted file and can be audited without any
private key:

```sh
grep -o 'age1[a-z0-9]*' secrets/clamav.sops.env | sort -u
```

The current `edge-server-1` service runs as root, uses
`--config /root/recon.yaml`, and already has:

```ini
Environment=SOPS_AGE_KEY_FILE=/etc/recon/age.txt
```

Keep that environment when updating the service. A packaged install that runs
as the `recon` user must set `SOPS_AGE_KEY_FILE=/etc/recon/age.txt` in its
service environment and make the private key readable by that user.

## 3. Manage the Basic Auth value

`CLAMAV_BASIC_AUTH_USERS` must contain the complete Traefik value, including
the username prefix:

```dotenv
CLAMAV_BASIC_AUTH_USERS='clamav:$2y$12$...'
```

The single quotes are required. They stop Compose from treating the `$`
characters in the bcrypt hash as variable references.

The old `compose.yaml` displays those dollars as `$$`; that is Compose's
escaping in the rendered model. The actual value stored in the running
container's label, which is what this encrypted file contains, uses single
`$` characters.

The checked-in `secrets/clamav.sops.env` was encrypted from the live value and
should not need to be recreated for the first migration. To replace or rotate
it, edit the encrypted file in place:

```sh
sops edit secrets/clamav.sops.env
git diff -- secrets/clamav.sops.env
git add secrets/clamav.sops.env
git commit -m "rotate ClamAV basic auth"
git push
```

SOPS opens plaintext in a private temporary file, encrypts it on save, and keeps
the recipients from `.sops.yaml`. Recon's watch loop will apply the new label
value after the commit is pushed.

If the encrypted file is missing, create it from the template:

```sh
cp secrets/clamav.env.example secrets/clamav.sops.env
$EDITOR secrets/clamav.sops.env
sops encrypt --in-place secrets/clamav.sops.env
```

Do not run any Git command between copying the plaintext template and encrypting
it. Confirm no plaintext is staged:

```sh
git status --short secrets
sops decrypt secrets/clamav.sops.env |
  sed -E 's/^(CLAMAV_BASIC_AUTH_USERS=).*/\1<redacted>/'
```

The second command prints only the variable name when decrypted with a
recipient key.

## 4. Install the Recon configuration

For the conventional packaged service, put the document at the path used by
`recon.service`:

```sh
sudo install -m 0644 recon.yaml /etc/recon/recon.yaml
sudo recon --config /etc/recon/recon.yaml --state-dir /var/lib/recon init
sudo recon --config /etc/recon/recon.yaml --state-dir /var/lib/recon status
sudo recon --config /etc/recon/recon.yaml --state-dir /var/lib/recon diff
```

On the current `edge-server-1` unit, use `/root/recon.yaml` instead and merge
the `clamav` stack into that document. Do not replace the other stack entries.
After updating the file, let the running watch process reconcile, or verify the
configuration and restart the service:

```sh
sudo recon --config /root/recon.yaml --state-dir /var/lib/recon status
sudo recon --config /root/recon.yaml --state-dir /var/lib/recon diff
sudo systemctl restart recon
sudo journalctl -u recon -f
```

## 5. Take over the existing Compose project

Stop using the ad-hoc file and `docker compose up` commands under
`/root/apps/clamav` once Recon manages the same project. Do not delete that
directory until the Recon deployment has been verified; it is a useful local
reference and rollback aid.

The first Recon apply will likely recreate both containers so it can add Recon's
deployment identity labels. The Compose project name, service names, container
names, and named volumes stay the same. Compose does not delete named volumes
during an apply, so the ClamAV database, runtime files, and logs remain in:

```
clamav_clamav
clamav_run-clamav
clamav_var-log-clamav
```

Verify the project and health before removing any manual fallback:

```sh
docker compose -p clamav ps
docker inspect clamav-rest --format '{{.State.Health.Status}}'
docker volume ls --filter label=com.docker.compose.project=clamav
```

Then confirm Recon sees the stack as healthy:

```sh
sudo recon --config /root/recon.yaml --state-dir /var/lib/recon status
```

The walkthrough has not changed anything on the live server. It only captures
the equivalent desired state and secret process.
