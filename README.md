[![CI](https://github.com/guidugli/ansible-role-package_manager/actions/workflows/CI.yml/badge.svg)](https://github.com/guidugli/ansible-role-package_manager/actions/workflows/CI.yml)
[![Release](https://img.shields.io/github/v/release/guidugli/ansible-role-package_manager?display_name=tag&sort=semver)](https://github.com/guidugli/ansible-role-package_manager/releases)
[![Galaxy](https://img.shields.io/badge/galaxy-guidugli.package_manager-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/guidugli/package_manager/)
[![License](https://img.shields.io/github/license/guidugli/ansible-role-package_manager)](LICENSE)

# Ansible Role: package_manager

`guidugli.package_manager` configures package-management behavior and manages package updates, installs, removals,
cleanup, GPG-check enforcement, and reboot signaling across supported Linux package managers.

## Requirements

- Ansible Core 2.17 for the included development and Molecule toolchain.
- Supported package managers: `apt`, `dnf`, `dnf5`, `yum`, `apk`, `pacman`, and `zypper`.
- Collections from `requirements.yml`:
  - `containers.podman >= 1.10.0` for Molecule container scenarios.
  - `community.general >= 9.0.0` for non-core package manager modules.
- Host privilege is controlled by the playbook or automation platform, not by this role.

## Variables

All role inputs are defined in `defaults/main.yml` and mirrored in `meta/argument_specs.yml`.

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `pm_update_system` | bool | `true` | Updates installed packages using the platform package manager. |
| `pm_apt_upgrade_command` | str | `dist` | Upgrade mode for APT hosts. Valid values are `dist`, `full`, `safe`, and `yes`. |
| `pm_apk_upgrade_available` | bool | `false` | Allows APK to replace or downgrade packages when the installed version is unavailable. |
| `pm_cache_valid_time` | int | `3600` | Package metadata cache validity in seconds for package managers that support it. |
| `pm_security_only` | bool | `false` | Applies security updates only on RedHat-family package-manager tasks. |
| `pm_autoremove` | bool | `true` | Removes unneeded dependency packages where supported by the package manager. |
| `pm_purge` | bool | `false` | Purges configuration files when packages are removed on Debian-family systems. |
| `pm_gpgcheck` | bool | `true` | Enforces `gpgcheck = 1` in RedHat-family package manager and repository configuration. |
| `pm_disable_rhnsd` | bool | `true` | Stops and disables `rhnsd` on supported RedHat-family hosts when the service task runs. |
| `pm_gpgcheck_excludes` | list(str) | `[]` | Repository files excluded from RedHat-family GPG-check enforcement. |
| `pm_install_packages` | list(str) | `[]` | Packages to install. |
| `pm_remove_packages` | list(str) | `[]` | Packages to remove. |
| `pm_reboot_after_update` | bool | `false` | Reboots when the platform indicates a reboot is required after updates. |

Internal variables are stored in `vars/main.yml`:

| Variable | Description |
| --- | --- |
| `int_default_value` | Internal sentinel value used for integer validation. |
| `default_packages` | Platform-specific package install and removal maps. Review before using in production. |
| `pkgmgr_redhat_conf` | RedHat-family package manager configuration file path. |

## Example Playbook

Package-management changes usually require elevated privileges on real hosts. Apply privilege at the play level:

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

## Molecule Testing

The role is aligned to shared Molecule playbooks:

- `molecule/shared/converge.yml` applies the role to every scenario target.
- `molecule/shared/verify.yml` collects package facts and validates that package facts are available.
- `molecule/default` exercises containerized distro targets.
- `molecule/systemd` prepares systemd-capable containers for service-related coverage.

Run locally with:

```bash
molecule test -s default
molecule test -s systemd
```

## Execution Notes

### Privilege model

This role does not set `become`, `become_user`, or `become_method`. Package, service, repository, reboot, and
filesystem operations are privileged on most real systems, so callers must set `become: true` externally when required.
Containerized Molecule targets run as root by default and do not require role-level privilege escalation.

### Container and systemd behavior

The role avoids forcing service-manager behavior. Reboot tasks are skipped for Docker and Podman connection contexts.
The `rhnsd` task uses the service module and may require a functional service manager; the systemd Molecule scenario
exists to exercise service-manager-sensitive behavior. If a target does not provide `rhnsd` or a compatible service
manager, the task is non-fatal by design.
