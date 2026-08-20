# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

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

# salt-controller (default SALT_VERSION=3006.27; supported: 3006.27, 3006.6)
podman build -t salt-controller:latest -t salt-controller:v3006.27 -f Containerfile.salt-controller .
podman build -t salt-controller:v3006.6 -f Containerfile.salt-controller --build-arg SALT_VERSION=3006.6 .

# salt-minion (default SALT_VERSION=3006.27; supported: 3006.27, 3006.6)
podman build -t salt-minion:latest -t salt-minion:v3006.27 -f Containerfile.salt-minion .
podman build -t salt-minion:v3006.6 -f Containerfile.salt-minion --build-arg SALT_VERSION=3006.6 .
```

Run a built image (needs an init-capable container):
```bash
podman run -d --name test --systemd=always localhost/systemd-almalinux:10
```

Lint (the files carry `hadolint ignore=` directives, so hadolint is the expected linter):
```bash
hadolint Containerfile.almalinux Containerfile.rockylinux Containerfile.fedora \
  Containerfile.debian Containerfile.ubuntu \
  Containerfile.salt-controller Containerfile.salt-minion
```

## Architecture & conventions

- **`VERSION` build-arg is declared twice on purpose.** A first `ARG VERSION` before `FROM` selects the base image tag; a second `ARG VERSION` after `FROM` re-introduces it into the build stage so the `COPY files/<distro>/${VERSION}/...` line resolves. Editing one without the other breaks versioned builds.

- **Each supported version needs a matching `files/` entry.** AlmaLinux/RockyLinux map to `files/<distro>/<VERSION>/local.repo`; Fedora maps to `files/fedora/<VERSION>/fedora.repo`; Debian/Ubuntu map to `files/<distro>/<VERSION>/{debian,ubuntu}.sources`. Adding a new version means adding both the build-arg support and the corresponding config file — the `COPY` will fail otherwise. AlmaLinux and RockyLinux version-pin their `gpgkey=` path (`RPM-GPG-KEY-AlmaLinux-<VERSION>`, `RPM-GPG-KEY-Rocky-<VERSION>`) — update it per file. Fedora instead templates its gpgkey by dnf variable (`RPM-GPG-KEY-fedora-$releasever-$basearch`), matching Fedora's own shipped repo file convention, so no per-version edit is needed there.

- **Internal infrastructure dependency.** The TKC Labs root CA is baked into every image from a static file, `files/ca/TKC-Labs-Root-CA-2026-2036.crt` (not fetched from Vault at build time — Vault currently serves an older, now-invalid root, so builds do not depend on it), then all package repos are repointed at the internal Nexus mirror `https://nexus.tkclabs.io:8443`. **Builds require network access to Nexus and will not work outside the TKC Labs network.** Nexus proxy repos follow a `<name>-proxy` naming convention (e.g. `almalinux-proxy`, `debian-trixie-proxy`), with dedicated `<name>-security-proxy` repos for apt-based distros' security suites — get this suffix wrong and the `COPY`'d `.repo`/`.sources` file will point at a repo that doesn't exist. The stock distro repos are commented out in the `files/` configs rather than deleted — uncomment them to build without the mirror. When the new root CA expires or rotates again, update `files/ca/` and the `COPY` line in all five Containerfiles together. The RPM family (AlmaLinux, RockyLinux, Fedora) is Nexus-only from the first layer. The APT family (Debian, Ubuntu) additionally needs one-shot public-archive access for the early `apt-get install -y ca-certificates` step, which runs before the stock sources are removed and the Nexus `.sources` file is copied in — after that point they're Nexus-only too.

- **systemd is aggressively trimmed.** All five Containerfiles `systemctl mask` a large, identical list of units (udevd, journald, logind, networkd, NetworkManager, resolved, auditd, firewalld, chronyd, crond, storage daemons, etc.) to avoid the file-descriptor explosion that kills systemd in containers. Keep this list in sync across all five files when changing it.

- **Logging goes to the console, not journald.** journald is masked, so a `journald.conf.d/container.conf` drop-in forwards remaining systemd output to `/dev/console` (`ForwardToConsole=yes`) so `podman logs` works. This is intentional given journald is disabled.

- **sshd, root password, and the test user are scaffolded but commented out.** The blocks for enabling sshd, setting `PermitRootLogin`, injecting `ROOT_PUBKEY`/`TEST_USER_PUBKEY`, and `ssh-keygen -A` are present but disabled. Uncomment and supply build-args (`ROOT_PUBKEY`, `TEST_USER`, `TEST_USER_PUBKEY`) to produce an SSH-reachable test target. The image does not expose SSH by default.

- **Keep the five Containerfiles parallel.** Within a family they're near-identical (Alma/Rocky/Fedora only differ in `FROM`, package-manager repo config, and gpgkey path — plus Fedora's `dnf` invocations also carry `--setopt=zchunk=False`, a required, non-optional workaround for the `fedora-proxy` Nexus repo not proxying Fedora's `.zck` metadata variant, not something a later parallelism cleanup should strip; Debian/Ubuntu only differ in `FROM` and the sources file). Across families they differ only where the distro forces it (dnf vs apt, `openssh-server` package names, AppStream/BaseOS/CRB/extras — absent on Fedora, whose repo layout is just releases/updates — vs Debian/Ubuntu components, CA path `/etc/pki/ca-trust/...` vs `/usr/local/share/ca-certificates/...`). Apply structural changes (masked-units list, journald drop-in, SSH scaffolding) to all five.

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
  keypair is generated at build time rather than committed, so no private key
  lands in git — but that is not the same as the key being secret. The generated
  `master.pem` is baked into an image layer and ships inside every copy of the
  published `salt-controller` image on Nexus: anyone with pull access to
  `nexus.tkclabs.io:8443/salt-controller` holds the master's private key and can
  stand up an impostor master that any minion which has already trusted that tag
  will accept silently. Treat the published image itself as holding a live
  credential, not just a test fixture. Nor does every build get a fresh identity:
  `salt-key --gen-keys` sits behind a fixed `RUN` line, so a rebuild that hits
  podman's layer cache (no change to anything above it in the Containerfile)
  reuses the cached layer and produces the *same* `master.pem` as before. A new
  master identity requires a cache miss on that layer — e.g. `podman build
  --no-cache` — not merely re-running the build command. Never point one of these
  controllers at anything whose compromise would matter.

- **The controller runs a minion too.** The target topology is tiered — tier-1
  controllers manage the minions running on tier-2 controllers, which manage the
  fleet — so a controller is itself a minion of an upstream controller. Minion
  config in both images is left at package defaults on purpose (`master: salt`,
  minion id from hostname); do not add `/etc/salt/minion.d/` drop-ins. Topology is
  configured at time of use or in a thin layer on top.

- **Images publish to the Nexus docker registry** at `nexus.tkclabs.io:8443`
  (Nexus `docker-hosted`, docker connector 8444 behind TLS on 8443,
  `forceBasicAuth`). Credentials are `NEXUS_USER_RW` / `NEXUS_PASS_RW` from
  `.envrc`, which is gitignored and must be created locally — it is not
  committed and there is no template for it in the repo. Never pass
  `NEXUS_PASS_RW` via `podman login -p` on the command line; it lands in shell
  history and `/proc/<pid>/cmdline`. Use `podman login --password-stdin`
  instead. Podman on the lab hosts runs rootless with
  `short-name-mode = enforcing`, so every image reference must be fully
  qualified — `localhost/...` locally, `nexus.tkclabs.io:8443/...` once published.
