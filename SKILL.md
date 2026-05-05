---
name: rhel-image-mode
description: >
  Build, deploy, and manage RHEL Image Mode (bootc) container images.
  Covers Containerfile authoring, bootc CLI operations, bootc-image-builder
  disk image creation, deployment to bare metal/VM/cloud, lifecycle management
  (upgrade/rollback/switch), filesystem model, air-gapped builds, embedded
  containers via Quadlet, and CI/CD pipeline patterns.
  Triggers: /rhel-image-mode, "image mode", "bootc", "rhel-bootc",
  "bootc image", "bootable container", "image mode containerfile",
  "bootc install", "bootc upgrade", "bootc-image-builder".
user_invocable: true
---

# rhel-image-mode — RHEL Image Mode Reference

## Architecture

Image mode delivers the OS as a bootc container image. The system separates into three layers:

| Layer | Contents | Mutability |
|-------|----------|------------|
| OS | Kernel, drivers, base utilities | Immutable at runtime |
| Application | Workload binaries, shared deps | Immutable at runtime |
| Configuration & Data | Node identity, persistent state | Mutable |

### What to bake into the image

- Packages universal to the workload class (e.g., Python, httpd)
- Baseline security configs, fleet-wide sudoers, systemd units
- OSCAP profiles, firewall rules, SELinux policy

### What to inject at deploy/boot time

- IP addresses, hostnames, SSH keys
- Environment-specific settings
- Drop-in configs (e.g., /etc/sudoers.d/)
- Registry credentials, secrets

### /etc shared responsibility model

- **Global policy (image)**: codify baseline security and system-wide config directly in the Containerfile
- **Local identity (automation)**: use cloud-init, Ansible, or ignition to manage node-specific drop-in files at boot

### Layered image strategy

Structure builds as a chain of responsibility when multiple teams own different concerns:

```
Base OS Image (sysadmin) -> Security Image (OSCAP) -> Application Image (app team)
```

Each team owns their layer. Changes flow down the chain via `FROM` inheritance.

## Base Images

| Version | Base Image | Status |
|---------|-----------|--------|
| RHEL 10 | `registry.redhat.io/rhel10/rhel-bootc:latest` | GA, native support |
| RHEL 9 | `registry.redhat.io/rhel9/rhel-bootc:latest` | GA (9.5+) |
| CentOS Stream 10 | `quay.io/centos-bootc/centos-bootc:stream10` | Upstream |

RHEL 10 is the primary target. RHEL 9 has feature parity but uses a different base image path.

## Build, Validate, and Push

```bash
# Build
podman build -t quay.io/<namespace>/<image>:<tag> .

# Lint the image for common issues
podman run --rm <image> bootc container lint --fatal-warnings

# Push to registry
podman login quay.io
podman push quay.io/<namespace>/<image>:<tag>

# Verify
podman images
```

`bootc container lint` checks for missing tmpfiles.d entries for /var paths, files incorrectly placed in /usr/etc, and other common mistakes.

### Tag promotion

Use registry tags for lifecycle control:

```
:dev -> :staging -> :prod
```

Tag images with commit SHAs for traceability: `<image>:$CI_COMMIT_SHORT_SHA`

## Filesystem Model

### Immutable at runtime

| Path | Behavior |
|------|----------|
| `/` (root) | composefs-backed, read-only |
| `/usr` | All OS content, read-only. `/bin` symlinks to `/usr/bin` |
| `/opt` | Plain directory, writable at build time, immutable at runtime |
| `/usr/local` | Same as /opt |

### Mutable / persistent

| Path | Behavior |
|------|----------|
| `/etc` | Mutable persistent state. 3-way merge on upgrades |
| `/var` | Persistent data volume. Content copied from image at initial install ONLY |

### /etc 3-way merge (on upgrade)

1. New default /etc from updated image used as base
2. Diff between current /etc and previous default applied to new
3. Locally modified files are retained

### /var behavior

- Content is copied from the container image at **initial install only**
- Subsequent image updates do **NOT** modify /var
- Acts as a persistent volume (like a Docker volume)
- Use `systemd tmpfiles.d` or `StateDirectory=` for pre-created directory structures
- Standard writable paths: `/var/log`, `/var/lib`

### Custom writable paths

If an application needs a path like `/application`, either:
- Symlink it: `/application -> /var/lib/application`
- Configure a systemd mount unit to attach external storage
- Use `BindPaths=` in the systemd unit: `BindPaths=/var/log/myapp:/opt/myapp/logs`

Failure to plan writable paths causes "Read-only file system" errors at runtime.

### State overlays

To make `/opt` (or other toplevel directories) writable across reboots while still receiving image updates:

```bash
systemctl enable ostree-state-overlay@opt.service
```

State overlay semantics: changes persist across reboots, but new container image files override locally modified versions during updates. Smaller mutable surface than transient root.

### Transient /etc mode

For fully config-managed systems where /etc should reset on every boot:

```ini
# /usr/lib/ostree/prepare-root.conf
[etc]
transient = true
```

This eliminates persistent /etc state entirely. Useful when all machine-specific configuration comes from kernel commandline, cloud-init, or ignition.

### /run, /proc, other API filesystems

No support for shipping content in these paths via container images.

### SELinux caveat for custom directories

Custom toplevel directories created during image build (e.g., `RUN mkdir /mydata`) may receive `default_t` SELinux labels, potentially blocking access. Define file contexts explicitly when needed.

## Users and Credentials

### Injection methods (ordered by preference)

1. **cloud-init / ignition** — metadata server provides SSH keys + users at boot. Best for cloud/virt.
2. **systemd credentials** — SMBIOS injection for local virtualization (qemu).
3. **Custom systemd unit** — pulls from FreeIPA or network-hosted credential source.
4. **Static in container build** — `RUN useradd someuser`. Caution: potential UID/GID drift.

### UID/GID drift prevention

- Prefer `DynamicUser=yes` in systemd service units
- Use `systemd-sysusers` with drop-in files: `COPY mycustom-user.conf /usr/lib/sysusers.d`
- Base images built by rpm-ostree have `nss-altfiles` enabled by default

### /var/home caveat

`/home -> /var/home` is persistent. Changes to /var in the container image after initial install are **not applied** on subsequent updates. Injecting `authorized_keys` into `/var/home/user/.ssh/` in the build will not update existing deployed systems.

## Air-Gapped / Disconnected Builds

Key constraints:
- Repository configs must be **inside** the container image, not on the host
- GPG keys must point to local paths (`file:///etc/pki/rpm-gpg/`) not remote URLs
- Pre-install `kernel-bootc` and `anaconda-dracut-modules` to avoid BIB download failures
- Use a local mirror registry for the base image: `FROM example.com:1234/rhel10/rhel-bootc:10.2`

See [containerfile-patterns.md](containerfile-patterns.md) for a complete air-gapped Containerfile example.

## Embedded Container Workloads (Quadlet)

Quadlet runs Podman containers as systemd services using declarative `.container` unit files.

### Pull strategies

| Strategy | Network Required | Startup Delay | Use Case |
|----------|-----------------|---------------|----------|
| On-demand | Yes | Yes | Simple, connected environments |
| Logically-bound | At install/update | No | Pre-pulled via `/usr/lib/bootc/bound-images.d/` symlinks |
| Physically-bound | Never | No | Air-gapped; embedded at build time via skopeo |

See [containerfile-patterns.md](containerfile-patterns.md) for Quadlet and bound image examples.

## bootc-image-builder

Converts bootc container images into deployable disk formats. Runs as a container itself.

### Install

```bash
sudo podman login registry.redhat.io
sudo podman pull registry.redhat.io/rhel10/bootc-image-builder
```

### Supported output formats

| Format | Use Case |
|--------|----------|
| QCOW2 | KVM/QEMU virtual machines |
| AMI | AWS EC2 |
| Raw | Generic disk image |
| VMI/VMDK | VMware vSphere |
| ISO | Disconnected bare metal install (Tech Preview) |

### Key constraint

bootc-image-builder uses **local container storage only**. It cannot pull from remote registries. Mount the host's container storage into the BIB container.

See [deployment-methods.md](deployment-methods.md) for complete BIB commands and all deployment paths.

## Post-Install Metadata

After installation, bootc writes `.bootc-aleph.json` to the physical filesystem root containing source/target image digests, OCI labels, build timestamp, kernel version, and SELinux state. Accessible post-boot at `/sysroot/.bootc-aleph.json`. Useful for auditing deployed systems.

## Known Gaps

These topics are not fully covered by this skill:

- **FIPS mode** — building FIPS-compliant bootc images
- **Satellite-as-registry** — concrete configuration steps for Satellite integration
- **Fleet monitoring** — tracking image versions across deployed systems at scale

## Additional References

- **Containerfile examples and anti-patterns**: [containerfile-patterns.md](containerfile-patterns.md)
- **All deployment methods with commands**: [deployment-methods.md](deployment-methods.md)
- **Lifecycle operations (upgrade/rollback/switch)**: [lifecycle-operations.md](lifecycle-operations.md)

### External documentation

- [RHEL 10 Image Mode docs](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index)
- [Upstream bootc project docs](https://bootc.dev/bootc/)
- [Image Mode use cases](https://developers.redhat.com/articles/2024/11/05/image-mode-rhel-4-key-use-cases-streamlining-your-os)
- [GitLab CI/CD pipeline for image mode](https://developers.redhat.com/articles/2026/02/12/how-build-image-mode-pipeline-gitlab)
- [Ansible system roles with image mode](https://developers.redhat.com/articles/2025/03/18/how-use-rhel-system-roles-image-mode)
- [Embedding containers in image mode](https://developers.redhat.com/articles/2025/05/29/how-embed-containers-image-mode-rhel)
