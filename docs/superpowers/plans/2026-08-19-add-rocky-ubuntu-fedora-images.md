# Add RockyLinux, Ubuntu, and Fedora systemd base images — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add RockyLinux 10, Ubuntu 26.04, and Fedora 44 as new systemd-enabled base images alongside the existing AlmaLinux and Debian ones, fix the pre-existing broken Nexus mirror paths, and replace the stale vault-fetched CA with the new baked-in TKC Labs root CA across all five Containerfiles.

**Architecture:** Each distro is its own `Containerfile.<distro>` (RPM/dnf family: almalinux, rockylinux, fedora; APT family: debian, ubuntu), parameterized by a `VERSION` build-arg, with per-version repo config under `files/<distro>/<version>/`. Trust is bootstrapped from a static CA file baked into the image via `COPY` (no more runtime fetch from Vault). Package mirrors are internal Nexus proxy repos.

**Tech Stack:** Podman/Containerfile (OCI), dnf (yum-format repos), apt (deb822 sources), Nexus Repository Manager 3 REST API.

There is no unit-test suite in this repo — the "test" for each task is a real `podman build` against the actual Nexus infrastructure (this repo requires TKC Labs network access to build at all, per CLAUDE.md), plus, for the Nexus task, a REST API round-trip. `hadolint` is not installed in this environment, so it is not used as a gating step here even though CLAUDE.md documents it as the expected linter.

## Global Constraints

- AlmaLinux default `VERSION=10`, Debian default `VERSION=13` — already correct, do not change.
- RockyLinux 10, Ubuntu 26.04, Fedora 44 each support exactly one version (no multi-version matrix like Alma/Debian have).
- Nexus proxy repos must follow the existing `<name>-proxy` naming convention.
- The new root CA is `files/ca/TKC-Labs-Root-CA-2026-2036.crt`, baked into every image via `COPY`. No Containerfile may depend on `vault.tkclabs.io` anymore.
- The masked-systemd-units list, journald-to-console drop-in, and commented-out sshd/root-password/test-user scaffolding must be byte-identical across all five Containerfiles.
- sshd/root-password/test-user provisioning stays commented out — do not enable it.
- Do not touch the `github` remote in `.git/config` (git config, out of scope).

---

### Task 1: Create the missing Nexus proxy repositories

**Files:** None (Nexus is external infrastructure; no repo files touched in this task).

**Interfaces:**
- Consumes: `NEXUS_USER_RW` / `NEXUS_PASS_RW` from `.envrc` (already present, gitignored).
- Produces: 7 new Nexus proxy repos that later tasks' `podman build` steps pull packages through: `debian-bullseye-proxy`, `debian-bullseye-security-proxy`, `debian-bookworm-security-proxy`, `debian-trixie-security-proxy`, `rockylinux-proxy`, `ubuntu-resolute-proxy`, `ubuntu-resolute-security-proxy`.

- [ ] **Step 1: Create the 4 new apt proxy repos**

Run (from the repo root, so `.envrc` is picked up):

```bash
source .envrc

for repo in \
  'debian-bullseye-proxy|bullseye|http://deb.debian.org/debian/' \
  'debian-bullseye-security-proxy|bullseye-security|http://deb.debian.org/debian-security/' \
  'debian-bookworm-security-proxy|bookworm-security|http://deb.debian.org/debian-security/' \
  'debian-trixie-security-proxy|trixie-security|http://deb.debian.org/debian-security/' \
  'ubuntu-resolute-proxy|resolute|http://archive.ubuntu.com/ubuntu/' \
  'ubuntu-resolute-security-proxy|resolute-security|http://security.ubuntu.com/ubuntu/'
do
  IFS='|' read -r name dist remote <<< "$repo"
  curl -s -k -u "${NEXUS_USER_RW}:${NEXUS_PASS_RW}" \
    -H "Content-Type: application/json" \
    -X POST "https://nexus.tkclabs.io:8443/service/rest/v1/repositories/apt/proxy" \
    -d "{
      \"name\": \"${name}\",
      \"online\": true,
      \"storage\": {\"blobStoreName\": \"default\", \"strictContentTypeValidation\": true},
      \"apt\": {\"distribution\": \"${dist}\", \"flat\": false},
      \"proxy\": {\"remoteUrl\": \"${remote}\", \"contentMaxAge\": 1440, \"metadataMaxAge\": 1440},
      \"negativeCache\": {\"enabled\": true, \"timeToLive\": 1440},
      \"httpClient\": {\"blocked\": false, \"autoBlock\": true, \"connection\": {\"enableCircularRedirects\": false, \"enableCookies\": false, \"useTrustStore\": false}}
    }" -w "\n%{http_code} ${name}\n"
done
```

Expected: each iteration prints `201 <name>` (Nexus returns 201 Created on successful repo creation — verified directly against this Nexus instance).

- [ ] **Step 2: Create the rockylinux-proxy yum repo**

```bash
source .envrc
curl -s -k -u "${NEXUS_USER_RW}:${NEXUS_PASS_RW}" \
  -H "Content-Type: application/json" \
  -X POST "https://nexus.tkclabs.io:8443/service/rest/v1/repositories/yum/proxy" \
  -d '{
    "name": "rockylinux-proxy",
    "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
    "proxy": {"remoteUrl": "https://download.rockylinux.org/pub/rocky/", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true, "connection": {"enableCircularRedirects": false, "enableCookies": false, "useTrustStore": false}}
  }' -w "\n%{http_code}\n"
```

Expected: `201`.

- [ ] **Step 3: Verify all 7 repos exist with the right remote URLs**

```bash
source .envrc
for name in debian-bullseye-proxy debian-bullseye-security-proxy debian-bookworm-security-proxy \
            debian-trixie-security-proxy rockylinux-proxy ubuntu-resolute-proxy ubuntu-resolute-security-proxy; do
  fmt=apt; [ "$name" = "rockylinux-proxy" ] && fmt=yum
  echo "== $name =="
  curl -s -k -u "${NEXUS_USER_RW}:${NEXUS_PASS_RW}" \
    "https://nexus.tkclabs.io:8443/service/rest/v1/repositories/${fmt}/proxy/${name}" \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['name'], d['proxy']['remoteUrl'])"
done
```

Expected: 7 lines, each printing the repo name and the remote URL from Step 1/2 — no errors.

- [ ] **Step 4: Commit**

No repo files changed in this task, so there is nothing to commit. Proceed to Task 2.

---

### Task 2: Add the new TKC Labs root CA to the repo

**Files:**
- Create: `files/ca/TKC-Labs-Root-CA-2026-2036.crt`

**Interfaces:**
- Consumes: the cert content available on this host at `/etc/pki/ca-trust/source/anchors/TKC-Labs-Root-CA-2026-2036.crt`.
- Produces: `files/ca/TKC-Labs-Root-CA-2026-2036.crt`, which Tasks 3–7 `COPY` into every Containerfile.

- [ ] **Step 1: Copy the CA file into the repo**

```bash
mkdir -p files/ca
cp /etc/pki/ca-trust/source/anchors/TKC-Labs-Root-CA-2026-2036.crt files/ca/TKC-Labs-Root-CA-2026-2036.crt
```

- [ ] **Step 2: Verify it's a valid, non-expired root CA cert with the expected fingerprint**

```bash
openssl x509 -in files/ca/TKC-Labs-Root-CA-2026-2036.crt -noout -subject -issuer -dates -fingerprint -sha256
sha256sum files/ca/TKC-Labs-Root-CA-2026-2036.crt
```

Expected:
- `subject=O=TKC Labs, CN=TKC Labs Root CA 2026-2036`
- `issuer=O=TKC Labs, CN=TKC Labs Root CA 2026-2036` (self-signed root)
- `notBefore=Aug 16 18:43:42 2026 GMT`, `notAfter=Aug 13 18:44:12 2036 GMT`
- sha256sum: `2d6ec462c5a3182373351b4621263ba39daa74a28e2e56b13e97a8cb7ad553ca`

- [ ] **Step 3: Commit**

```bash
git add files/ca/TKC-Labs-Root-CA-2026-2036.crt
git commit -m "$(cat <<'EOF'
feat: add TKC Labs Root CA 2026-2036 for baked-in image trust

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Fix Containerfile.almalinux and its local.repo files

**Files:**
- Modify: `Containerfile.almalinux`
- Modify: `files/almalinux/9/local.repo`
- Modify: `files/almalinux/10/local.repo`

**Interfaces:**
- Consumes: `files/ca/TKC-Labs-Root-CA-2026-2036.crt` (Task 2), `almalinux-proxy` Nexus repo (already existed).
- Produces: working `systemd-almalinux:9` / `systemd-almalinux:10` images.

- [ ] **Step 1: Replace the vault CA fetch with the baked-in CA in Containerfile.almalinux**

Edit `Containerfile.almalinux`, replacing:

```dockerfile
# Setup our local repo mirrors
# hadolint ignore=SC2005
RUN rm -f /etc/yum.repos.d/*.repo \
  && echo "$(curl -s --insecure https://vault.tkclabs.io:8200/v1/root_ca/ca/pem && echo)" > /etc/pki/ca-trust/source/anchors/tkclabs-root-ca.pem \
  && update-ca-trust

COPY files/almalinux/${VERSION}/local.repo /etc/yum.repos.d/local.repo
```

with:

```dockerfile
# Setup our local repo mirrors
COPY files/ca/TKC-Labs-Root-CA-2026-2036.crt /etc/pki/ca-trust/source/anchors/TKC-Labs-Root-CA-2026-2036.crt

RUN rm -f /etc/yum.repos.d/*.repo \
  && update-ca-trust

COPY files/almalinux/${VERSION}/local.repo /etc/yum.repos.d/local.repo
```

- [ ] **Step 2: Fix the broken Nexus path in files/almalinux/9/local.repo**

Replace the file's content with (only the `baseurl` lines changed — `/repository/almalinux/` → `/repository/almalinux-proxy/`; `gpgkey` stays version-9):

```ini
[appstream]
name=AlmaLinux $releasever - AppStream
mirrorlist=https://mirrors.almalinux.org/mirrorlist/$releasever/appstream
# baseurl=https://repo.almalinux.org/almalinux/$releasever/AppStream/$basearch/os/
baseurl=https://nexus.tkclabs.io:8443/repository/almalinux-proxy/$releasever/AppStream/$basearch/os/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux-9
metadata_expire=86400
enabled_metadata=1

[baseos]
name=AlmaLinux $releasever - BaseOS
# mirrorlist=https://mirrors.almalinux.org/mirrorlist/$releasever/baseos
# baseurl=https://repo.almalinux.org/almalinux/$releasever/BaseOS/$basearch/os/
baseurl=https://nexus.tkclabs.io:8443/repository/almalinux-proxy/$releasever/BaseOS/$basearch/os/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux-9
metadata_expire=86400
enabled_metadata=1

[crb]
name=AlmaLinux $releasever - CRB
mirrorlist=https://mirrors.almalinux.org/mirrorlist/$releasever/crb
# baseurl=https://repo.almalinux.org/almalinux/$releasever/CRB/$basearch/os/
baseurl=https://nexus.tkclabs.io:8443/repository/almalinux-proxy/$releasever/CRB/$basearch/os/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux-9
metadata_expire=86400
enabled_metadata=0

[extras]
name=AlmaLinux $releasever - Extras
mirrorlist=https://mirrors.almalinux.org/mirrorlist/$releasever/extras
# baseurl=https://repo.almalinux.org/almalinux/$releasever/extras/$basearch/os/
baseurl=https://nexus.tkclabs.io:8443/repository/almalinux-proxy/$releasever/extras/$basearch/os/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux-9
metadata_expire=86400
enabled_metadata=0
```

- [ ] **Step 3: Fix the broken Nexus path in files/almalinux/10/local.repo**

Same as Step 2 but with `gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux-10` on all four sections instead of `-9`.

- [ ] **Step 4: Build and verify**

```bash
podman build -t systemd-almalinux:latest -t systemd-almalinux:10 -f Containerfile.almalinux .
podman build -t systemd-almalinux:9 -f Containerfile.almalinux --build-arg VERSION=9 .
```

Expected: both builds complete with exit code 0 (may take a few minutes — package mirror sync through Nexus). If either fails with a TLS or 404/repo error, the CA or Nexus path fix is wrong — re-check Steps 1–3 before proceeding.

- [ ] **Step 5: Commit**

```bash
git add Containerfile.almalinux files/almalinux/9/local.repo files/almalinux/10/local.repo
git commit -m "$(cat <<'EOF'
fix: correct AlmaLinux Nexus mirror paths and switch to baked-in CA

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Fix Containerfile.debian and its debian.sources files

**Files:**
- Modify: `Containerfile.debian`
- Modify: `files/debian/11/debian.sources`
- Modify: `files/debian/12/debian.sources`
- Modify: `files/debian/13/debian.sources`

**Interfaces:**
- Consumes: `files/ca/TKC-Labs-Root-CA-2026-2036.crt` (Task 2); `debian-bookworm-proxy`, `debian-trixie-proxy` (pre-existing); `debian-bullseye-proxy`, `debian-bullseye-security-proxy`, `debian-bookworm-security-proxy`, `debian-trixie-security-proxy` (Task 1).
- Produces: working `systemd-debian:11` / `:12` / `:13` images.

- [ ] **Step 1: Replace the vault CA fetch with the baked-in CA in Containerfile.debian**

Edit `Containerfile.debian`, replacing:

```dockerfile
# Download curl so we can get the root ca
RUN DEBIAN_FRONTEND=noninteractive apt-get update && \
  apt-get install curl -y

# Setup our local repo mirrors
RUN echo "$(curl -s --insecure https://vault.tkclabs.io:8200/v1/root_ca/ca/pem && echo)" \
  > /usr/local/share/ca-certificates/tkclabs-root-ca.crt && \
  update-ca-certificates && \
  apt-get clean && \
  rm -rf /etc/apt/sources.list

# Deploy sources file for local cache
COPY files/debian/${VERSION}/debian.sources /etc/apt/sources.list.d/debian.sources
```

with:

```dockerfile
# Setup our local repo mirrors
COPY files/ca/TKC-Labs-Root-CA-2026-2036.crt /usr/local/share/ca-certificates/TKC-Labs-Root-CA-2026-2036.crt

RUN update-ca-certificates && \
  rm -rf /etc/apt/sources.list

# Deploy sources file for local cache
COPY files/debian/${VERSION}/debian.sources /etc/apt/sources.list.d/debian.sources
```

- [ ] **Step 2: Fix files/debian/11/debian.sources**

Replace the file's content with:

```
Types: deb
URIs: https://nexus.tkclabs.io:8443/repository/debian-bullseye-proxy/
Suites: bullseye bullseye-updates
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://nexus.tkclabs.io:8443/repository/debian-bullseye-security-proxy/
Suites: bullseye-security
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

- [ ] **Step 3: Fix files/debian/12/debian.sources**

Replace the file's content with:

```
Types: deb
# http://snapshot.debian.org/archive/debian/20251117T000000Z
# URIs: http://deb.debian.org/debian
URIs: https://nexus.tkclabs.io:8443/repository/debian-bookworm-proxy/
Suites: bookworm bookworm-updates
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
# http://snapshot.debian.org/archive/debian-security/20251117T000000Z
# URIs: http://deb.debian.org/debian-security
URIs: https://nexus.tkclabs.io:8443/repository/debian-bookworm-security-proxy/
Suites: bookworm-security
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

- [ ] **Step 4: Fix files/debian/13/debian.sources**

Replace the file's content with:

```
Types: deb deb-src
URIs: https://nexus.tkclabs.io:8443/repository/debian-trixie-proxy/
Suites: trixie trixie-updates trixie-backports
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb deb-src
URIs: https://nexus.tkclabs.io:8443/repository/debian-trixie-security-proxy/
Suites: trixie-security
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

- [ ] **Step 5: Build and verify**

```bash
podman build -t systemd-debian:latest -t systemd-debian:13 -f Containerfile.debian .
podman build -t systemd-debian:12 -f Containerfile.debian --build-arg VERSION=12 .
podman build -t systemd-debian:11 -f Containerfile.debian --build-arg VERSION=11 .
```

Expected: all three builds complete with exit code 0.

- [ ] **Step 6: Commit**

```bash
git add Containerfile.debian files/debian/11/debian.sources files/debian/12/debian.sources files/debian/13/debian.sources
git commit -m "$(cat <<'EOF'
fix: correct Debian Nexus mirror paths, add security repos, switch to baked-in CA

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Add Containerfile.rockylinux

**Files:**
- Create: `Containerfile.rockylinux`
- Create: `files/rockylinux/10/local.repo`

**Interfaces:**
- Consumes: `files/ca/TKC-Labs-Root-CA-2026-2036.crt` (Task 2), `rockylinux-proxy` Nexus repo (Task 1).
- Produces: working `systemd-rockylinux:10` image.

- [ ] **Step 1: Create files/rockylinux/10/local.repo**

```bash
mkdir -p files/rockylinux/10
```

Write `files/rockylinux/10/local.repo`:

```ini
[appstream]
name=Rocky Linux $releasever - AppStream
# mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=AppStream-$releasever
# baseurl=https://download.rockylinux.org/pub/rocky/$releasever/AppStream/$basearch/os/
baseurl=https://nexus.tkclabs.io:8443/repository/rockylinux-proxy/$releasever/AppStream/$basearch/os/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-10
metadata_expire=86400
enabled_metadata=1

[baseos]
name=Rocky Linux $releasever - BaseOS
# mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=BaseOS-$releasever
# baseurl=https://download.rockylinux.org/pub/rocky/$releasever/BaseOS/$basearch/os/
baseurl=https://nexus.tkclabs.io:8443/repository/rockylinux-proxy/$releasever/BaseOS/$basearch/os/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-10
metadata_expire=86400
enabled_metadata=1

[crb]
name=Rocky Linux $releasever - CRB
# mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=CRB-$releasever
# baseurl=https://download.rockylinux.org/pub/rocky/$releasever/CRB/$basearch/os/
baseurl=https://nexus.tkclabs.io:8443/repository/rockylinux-proxy/$releasever/CRB/$basearch/os/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-10
metadata_expire=86400
enabled_metadata=0

[extras]
name=Rocky Linux $releasever - Extras
# mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=extras-$releasever
# baseurl=https://download.rockylinux.org/pub/rocky/$releasever/extras/$basearch/os/
baseurl=https://nexus.tkclabs.io:8443/repository/rockylinux-proxy/$releasever/extras/$basearch/os/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-10
metadata_expire=86400
enabled_metadata=0
```

- [ ] **Step 2: Create Containerfile.rockylinux**

```dockerfile
# systemd-rockylinux
#
# Build the default image with:
#   podman build -t systemd-rockylinux:latest -t systemd-rockylinux:10 -f Containerfile.rockylinux .
#
ARG VERSION=10

FROM rockylinux/rockylinux:${VERSION}

ARG VERSION=10

# Setup our local repo mirrors
COPY files/ca/TKC-Labs-Root-CA-2026-2036.crt /etc/pki/ca-trust/source/anchors/TKC-Labs-Root-CA-2026-2036.crt

RUN rm -f /etc/yum.repos.d/*.repo \
  && update-ca-trust

COPY files/rockylinux/${VERSION}/local.repo /etc/yum.repos.d/local.repo

# Update the image
# hadolint ignore=DL3041
RUN dnf update -y \
  # Install necessary packages
  && dnf install -y git \
  iproute \
  openssh-clients \
  openssh-server \
  procps-ng \
  python3 \
  sudo \
  systemd \
  which \
  # Cleanup the image
  && dnf clean all \
  && rm -rf /var/cache/dnf

# ---------------------------------------------------------------------------
# Mask systemd units that are useless inside a container and are the
# biggest contributors to the open-file explosion:
#
# systemd-udevd          – hardware device manager, no real hardware here
# systemd-journald       – logging daemon, high fd usage; use docker logs instead
# systemd-logind         – session/seat management, irrelevant in containers
# systemd-networkd       – network config; container runtime owns networking
# NetworkManager         – same as above
# auditd                 – kernel audit subsystem, not available in containers
# firewalld              – iptables management handled outside container
# tuned                  – performance tuning daemon, unneeded
# kdump                  – kernel crash dumps, irrelevant
# sssd                   – identity/auth provider, unneeded for testing
# rhsmcertd / insights   – Red Hat subscription/telemetry daemons
# systemd-resolved       – DNS resolver; /etc/resolv.conf comes from runtime
# chronyd / ntpd         – time sync, container inherits host clock
# crond                  – cron, not needed for test targets
# mdmonitor / lvm2-*     – storage daemons, no block devices
# ---------------------------------------------------------------------------
RUN systemctl mask \
  systemd-udevd.service \
  systemd-udevd-kernel.socket \
  systemd-udevd-control.socket \
  systemd-journald.service \
  systemd-journald.socket \
  systemd-journald-dev-log.socket \
  systemd-journald-audit.socket \
  systemd-logind.service \
  systemd-networkd.service \
  systemd-networkd.socket \
  systemd-networkd-wait-online.service \
  systemd-resolved.service \
  NetworkManager.service \
  NetworkManager-wait-online.service \
  auditd.service \
  firewalld.service \
  tuned.service \
  kdump.service \
  sssd.service \
  rhsmcertd.service \
  insights-client.service \
  chronyd.service \
  crond.service \
  mdmonitor.service \
  lvm2-monitor.service \
  dm-event.service \
  multipathd.service \
  systemd-hwdb-update.service \
  sys-kernel-debug.mount \
  kbrequest.target \
  dev-hugepages.mount

# ---------------------------------------------------------------------------
# Enable only what we actually need: sshd
# ---------------------------------------------------------------------------
# hadolint ignore=DL3059
#RUN systemctl enable sshd.service

# ---------------------------------------------------------------------------
# SSH configuration for ephemeral test containers.
# - Allow root login (Ansible/pyinfra often default to root in test labs)
# - Disable slow reverse-DNS lookups (cuts connect time dramatically)
# - Disable PAM session noise that can stall logins without a real tty
# ---------------------------------------------------------------------------
# RUN mkdir -p /etc/ssh/sshd_config.d && \
#     printf '%s\n' \
#         'PermitRootLogin yes' \
#         'PasswordAuthentication yes' \
#         'UseDNS no' \
#         'UsePAM yes' \
#         'PrintMotd no' \
#         'PrintLastLog no' \
#         'GSSAPIAuthentication no' \
#         'X11Forwarding no' \
#         > /etc/ssh/sshd_config.d/99-container.conf

# ---------------------------------------------------------------------------
# Set a known root password (change or replace with key injection).
# For key-based auth: drop your pubkey into /root/.ssh/authorized_keys
# ---------------------------------------------------------------------------
# ARG ROOT_PUBKEY="VALID_KEY"
#   # hadolint ignore=DL4006
# RUN echo 'root:TestPass!' | chpasswd && \
#     mkdir -p /root/.ssh && \
#     echo "${ROOT_PUBKEY}" > /root/.ssh/authorized_keys

# Create a non-root test user for sudo-based testing
# ARG TEST_USER=tkcadmin
# ARG TEST_USER_PUBKEY="VALID_KEY"
# # hadolint ignore=DL4006
# RUN useradd -m -s /bin/bash ${TEST_USER} && \
#     echo "${TEST_USER}:TestPass!" | chpasswd && \
#     echo "${TEST_USER} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${TEST_USER} && \
#     mkdir -p /home/${TEST_USER}/.ssh && \
#     echo "${TEST_USER_PUBKEY}" > /home/${TEST_USER}/.ssh/authorized_keys

# ---------------------------------------------------------------------------
# Ensure SSH host keys are generated at build time so the first
# container start is not slowed by key generation.
# ---------------------------------------------------------------------------
# RUN ssh-keygen -A

# ---------------------------------------------------------------------------
# Journal / logging: redirect to stdout so `podman logs` works even
# without journald. A minimal virtual console journal config keeps
# any remaining systemd log output from eating fds.
# ---------------------------------------------------------------------------
RUN mkdir -p /etc/systemd/journald.conf.d && \
    printf '%s' \
      '[Journal]\n' \
      'Storage=volatile\n' \
      'Compress=no\n' \
      'SplitMode=none\n' \
      'RateLimitIntervalSec=0\n' \
      'RateLimitBurst=0\n' \
      'ForwardToSyslog=no\n' \
      'ForwardToConsole=yes\n' \
      'TTYPath=/dev/console\n' \
      'MaxLevelConsole=warning\n' \
      > /etc/systemd/journald.conf.d/container.conf

WORKDIR /root
#EXPOSE 22
CMD [ "/sbin/init" ]
```

- [ ] **Step 3: Build and verify**

```bash
podman build -t systemd-rockylinux:latest -t systemd-rockylinux:10 -f Containerfile.rockylinux .
```

Expected: build completes with exit code 0. If `dnf install` fails on a package name, check whether that package differs on Rocky (it shouldn't for this list, but verify the error message names the actual missing package).

- [ ] **Step 4: Commit**

```bash
git add Containerfile.rockylinux files/rockylinux/10/local.repo
git commit -m "$(cat <<'EOF'
feat: add systemd-rockylinux base image (RockyLinux 10)

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: Add Containerfile.fedora

**Files:**
- Create: `Containerfile.fedora`
- Create: `files/fedora/44/fedora.repo`

**Interfaces:**
- Consumes: `files/ca/TKC-Labs-Root-CA-2026-2036.crt` (Task 2), `fedora-proxy` Nexus repo (pre-existing).
- Produces: working `systemd-fedora:44` image.

- [ ] **Step 1: Create files/fedora/44/fedora.repo**

```bash
mkdir -p files/fedora/44
```

Write `files/fedora/44/fedora.repo`:

```ini
[fedora]
name=Fedora $releasever - $basearch
# metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-$releasever&arch=$basearch
# baseurl=https://download.fedoraproject.org/pub/fedora/linux/releases/$releasever/Everything/$basearch/os/
baseurl=https://nexus.tkclabs.io:8443/repository/fedora-proxy/releases/$releasever/Everything/$basearch/os/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch
metadata_expire=86400
skip_if_unavailable=False

[updates]
name=Fedora $releasever - $basearch - Updates
# metalink=https://mirrors.fedoraproject.org/metalink?repo=updates-released-f$releasever&arch=$basearch
# baseurl=https://download.fedoraproject.org/pub/fedora/linux/updates/$releasever/Everything/$basearch/
baseurl=https://nexus.tkclabs.io:8443/repository/fedora-proxy/updates/$releasever/Everything/$basearch/
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch
metadata_expire=21600
skip_if_unavailable=False
```

- [ ] **Step 2: Create Containerfile.fedora**

```dockerfile
# systemd-fedora
#
# Build the default image with:
#   podman build -t systemd-fedora:latest -t systemd-fedora:44 -f Containerfile.fedora .
#
ARG VERSION=44

FROM fedora:${VERSION}

ARG VERSION=44

# Setup our local repo mirrors
COPY files/ca/TKC-Labs-Root-CA-2026-2036.crt /etc/pki/ca-trust/source/anchors/TKC-Labs-Root-CA-2026-2036.crt

RUN rm -f /etc/yum.repos.d/*.repo \
  && update-ca-trust

COPY files/fedora/${VERSION}/fedora.repo /etc/yum.repos.d/fedora.repo

# Update the image
# hadolint ignore=DL3041
RUN dnf update -y \
  # Install necessary packages
  && dnf install -y git \
  iproute \
  openssh-clients \
  openssh-server \
  procps-ng \
  python3 \
  sudo \
  systemd \
  which \
  # Cleanup the image
  && dnf clean all \
  && rm -rf /var/cache/dnf

# ---------------------------------------------------------------------------
# Mask systemd units that are useless inside a container and are the
# biggest contributors to the open-file explosion:
#
# systemd-udevd          – hardware device manager, no real hardware here
# systemd-journald       – logging daemon, high fd usage; use docker logs instead
# systemd-logind         – session/seat management, irrelevant in containers
# systemd-networkd       – network config; container runtime owns networking
# NetworkManager         – same as above
# auditd                 – kernel audit subsystem, not available in containers
# firewalld              – iptables management handled outside container
# tuned                  – performance tuning daemon, unneeded
# kdump                  – kernel crash dumps, irrelevant
# sssd                   – identity/auth provider, unneeded for testing
# rhsmcertd / insights   – Red Hat subscription/telemetry daemons
# systemd-resolved       – DNS resolver; /etc/resolv.conf comes from runtime
# chronyd / ntpd         – time sync, container inherits host clock
# crond                  – cron, not needed for test targets
# mdmonitor / lvm2-*     – storage daemons, no block devices
# ---------------------------------------------------------------------------
RUN systemctl mask \
  systemd-udevd.service \
  systemd-udevd-kernel.socket \
  systemd-udevd-control.socket \
  systemd-journald.service \
  systemd-journald.socket \
  systemd-journald-dev-log.socket \
  systemd-journald-audit.socket \
  systemd-logind.service \
  systemd-networkd.service \
  systemd-networkd.socket \
  systemd-networkd-wait-online.service \
  systemd-resolved.service \
  NetworkManager.service \
  NetworkManager-wait-online.service \
  auditd.service \
  firewalld.service \
  tuned.service \
  kdump.service \
  sssd.service \
  rhsmcertd.service \
  insights-client.service \
  chronyd.service \
  crond.service \
  mdmonitor.service \
  lvm2-monitor.service \
  dm-event.service \
  multipathd.service \
  systemd-hwdb-update.service \
  sys-kernel-debug.mount \
  kbrequest.target \
  dev-hugepages.mount

# ---------------------------------------------------------------------------
# Enable only what we actually need: sshd
# ---------------------------------------------------------------------------
# hadolint ignore=DL3059
#RUN systemctl enable sshd.service

# ---------------------------------------------------------------------------
# SSH configuration for ephemeral test containers.
# - Allow root login (Ansible/pyinfra often default to root in test labs)
# - Disable slow reverse-DNS lookups (cuts connect time dramatically)
# - Disable PAM session noise that can stall logins without a real tty
# ---------------------------------------------------------------------------
# RUN mkdir -p /etc/ssh/sshd_config.d && \
#     printf '%s\n' \
#         'PermitRootLogin yes' \
#         'PasswordAuthentication yes' \
#         'UseDNS no' \
#         'UsePAM yes' \
#         'PrintMotd no' \
#         'PrintLastLog no' \
#         'GSSAPIAuthentication no' \
#         'X11Forwarding no' \
#         > /etc/ssh/sshd_config.d/99-container.conf

# ---------------------------------------------------------------------------
# Set a known root password (change or replace with key injection).
# For key-based auth: drop your pubkey into /root/.ssh/authorized_keys
# ---------------------------------------------------------------------------
# ARG ROOT_PUBKEY="VALID_KEY"
#   # hadolint ignore=DL4006
# RUN echo 'root:TestPass!' | chpasswd && \
#     mkdir -p /root/.ssh && \
#     echo "${ROOT_PUBKEY}" > /root/.ssh/authorized_keys

# Create a non-root test user for sudo-based testing
# ARG TEST_USER=tkcadmin
# ARG TEST_USER_PUBKEY="VALID_KEY"
# # hadolint ignore=DL4006
# RUN useradd -m -s /bin/bash ${TEST_USER} && \
#     echo "${TEST_USER}:TestPass!" | chpasswd && \
#     echo "${TEST_USER} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${TEST_USER} && \
#     mkdir -p /home/${TEST_USER}/.ssh && \
#     echo "${TEST_USER_PUBKEY}" > /home/${TEST_USER}/.ssh/authorized_keys

# ---------------------------------------------------------------------------
# Ensure SSH host keys are generated at build time so the first
# container start is not slowed by key generation.
# ---------------------------------------------------------------------------
# RUN ssh-keygen -A

# ---------------------------------------------------------------------------
# Journal / logging: redirect to stdout so `podman logs` works even
# without journald. A minimal virtual console journal config keeps
# any remaining systemd log output from eating fds.
# ---------------------------------------------------------------------------
RUN mkdir -p /etc/systemd/journald.conf.d && \
    printf '%s' \
      '[Journal]\n' \
      'Storage=volatile\n' \
      'Compress=no\n' \
      'SplitMode=none\n' \
      'RateLimitIntervalSec=0\n' \
      'RateLimitBurst=0\n' \
      'ForwardToSyslog=no\n' \
      'ForwardToConsole=yes\n' \
      'TTYPath=/dev/console\n' \
      'MaxLevelConsole=warning\n' \
      > /etc/systemd/journald.conf.d/container.conf

WORKDIR /root
#EXPOSE 22
CMD [ "/sbin/init" ]
```

- [ ] **Step 3: Build and verify**

```bash
podman build -t systemd-fedora:latest -t systemd-fedora:44 -f Containerfile.fedora .
```

Expected: build completes with exit code 0. If gpgcheck fails, verify `RPM-GPG-KEY-fedora-44-x86_64` (or the relevant `$basearch`) actually exists inside the built rootfs at `/etc/pki/rpm-gpg/` — Fedora's key-file naming has shifted between releases, so this is the most likely failure point; adjust the `gpgkey=` line in `files/fedora/44/fedora.repo` to match if needed.

- [ ] **Step 4: Commit**

```bash
git add Containerfile.fedora files/fedora/44/fedora.repo
git commit -m "$(cat <<'EOF'
feat: add systemd-fedora base image (Fedora 44)

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 7: Add Containerfile.ubuntu

**Files:**
- Create: `Containerfile.ubuntu`
- Create: `files/ubuntu/26.04/ubuntu.sources`

**Interfaces:**
- Consumes: `files/ca/TKC-Labs-Root-CA-2026-2036.crt` (Task 2), `ubuntu-resolute-proxy` / `ubuntu-resolute-security-proxy` Nexus repos (Task 1).
- Produces: working `systemd-ubuntu:26.04` image.

- [ ] **Step 1: Create files/ubuntu/26.04/ubuntu.sources**

```bash
mkdir -p files/ubuntu/26.04
```

Write `files/ubuntu/26.04/ubuntu.sources`:

```
Types: deb
URIs: https://nexus.tkclabs.io:8443/repository/ubuntu-resolute-proxy/
Suites: resolute resolute-updates resolute-backports
Components: main
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: https://nexus.tkclabs.io:8443/repository/ubuntu-resolute-security-proxy/
Suites: resolute-security
Components: main
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

- [ ] **Step 2: Create Containerfile.ubuntu**

```dockerfile
# systemd-ubuntu
#
# Build the default image with:
#   podman build -t systemd-ubuntu:latest -t systemd-ubuntu:26.04 -f Containerfile.ubuntu .
#
ARG VERSION=26.04

FROM ubuntu:${VERSION}

ARG VERSION=26.04

# Setup our local repo mirrors
COPY files/ca/TKC-Labs-Root-CA-2026-2036.crt /usr/local/share/ca-certificates/TKC-Labs-Root-CA-2026-2036.crt

RUN update-ca-certificates && \
  rm -rf /etc/apt/sources.list

# Deploy sources file for local cache
COPY files/ubuntu/${VERSION}/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources

# Install necessary packages and clean up the image
RUN DEBIAN_FRONTEND=noninteractive apt-get update && apt-get upgrade -y && \
  apt-get install -y git \
  iproute2 \
  openssh-client \
  openssh-server \
  procps \
  python3 \
  python3-venv \
  sudo \
  systemd \
  vim-tiny && \
  # Clean up the image
  rm -rf /var/lib/apt/lists/* && \
  apt-get clean

# ---------------------------------------------------------------------------
# Mask systemd units that are useless inside a container and are the
# biggest contributors to the open-file explosion:
#
# systemd-udevd          – hardware device manager, no real hardware here
# systemd-journald       – logging daemon, high fd usage; use docker logs instead
# systemd-logind         – session/seat management, irrelevant in containers
# systemd-networkd       – network config; container runtime owns networking
# NetworkManager         – same as above
# auditd                 – kernel audit subsystem, not available in containers
# firewalld              – iptables management handled outside container
# tuned                  – performance tuning daemon, unneeded
# kdump                  – kernel crash dumps, irrelevant
# sssd                   – identity/auth provider, unneeded for testing
# rhsmcertd / insights   – Red Hat subscription/telemetry daemons
# systemd-resolved       – DNS resolver; /etc/resolv.conf comes from runtime
# chronyd / ntpd         – time sync, container inherits host clock
# crond                  – cron, not needed for test targets
# mdmonitor / lvm2-*     – storage daemons, no block devices
# ---------------------------------------------------------------------------
RUN systemctl mask \
  systemd-udevd.service \
  systemd-udevd-kernel.socket \
  systemd-udevd-control.socket \
  systemd-journald.service \
  systemd-journald.socket \
  systemd-journald-dev-log.socket \
  systemd-journald-audit.socket \
  systemd-logind.service \
  systemd-networkd.service \
  systemd-networkd.socket \
  systemd-networkd-wait-online.service \
  systemd-resolved.service \
  NetworkManager.service \
  NetworkManager-wait-online.service \
  auditd.service \
  firewalld.service \
  tuned.service \
  kdump.service \
  sssd.service \
  rhsmcertd.service \
  insights-client.service \
  chronyd.service \
  crond.service \
  mdmonitor.service \
  lvm2-monitor.service \
  dm-event.service \
  multipathd.service \
  systemd-hwdb-update.service \
  sys-kernel-debug.mount \
  kbrequest.target \
  dev-hugepages.mount

# ---------------------------------------------------------------------------
# Enable only what we actually need: sshd
# ---------------------------------------------------------------------------
# hadolint ignore=DL3059
# RUN systemctl enable ssh.service

# ---------------------------------------------------------------------------
# SSH configuration for ephemeral test containers.
# - Allow root login (Ansible/pyinfra often default to root in test labs)
# - Disable slow reverse-DNS lookups (cuts connect time dramatically)
# - Disable PAM session noise that can stall logins without a real tty
# ---------------------------------------------------------------------------
# RUN mkdir -p /etc/ssh/sshd_config.d && \
#     printf '%s\n' \
#         'PermitRootLogin yes' \
#         'PasswordAuthentication yes' \
#         'UseDNS no' \
#         'UsePAM yes' \
#         'PrintMotd no' \
#         'PrintLastLog no' \
#         'GSSAPIAuthentication no' \
#         'X11Forwarding no' \
#         > /etc/ssh/sshd_config.d/99-container.conf

# ---------------------------------------------------------------------------
# Set a known root password (change or replace with key injection).
# For key-based auth: drop your pubkey into /root/.ssh/authorized_keys
# ---------------------------------------------------------------------------
# ARG ROOT_PUBKEY="VALID_KEY"
#   # hadolint ignore=DL4006
# RUN echo 'root:TestPass!' | chpasswd && \
#     mkdir -p /root/.ssh && \
#     echo "${ROOT_PUBKEY}" > /root/.ssh/authorized_keys

# Create a non-root test user for sudo-based testing
# ARG TEST_USER=tkcadmin
# ARG TEST_USER_PUBKEY="VALID_KEY"
# # hadolint ignore=DL4006
# RUN useradd -m -s /bin/bash ${TEST_USER} && \
#     echo "${TEST_USER}:TestPass!" | chpasswd && \
#     echo "${TEST_USER} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${TEST_USER} && \
#     mkdir -p /home/${TEST_USER}/.ssh && \
#     echo "${TEST_USER_PUBKEY}" > /home/${TEST_USER}/.ssh/authorized_keys

# ---------------------------------------------------------------------------
# Ensure SSH host keys are generated at build time so the first
# container start is not slowed by key generation.
# ---------------------------------------------------------------------------
# RUN ssh-keygen -A

# ---------------------------------------------------------------------------
# Journal / logging: redirect to stdout so `podman logs` works even
# without journald. A minimal virtual console journal config keeps
# any remaining systemd log output from eating fds.
# ---------------------------------------------------------------------------
RUN mkdir -p /etc/systemd/journald.conf.d && \
    printf '%s' \
      '[Journal]\n' \
      'Storage=volatile\n' \
      'Compress=no\n' \
      'SplitMode=none\n' \
      'RateLimitIntervalSec=0\n' \
      'RateLimitBurst=0\n' \
      'ForwardToSyslog=no\n' \
      'ForwardToConsole=yes\n' \
      'TTYPath=/dev/console\n' \
      'MaxLevelConsole=warning\n' \
      > /etc/systemd/journald.conf.d/container.conf

WORKDIR /root
# EXPOSE 22
CMD [ "/sbin/init" ]
```

- [ ] **Step 3: Build and verify**

```bash
podman build -t systemd-ubuntu:latest -t systemd-ubuntu:26.04 -f Containerfile.ubuntu .
```

Expected: build completes with exit code 0. If `apt-get install` fails to locate a package, check whether it moved to the `universe` component on Ubuntu — add `Components: main universe` to `files/ubuntu/26.04/ubuntu.sources` if so.

- [ ] **Step 4: Commit**

```bash
git add Containerfile.ubuntu files/ubuntu/26.04/ubuntu.sources
git commit -m "$(cat <<'EOF'
feat: add systemd-ubuntu base image (Ubuntu 26.04 Resolute Raccoon)

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 8: Update README.md

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: nothing (pure documentation).
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Add build sections for the three new images**

Edit `README.md`, replacing:

```markdown
### systemd-debian

```bash
# Build default image (Debian 13):
podman build -t systemd-debian:latest -t systemd-debian:13 -f Containerfile.debian .

# Build specific version image (Debian 11 or 12):
podman build -t systemd-debian:12 -f Containerfile.debian --build-arg VERSION=12 .
podman build -t systemd-debian:11 -f Containerfile.debian --build-arg VERSION=11 .
```
```

with:

```markdown
### systemd-debian

```bash
# Build default image (Debian 13):
podman build -t systemd-debian:latest -t systemd-debian:13 -f Containerfile.debian .

# Build specific version image (Debian 11 or 12):
podman build -t systemd-debian:12 -f Containerfile.debian --build-arg VERSION=12 .
podman build -t systemd-debian:11 -f Containerfile.debian --build-arg VERSION=11 .
```

### systemd-rockylinux

```bash
# Build default image (RockyLinux 10):
podman build -t systemd-rockylinux:latest -t systemd-rockylinux:10 -f Containerfile.rockylinux .
```

### systemd-fedora

```bash
# Build default image (Fedora 44):
podman build -t systemd-fedora:latest -t systemd-fedora:44 -f Containerfile.fedora .
```

### systemd-ubuntu

```bash
# Build default image (Ubuntu 26.04 "Resolute Raccoon"):
podman build -t systemd-ubuntu:latest -t systemd-ubuntu:26.04 -f Containerfile.ubuntu .
```
```

- [ ] **Step 2: Verify the file renders sensibly**

```bash
cat README.md
```

Expected: 5 distro sections total (almalinux, debian, rockylinux, fedora, ubuntu), each with a fenced `bash` build example, no broken markdown fences.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "$(cat <<'EOF'
docs: document build commands for RockyLinux, Fedora, and Ubuntu images

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 9: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: nothing (pure documentation).
- Produces: nothing consumed by later tasks — this is the final task.

- [ ] **Step 1: Update the "What this is" section**

Replace:

```markdown
A pair of Containerfiles that build **systemd-enabled base images** (AlmaLinux and Debian) for use as test targets — e.g. SSH-reachable hosts for Ansible/pyinfra/Salt runs. The images run `/sbin/init` as PID 1 so real systemd units can be exercised inside the container. There is no application code; the "source" is the two Containerfiles plus per-version repo/sources config under `files/`.
```

with:

```markdown
Five Containerfiles that build **systemd-enabled base images** (AlmaLinux, RockyLinux, Fedora, Debian, Ubuntu) for use as test targets — e.g. SSH-reachable hosts for Ansible/pyinfra/Salt runs. The images run `/sbin/init` as PID 1 so real systemd units can be exercised inside the container. There is no application code; the "source" is the five Containerfiles plus per-version repo/sources config under `files/`.

The five split into two families that share structure:
- **RPM/dnf** (`Containerfile.almalinux`, `Containerfile.rockylinux`, `Containerfile.fedora`) — dnf install, `.repo` files under `files/<distro>/<version>/`, CA trusted via `/etc/pki/ca-trust/source/anchors/`.
- **APT** (`Containerfile.debian`, `Containerfile.ubuntu`) — apt install, deb822 `.sources` files under `files/<distro>/<version>/`, CA trusted via `/usr/local/share/ca-certificates/`.
```

- [ ] **Step 2: Update the Build commands section**

Replace:

```markdown
## Build commands

```bash
# AlmaLinux (default VERSION=10; supported: 9, 10)
podman build -t systemd-almalinux:latest -t systemd-almalinux:10 -f Containerfile.almalinux .
podman build -t systemd-almalinux:9 -f Containerfile.almalinux --build-arg VERSION=9 .

# Debian (default VERSION=13; supported: 11, 12, 13)
podman build -t systemd-debian:latest -t systemd-debian:13 -f Containerfile.debian .
podman build -t systemd-debian:12 -f Containerfile.debian --build-arg VERSION=12 .
```

Run a built image (needs an init-capable container):
```bash
podman run -d --name test --systemd=always systemd-almalinux:10
```

Lint (the files carry `hadolint ignore=` directives, so hadolint is the expected linter):
```bash
hadolint Containerfile.almalinux Containerfile.debian
```
```

with:

```markdown
## Build commands

```bash
# AlmaLinux (default VERSION=10; supported: 9, 10)
podman build -t systemd-almalinux:latest -t systemd-almalinux:10 -f Containerfile.almalinux .
podman build -t systemd-almalinux:9 -f Containerfile.almalinux --build-arg VERSION=9 .

# Debian (default VERSION=13; supported: 11, 12, 13)
podman build -t systemd-debian:latest -t systemd-debian:13 -f Containerfile.debian .
podman build -t systemd-debian:12 -f Containerfile.debian --build-arg VERSION=12 .

# RockyLinux (default and only supported VERSION=10)
podman build -t systemd-rockylinux:latest -t systemd-rockylinux:10 -f Containerfile.rockylinux .

# Fedora (default and only supported VERSION=44)
podman build -t systemd-fedora:latest -t systemd-fedora:44 -f Containerfile.fedora .

# Ubuntu (default and only supported VERSION=26.04)
podman build -t systemd-ubuntu:latest -t systemd-ubuntu:26.04 -f Containerfile.ubuntu .
```

Run a built image (needs an init-capable container):
```bash
podman run -d --name test --systemd=always systemd-almalinux:10
```

Lint (the files carry `hadolint ignore=` directives, so hadolint is the expected linter):
```bash
hadolint Containerfile.almalinux Containerfile.rockylinux Containerfile.fedora Containerfile.debian Containerfile.ubuntu
```
```

- [ ] **Step 3: Update the "Each supported version needs a matching files/ entry" bullet**

Replace:

```markdown
- **Each supported version needs a matching `files/` entry.** AlmaLinux maps to `files/almalinux/<VERSION>/local.repo`; Debian maps to `files/debian/<VERSION>/debian.sources`. Adding a new version means adding both the build-arg support and the corresponding config file — the `COPY` will fail otherwise. The `gpgkey=` path in `local.repo` is version-pinned (`RPM-GPG-KEY-AlmaLinux-<VERSION>`), so it must be updated per file.
```

with:

```markdown
- **Each supported version needs a matching `files/` entry.** AlmaLinux/RockyLinux map to `files/<distro>/<VERSION>/local.repo`; Fedora maps to `files/fedora/<VERSION>/fedora.repo`; Debian/Ubuntu map to `files/<distro>/<VERSION>/{debian,ubuntu}.sources`. Adding a new version means adding both the build-arg support and the corresponding config file — the `COPY` will fail otherwise. AlmaLinux and RockyLinux version-pin their `gpgkey=` path (`RPM-GPG-KEY-AlmaLinux-<VERSION>`, `RPM-GPG-KEY-Rocky-<VERSION>`) — update it per file. Fedora instead templates its gpgkey by dnf variable (`RPM-GPG-KEY-fedora-$releasever-$basearch`), matching Fedora's own shipped repo file convention, so no per-version edit is needed there.
```

- [ ] **Step 4: Update the "Internal infrastructure dependency" bullet**

Replace:

```markdown
- **Internal infrastructure dependency.** Builds pull the TKC Labs root CA from `https://vault.tkclabs.io:8200` (with `--insecure`) and install it system-wide, then repoint all package repos at the internal Nexus mirror `https://nexus.tkclabs.io:8443`. **Builds require network access to these hosts and will not work outside the TKC Labs network.** The stock distro repos are commented out in the `files/` configs rather than deleted — uncomment them to build without the mirror.
```

with:

```markdown
- **Internal infrastructure dependency.** The TKC Labs root CA is baked into every image from a static file, `files/ca/TKC-Labs-Root-CA-2026-2036.crt` (not fetched from Vault at build time — Vault currently serves an older, now-invalid root, so builds do not depend on it), then all package repos are repointed at the internal Nexus mirror `https://nexus.tkclabs.io:8443`. **Builds require network access to Nexus and will not work outside the TKC Labs network.** Nexus proxy repos follow a `<name>-proxy` naming convention (e.g. `almalinux-proxy`, `debian-trixie-proxy`), with dedicated `<name>-security-proxy` repos for apt-based distros' security suites — get this suffix wrong and the `COPY`'d `.repo`/`.sources` file will point at a repo that doesn't exist. The stock distro repos are commented out in the `files/` configs rather than deleted — uncomment them to build without the mirror. When the new root CA expires or rotates again, update `files/ca/` and the `COPY` line in all five Containerfiles together.
```

- [ ] **Step 5: Update the "systemd is aggressively trimmed" bullet**

Replace:

```markdown
- **systemd is aggressively trimmed.** Both Containerfiles `systemctl mask` a large, identical list of units (udevd, journald, logind, networkd, NetworkManager, resolved, auditd, firewalld, chronyd, crond, storage daemons, etc.) to avoid the file-descriptor explosion that kills systemd in containers. Keep this list in sync between the two files when changing it.
```

with:

```markdown
- **systemd is aggressively trimmed.** All five Containerfiles `systemctl mask` a large, identical list of units (udevd, journald, logind, networkd, NetworkManager, resolved, auditd, firewalld, chronyd, crond, storage daemons, etc.) to avoid the file-descriptor explosion that kills systemd in containers. Keep this list in sync across all five files when changing it.
```

- [ ] **Step 6: Update the "Keep the two Containerfiles parallel" bullet**

Replace:

```markdown
- **Keep the two Containerfiles parallel.** They differ only where the distro forces it (dnf vs apt, `openssh-server` package names, AppStream/CRB vs Debian components, CA path `/etc/pki/ca-trust/...` vs `/usr/local/share/ca-certificates/...`). Apply structural changes to both.
```

with:

```markdown
- **Keep the five Containerfiles parallel.** Within a family they're near-identical (Alma/Rocky/Fedora only differ in `FROM`, package-manager repo config, and gpgkey path; Debian/Ubuntu only differ in `FROM` and the sources file). Across families they differ only where the distro forces it (dnf vs apt, `openssh-server` package names, AppStream/BaseOS/CRB/extras — absent on Fedora, whose repo layout is just releases/updates — vs Debian/Ubuntu components, CA path `/etc/pki/ca-trust/...` vs `/usr/local/share/ca-certificates/...`). Apply structural changes (masked-units list, journald drop-in, SSH scaffolding) to all five.
```

- [ ] **Step 7: Verify the doc is internally consistent**

```bash
grep -n "pair of Containerfiles\|the two Containerfiles\|both Containerfiles" CLAUDE.md
```

Expected: no matches (all such phrasing has been updated to reflect five files).

- [ ] **Step 8: Commit**

```bash
git add CLAUDE.md
git commit -m "$(cat <<'EOF'
docs: update CLAUDE.md for the five-Containerfile architecture and CA change

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```
