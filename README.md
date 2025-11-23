# Systemd podman containerfiles

## Building Images

*Trailing . is intentional*

```bash
# Default Debian version is 12
podman build -f Containerfile.debian -t systemd:debian12 .

# Build for other Debian version, like 11
podman build -f Containerfile.debian --build-arg VERSION=11 -t systemd:debian11 .

# Default almalinux version is 10
podman build -f Containerfile.almalinux -t systemd:almalinux10 .

# Build for other AlmaLinux version, like 9
podman build -f Containerfile.almalinux -t systemd:almalinux9 .
```
