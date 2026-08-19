# Add RockyLinux, Ubuntu, and Fedora systemd base images

Date: 2026-08-19

## Context

This repo builds systemd-enabled base container images (currently AlmaLinux
and Debian) used as test targets for Ansible/pyinfra/Salt runs. Each distro
gets its own `Containerfile.<distro>`, parameterized by a `VERSION` build-arg,
with per-version repo/mirror config under `files/<distro>/<version>/`.

Goal: add RockyLinux 10, Ubuntu 26.04, and Fedora 44 as additional distro
options, following the existing one-Containerfile-per-distro convention.
AlmaLinux and Debian's defaults (10 and 13 respectively) already match the
target state and need no change.

While investigating the Nexus mirror paths needed for the new distros, we
discovered the *existing* `files/almalinux/*/local.repo` and
`files/debian/*/debian.sources` files reference Nexus repository names that
don't actually exist (missing a `-proxy` suffix, and no `debian-security-*`
proxy exists at all for any Debian version). This is a pre-existing bug —
current builds cannot actually reach these mirrors. Fixing it is in scope for
this work since it establishes the correct pattern the new distro files also
need to follow.

There is no `github.com` URL for this repo anywhere in tracked files — only a
`github` git remote in `.git/config` (unrelated to documentation, and not
touched here since it's git configuration, not a documentation reference).

Separately, while verifying Nexus we found the TKC Labs PKI has already
rotated: `nexus.tkclabs.io`'s live TLS cert now chains through
`TKC Labs Intermediate CA 2026-2031` up to `TKC Labs Root CA 2026-2036`
(issued 2026-08-16), but `vault.tkclabs.io:8200/v1/root_ca/ca/pem` — the
endpoint every existing Containerfile currently fetches its trusted root
from at build time — still serves the old `TKC Labs - Root CA`
(valid Feb 2026–Feb 2027). As committed, **every build (including the
existing AlmaLinux and Debian images) currently fails TLS validation against
Nexus.** This is fixed as part of this work — see "Root CA bootstrap change"
below.

## Root CA bootstrap change

Vault has not been updated with the new root CA yet, so the fix does not
depend on it. The new root CA content is available locally at
`/etc/pki/ca-trust/source/anchors/TKC-Labs-Root-CA-2026-2036.crt` on this
host; it will be copied into the repo as `files/ca/TKC-Labs-Root-CA-2026-2036.crt`
and baked into every image at build time via `COPY`, replacing the
`curl --insecure https://vault.tkclabs.io:8200/...` step entirely in **all
five** Containerfiles (the two existing ones and the three new ones) — the
old CA is expiring soon regardless (Feb 2027) and Nexus's current chain only
validates against the new root. Trusting just the root is sufficient: Nexus
sends the intermediate cert during the TLS handshake, so the client only
needs the root pinned locally.

- RPM family (almalinux, rockylinux, fedora):
  `COPY files/ca/TKC-Labs-Root-CA-2026-2036.crt /etc/pki/ca-trust/source/anchors/` +
  `update-ca-trust`
- APT family (debian, ubuntu):
  `COPY files/ca/TKC-Labs-Root-CA-2026-2036.crt /usr/local/share/ca-certificates/` +
  `update-ca-certificates`

This removes the `apt-get install curl` step from `Containerfile.debian` /
`Containerfile.ubuntu` too, since curl was only needed to fetch the CA from
vault.

## Nexus infrastructure changes

Using the RW credentials in `.envrc`, create the following proxy repositories
in Nexus (`https://nexus.tkclabs.io:8443`), matching the existing `<name>-proxy`
naming convention:

| Repo | Format | Remote URL |
|---|---|---|
| `debian-bullseye-proxy` | apt (proxy) | `http://deb.debian.org/debian/` |
| `debian-bullseye-security-proxy` | apt (proxy) | `http://deb.debian.org/debian-security/` |
| `debian-bookworm-security-proxy` | apt (proxy) | `http://deb.debian.org/debian-security/` |
| `debian-trixie-security-proxy` | apt (proxy) | `http://deb.debian.org/debian-security/` |
| `rockylinux-proxy` | yum (proxy) | `https://download.rockylinux.org/pub/rocky/` |
| `ubuntu-resolute-proxy` | apt (proxy) | `http://archive.ubuntu.com/ubuntu/` |
| `ubuntu-resolute-security-proxy` | apt (proxy) | `http://security.ubuntu.com/ubuntu/` |

Reused without changes: `almalinux-proxy`, `debian-bookworm-proxy`,
`debian-trixie-proxy`, `fedora-proxy` (Fedora's `releases/` and `updates/`
paths both live under the one proxied root `pub/fedora/linux/`, so a single
proxy repo covers both).

## Fixing existing broken paths

Update `baseurl`/`URIs` in the existing files to point at the real proxy
names, and split off security suites to their new dedicated proxies:

- `files/almalinux/9/local.repo`, `files/almalinux/10/local.repo`:
  `.../repository/almalinux/...` → `.../repository/almalinux-proxy/...`
- `files/debian/11/debian.sources`:
  `.../repository/debian-bullseye` → `.../repository/debian-bullseye-proxy/`;
  `.../repository/debian-security-bullseye/` → `.../repository/debian-bullseye-security-proxy/`
- `files/debian/12/debian.sources`:
  `.../repository/debian-bookworm/` → `.../repository/debian-bookworm-proxy/`;
  `.../repository/debian-security-bookworm/` → `.../repository/debian-bookworm-security-proxy/`
- `files/debian/13/debian.sources`:
  `.../repository/debian-trixie/` → `.../repository/debian-trixie-proxy/`;
  `.../repository/debian-security-trixie/` → `.../repository/debian-trixie-security-proxy/`

## New distro files

Each new distro gets a `Containerfile.<distro>` structurally identical to the
closest existing family member (same static-CA bootstrap per the "Root CA
bootstrap change" above, same package install step, same masked-systemd-units
list, same journald-to-console drop-in, same commented-out
sshd/root-password/test-user scaffolding), differing only where the distro
forces it. Each supports a single version (matching the scope of
the request), still via the double `ARG VERSION` pattern for consistency with
the existing files.

### RockyLinux 10 (RPM/dnf family, modeled on Containerfile.almalinux)

- `Containerfile.rockylinux`: `ARG VERSION=10`, `FROM rockylinux/rockylinux:${VERSION}`
  (Rocky's official Docker Hub image is under the `rockylinux/rockylinux`
  namespace, not bare `rockylinux`, which has no `10` tag). Same dnf package
  list as almalinux (git, iproute, openssh-clients, openssh-server,
  procps-ng, python3, sudo, systemd, which).
- `files/rockylinux/10/local.repo`: same 4 sections as Alma's (baseos,
  appstream, crb, extras), `baseurl` under `rockylinux-proxy`,
  `gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-10` (Rocky version-pins
  this key file, same convention as AlmaLinux's `RPM-GPG-KEY-AlmaLinux-<N>`).

### Fedora 44 (RPM/dnf family, modeled on Containerfile.almalinux)

- `Containerfile.fedora`: `ARG VERSION=44`, `FROM fedora:${VERSION}`. Same
  dnf package list as almalinux/rockylinux.
- `files/fedora/44/fedora.repo`: 2 sections (fedora, fedora-updates) instead
  of Alma/Rocky's 4 — Fedora's repo layout doesn't have AppStream/BaseOS/CRB.
  `baseurl` under `fedora-proxy` with `releases/$releasever/Everything/$basearch/os/`
  and `updates/$releasever/Everything/$basearch/`.
  `gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch` —
  note this is templated by dnf variable rather than hardcoded like Alma/Rocky,
  matching Fedora's own shipped repo file convention.

### Ubuntu 26.04 "Resolute Raccoon" (apt family, modeled on Containerfile.debian)

- `Containerfile.ubuntu`: `ARG VERSION=26.04`, `FROM ubuntu:${VERSION}`. Same
  apt package list as debian (git, iproute2, openssh-client, openssh-server,
  procps, python3, python3-venv, sudo, systemd, vim-tiny) — all present on
  Ubuntu unchanged.
- `files/ubuntu/26.04/ubuntu.sources`: deb822 format like Debian's, two
  stanzas — main (`Suites: resolute resolute-updates resolute-backports`,
  `URIs: .../ubuntu-resolute-proxy/`) and security (`Suites: resolute-security`,
  `URIs: .../ubuntu-resolute-security-proxy/`).
  `Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg` — Ubuntu ships
  its own keyring package/path, distinct from Debian's
  `debian-archive-keyring.gpg`.

## Documentation updates

- **README.md**: add a build-command section for each new image (default-tag
  build example only, since each has a single supported version), matching
  existing style.
- **CLAUDE.md**: update "a pair of Containerfiles" language to describe five
  files in two families (RPM/dnf: almalinux, rockylinux, fedora; APT: debian,
  ubuntu); extend the "keep the masked-units list in sync" and "keep
  Containerfiles parallel" notes to cover all five; document the corrected
  Nexus proxy-naming convention (`<name>-proxy`, with dedicated
  `*-security-proxy` repos for apt-based distros) since this was previously
  undocumented and the existing files got it wrong; note Fedora's gpgkey path
  is variable-templated rather than version-pinned like Alma/Rocky; replace
  the "Internal infrastructure dependency" bullet's description of the
  vault-fetched CA with the new static-CA-bake-in approach (`files/ca/`,
  no more `vault.tkclabs.io` build-time dependency, no more `--insecure`
  curl of a root CA).

## Out of scope

- The `github` git remote in `.git/config` — not documentation, not touched
  without an explicit separate request.
- Enabling sshd / root-password / test-user provisioning — stays
  commented-out scaffolding in the new Containerfiles, same as the existing
  two.
- Any Nexus repos/versions beyond the ones listed above (e.g. no
  `ubuntu-jammy-security-proxy`, no additional Rocky/Fedora versions).
