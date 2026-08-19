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
