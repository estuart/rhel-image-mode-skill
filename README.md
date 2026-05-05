# RHEL Image Mode Skill for Claude Code

A [Claude Code skill](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/overview) that provides comprehensive guidance for building, deploying, and managing RHEL Image Mode (bootc) container images.

## What This Skill Covers

- **Containerfile authoring** -- patterns, anti-patterns, multi-stage builds, layered image strategies
- **Build and validation** -- podman workflows, `bootc container lint`, tag promotion
- **Deployment methods** -- bare metal, VM (KVM, vSphere), cloud (AWS), PXE, Kickstart, loopback images
- **Lifecycle management** -- `bootc upgrade`, `bootc rollback`, `bootc switch`, auto-update timer control
- **Filesystem model** -- /etc 3-way merge, /var persistence, state overlays, transient /etc
- **Air-gapped/disconnected** -- local repos, GPG key handling, physically-bound container images
- **Embedded containers** -- Quadlet units, logically-bound and physically-bound pull strategies
- **CI/CD pipelines** -- GitLab 3-stage pipeline (build/package/deploy), bootc-image-builder
- **Users and credentials** -- cloud-init, systemd credentials, UID drift prevention
- **Fleet operations** -- canary deployments, Ansible orchestration, N-1 compatibility rule

## Sources

Built from:
- [RHEL 10 Image Mode official documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index)
- [Upstream bootc project documentation](https://bootc.dev/bootc/)
- Internal Red Hat field best practices (RHEL Image Mode Best Practices v2)
- Red Hat Developer blog posts on [CI/CD pipelines](https://developers.redhat.com/articles/2026/02/12/how-build-image-mode-pipeline-gitlab), [Ansible integration](https://developers.redhat.com/articles/2025/03/18/how-use-rhel-system-roles-image-mode), and [embedded containers](https://developers.redhat.com/articles/2025/05/29/how-embed-containers-image-mode-rhel)

## Installation

### Option 1: Clone directly into Claude Code skills directory

```bash
git clone https://github.com/estuart/rhel-image-mode-skill.git \
    ~/.claude/skills/building-rhel-image-mode
```

### Option 2: Clone and symlink

```bash
git clone https://github.com/estuart/rhel-image-mode-skill.git ~/Projects/rhel-image-mode-skill
ln -s ~/Projects/rhel-image-mode-skill ~/.claude/skills/building-rhel-image-mode
```

### Verify

After installation, the skill should appear in Claude Code's available skills list. You can invoke it with:

```
/building-rhel-image-mode
```

Or by asking about image mode topics naturally -- Claude will discover the skill when you mention "bootc", "image mode", "bootable container", etc.

## File Structure

```
SKILL.md                   # Main reference (architecture, filesystem, build workflow)
containerfile-patterns.md  # Containerfile examples, anti-patterns, linting
deployment-methods.md      # All deployment paths with commands
lifecycle-operations.md    # Upgrade, rollback, switch, fleet management
```

The skill uses progressive disclosure: `SKILL.md` is loaded when the skill triggers, and the reference files are loaded on-demand when Claude needs deeper detail on a specific topic.

## Usage Examples

Once installed, ask Claude Code things like:

- "Help me write a Containerfile for a hardened web server image"
- "How do I deploy a bootc image to vSphere?"
- "What's the rollback procedure if an upgrade breaks something?"
- "Set up a GitLab CI pipeline for building bootc images"
- "How does /etc work in image mode?"
- "Build an air-gapped bootc image for a disconnected environment"

## Contributing

PRs welcome. If you find gaps or inaccuracies, open an issue or submit a fix. Key areas where contributions would be valuable:

- FIPS mode / hardened bootc image patterns
- Satellite-as-registry configuration
- Fleet monitoring and drift detection at scale
- Additional CI/CD platform examples (GitHub Actions, Tekton)

## License

This skill is provided as-is for internal Red Hat use and community sharing. The content is derived from publicly available Red Hat documentation and upstream bootc project docs.
