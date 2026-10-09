# system_base

Prepares a supported Ubuntu host for reusable infrastructure roles.

## Scope

The role first detects the operating system and APT without Python. It
supports Ubuntu only and fails before any package installation on other
operating systems. It bootstraps Python 3 with a raw APT command when Python is
missing, then:

- verifies Ubuntu,
- installs the Python runtime and common Python libraries required by Ansible,

- installs the standard base, Ansible, and troubleshooting package set,
- configures the system timezone,
- enables and starts the configured NTP service,
- optionally configures the hostname.

The role does not create VMs, install Docker, configure SSH hardening, mount
data disks, or deploy application containers.

## Defaults

```yaml
system_base_packages:
  - ca-certificates
  - curl
  - dnsutils
  - gnupg
  - iproute2
  - iputils-ping
  - jq
  - locales
  - netcat-openbsd
  - openssl
  - python3
  - python3-apt
  - python3-jmespath
  - systemd-timesyncd
  - unzip
  - vim

system_base_timezone: "{{ system_timezone }}"
system_base_locale: "en_US.UTF-8"
system_base_manage_hostname: false
system_base_hostname: "{{ inventory_hostname }}"
system_base_disk_reserve_enabled: false
system_base_disk_reserve_path: /var/lib/system-base/disk-reserve
system_base_disk_reserve_size: 1GiB
```

Hostname management is disabled by default because cloud-init, inventory, or a
cloud provider may own the hostname. Enable it only when this role should own
hostname configuration.

The standard base setup installs `systemd-timesyncd`, configures the shared
system timezone, generates the standard UTF-8 locale, and enables time
synchronization by default. It also includes basic troubleshooting tools such
as `dig`, `nslookup`, `ping`, `ip`, `nc`, `openssl`, and `jq`. It sets `LANG`
but intentionally does not set
`LC_ALL` globally.

## Emergency disk reserve

The role can optionally create a root-owned reserve file that can be deleted
when a host filesystem is full:

```yaml
system_base_disk_reserve_enabled: true
system_base_disk_reserve_path: /var/lib/system-base/disk-reserve
system_base_disk_reserve_size: 1GiB
```

This feature is disabled by default. Delete the file manually to release the
reserved space; do not enable it as a replacement for disk monitoring or log
retention.

## Usage

```yaml
- name: Prepare infrastructure hosts
  hosts: grp_docker_hosts
  become: true
  roles:
    - role: system4u.infra.system_base
```

Run this role before `system_updates`, Docker, and service roles. VM creation
itself belongs to Terraform, OpenTofu, cloud-init, or the relevant
infrastructure provider.
