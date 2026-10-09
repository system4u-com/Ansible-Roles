# system_updates

Explicitly updates Ubuntu packages and optionally reboots the host.

All potentially disruptive operations are disabled by default.

## Defaults

```yaml
system_updates_update_cache: true
system_updates_cache_valid_time: 3600
system_updates_upgrade: false
system_updates_upgrade_type: safe
system_updates_autoremove: false
system_updates_reboot: false
system_updates_reboot_timeout: 600
```

The role checks `/var/run/reboot-required` and reboots only when both
`system_updates_reboot: true` and a reboot is required.

## Usage

```yaml
- name: Update infrastructure hosts
  hosts: grp_docker_hosts
  become: true
  roles:
    - role: system4u.infra.system_updates
      vars:
        system_updates_upgrade: true
        system_updates_reboot: false
```

For a full upgrade:

```yaml
system_updates_upgrade: true
system_updates_upgrade_type: full
```

Run this role as an explicit maintenance or provisioning step, not implicitly
from application roles. The package cache is refreshed when explicitly enabled
or whenever a package upgrade is requested.
