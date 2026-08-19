# Salt systemd containers — design

Date: 2026-08-19

## Goal

Add two systemd-enabled Salt test images built on the existing AlmaLinux 10 base,
default Salt `3006.27` with `3006.6` as a second supported version, installed from
the TKC Labs Nexus mirror. Publish this repo's images to the Nexus private docker
registry so the derived images have a resolvable base on any host.

- **salt-controller** — `salt`, `salt-master`, `salt-minion`, `salt-syndic`, `salt-api`;
  master keypair baked in; `auto_accept` enabled.
- **salt-minion** — `salt`, `salt-minion`.

Both run `salt-minion.service`: the target topology is tiered (tier-1 controllers
manage the minions running on tier-2 controllers; tier-2 controllers manage the
fleet), so a controller is itself a minion of an upstream controller.

## Non-goals

- No Salt topology configuration. Minion config is left at package defaults
  (`master: salt`, minion id from hostname). Tiering is configured at time of use
  or in a thin config layer on top of these images.
- No `salt-api` or `salt-syndic` enablement — both fail-loop without config that
  depends on the deployed topology.
- No version-locking plugin. The images are immutable; nothing upgrades in place.

## New files

| Path | Purpose |
|---|---|
| `Containerfile.salt-controller` | master / syndic / api image |
| `Containerfile.salt-minion` | minion image |
| `files/salt/salt.repo` | dnf repo pointing at `saltproject-rpm-proxy` |
| `files/salt/SALT-PROJECT-GPG-PUBKEY.pgp` | vendored Salt Project signing key |

### Why the GPG key is vendored

Nexus proxies `https://packages.broadcom.com/artifactory/saltproject-rpm/`. The
signing key lives at `https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public`,
outside that path, so it is not reachable through the mirror. It is committed as a
static file and `COPY`'d in, exactly like `files/ca/TKC-Labs-Root-CA-2026-2036.crt`.

Key: `Salt Project Packaging <saltproject-packaging@vmware.com>`,
fingerprint `10857FFDD3F91EAE577A21D664CBBC8173D76B3F`, RSA 3072.

### Why there is no `files/salt/<version>/` split

Unlike the distro repos, the Broadcom Salt repo is flat and version-agnostic — one
`repodata/` indexing every release. Version pinning happens at `dnf install` time
via `salt-master-${SALT_VERSION}`, not in the repo file, so a single `salt.repo`
serves every supported Salt version.

## Build args

```dockerfile
ARG BASE_REGISTRY=nexus.tkclabs.io:8443/
ARG VERSION=10
FROM ${BASE_REGISTRY}systemd-almalinux:${VERSION}
ARG SALT_VERSION=3006.27
```

`BASE_REGISTRY` defaults to Nexus so a clean host can build without first building
the base locally. Pass `--build-arg BASE_REGISTRY=` (empty) to build against a
local `systemd-almalinux:${VERSION}`.

`VERSION` selects the AlmaLinux base tag, following the existing convention.
`SALT_VERSION` is declared after `FROM` because it is only consumed by `RUN`.

Supported `SALT_VERSION`: `3006.27` (default), `3006.6`. Both verified present in
`saltproject-rpm-proxy` for x86_64 and aarch64 across all five packages.

## Inherited from the base image

`systemd-almalinux` already provides the TKC Labs root CA, the Nexus AlmaLinux
repo config, the masked-units list, and the journald→console drop-in. The Salt
Containerfiles do **not** repeat any of it. This is the first derived-image family
in the repo; the "keep the Containerfiles parallel" rule applies within the Salt
pair, not between the Salt pair and the five base images.

## salt-controller

1. `COPY` `salt.repo` and the signing key; `rpm --import` the key.
2. `dnf install -y salt-${SALT_VERSION} salt-master-${SALT_VERSION} salt-minion-${SALT_VERSION} salt-syndic-${SALT_VERSION} salt-api-${SALT_VERSION}`, then `dnf clean all` and `rm -rf /var/cache/dnf`.
3. `salt-key --gen-keys=master --gen-keys-dir=/etc/salt/pki/master`; `chmod 0400 master.pem`, `0644 master.pub`.
4. `/etc/salt/master.d/auto-accept.conf` → `auto_accept: True`.
5. `systemctl enable salt-master.service salt-minion.service`.
6. `EXPOSE 4505 4506`.

The master keypair is generated per build rather than committed, so no private key
enters git. Each build gets its own master identity; that is acceptable because the
containers are ephemeral test targets.

`auto_accept` is deliberate and test-only: it makes any minion that connects get
its key accepted without operator action. These images are never intended for a
production master.

## salt-minion

1. Same repo + key setup.
2. `dnf install -y salt-${SALT_VERSION} salt-minion-${SALT_VERSION}`, same cleanup.
3. `systemctl enable salt-minion.service`.

No configuration. With the package default `master: salt`, running the controller
with `--network-alias salt` is enough for a minimal two-container test.

## Build commands

```bash
# salt-controller (default SALT_VERSION=3006.27; supported: 3006.27, 3006.6)
podman build -t salt-controller:latest -t salt-controller:v3006.27 -f Containerfile.salt-controller .
podman build -t salt-controller:v3006.6 -f Containerfile.salt-controller --build-arg SALT_VERSION=3006.6 .

# salt-minion (default SALT_VERSION=3006.27; supported: 3006.27, 3006.6)
podman build -t salt-minion:latest -t salt-minion:v3006.27 -f Containerfile.salt-minion .
podman build -t salt-minion:v3006.6 -f Containerfile.salt-minion --build-arg SALT_VERSION=3006.6 .
```

Note the tags carry a `v` prefix (`v3006.27`) while the build arg does not
(`3006.27`), matching the package version string.

## Publishing to Nexus

Registry: `nexus.tkclabs.io:8443` (Nexus `docker-hosted`, docker connector 8444
behind TLS on 8443, `forceBasicAuth`). Credentials come from `.envrc`
(`NEXUS_USER_RW` / `NEXUS_PASS_RW`).

```bash
podman login nexus.tkclabs.io:8443 -u "$NEXUS_USER_RW" -p "$NEXUS_PASS_RW"
podman tag  salt-controller:v3006.27 nexus.tkclabs.io:8443/salt-controller:v3006.27
podman push nexus.tkclabs.io:8443/salt-controller:v3006.27
```

Images and tags to publish:

| Image | Tags |
|---|---|
| `systemd-almalinux` | `latest`, `10` |
| `systemd-rockylinux` | `latest`, `10` |
| `systemd-fedora` | `latest`, `44` |
| `systemd-debian` | `latest`, `13` |
| `systemd-ubuntu` | `latest`, `26.04` |
| `salt-controller` | `latest`, `v3006.27`, `v3006.6` |
| `salt-minion` | `latest`, `v3006.27`, `v3006.6` |

`latest` tracks the default version for every image; for the Salt pair that is
`v3006.27`. Non-default distro versions (`almalinux:9`, `debian:11`, `debian:12`)
are out of scope for this pass.

## Documentation updates

- `README.md` — build sections for both Salt images; a publishing section with the
  login/tag/push commands.
- `CLAUDE.md` — seven Containerfiles, not five; the derived-image family and what
  it inherits; `saltproject-rpm-proxy` and the flat-repo pinning model; the
  vendored Salt key and when it must be rotated; `auto_accept` as an intentional
  test-only choice; the Nexus docker registry and the `BASE_REGISTRY` escape hatch.

## Verification

1. `hadolint Containerfile.salt-controller Containerfile.salt-minion` clean.
2. All four Salt images build.
3. `podman run -d --systemd=always` the controller; confirm `salt-master.service`
   and `salt-minion.service` are active and `/etc/salt/pki/master/master.{pem,pub}`
   exist with the right modes.
4. Run a controller with `--network-alias salt` plus a minion on the same network;
   confirm the minion key is auto-accepted and `salt '*' test.ping` answers.
5. Repeat 2–4 for `SALT_VERSION=3006.6`; confirm `salt --version` reports it.
6. Confirm every published tag is listed by the Nexus `/v2/_catalog` and
   `/v2/<name>/tags/list` endpoints.
