# Salt systemd containers Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `salt-controller` and `salt-minion` systemd container images derived from `systemd-almalinux:10`, defaulting to Salt 3006.27 with 3006.6 as an overridable second version, and publish this repo's images to the TKC Labs Nexus docker registry.

**Architecture:** Both new Containerfiles are thin derived layers on top of the existing `systemd-almalinux` image, so the CA trust, Nexus dnf repos, masked-units list and journald→console drop-in are all inherited rather than duplicated. Salt RPMs come from `saltproject-rpm-proxy` on Nexus — a flat, version-agnostic repo, so the version pin lives on the `dnf install` command line via a `SALT_VERSION` build arg rather than in the repo file. The base image reference is itself an overridable build arg so a clean host can build without first building the base locally.

**Tech Stack:** podman 5.8.2 (rootless), AlmaLinux 10, Salt 3006.x (Broadcom onedir RPMs), Sonatype Nexus 3.93 (yum proxy + docker-hosted registry), hadolint 2.15.1 (run as a container).

## Global Constraints

- Design spec: `docs/superpowers/specs/2026-08-19-salt-systemd-containers-design.md`. Read it first.
- Default `SALT_VERSION=3006.27`. Second supported version: `3006.6`. No others.
- Image tags carry a `v` prefix (`v3006.27`); the build arg does not (`3006.27`).
- All package installs go through `https://nexus.tkclabs.io:8443`. Never point a repo file at a public archive; leave stock URLs present but commented out, matching every existing file under `files/`.
- Nexus credentials come from `.envrc` (`NEXUS_USER_RW` / `NEXUS_PASS_RW`). `source .envrc` before any Nexus command. Never hardcode or echo them.
- Podman here is **rootless** with `short-name-mode = enforcing`. Every image reference in a command or doc must be fully qualified: `localhost/…` for local builds, `nexus.tkclabs.io:8443/…` for published ones.
- hadolint is not installed as a binary. Lint with:
  `podman run --rm -i docker.io/hadolint/hadolint:v2.15.1 hadolint - < Containerfile.x`
- Do not repeat base-image setup (CA copy, `systemctl mask` list, journald drop-in) in the derived Containerfiles.
- Do not write any config into `/etc/salt/minion.d/` in either image. Minion config stays at package defaults.
- Existing unrelated local images (`localhost/salt-minion:3006.27`, `localhost/salt-dupe-master:*`, etc.) are from earlier ad-hoc work. Do not delete them. Our tags are `v`-prefixed and will not collide.

## File Structure

| Path | Responsibility |
|---|---|
| `files/salt/SALT-PROJECT-GPG-PUBKEY.pgp` | Salt Project signing key, vendored because it lives outside the proxied path |
| `files/salt/salt.repo` | dnf repo definition pointing at `saltproject-rpm-proxy` |
| `Containerfile.salt-controller` | master + syndic + api + minion image |
| `Containerfile.salt-minion` | minion-only image |
| `README.md` | build / run / publish commands |
| `CLAUDE.md` | architecture and conventions |

---

### Task 1: Vendor the Salt repo config and signing key

**Files:**
- Create: `files/salt/SALT-PROJECT-GPG-PUBKEY.pgp`
- Create: `files/salt/salt.repo`

**Interfaces:**
- Consumes: nothing.
- Produces: `/etc/pki/rpm-gpg/SALT-PROJECT-GPG-PUBKEY.pgp` and `/etc/yum.repos.d/salt.repo` as the `COPY` sources used by Tasks 3 and 4.

- [ ] **Step 1: Fetch the Salt Project signing key**

```bash
mkdir -p files/salt
curl -fsSL -o files/salt/SALT-PROJECT-GPG-PUBKEY.pgp \
  https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public
```

- [ ] **Step 2: Verify the key is the expected one**

```bash
gpg --show-keys files/salt/SALT-PROJECT-GPG-PUBKEY.pgp
```

Expected: fingerprint `10857FFDD3F91EAE577A21D664CBBC8173D76B3F`, uid `Salt Project Packaging <saltproject-packaging@vmware.com>`, `rsa3072`.

**If the fingerprint does not match, stop and report it. Do not continue.**

- [ ] **Step 3: Write the repo file**

Create `files/salt/salt.repo`:

```ini
[salt]
name=Salt Project $releasever - $basearch
# baseurl=https://packages.broadcom.com/artifactory/saltproject-rpm/
baseurl=https://nexus.tkclabs.io:8443/repository/saltproject-rpm-proxy/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/SALT-PROJECT-GPG-PUBKEY.pgp
metadata_expire=86400
enabled_metadata=1
```

The Broadcom repo is flat — one `repodata/` at the root indexing every release — so there is no `$releasever`/`$basearch` path component and no per-version repo file.

- [ ] **Step 4: Verify the mirror actually serves the pinned versions**

```bash
curl -fsS -o /dev/null -w '%{http_code}\n' \
  https://nexus.tkclabs.io:8443/repository/saltproject-rpm-proxy/repodata/repomd.xml
```

Expected: `200`.

- [ ] **Step 5: Commit**

```bash
git add files/salt
git commit -m "feat: add Salt Project repo config and signing key for Nexus mirror"
```

---

### Task 2: Publish the base image so the derived FROM resolves

**Files:**
- None modified. This task only builds and pushes.

**Interfaces:**
- Consumes: `Containerfile.almalinux` (existing, unchanged).
- Produces: `nexus.tkclabs.io:8443/systemd-almalinux:10` and `:latest`, which Tasks 3 and 4 use as the default `FROM`.

- [ ] **Step 1: Build the base image**

```bash
podman build -t systemd-almalinux:latest -t systemd-almalinux:10 -f Containerfile.almalinux .
```

Expected: build succeeds. Requires network access to `nexus.tkclabs.io:8443`.

- [ ] **Step 2: Verify it runs as an init container**

```bash
podman run -d --name base-smoke --systemd=always localhost/systemd-almalinux:10
sleep 5
podman exec base-smoke systemctl is-system-running --wait
podman rm -f base-smoke
```

Expected: `running` or `degraded`. `degraded` is acceptable — masked units count as failed-to-start for some targets. Anything else, stop and report.

- [ ] **Step 3: Log in to the Nexus registry**

```bash
source .envrc
podman login nexus.tkclabs.io:8443 -u "$NEXUS_USER_RW" -p "$NEXUS_PASS_RW"
```

Expected: `Login Succeeded!`

- [ ] **Step 4: Tag and push**

```bash
podman tag localhost/systemd-almalinux:10 nexus.tkclabs.io:8443/systemd-almalinux:10
podman tag localhost/systemd-almalinux:latest nexus.tkclabs.io:8443/systemd-almalinux:latest
podman push nexus.tkclabs.io:8443/systemd-almalinux:10
podman push nexus.tkclabs.io:8443/systemd-almalinux:latest
```

- [ ] **Step 5: Verify the push landed**

```bash
source .envrc
curl -fsS -u "$NEXUS_USER_RW:$NEXUS_PASS_RW" \
  https://nexus.tkclabs.io:8443/v2/systemd-almalinux/tags/list
```

Expected: `{"name":"systemd-almalinux","tags":["10","latest"]}` (tag order may vary).

- [ ] **Step 6: Verify a clean pull works**

```bash
podman rmi nexus.tkclabs.io:8443/systemd-almalinux:10
podman pull nexus.tkclabs.io:8443/systemd-almalinux:10
```

Expected: pull succeeds. This is what proves the derived Containerfiles' default `FROM` will resolve on a clean host.

No commit — this task changes no files.

---

### Task 3: Containerfile.salt-minion

**Files:**
- Create: `Containerfile.salt-minion`

**Interfaces:**
- Consumes: `files/salt/salt.repo` and `files/salt/SALT-PROJECT-GPG-PUBKEY.pgp` from Task 1; `nexus.tkclabs.io:8443/systemd-almalinux:10` from Task 2.
- Produces: build args `BASE_REGISTRY` (default `nexus.tkclabs.io:8443/`), `VERSION` (default `10`), `SALT_VERSION` (default `3006.27`). Task 4 uses the identical arg names and defaults.

- [ ] **Step 1: Write the Containerfile**

Create `Containerfile.salt-minion`:

```dockerfile
# salt-minion
#
# A systemd-almalinux test target running salt-minion. Minion configuration is
# deliberately left at package defaults (master: salt, minion id from hostname);
# configure it at run time or in a thin layer on top of this image.
#
# Build the default image with:
#   podman build -t salt-minion:latest -t salt-minion:v3006.27 -f Containerfile.salt-minion .
#
# Build versioned image with:
#   podman build -t salt-minion:v3006.6 -f Containerfile.salt-minion --build-arg SALT_VERSION=3006.6 .
#
# Build against a locally built base instead of the Nexus registry with:
#   podman build -t salt-minion:latest -f Containerfile.salt-minion --build-arg BASE_REGISTRY= .
#
ARG BASE_REGISTRY=nexus.tkclabs.io:8443/
ARG VERSION=10

FROM ${BASE_REGISTRY}systemd-almalinux:${VERSION}

ARG SALT_VERSION=3006.27

# ---------------------------------------------------------------------------
# The TKC Labs root CA, the Nexus AlmaLinux repos, the masked-units list and the
# journald->console drop-in all come from the base image and are not repeated.
#
# Salt RPMs come from the Nexus proxy of the Broadcom repo. That repo is flat and
# version-agnostic, so the version pin lives on the dnf command line rather than
# in salt.repo, and one repo file serves every supported Salt version.
# ---------------------------------------------------------------------------
COPY files/salt/SALT-PROJECT-GPG-PUBKEY.pgp /etc/pki/rpm-gpg/SALT-PROJECT-GPG-PUBKEY.pgp
COPY files/salt/salt.repo /etc/yum.repos.d/salt.repo

RUN dnf install -y salt-${SALT_VERSION} \
  salt-minion-${SALT_VERSION} \
  && dnf clean all \
  && rm -rf /var/cache/dnf

# hadolint ignore=DL3059
RUN systemctl enable salt-minion.service

WORKDIR /root
CMD [ "/sbin/init" ]
```

- [ ] **Step 2: Lint it**

```bash
podman run --rm -i docker.io/hadolint/hadolint:v2.15.1 hadolint - < Containerfile.salt-minion
```

Expected: no output, exit 0. If `DL3059` does not fire, delete the `# hadolint ignore=DL3059` line and re-lint — do not carry an unnecessary ignore.

- [ ] **Step 3: Build it**

```bash
podman build -t salt-minion:latest -t salt-minion:v3006.27 -f Containerfile.salt-minion .
```

Expected: build succeeds.

- [ ] **Step 4: Verify the pinned version and the enabled unit**

```bash
podman run --rm --entrypoint="" localhost/salt-minion:v3006.27 salt-minion --version
podman run --rm --entrypoint="" localhost/salt-minion:v3006.27 \
  systemctl is-enabled salt-minion.service
```

Expected: `salt-minion 3006.27` and `enabled`.

- [ ] **Step 5: Verify no minion config was written**

```bash
podman run --rm --entrypoint="" localhost/salt-minion:v3006.27 \
  sh -c 'ls -A /etc/salt/minion.d/ 2>/dev/null | wc -l'
```

Expected: `0`. The spec requires package-default minion config; a non-empty `minion.d` means something leaked in.

- [ ] **Step 6: Commit**

```bash
git add Containerfile.salt-minion
git commit -m "feat: add salt-minion systemd image, default Salt 3006.27"
```

---

### Task 4: Containerfile.salt-controller

**Files:**
- Create: `Containerfile.salt-controller`

**Interfaces:**
- Consumes: `files/salt/salt.repo` and `files/salt/SALT-PROJECT-GPG-PUBKEY.pgp` from Task 1; `nexus.tkclabs.io:8443/systemd-almalinux:10` from Task 2.
- Produces: same build arg names and defaults as Task 3 — `BASE_REGISTRY`, `VERSION`, `SALT_VERSION`. Also produces `/etc/salt/pki/master/master.pem` (0400) and `master.pub` (0644), and `/etc/salt/master.d/auto-accept.conf`.

- [ ] **Step 1: Write the Containerfile**

Create `Containerfile.salt-controller`:

```dockerfile
# salt-controller
#
# A systemd-almalinux test target running salt-master and salt-minion. The minion
# runs here too because the target topology is tiered: a tier-2 controller is
# itself a minion of a tier-1 controller. Minion configuration is deliberately
# left at package defaults (master: salt, minion id from hostname); configure it
# at run time or in a thin layer on top of this image.
#
# salt-syndic and salt-api are installed but not enabled — both need topology
# specific config (syndic_master, rest_cherrypy + eauth) and fail-loop without it.
#
# Build the default image with:
#   podman build -t salt-controller:latest -t salt-controller:v3006.27 -f Containerfile.salt-controller .
#
# Build versioned image with:
#   podman build -t salt-controller:v3006.6 -f Containerfile.salt-controller --build-arg SALT_VERSION=3006.6 .
#
# Build against a locally built base instead of the Nexus registry with:
#   podman build -t salt-controller:latest -f Containerfile.salt-controller --build-arg BASE_REGISTRY= .
#
ARG BASE_REGISTRY=nexus.tkclabs.io:8443/
ARG VERSION=10

FROM ${BASE_REGISTRY}systemd-almalinux:${VERSION}

ARG SALT_VERSION=3006.27

# ---------------------------------------------------------------------------
# The TKC Labs root CA, the Nexus AlmaLinux repos, the masked-units list and the
# journald->console drop-in all come from the base image and are not repeated.
#
# Salt RPMs come from the Nexus proxy of the Broadcom repo. That repo is flat and
# version-agnostic, so the version pin lives on the dnf command line rather than
# in salt.repo, and one repo file serves every supported Salt version.
# ---------------------------------------------------------------------------
COPY files/salt/SALT-PROJECT-GPG-PUBKEY.pgp /etc/pki/rpm-gpg/SALT-PROJECT-GPG-PUBKEY.pgp
COPY files/salt/salt.repo /etc/yum.repos.d/salt.repo

RUN dnf install -y salt-${SALT_VERSION} \
  salt-master-${SALT_VERSION} \
  salt-minion-${SALT_VERSION} \
  salt-syndic-${SALT_VERSION} \
  salt-api-${SALT_VERSION} \
  && dnf clean all \
  && rm -rf /var/cache/dnf

# ---------------------------------------------------------------------------
# Bake in a master keypair so the master identity exists before first start.
# Generated per build rather than committed, so no private key lands in git.
# Each build therefore gets its own master identity, which is fine for the
# ephemeral test containers these images are for.
# ---------------------------------------------------------------------------
RUN mkdir -p /etc/salt/pki/master \
  && salt-key --gen-keys=master --gen-keys-dir=/etc/salt/pki/master \
  && chmod 0400 /etc/salt/pki/master/master.pem \
  && chmod 0644 /etc/salt/pki/master/master.pub

# ---------------------------------------------------------------------------
# Auto-accept minion keys. This is deliberate and TEST ONLY: any minion that
# connects gets its key accepted with no operator action. Never use this image
# as a production master.
# ---------------------------------------------------------------------------
RUN mkdir -p /etc/salt/master.d && \
    printf '%s\n' 'auto_accept: True' > /etc/salt/master.d/auto-accept.conf

# hadolint ignore=DL3059
RUN systemctl enable salt-master.service salt-minion.service

WORKDIR /root
EXPOSE 4505 4506
CMD [ "/sbin/init" ]
```

- [ ] **Step 2: Lint it**

```bash
podman run --rm -i docker.io/hadolint/hadolint:v2.15.1 hadolint - < Containerfile.salt-controller
```

Expected: no output, exit 0. If `DL3059` does not fire, delete the `# hadolint ignore=DL3059` line and re-lint.

- [ ] **Step 3: Build it**

```bash
podman build -t salt-controller:latest -t salt-controller:v3006.27 -f Containerfile.salt-controller .
```

Expected: build succeeds.

- [ ] **Step 4: Verify version, keys, modes, auto-accept and enabled units**

```bash
podman run --rm --entrypoint="" localhost/salt-controller:v3006.27 sh -c '
  salt-master --version
  stat -c "%n %a" /etc/salt/pki/master/master.pem /etc/salt/pki/master/master.pub
  cat /etc/salt/master.d/auto-accept.conf
  systemctl is-enabled salt-master.service salt-minion.service
  systemctl is-enabled salt-api.service salt-syndic.service || true
  ls -A /etc/salt/minion.d/ | wc -l
'
```

Expected:
- `salt-master 3006.27`
- `/etc/salt/pki/master/master.pem 400` and `/etc/salt/pki/master/master.pub 644`
- `auto_accept: True`
- `enabled` twice for master and minion
- `disabled` for api and syndic
- `0` for the `minion.d` count

- [ ] **Step 5: Verify the master and minion actually start under systemd**

```bash
podman run -d --name ctl-smoke --systemd=always localhost/salt-controller:v3006.27
for _ in $(seq 60); do
  podman exec ctl-smoke systemctl is-active salt-master.service >/dev/null 2>&1 && break
  sleep 1
done
podman exec ctl-smoke systemctl is-active salt-master.service salt-minion.service
podman rm -f ctl-smoke
```

Expected: `active` for both.

- [ ] **Step 6: Commit**

```bash
git add Containerfile.salt-controller
git commit -m "feat: add salt-controller systemd image with baked master key and auto-accept"
```

---

### Task 5: Two-container integration test, then the 3006.6 builds

**Files:**
- None modified. This task validates Tasks 3 and 4 end to end and produces the second version's images.

**Interfaces:**
- Consumes: `localhost/salt-controller:v3006.27`, `localhost/salt-minion:v3006.27`.
- Produces: `localhost/salt-controller:v3006.6`, `localhost/salt-minion:v3006.6`.

- [ ] **Step 1: Create the test network**

```bash
podman network exists saltdev || podman network create saltdev
```

- [ ] **Step 2: Start a controller and a minion on it**

```bash
podman run -d --name salt-controller --network saltdev --network-alias salt \
  --systemd=always localhost/salt-controller:v3006.27
podman run -d --name minion01 --network saltdev \
  --systemd=always localhost/salt-minion:v3006.27
```

`saltdev` is a user-defined network, which is what gives containers DNS resolution
by `--name`. `--network-alias salt` adds a second name so the minions' untouched
default `master: salt` resolves without configuring them.

- [ ] **Step 3: Wait for the minion key to be auto-accepted**

```bash
for _ in $(seq 90); do
  podman exec salt-controller salt-key -L 2>/dev/null | grep -q minion01 && break
  sleep 2
done
podman exec salt-controller salt-key -L
```

Expected: `minion01` and the controller's own minion both listed under `Accepted Keys:`, and `Unaccepted Keys:` empty. An entry under `Unaccepted Keys` means `auto_accept` did not take effect.

- [ ] **Step 4: Ping both minions**

```bash
podman exec salt-controller salt '*' test.ping
```

Expected: `True` for `minion01` and for the controller's own minion id.

- [ ] **Step 5: Tear down**

```bash
podman rm -f salt-controller minion01
```

- [ ] **Step 6: Build both images at 3006.6**

```bash
podman build -t salt-controller:v3006.6 -f Containerfile.salt-controller --build-arg SALT_VERSION=3006.6 .
podman build -t salt-minion:v3006.6 -f Containerfile.salt-minion --build-arg SALT_VERSION=3006.6 .
```

Expected: both succeed.

- [ ] **Step 7: Verify the 3006.6 pin took**

```bash
podman run --rm --entrypoint="" localhost/salt-controller:v3006.6 salt-master --version
podman run --rm --entrypoint="" localhost/salt-minion:v3006.6 salt-minion --version
```

Expected: `salt-master 3006.6` and `salt-minion 3006.6`.

- [ ] **Step 8: Repeat the integration test at 3006.6**

```bash
podman run -d --name salt-controller --network saltdev --network-alias salt \
  --systemd=always localhost/salt-controller:v3006.6
podman run -d --name minion01 --network saltdev \
  --systemd=always localhost/salt-minion:v3006.6
for _ in $(seq 90); do
  podman exec salt-controller salt-key -L 2>/dev/null | grep -q minion01 && break
  sleep 2
done
podman exec salt-controller salt '*' test.ping
podman rm -f salt-controller minion01
```

Expected: same as Steps 3–4 — both minions accepted and answering `True`.

No commit — this task changes no files. Report any failure rather than working around it.

---

### Task 6: Documentation

**Files:**
- Modify: `README.md`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: the build args and tags established in Tasks 3–5.
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Add the Salt build sections to README.md**

Append after the existing `### systemd-ubuntu` section:

````markdown
### salt-controller

AlmaLinux 10 systemd container running `salt-master` and `salt-minion`, with a
master keypair baked in and `auto_accept` enabled. `salt-syndic` and `salt-api`
are installed but not enabled. Auto-accept is test-only — never use this as a
production master.

```bash
# Build default image (Salt 3006.27):
podman build -t salt-controller:latest -t salt-controller:v3006.27 -f Containerfile.salt-controller .

# Build specific version image (Salt 3006.6)
podman build -t salt-controller:v3006.6 -f Containerfile.salt-controller --build-arg SALT_VERSION=3006.6 .

# Build against a locally built base instead of the Nexus registry
podman build -t salt-controller:latest -f Containerfile.salt-controller --build-arg BASE_REGISTRY= .
```

### salt-minion

AlmaLinux 10 systemd container running `salt-minion`. Minion configuration is left
at package defaults (`master: salt`, minion id from hostname).

```bash
# Build default image (Salt 3006.27):
podman build -t salt-minion:latest -t salt-minion:v3006.27 -f Containerfile.salt-minion .

# Build specific version image (Salt 3006.6)
podman build -t salt-minion:v3006.6 -f Containerfile.salt-minion --build-arg SALT_VERSION=3006.6 .

# Build against a locally built base instead of the Nexus registry
podman build -t salt-minion:latest -f Containerfile.salt-minion --build-arg BASE_REGISTRY= .
```

## Running the images

Podman on the lab hosts is rootless with `short-name-mode = enforcing`, so image
references must be fully qualified — `localhost/...` for a local build,
`nexus.tkclabs.io:8443/...` for a published one. A bare `salt-controller:v3006.27`
fails with `short-name resolution enforced but cannot prompt without a TTY`.

```bash
podman run -d --name test --systemd=always localhost/systemd-almalinux:10
```

A minimal two-container Salt lab. The user-defined network is what gives the
containers DNS resolution by `--name`; the `salt` alias satisfies the minions'
untouched default `master: salt`.

```bash
podman network create saltdev

podman run -d --name salt-controller --network saltdev --network-alias salt \
  --systemd=always localhost/salt-controller:v3006.27

podman run -d --name minion01 --network saltdev \
  --systemd=always localhost/salt-minion:v3006.27

podman exec salt-controller salt-key -L
podman exec salt-controller salt '*' test.ping
```

## Publishing to Nexus

Images are published to the Nexus private docker registry at
`nexus.tkclabs.io:8443`. Credentials come from `.envrc`.

```bash
source .envrc
podman login nexus.tkclabs.io:8443 -u "$NEXUS_USER_RW" -p "$NEXUS_PASS_RW"

for ref in \
  systemd-almalinux:latest systemd-almalinux:10 \
  systemd-rockylinux:latest systemd-rockylinux:10 \
  systemd-fedora:latest systemd-fedora:44 \
  systemd-debian:latest systemd-debian:13 \
  systemd-ubuntu:latest systemd-ubuntu:26.04 \
  salt-controller:latest salt-controller:v3006.27 salt-controller:v3006.6 \
  salt-minion:latest salt-minion:v3006.27 salt-minion:v3006.6 \
; do
  podman tag "localhost/${ref}" "nexus.tkclabs.io:8443/${ref}"
  podman push "nexus.tkclabs.io:8443/${ref}"
done
```
````

- [ ] **Step 2: Update the "What this is" section of CLAUDE.md**

Replace the existing `## What this is` body with:

```markdown
Seven Containerfiles that build **systemd-enabled base images** (AlmaLinux,
RockyLinux, Fedora, Debian, Ubuntu) and two **Salt test images** derived from the
AlmaLinux one, for use as test targets — e.g. SSH-reachable hosts for
Ansible/pyinfra/Salt runs. The images run `/sbin/init` as PID 1 so real systemd
units can be exercised inside the container. There is no application code; the
"source" is the seven Containerfiles plus per-version repo/sources config under
`files/`.

The five base images split into two families that share structure:
- **RPM/dnf** (`Containerfile.almalinux`, `Containerfile.rockylinux`, `Containerfile.fedora`) — dnf install, `.repo` files under `files/<distro>/<version>/`, CA trusted via `/etc/pki/ca-trust/source/anchors/`.
- **APT** (`Containerfile.debian`, `Containerfile.ubuntu`) — apt install, deb822 `.sources` files under `files/<distro>/<version>/`, CA trusted via `/usr/local/share/ca-certificates/`.

The two Salt images (`Containerfile.salt-controller`, `Containerfile.salt-minion`)
are a third, **derived** family: they build `FROM systemd-almalinux` rather than
from a distro base, and inherit its CA trust, Nexus repos, masked-units list and
journald drop-in instead of repeating any of it.
```

- [ ] **Step 3: Add the Salt build commands to CLAUDE.md**

Append to the `## Build commands` fenced block, after the Ubuntu lines:

```bash
# salt-controller (default SALT_VERSION=3006.27; supported: 3006.27, 3006.6)
podman build -t salt-controller:latest -t salt-controller:v3006.27 -f Containerfile.salt-controller .
podman build -t salt-controller:v3006.6 -f Containerfile.salt-controller --build-arg SALT_VERSION=3006.6 .

# salt-minion (default SALT_VERSION=3006.27; supported: 3006.27, 3006.6)
podman build -t salt-minion:latest -t salt-minion:v3006.27 -f Containerfile.salt-minion .
podman build -t salt-minion:v3006.6 -f Containerfile.salt-minion --build-arg SALT_VERSION=3006.6 .
```

- [ ] **Step 4: Fix the run example and the lint command in CLAUDE.md**

The existing run example uses a bare short name, which fails under enforcing
short-name mode. Replace:

```bash
podman run -d --name test --systemd=always systemd-almalinux:10
```

with:

```bash
podman run -d --name test --systemd=always localhost/systemd-almalinux:10
```

And extend the lint command to cover all seven files:

```bash
hadolint Containerfile.almalinux Containerfile.rockylinux Containerfile.fedora \
  Containerfile.debian Containerfile.ubuntu \
  Containerfile.salt-controller Containerfile.salt-minion
```

- [ ] **Step 5: Add the Salt conventions to the CLAUDE.md "Architecture & conventions" list**

Append these bullets:

```markdown
- **The Salt images are derived, not parallel.** `Containerfile.salt-controller`
  and `Containerfile.salt-minion` build `FROM ${BASE_REGISTRY}systemd-almalinux:${VERSION}`.
  Do **not** copy the CA, the `systemctl mask` list or the journald drop-in into
  them — those are inherited, and duplicating them would create a second copy to
  keep in sync. The "keep the Containerfiles parallel" rule applies *within* the
  Salt pair, which differs only by package set, master key generation, the
  auto-accept drop-in, `systemctl enable salt-master`, and `EXPOSE`.

- **`BASE_REGISTRY` defaults to the Nexus docker registry.** The Salt images
  default to `nexus.tkclabs.io:8443/` so a clean host can build them without
  building the base first. Pass `--build-arg BASE_REGISTRY=` (empty) to build
  against a local `systemd-almalinux:${VERSION}`. Changing the base image means
  rebuilding *and re-pushing* it before the Salt images pick it up.

- **Salt versions are pinned on the dnf command line, not in the repo file.**
  Unlike the distro repos, `saltproject-rpm-proxy` mirrors a flat, version-agnostic
  Broadcom repo — one `repodata/` indexing every release. So there is no
  `files/salt/<version>/`; a single `files/salt/salt.repo` serves every version and
  `SALT_VERSION` is interpolated into `dnf install salt-master-${SALT_VERSION}`.
  Adding a new supported version needs no new files, but do verify the packages
  exist in the mirror first.

- **The Salt signing key is vendored.** Nexus proxies
  `packages.broadcom.com/artifactory/saltproject-rpm/`, but the signing key lives
  at `/artifactory/api/security/keypair/SaltProjectKey/public` — outside that path
  and therefore not reachable through the mirror. It is committed as
  `files/salt/SALT-PROJECT-GPG-PUBKEY.pgp` (fingerprint
  `10857FFDD3F91EAE577A21D664CBBC8173D76B3F`) and `COPY`'d to
  `/etc/pki/rpm-gpg/`, the same pattern as the root CA. Re-fetch and re-verify the
  fingerprint if Salt rotates it.

- **`auto_accept: True` on the controller is deliberate and test-only.** Any
  minion that connects gets its key accepted with no operator action. These images
  exist to be thrown away; never use one as a production master. The master
  keypair is generated at build time rather than committed, so every build has a
  distinct master identity and no private key lives in git.

- **The controller runs a minion too.** The target topology is tiered — tier-1
  controllers manage the minions running on tier-2 controllers, which manage the
  fleet — so a controller is itself a minion of an upstream controller. Minion
  config in both images is left at package defaults on purpose (`master: salt`,
  minion id from hostname); do not add `/etc/salt/minion.d/` drop-ins. Topology is
  configured at time of use or in a thin layer on top.

- **Images publish to the Nexus docker registry** at `nexus.tkclabs.io:8443`
  (Nexus `docker-hosted`, docker connector 8444 behind TLS on 8443,
  `forceBasicAuth`). Credentials are `NEXUS_USER_RW` / `NEXUS_PASS_RW` from
  `.envrc`. Podman on the lab hosts runs rootless with
  `short-name-mode = enforcing`, so every image reference must be fully
  qualified — `localhost/...` locally, `nexus.tkclabs.io:8443/...` once published.
```

- [ ] **Step 6: Commit**

```bash
git add README.md CLAUDE.md
git commit -m "docs: document salt images, Nexus publishing, and short-name requirements"
```

---

### Task 7: Build and publish the remaining images

**Files:**
- None modified.

**Interfaces:**
- Consumes: every Containerfile in the repo.
- Produces: the full published tag set on `nexus.tkclabs.io:8443`.

- [ ] **Step 1: Build the four remaining base images**

```bash
podman build -t systemd-rockylinux:latest -t systemd-rockylinux:10 -f Containerfile.rockylinux .
podman build -t systemd-fedora:latest -t systemd-fedora:44 -f Containerfile.fedora .
podman build -t systemd-debian:latest -t systemd-debian:13 -f Containerfile.debian .
podman build -t systemd-ubuntu:latest -t systemd-ubuntu:26.04 -f Containerfile.ubuntu .
```

Expected: all four succeed. `systemd-almalinux` was already built in Task 2.

- [ ] **Step 2: Tag `latest` on the Salt images**

Task 3 and 4 tagged `latest` at 3006.27 already; confirm rather than re-tag:

```bash
podman images --format '{{.Repository}}:{{.Tag}} {{.ID}}' \
  | grep -E 'localhost/salt-(controller|minion)'
```

Expected: `latest` and `v3006.27` share an image ID for both repositories; `v3006.6` is a distinct ID.

- [ ] **Step 3: Push everything**

```bash
source .envrc
podman login nexus.tkclabs.io:8443 -u "$NEXUS_USER_RW" -p "$NEXUS_PASS_RW"

for ref in \
  systemd-almalinux:latest systemd-almalinux:10 \
  systemd-rockylinux:latest systemd-rockylinux:10 \
  systemd-fedora:latest systemd-fedora:44 \
  systemd-debian:latest systemd-debian:13 \
  systemd-ubuntu:latest systemd-ubuntu:26.04 \
  salt-controller:latest salt-controller:v3006.27 salt-controller:v3006.6 \
  salt-minion:latest salt-minion:v3006.27 salt-minion:v3006.6 \
; do
  podman tag "localhost/${ref}" "nexus.tkclabs.io:8443/${ref}"
  podman push "nexus.tkclabs.io:8443/${ref}"
done
```

- [ ] **Step 4: Verify the full published set**

```bash
source .envrc
for name in systemd-almalinux systemd-rockylinux systemd-fedora systemd-debian \
            systemd-ubuntu salt-controller salt-minion; do
  echo -n "$name: "
  curl -fsS -u "$NEXUS_USER_RW:$NEXUS_PASS_RW" \
    "https://nexus.tkclabs.io:8443/v2/${name}/tags/list"
  echo
done
```

Expected, tag order may vary:
- `systemd-almalinux` → `10`, `latest`
- `systemd-rockylinux` → `10`, `latest`
- `systemd-fedora` → `44`, `latest`
- `systemd-debian` → `13`, `latest`
- `systemd-ubuntu` → `26.04`, `latest`
- `salt-controller` → `latest`, `v3006.27`, `v3006.6`
- `salt-minion` → `latest`, `v3006.27`, `v3006.6`

- [ ] **Step 5: Verify a published Salt image pulls and runs clean**

```bash
podman rmi nexus.tkclabs.io:8443/salt-minion:v3006.27
podman pull nexus.tkclabs.io:8443/salt-minion:v3006.27
podman run --rm --entrypoint="" nexus.tkclabs.io:8443/salt-minion:v3006.27 salt-minion --version
```

Expected: `salt-minion 3006.27`.

No commit — this task changes no files.

---

## Out of scope

Recorded so a later reader knows these were decisions, not oversights:

- Non-default distro version tags (`systemd-almalinux:9`, `systemd-debian:11`, `systemd-debian:12`) are not built or published in this pass.
- No `dnf versionlock`. The images are immutable and nothing upgrades in place.
- No Salt topology config, no syndic or api enablement.
- No CI automation for the build/publish loop.
