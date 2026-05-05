# Containerfile Patterns for RHEL Image Mode

## Contents

- General structure and ignored directives
- Cloud-init enabled image
- Air-gapped / disconnected build
- Multi-stage build for Ansible system roles
- Layered image strategy
- Embedding containers with Quadlet
- Logically-bound and physically-bound images
- Kernel argument injection
- Third-party driver installation
- Anti-patterns

## General Containerfile Structure

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest

RUN dnf -y install [packages] && dnf clean all

ADD [application files]
ADD [configuration files]
RUN [config scripts]
```

### Ignored directives (when installed to a system)

These Containerfile instructions are ignored when the bootc image is deployed to disk. They work normally when running the image as a container with podman/docker.

| Directive | Instead use |
|-----------|-------------|
| `ENTRYPOINT` / `CMD` | Set `CMD /sbin/init` |
| `ENV` | Configure systemd environment files |
| `EXPOSE` | Configure firewall rules in the image |
| `USER` | Configure individual services as unprivileged |

## Cloud-Init Enabled Image

For cloud and virtualized deployments where metadata servers provide identity:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest
RUN dnf -y install cloud-init && \
    ln -s ../cloud-init.target /usr/lib/systemd/system/default.target.wants && \
    dnf clean all
```

This enables automatic SSH key injection and instance configuration from infrastructure metadata (AWS, GCP, KVM with metadata).

## Air-Gapped / Disconnected Build

```dockerfile
FROM example.com:1234/rhel10/rhel-bootc:10.2

RUN echo -e "[local-baseos]\n\
name=Local RHEL 10 BaseOS\n\
baseurl=file:///path/to/repo/in/image/BaseOS\n\
enabled=1\n\
gpgcheck=0" > /etc/yum.repos.d/local.repo

RUN dnf install -y firewalld && dnf clean all

# Pre-install to avoid BIB download failures
RUN dnf install -y kernel-bootc anaconda-dracut-modules && dnf clean all
```

### Air-gapped constraints

- Repo configs must be **inside** the container image, not on the host
- `gpgkey` must point to a local file path (`file:///etc/pki/rpm-gpg/`) or reachable local HTTP server
- bootc-image-builder re-validates GPG keys during ISO creation even if `dnf install` succeeded during build
- Store GPG keys locally in the image and reference them by `file:/` in `.repo` files

## Multi-Stage Build for Ansible System Roles

Use a multi-stage build to discover system role dependencies without polluting the final image:

```dockerfile
FROM quay.io/centos-bootc/centos-bootc:stream10 as ansible-stage
RUN dnf -y install ansible-core
RUN ansible-galaxy collection install fedora.linux_system_roles
RUN mkdir -p /deps
RUN /root/.ansible/collections/ansible_collections/fedora/linux_system_roles/\
roles/podman/.ostree/get_ostree_data.sh \
    packages runtime centos-10 raw > /deps/ansible.txt || true

FROM quay.io/centos-bootc/centos-bootc:stream10
RUN --mount=type=bind,from=ansible-stage,source=/deps/,target=/deps \
    cat /deps/ansible.txt | xargs dnf -y install
```

Each system role provides a `get_ostree_data.sh` script to extract required packages. Available roles include SELinux, firewall, Cockpit, SAP, and Podman.

## Layered Image Strategy

Structure builds as a chain when teams own different concerns:

```dockerfile
# Layer 1: Base OS (maintained by sysadmin team)
FROM registry.redhat.io/rhel10/rhel-bootc:latest AS base
RUN dnf -y install standard-tools logging-agent && dnf clean all
COPY base-sudoers /etc/sudoers.d/base

# Layer 2: Security (maintained by security team)
FROM base AS secured
RUN dnf -y install openscap-scanner scap-security-guide && dnf clean all
RUN oscap xccdf eval --remediate --profile xccdf_org.ssgproject.content_profile_stig \
    /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml || true

# Layer 3: Application (maintained by app team)
FROM secured
RUN dnf -y install httpd mod_ssl && dnf clean all
COPY httpd.conf /etc/httpd/conf/httpd.conf
RUN systemctl enable httpd
```

In practice, each layer is usually its own Containerfile + registry image. The downstream layer uses `FROM <registry>/secured-base:prod` rather than a multi-stage build.

## Embedding Containers with Quadlet

Quadlet runs Podman containers as systemd services. Place `.container` unit files in `/etc/containers/systemd/`:

### Containerfile

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest
RUN mkdir -p /etc/containers/systemd
COPY my-app.container /etc/containers/systemd/
```

### Quadlet unit file (my-app.container)

```ini
[Unit]
Description=My application container

[Container]
Image=registry.example.com/my-app:latest
Exec=/usr/bin/my-app

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

## Logically-Bound Images (Pre-Pulled)

Images are pre-pulled during install/update. Avoids startup delay without full air-gap embedding.

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest
COPY my-app.container /usr/share/containers/systemd/my-app.container
RUN ln -s /usr/share/containers/systemd/my-app.container \
    /usr/lib/bootc/bound-images.d/my-app.container
```

The Quadlet unit must include:

```ini
GlobalArgs=--storage-opt=additionalimagestore=/usr/lib/bootc/storage
```

## Physically-Bound Images (Air-Gapped)

Fully embed container images at build time using skopeo:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest

# Embed at build time
RUN skopeo copy --preserve-digests \
    docker://registry.example.com/my-app:latest \
    dir:/usr/lib/containers-image-cache/my-app

# Copy into container storage at runtime
RUN skopeo copy --preserve-digests \
    dir:/usr/lib/containers-image-cache/my-app \
    containers-storage:registry.example.com/my-app:latest
```

## Kernel Argument Injection

### Via kargs.d (preferred for image-embedded args)

Create a TOML file at `/usr/lib/bootc/kargs.d/`:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest
COPY 10-console.toml /usr/lib/bootc/kargs.d/
```

```toml
# /usr/lib/bootc/kargs.d/10-console.toml
kargs = ["console=ttyS0,115200n8"]
```

### At install time

```bash
bootc install to-filesystem --karg=root=UUID=<uuid> --imgref $self /mnt
```

### Post-install modification

```bash
rpm-ostree kargs --append debug
```

## Third-Party Driver Installation

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest
COPY my-driver.rpm /tmp/
RUN dnf install -y /tmp/my-driver.rpm && rm /tmp/my-driver.rpm
```

RPM `%post` scripts trigger `depmod` automatically inside the image.

## Anti-Patterns

| Do NOT | Instead |
|--------|---------|
| Embed static IPs in the Containerfile | Use cloud-init or Ansible at boot |
| Embed hostnames | Use cloud-init `local-hostname` or DHCP |
| Embed SSH private keys | Inject via metadata server or systemd credentials |
| Put node-specific configs in the image | Use drop-in directories managed by automation |
| Use `dnf update` on running systems | Build a new image, push to registry, `bootc upgrade` |
| Treat the running disk as source of truth | The Containerfile is the source of truth |
| Write application data to immutable paths | Plan writable paths under `/var` or use mount units |
| Skip `dnf clean all` in RUN steps | Always clean to reduce image size |
| Use `ENV` for system-wide environment | Use systemd environment files |
