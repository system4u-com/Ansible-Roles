# system4u.infra

Shared Ansible roles for infrastructure provisioning.

## Requirements

- Ansible Core >= 2.15
- Ubuntu 22.04 or 24.04 for the `docker` role
- A supported Docker architecture (for example `amd64` or `arm64`)

Install the collection dependencies with:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Usage

After installing the collection, use the Docker role with its fully qualified
collection name:

```yaml
---
- name: Configure Docker hosts
  hosts: docker_hosts
  become: true
  roles:
    - role: system4u.infra.docker
```

## Docker role

The role installs and configures Docker and creates a dedicated system user
and group for consistent ownership of bind-mounted data across containers.
The user is not granted access to the Docker CLI.

### Main variables

| Variable | Default | Description |
| --- | --- | --- |
| `docker_user` | `docker4u` | System user created by the role |
| `docker_uid` | `9000` | UID assigned to the system user |
| `docker_group` | `docker4u` | Primary system group created by the role |
| `docker_gid` | `9000` | GID assigned to the system group |
| `docker_apt_keyring` | `/etc/apt/keyrings/docker.asc` | Docker APT repository signing key |
| `docker_apt_keyring` | `/etc/apt/keyrings/docker.asc` | Docker APT repository signing key |
| `docker_apt_cache_ttl` | `3600` | APT cache validity in seconds |

### Bind-mount ownership

The role creates the host user and group with configurable numeric UID/GID.
Containers can use the same numeric identity to keep ownership consistent for
bind-mounted data:

```yaml
services:
  app:
    image: example/app
    user: "9000:9000"
    volumes:
      - /srv/app-data:/var/lib/app
```

The container does not need to contain a user named `docker4u`; bind-mount
permissions are based on the numeric UID/GID. The `docker4u` host user is not
added to the `docker` group and therefore cannot use the Docker CLI by default.

## Docker maintenance role

See [`roles/docker_maintenance/README.md`](roles/docker_maintenance/README.md)
for explicit Docker resource pruning and retention filters.

## Docker Traefik role

See [`roles/docker_traefik/README.md`](roles/docker_traefik/README.md) for
backend network isolation, dashboard labels, and the optional Whoami debug
container.

## Agentgateway role

See [`roles/docker_agentgateway/README.md`](roles/docker_agentgateway/README.md)
for Traefik integration and isolated ingress, egress, and downstream/MCP
networks.

## System4u MCP server role

See [`roles/docker_s4u_mcp_server/README.md`](roles/docker_s4u_mcp_server/README.md)
for plural instance deployment, derived resource names, isolated networks, and
deployment-managed configuration.

## System4u agents runtime role

See [`roles/docker_s4u_agents_runtime/README.md`](roles/docker_s4u_agents_runtime/README.md)
for plural instance deployment, derived resource names, isolated networks, and
deployment-managed configuration.

## Redis cache role

See [`roles/docker_redis_cache/README.md`](roles/docker_redis_cache/README.md) for
isolated network access, memory limits, eviction policy, authentication, and
ephemeral cache behavior.

## Helper role

The `helper_merge_kv` role merges string key-value lists and supports
last-write-wins overrides and key removal. See
[`roles/helper_merge_kv/README.md`](roles/helper_merge_kv/README.md) for usage
and variable documentation.
