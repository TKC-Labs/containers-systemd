# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A pair of Containerfiles that build **systemd-enabled base images** (AlmaLinux and Debian) for use as test targets — e.g. SSH-reachable hosts for Ansible/pyinfra/Salt runs. The images run `/sbin/init` as PID 1 so real systemd units can be exercised inside the container. There is no application code; the "source" is the two Containerfiles plus per-version repo/sources config under `files/`.

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

## Architecture & conventions

- **`VERSION` build-arg is declared twice on purpose.** A first `ARG VERSION` before `FROM` selects the base image tag; a second `ARG VERSION` after `FROM` re-introduces it into the build stage so the `COPY files/<distro>/${VERSION}/...` line resolves. Editing one without the other breaks versioned builds.

- **Each supported version needs a matching `files/` entry.** AlmaLinux maps to `files/almalinux/<VERSION>/local.repo`; Debian maps to `files/debian/<VERSION>/debian.sources`. Adding a new version means adding both the build-arg support and the corresponding config file — the `COPY` will fail otherwise. The `gpgkey=` path in `local.repo` is version-pinned (`RPM-GPG-KEY-AlmaLinux-<VERSION>`), so it must be updated per file.

- **Internal infrastructure dependency.** Builds pull the TKC Labs root CA from `https://vault.tkclabs.io:8200` (with `--insecure`) and install it system-wide, then repoint all package repos at the internal Nexus mirror `https://nexus.tkclabs.io:8443`. **Builds require network access to these hosts and will not work outside the TKC Labs network.** The stock distro repos are commented out in the `files/` configs rather than deleted — uncomment them to build without the mirror.

- **systemd is aggressively trimmed.** Both Containerfiles `systemctl mask` a large, identical list of units (udevd, journald, logind, networkd, NetworkManager, resolved, auditd, firewalld, chronyd, crond, storage daemons, etc.) to avoid the file-descriptor explosion that kills systemd in containers. Keep this list in sync between the two files when changing it.

- **Logging goes to the console, not journald.** journald is masked, so a `journald.conf.d/container.conf` drop-in forwards remaining systemd output to `/dev/console` (`ForwardToConsole=yes`) so `podman logs` works. This is intentional given journald is disabled.

- **sshd, root password, and the test user are scaffolded but commented out.** The blocks for enabling sshd, setting `PermitRootLogin`, injecting `ROOT_PUBKEY`/`TEST_USER_PUBKEY`, and `ssh-keygen -A` are present but disabled. Uncomment and supply build-args (`ROOT_PUBKEY`, `TEST_USER`, `TEST_USER_PUBKEY`) to produce an SSH-reachable test target. The image does not expose SSH by default.

- **Keep the two Containerfiles parallel.** They differ only where the distro forces it (dnf vs apt, `openssh-server` package names, AppStream/CRB vs Debian components, CA path `/etc/pki/ca-trust/...` vs `/usr/local/share/ca-certificates/...`). Apply structural changes to both.
