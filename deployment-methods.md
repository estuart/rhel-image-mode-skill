# Deployment Methods for RHEL Image Mode

## Contents

- Deployment overview
- bootc install to-disk (bare metal / VM)
- bootc install to-filesystem (custom partitioning)
- system-reinstall-bootc (cloud instance conversion)
- KVM with QCOW2
- vSphere with VMDK
- AWS with AMI
- Anaconda / Kickstart (Tech Preview)
- PXE boot with ISO
- bootc-image-builder commands
- CI/CD pipeline structure

## Deployment Overview

Installation happens **once**. After initial deployment, all future updates come directly from the container registry via `bootc upgrade`.

| Method | Target | Maturity |
|--------|--------|----------|
| `bootc install to-disk` | Bare metal, VM | GA |
| `bootc install to-filesystem` | Custom partitioning | GA |
| `system-reinstall-bootc` | Running cloud instance | GA (RHEL 9.6+/10.0+) |
| bootc-image-builder QCOW2 | KVM/QEMU VMs | GA |
| bootc-image-builder AMI | AWS EC2 | GA |
| bootc-image-builder VMDK | VMware vSphere | GA |
| bootc-image-builder Raw | Generic disk | GA |
| bootc-image-builder ISO | Disconnected bare metal | Tech Preview |
| Anaconda/Kickstart | Bare metal, VM, cloud | Tech Preview |
| PXE boot | Bare metal | Tech Preview |

## bootc install to-disk

Installs a bootc image directly to a block device. Handles partitioning, bootloader setup, and image extraction.

```bash
podman run --rm --privileged --pid=host \
    -v /dev:/dev \
    -v /var/lib/containers:/var/lib/containers \
    --security-opt label=type:unconfined_t \
    <image> \
    bootc install to-disk <path-to-disk>
```

The image itself contains the installer. No separate install media needed beyond the container runtime.

## bootc install to-filesystem

For custom partition layouts (e.g., LVM). You prepare the filesystem, then bootc populates it.

```bash
# Prepare your partitions and mount at /mnt, then:
podman run --rm --privileged --pid=host \
    -v /:/target \
    -v /dev:/dev \
    -v /var/lib/containers:/var/lib/containers \
    --security-opt label=type:unconfined_t \
    <image> \
    bootc install to-filesystem /mnt
```

Under the hood, `to-disk` is equivalent to:
```bash
mkfs.$fs /dev/disk
mount /dev/disk /mnt
bootc install to-filesystem --karg=root=UUID=<uuid of /mnt> --imgref $self /mnt
```

## system-reinstall-bootc (Cloud Instance Conversion)

Converts a running package-mode RHEL instance to image mode with a single command. Wraps `bootc install to-existing-root`.

```bash
# SSH into the cloud instance, then:
sudo system-reinstall-bootc <registry>/<image>:<tag>
```

Requirements:
- Package-based RHEL 9.6+ or 10.0+
- Destructive: replaces the existing OS
- SSH key selected at instance launch is preserved

## KVM with QCOW2

First create a QCOW2 disk image with bootc-image-builder (see below), then launch with virt-install:

```bash
sudo virt-install \
    --name bootc \
    --memory 4096 \
    --vcpus 2 \
    --disk <qcow2/disk.qcow2> \
    --import
```

## vSphere with VMDK

Create a VMDK with bootc-image-builder, then deploy using govc CLI.

### Prepare cloud-init data

```yaml
# metadata.yaml
instance-id: cloud-vm
local-hostname: vmname
```

```yaml
# userdata.yaml
#cloud-config
users:
  - name: admin
    sudo: "ALL=(ALL) NOPASSWD:ALL"
    ssh_authorized_keys:
      - ssh-rsa AAA...fhHQ== your.email@example.com
```

### Deploy

```bash
# Set govc environment
export GOVC_URL=...
export GOVC_DATACENTER=...
export GOVC_FOLDER=...
export GOVC_DATASTORE=...
export GOVC_RESOURCE_POOL=...
export GOVC_NETWORK=...

# Export cloud-init data as base64-encoded gzip
export METADATA=$(gzip -c metadata.yaml | base64)
export USERDATA=$(gzip -c userdata.yaml | base64)

# Upload VMDK and create VM using govc commands
```

## AWS with AMI

Create an AMI with bootc-image-builder and upload to AWS. See BIB commands below for the build step. The CI/CD pipeline section shows the full automated flow.

## Anaconda / Kickstart (Tech Preview)

Use `ostreecontainer` or the new `bootc` kickstart command to install from a registry image:

```
# In your kickstart file, replace the %packages section with:
ostreecontainer --url registry.example.com/my-image:prod
```

Not recommended for production (Technology Preview). Both `ostreecontainer` and `bootc` kickstart commands are functionally similar; the `bootc` command is newer.

## PXE Boot with ISO

1. Create an ISO with bootc-image-builder
2. Configure PXE server (DHCP + TFTP or HTTP)
3. Boot client from network, select PXE boot source
4. Installation proceeds from the ISO

Standard RHEL PXE infrastructure works; the only difference is the ISO contains a bootc image instead of a traditional installer.

## bootc-image-builder Commands

### Install BIB

```bash
sudo podman login registry.redhat.io
sudo podman pull registry.redhat.io/rhel10/bootc-image-builder
```

### Create a QCOW2 image

```bash
# First, ensure the bootc image is in local storage
podman pull <your-image>

# Run BIB
sudo podman run --rm --privileged \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    registry.redhat.io/rhel10/bootc-image-builder \
    --type qcow2 \
    --local \
    <your-image>
```

### Create other formats

Replace `--type qcow2` with: `ami`, `vmdk`, `raw`, or `iso`.

### With customizations (config file)

BIB accepts a TOML or JSON config file for customizations (users, SSH keys, filesystem layout, partitioning, timezone, locale):

```bash
sudo podman run --rm --privileged \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    -v ./config.toml:/config.toml:ro \
    registry.redhat.io/rhel10/bootc-image-builder \
    --type qcow2 \
    --config /config.toml \
    --local \
    <your-image>
```

### Key constraints

- BIB uses **local container storage only** — cannot pull from remote registries
- Mount host storage: `-v /var/lib/containers/storage:/var/lib/containers/storage`
- ISO creation requires `--privileged`
- Generic base images have no default passwords or SSH keys — add them via BIB config or cloud-init in the Containerfile

## CI/CD Pipeline Structure (GitLab)

Three-stage pipeline:

```yaml
stages:
  - build
  - package
  - deploy

variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE/my-rhel-os
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
```

### Stage 1: Build (container image)

- Image: `registry.access.redhat.com/ubi10/podman:latest`
- Authenticate to GitLab registry and Red Hat registry
- Set up RHSM secrets at `/run/secrets/` for subscription-manager
- `podman build -t $IMAGE_NAME:$IMAGE_TAG .`
- `podman push $IMAGE_NAME:$IMAGE_TAG`

### Stage 2: Package (disk image)

- Image: `quay.io/centos-bootc/bootc-image-builder:latest`
- Requires `--privileged` (shared GitLab runners won't work)
- `bootc-image-builder $IMAGE_NAME:$IMAGE_TAG --type qcow2 --output ./output`

### Stage 3: Deploy (upload artifact)

- Upload disk image to object storage (S3, GCS, etc.)
- Runs only on main branch commits

```bash
aws s3 cp $FILE_PATH s3://your-os-artifacts-bucket/rhel-image-mode/$IMAGE_TAG.qcow2
```

### Required CI/CD variables

- `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` — GitLab registry auth
- `RH_REGISTRY_USER` / `RH_REGISTRY_PASSWORD` — Red Hat registry auth
- `RHN_USER` / `RHN_PASSWORD` — subscription-manager entitlements
