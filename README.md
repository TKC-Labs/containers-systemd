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
