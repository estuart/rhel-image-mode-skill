# Lifecycle Operations for RHEL Image Mode

## Contents

- Update workflow overview
- bootc upgrade
- bootc rollback
- bootc switch
- bootc status
- Auto-update timer management
- Download-only mode
- Fleet update patterns
- N-1 compatibility rule
- Troubleshooting

## Update Workflow Overview

The operational model replaces traditional patching with image deployment:

```
Edit Containerfile -> Build new image -> Push to registry -> bootc upgrade on targets
```

| Traditional | Image Mode |
|-------------|------------|
| `dnf update` on live systems | `bootc upgrade` pulls new image |
| Patch individual packages | Replace entire OS layer atomically |
| Risk of configuration drift | Identical state across fleet |
| Troubleshoot the running system | Troubleshoot the Containerfile |

## bootc upgrade

Fetches an updated image from the registry and stages it for next boot.

```bash
bootc upgrade
```

The image is staged but the running system is **not changed** until reboot.

### Auto-apply (stage + reboot)

```bash
bootc upgrade --apply
```

The `--apply` flag triggers an automatic reboot if the system has changed.

### Default behavior

RHEL bootc images ship with `bootc-fetch-apply-updates.timer` **enabled by default**. This periodically checks the registry and auto-upgrades. See "Auto-update timer management" below to control this.

## bootc rollback

Reverts to the previous boot entry. Reorders the bootloader so the prior deployment is next.

```bash
bootc rollback
```

### Critical caveats

1. **/etc reverts completely**: Changes to `/etc` do NOT carry over to the rolled-back deployment. The /etc directory reverts to the state from the previous deployment. No 3-way merge occurs on rollback (unlike upgrade).

2. **Auto-update will re-upgrade**: The update timer will automatically pull the newer image again within 1-3 hours unless you mask it first:
   ```bash
   systemctl mask bootc-fetch-apply-updates.timer
   ```

3. **/var is untouched**: Persistent data in /var is NOT rolled back. Database schemas, application data, and logs remain in their current state.

4. **Preserving /etc files**: To keep a modified /etc file across rollback, copy it to /var first:
   ```bash
   cp /etc/my-config /var/root/my-config-backup
   ```

### Rollback vs switch

`bootc rollback` reorders existing deployments without creating new ones. If you need /etc changes to be merged properly (like a normal upgrade), use `bootc switch` instead to revert to an older image tag.

## bootc switch

Switches to a completely different image reference. Performs the full /etc 3-way merge (unlike rollback).

```bash
bootc switch <new-image-reference>
```

Use cases:
- Reverting to an older image version while preserving /etc merge behavior
- Moving a system from one image stream to another (e.g., web-tier to db-tier)
- Switching from CentOS Stream to RHEL base image

### Switching to a locally-built image

For dev/testing, switch to an image in local podman storage without a registry:

```bash
bootc switch --transport containers-storage localhost/my-test-image:latest
```

## bootc status

Shows the current deployment state.

```bash
bootc status
```

Displays:
- Currently booted image reference and version
- Staged image (if an upgrade is pending)
- Rollback image (previous deployment)

## Auto-Update Timer Management

### Disable auto-updates entirely

```bash
systemctl mask bootc-fetch-apply-updates.timer
```

### Change update schedule

Use a systemd drop-in to override the timer. Example for weekly updates:

```bash
mkdir -p /usr/lib/systemd/system/bootc-fetch-apply-updates.timer.d
```

Create `/usr/lib/systemd/system/bootc-fetch-apply-updates.timer.d/updates.conf`:

```ini
[Timer]
# Clear previous timers
OnBootSec=
OnBootSec=1w
OnUnitInactiveSec=1w
```

### Disable auto-updates in the Containerfile

To build images with auto-updates disabled by default:

```dockerfile
RUN systemctl mask bootc-fetch-apply-updates.timer
```

### Fleet-managed updates pattern

For centrally managed fleets, disable the auto-update timer and use a management agent:

```dockerfile
# Disable automatic updates
RUN systemctl mask bootc-fetch-apply-updates.timer

# Install management client
RUN dnf install -y <management-agent> && dnf clean all

# Inject credentials and enable registration service
COPY mgmt-credentials /etc/mgmt/credentials
RUN ln -s /etc/systemd/system/mgmt-register.service \
    /etc/systemd/system/multi-user.target.wants/mgmt-register.service
```

The management service determines when to upgrade each system based on fleet policy.

## Download-Only Mode

Stage an update without applying it. Useful for pre-positioning during maintenance windows.

```bash
# Download and stage
bootc upgrade

# Later, when ready to apply
systemctl reboot
```

The default `bootc upgrade` (without `--apply`) already operates in download-only mode. The update is staged and applied on the next reboot.

## Fleet Update Patterns

### Ansible-orchestrated rolling update

Use Ansible to coordinate updates across a fleet with controlled reboots:

1. Update the Containerfile with new packages/patches
2. Build and push the new image to the registry
3. Use Ansible to run `bootc upgrade` on target groups
4. Schedule reboots during maintenance windows
5. Verify with `bootc status` after reboot

### Canary deployments

Use registry tags to implement canary rollouts:

1. Push the new image with a canary tag: `<image>:canary`
2. `bootc switch <image>:canary` on a small test group
3. Monitor for issues
4. If successful, re-tag as `:prod` and roll out to the fleet
5. If issues found, `bootc switch <image>:prod` on the canary group to revert

### Controlled rollouts via registry

Tag promotion controls which systems see which version:

```
:dev    -> development/test systems auto-pull
:staging -> pre-production validation
:prod   -> production fleet
```

Systems are configured to track their tier's tag. Promotion is a registry-side tag operation, not a per-system command.

## N-1 Compatibility Rule

OS rollback restores the operating system and packaged applications but does **not** revert persistent application data.

### The problem

A database schema migration applied during an upgrade leaves data in a format the previous application version cannot read. Rolling back the OS gives you the old application but the new data schema.

### The mitigation

- Design application updates to maintain backward compatibility with the previous data schema version
- Take data-level backups or snapshots before deploying images with major application upgrades
- Recovery requires **both** an image rollback and a data restoration

### Example scenario

```
v1 image (schema v1) -> upgrade to v2 image (migrates schema to v2)
                      -> rollback to v1 image
                      -> v1 app cannot read schema v2 data
```

Solution: ensure v1 app can read schema v2 data, OR restore data snapshot alongside rollback.

## Auditing Deployments

### .bootc-aleph.json

After installation, bootc writes metadata to `/sysroot/.bootc-aleph.json` containing:
- Source and target image digests
- OCI labels from the container image
- Installation timestamp
- bootc version used for install
- Kernel version
- SELinux state

Use this for auditing which image was originally installed on a system.

### bootc status

Check current deployment state, staged updates, and rollback targets:

```bash
bootc status
```

## Troubleshooting

### "Read-only file system" errors at runtime

Application is writing to an immutable path. Fix by:
- Moving data to `/var/lib/<app>`
- Creating a symlink: `/application -> /var/lib/application`
- Adding a systemd mount unit for external storage

### Rollback doesn't fix the issue

Remember that /var persists across rollback. If the problem is in persistent data (not the OS), rollback alone won't help. Restore from data backup.

### Auto-update keeps reverting a rollback

Mask the timer before rolling back:
```bash
systemctl mask bootc-fetch-apply-updates.timer
bootc rollback
systemctl reboot
```

### /etc changes lost after rollback

Expected behavior. /etc reverts to the previous deployment's state. Use `bootc switch <older-image-tag>` instead if you need /etc merge behavior.
