[![CI](https://github.com/guidugli/ansible-role-package_manager/actions/workflows/CI.yml/badge.svg)](https://github.com/guidugli/ansible-role-package_manager/actions/workflows/CI.yml)
[![Release](https://img.shields.io/github/v/release/guidugli/ansible-role-package_manager?display_name=tag&sort=semver)](https://github.com/guidugli/ansible-role-package_manager/releases)
[![Galaxy](https://img.shields.io/badge/galaxy-guidugli.package_manager-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/guidugli/package_manager/)
[![License](https://img.shields.io/github/license/guidugli/ansible-role-package_manager)](LICENSE)

# Ansible Role: package_manager

`guidugli.package_manager` configures Linux package management across supported package managers. It can refresh package metadata, apply package updates, install packages, remove packages, clean package-manager caches, enforce RedHat-family GPG-check settings, and report whether a reboot is required.

The role is designed to stay execution-context aware: it does not enforce privilege escalation, avoids rebooting container targets, and keeps optional validation tasks non-disruptive.

## Requirements

- Ansible Core 2.17 for the included development and Molecule toolchain.
- Supported package managers:
  - `apt`
  - `dnf`
  - `dnf5`
  - `yum`
  - `apk`
  - `pacman`
  - `zypper`
- Collections from `requirements.yml`:
  - `containers.podman >= 1.10.0` for Molecule container scenarios.
  - `community.general >= 9.0.0` for package-manager modules that are not part of `ansible.builtin`.
- Privileged operations, such as package installation, service management, repository configuration, and reboot actions, must be authorized by the caller through the playbook or automation platform.

## Variables

All role inputs are defined in `defaults/main.yml` and should be mirrored in `meta/argument_specs.yml`.

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `pm_update_system` | bool | `true` | Updates installed packages using the detected package manager. |
| `pm_apt_upgrade_command` | str | `dist` | Upgrade mode for APT systems. Valid values are `dist`, `full`, `safe`, and `yes`. |
| `pm_apk_upgrade_available` | bool | `false` | Allows APK to replace or downgrade packages when the currently installed version is no longer available from repositories. |
| `pm_cache_valid_time` | int | `3600` | Package metadata cache validity in seconds for package managers that support cache age checks. |
| `pm_security_only` | bool | `false` | Applies security updates only on RedHat-family package-manager tasks. |
| `pm_autoremove` | bool | `true` | Removes unneeded dependency packages where supported by the package manager. |
| `pm_purge` | bool | `false` | Purges package configuration files when packages are removed on Debian-family systems. |
| `pm_gpgcheck` | bool | `true` | Enforces `gpgcheck = 1` in RedHat-family package manager and repository configuration. |
| `pm_disable_rhnsd` | bool | `true` | Stops and disables `rhnsd` on supported RedHat-family hosts when the service task runs. |
| `pm_gpgcheck_excludes` | list(str) | `[]` | RedHat-family repository files excluded from automatic `gpgcheck` enforcement. |
| `pm_install_packages` | list(str) | `[]` | Packages to install. Package names are passed directly to the detected package manager. |
| `pm_remove_packages` | list(str) | `[]` | Packages to remove. Package names are passed directly to the detected package manager. |
| `pm_reboot_after_update` | bool | `false` | Reboots the host when the platform indicates that a reboot is required after package updates. Container connections are skipped. |

Internal variables are stored in `vars/main.yml` and normally do not need to be overridden.

| Variable | Description |
| --- | --- |
| `int_default_value` | Internal sentinel value used for integer validation. |
| `default_packages` | Platform-specific package maps retained for role data and future extension. Review before using in production. |
| `pkgmgr_redhat_conf` | RedHat-family package manager configuration file path. |

## Example Playbook

Package-management changes usually require elevated privileges on real hosts. Apply privilege at the play level or from your automation platform, not inside the role.

```yaml
---
- name: Manage baseline packages
  hosts: servers
  become: true
  roles:
    - role: guidugli.package_manager
      vars:
        pm_update_system: true
        pm_security_only: false
        pm_install_packages:
          - vim
          - curl
        pm_remove_packages: []
        pm_reboot_after_update: false
```

To enable optional APT key fingerprint inspection on systems where `gpg` is not already installed, install GnuPG explicitly:

```yaml
---
- name: Manage APT hosts with optional key fingerprint review
  hosts: debian_servers
  become: true
  roles:
    - role: guidugli.package_manager
      vars:
        pm_install_packages:
          - gnupg
```

## Molecule Testing

The role uses shared Molecule playbooks so the same converge and verify logic can be reused by multiple scenarios.

- `molecule/shared/converge.yml` applies the role to every scenario target.
- `molecule/shared/verify.yml` validates package manager detection, package facts, APT package index availability, optional GPG fingerprint capability, and RedHat-family package manager configuration presence.
- `molecule/default` exercises standard containerized distro targets.
- `molecule/systemd` prepares systemd-capable containers for service-manager-sensitive coverage.

Run the scenarios with:

```bash
molecule test -s default
molecule test -s systemd
```

For a faster local iteration after containers already exist, run:

```bash
molecule converge -s default
molecule verify -s default
```

## Execution Notes

### Privilege model

This role does not set `become`, `become_user`, or `become_method`. Package, service, repository, reboot, and filesystem operations are privileged on most real systems, so callers must set `become: true` externally when required.

Molecule containers run as root by default, so the Molecule converge play does not need role-level privilege escalation.

### Container and systemd behavior

The role avoids forcing service-manager behavior. Reboot tasks are skipped for Docker and Podman connection contexts.

The `rhnsd` task uses the service module and may require a functional service manager. The `molecule/systemd` scenario exists to provide coverage for service-manager-sensitive behavior. If a target does not provide `rhnsd` or a compatible service manager, the task is non-fatal by design.

### APT cache handling

Minimal Debian and Ubuntu container images may have an empty `/var/lib/apt/lists` directory after preparation or cleanup. The role detects whether APT package index files exist before installing or updating packages.

- If indexes are missing, the role refreshes the APT cache without relying on `pm_cache_valid_time`.
- If indexes are present, the role uses `pm_cache_valid_time` to avoid unnecessary cache refreshes.

This keeps APT behavior reliable in minimal containers while preserving cache efficiency on normal hosts.

### Optional GPG fingerprint review

APT key fingerprint review is optional. The role checks whether the `gpg` command is available before running fingerprint inspection. If `gpg` is not installed, fingerprint review is skipped without failing the role.

Install GnuPG externally or include it in `pm_install_packages` when fingerprint inspection is required:

```yaml
pm_install_packages:
  - gnupg
```

### Reboot behavior

The role reports reboot requirements where supported by the platform. It only reboots when all of the following are true:

- `pm_reboot_after_update` is `true`.
- The platform reports that a reboot is required.
- The Ansible connection is not a Docker or Podman container connection.

### Package removal safety

`pm_remove_packages` defaults to an empty list. Package removals happen only when the caller explicitly provides package names. Review removal lists carefully before using them on production systems.
