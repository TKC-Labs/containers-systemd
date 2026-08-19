# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Five Containerfiles that build **systemd-enabled base images** (AlmaLinux, RockyLinux, Fedora, Debian, Ubuntu) for use as test targets — e.g. SSH-reachable hosts for Ansible/pyinfra/Salt runs. The images run `/sbin/init` as PID 1 so real systemd units can be exercised inside the container. There is no application code; the "source" is the five Containerfiles plus per-version repo/sources config under `files/`.

The five split into two families that share structure:
- **RPM/dnf** (`Containerfile.almalinux`, `Containerfile.rockylinux`, `Containerfile.fedora`) — dnf install, `.repo` files under `files/<distro>/<version>/`, CA trusted via `/etc/pki/ca-trust/source/anchors/`.
- **APT** (`Containerfile.debian`, `Containerfile.ubuntu`) — apt install, deb822 `.sources` files under `files/<distro>/<version>/`, CA trusted via `/usr/local/share/ca-certificates/`.

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

## Architecture & conventions

- **`VERSION` build-arg is declared twice on purpose.** A first `ARG VERSION` before `FROM` selects the base image tag; a second `ARG VERSION` after `FROM` re-introduces it into the build stage so the `COPY files/<distro>/${VERSION}/...` line resolves. Editing one without the other breaks versioned builds.

- **Each supported version needs a matching `files/` entry.** AlmaLinux/RockyLinux map to `files/<distro>/<VERSION>/local.repo`; Fedora maps to `files/fedora/<VERSION>/fedora.repo`; Debian/Ubuntu map to `files/<distro>/<VERSION>/{debian,ubuntu}.sources`. Adding a new version means adding both the build-arg support and the corresponding config file — the `COPY` will fail otherwise. AlmaLinux and RockyLinux version-pin their `gpgkey=` path (`RPM-GPG-KEY-AlmaLinux-<VERSION>`, `RPM-GPG-KEY-Rocky-<VERSION>`) — update it per file. Fedora instead templates its gpgkey by dnf variable (`RPM-GPG-KEY-fedora-$releasever-$basearch`), matching Fedora's own shipped repo file convention, so no per-version edit is needed there.

- **Internal infrastructure dependency.** The TKC Labs root CA is baked into every image from a static file, `files/ca/TKC-Labs-Root-CA-2026-2036.crt` (not fetched from Vault at build time — Vault currently serves an older, now-invalid root, so builds do not depend on it), then all package repos are repointed at the internal Nexus mirror `https://nexus.tkclabs.io:8443`. **Builds require network access to Nexus and will not work outside the TKC Labs network.** Nexus proxy repos follow a `<name>-proxy` naming convention (e.g. `almalinux-proxy`, `debian-trixie-proxy`), with dedicated `<name>-security-proxy` repos for apt-based distros' security suites — get this suffix wrong and the `COPY`'d `.repo`/`.sources` file will point at a repo that doesn't exist. The stock distro repos are commented out in the `files/` configs rather than deleted — uncomment them to build without the mirror. When the new root CA expires or rotates again, update `files/ca/` and the `COPY` line in all five Containerfiles together.

- **systemd is aggressively trimmed.** All five Containerfiles `systemctl mask` a large, identical list of units (udevd, journald, logind, networkd, NetworkManager, resolved, auditd, firewalld, chronyd, crond, storage daemons, etc.) to avoid the file-descriptor explosion that kills systemd in containers. Keep this list in sync across all five files when changing it.

- **Logging goes to the console, not journald.** journald is masked, so a `journald.conf.d/container.conf` drop-in forwards remaining systemd output to `/dev/console` (`ForwardToConsole=yes`) so `podman logs` works. This is intentional given journald is disabled.

- **sshd, root password, and the test user are scaffolded but commented out.** The blocks for enabling sshd, setting `PermitRootLogin`, injecting `ROOT_PUBKEY`/`TEST_USER_PUBKEY`, and `ssh-keygen -A` are present but disabled. Uncomment and supply build-args (`ROOT_PUBKEY`, `TEST_USER`, `TEST_USER_PUBKEY`) to produce an SSH-reachable test target. The image does not expose SSH by default.

- **Keep the five Containerfiles parallel.** Within a family they're near-identical (Alma/Rocky/Fedora only differ in `FROM`, package-manager repo config, and gpgkey path; Debian/Ubuntu only differ in `FROM` and the sources file). Across families they differ only where the distro forces it (dnf vs apt, `openssh-server` package names, AppStream/BaseOS/CRB/extras — absent on Fedora, whose repo layout is just releases/updates — vs Debian/Ubuntu components, CA path `/etc/pki/ca-trust/...` vs `/usr/local/share/ca-certificates/...`). Apply structural changes (masked-units list, journald drop-in, SSH scaffolding) to all five.
