Ansible Role: package_manager
=========

An Ansible Role that configure package management software on RHEL/CentOS, Fedora and Debian/Ubuntu. Also install and remove software according to configuration.

Requirements
------------

No requirements.

Role Variables
--------------

**Available variables are listed below, along with default values (see defaults/main.yml):**

    pm_update_system: yes

Update packages to latest version whenever this role is executed

    pm_apt_upgrade_command: dist

For APT only: can be dist, full, yes or safe

    pm_apk_upgrade_available: no

If set to yes, during upgrade, prefer replacing or downgrading packages (instead of holding them) if the currently installed package is no longer available from any repository.

pm_cache_valid_time: 3600

Update cache if older than this variable value (in seconds)

    pm_security_only: no

Apply only security updates (RedHat family only)

    pm_autoremove: yes

Automatically remove unneeded packages installed as dependencies

    pm_purge: no

Force purging configuration files if package is removed (Debian family only)

    pm_gpgcheck: yes

Make all repositories to validate gpg signature (RedHat family only)

    pm_disable_rhnsd: yes

Do not disable the service if you plan on connecting to a Sattelite ((RedHat family only)

    pm_install_packages: "{{ default_packages['install']['COMMON'] |
                        default([], true) + default_packages['install'][ansible_os_family] | default([], true) }}"

Packages to install. Default is no packages to install. Update this variable to specify the desired packages to install.

    pm_remove_packages: "{{ default_packages['remove']['COMMON'] |
                        default([], true) + default_packages['remove'][ansible_os_family] | default([], true) }}"

Packages to remove: default list is based on CIS recommendations of software to uninstall if not used. Check `vars/main.yml` for the list of packages of each operating system. By default it will remove specialized software that may run on a server like samba, nfs, http server, etc, since most installation don´t use them, and they may act as a vector for vulnerabilities. Ensure you review and adjust the list on each system.

    pm_reboot_after_update: no

Should the system be rebooted after update?

    #  pm_gpgcheck_excludes:
    #    - /etc/yum.repos.d/virtio-win

RedHat family only: repositories to skip gpg check. Some repositories (like virtio-win) may not have a gpg signature available. Use this option to disable gpg check of the problematic repository.

**The variables listed below do not need to be changed for targeted systems (see vars/main.yml):**

    int_default_value: -1

Internal variable that define the default value of undefined integers. This value should not match any valid value of integer variables. Do not change it unless you know the internals of this role.

    default_packages:

This is a complex structure that defines the default packages to be installed and removed based on the operating system type and version.

    pkgmgr_redhat_conf:

Configuration file location and name for RedHat based systems.

Dependencies
------------

No dependencies.

Example Playbook
----------------

    - hosts: servers
      roles:
         - { role: guidugli.package_manager }

License
-------

MIT / BSD

Author Information
------------------

This role was created in 2020 by Carlos Guidugli.
