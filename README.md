# Systemd containerfiles

Sytemd containers for testing

## Building Images

### systemd-almalinux

```bash
# Build default image (AlmaLinux 10):
podman build -t systemd-almalinux:latest -t systemd-almalinux:10 -f Containerfile.almalinux .

# Build specific version image (AlmaLinux 9)
podman build -t systemd-almalinux:9 -f Containerfile.almalinux --build-arg VERSION=9 .
```

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

### salt-controller

AlmaLinux 10 systemd container running `salt-master` and `salt-minion`, with a
master keypair baked in and `auto_accept` enabled. `salt-syndic` and `salt-api`
are installed but not enabled. Auto-accept is test-only — never use this as a
production master. The baked master private key ships inside the published
image: anyone with Nexus pull access to `salt-controller` can extract it and
impersonate the master, and rebuilding does not rotate it unless the key-gen
layer is actually cache-missed (`podman build --no-cache`). Never point this
image at anything whose compromise matters.

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

Pass `--hostname` as well as `--name`. Minion config is at package defaults, so a
minion derives its id from the container's hostname — and podman does *not* set
the hostname from `--name`, it defaults to the container ID. Without `--hostname`
your minions show up in `salt-key -L` as hex ids like `28016f48cf37`.

```bash
podman network create saltdev

podman run -d --name salt-controller --hostname salt-controller \
  --network saltdev --network-alias salt \
  --systemd=always localhost/salt-controller:v3006.27

podman run -d --name minion01 --hostname minion01 --network saltdev \
  --systemd=always localhost/salt-minion:v3006.27

podman exec salt-controller salt-key -L
podman exec salt-controller salt '*' test.ping
```

`salt-key -L` will list two accepted minions — `minion01` and `salt-controller`.
The controller runs a minion of its own by design.

## Publishing to Nexus

Images are published to the Nexus private docker registry at
`nexus.tkclabs.io:8443`. Credentials come from `.envrc`, which is gitignored
and **not** part of a fresh clone — create it yourself with:

```bash
export NEXUS_USER_RW=<your-nexus-rw-username>
export NEXUS_PASS_RW=<your-nexus-rw-password>
```

Get the actual read-write credential values out of band; do not commit
`.envrc` or remove it from `.gitignore`.

All sixteen references below must exist locally before publishing — in
particular the `3006.6` builds are a separate command under "Build specific
version image" above and won't exist if you only ran the default builds.

```bash
source .envrc
printf '%s' "$NEXUS_PASS_RW" | podman login nexus.tkclabs.io:8443 -u "$NEXUS_USER_RW" --password-stdin

set -e
for ref in \
  systemd-almalinux:latest systemd-almalinux:10 \
  systemd-rockylinux:latest systemd-rockylinux:10 \
  systemd-fedora:latest systemd-fedora:44 \
  systemd-debian:latest systemd-debian:13 \
  systemd-ubuntu:latest systemd-ubuntu:26.04 \
  salt-controller:latest salt-controller:v3006.27 salt-controller:v3006.6 \
  salt-minion:latest salt-minion:v3006.27 salt-minion:v3006.6 \
; do
  podman tag "localhost/${ref}" "nexus.tkclabs.io:8443/${ref}" && \
    podman push "nexus.tkclabs.io:8443/${ref}"
done
```
