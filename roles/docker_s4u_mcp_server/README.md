# docker_s4u_mcp_server

Deploys one or more deployment-specific System4u MCP server containers. The
role manages one container per item in `docker_s4u_mcp_server_instances`.

The role is intended for MCP servers behind an agentgateway. It does not add
Traefik labels, connect MCP servers directly to Traefik, or publish backend
ports on the host.

## Usage

Define the instances in the deployment project and invoke the role once:

```yaml
# group_vars/grp_docker_hosts/mcp_servers.yml
docker_s4u_mcp_server_instances:
  - name: s4u-mcp-vivantio
    image: ghcr.io/system4u-com/s4u-mcp-vivantio:2026.10.0
    ingress_network: s4u-mcp-vivantio-ingress-net
    egress_network: s4u-mcp-vivantio-egress-net
    config_source_path: >-
      {{ playbook_dir }}/templates/s4u-mcp-vivantio/config
    dependency_networks:
      - s4u-mcp-vivantio-redis-net
    environment:
      - key: TZ
        value: "{{ system_timezone }}"
      - key: S4U_MCP_REDIS_URL
        value: redis://redis-cache:6379

  - name: s4u-mcp-bookstack
    image: ghcr.io/system4u-com/s4u-mcp-bookstack:2026.10.0
    ingress_network: s4u-mcp-bookstack-ingress-net
    egress_network: s4u-mcp-bookstack-egress-net
    config_source_path: >-
      {{ playbook_dir }}/templates/s4u-mcp-bookstack/config
    dependency_networks:
      - s4u-mcp-bookstack-redis-net
```

```yaml
- name: Deploy MCP servers
  hosts: grp_docker_hosts
  become: true
  roles:
    - role: system4u.infra.docker_s4u_mcp_server
```

Images are built and published by the deployment project's CI, for example to
GHCR. Use an immutable release tag or digest rather than `latest` in production.

## Resource names

The `name`, `ingress_network`, and `egress_network` fields are explicit final
Docker resource names. The role does not add prefixes or suffixes. This keeps
the network contract with `docker_agentgateway` visible in the deployment
configuration.

## Instance fields

The public variable is a list. Each item supports the following fields:

| Field | Required | Description |
| --- | --- | --- |
| `name` | yes | Final Docker container name |
| `image` | yes | Container image |
| `ingress_network` | yes | Final internal frontend network owned by agentgateway |
| `egress_network` | yes | Final dedicated egress network created by this role |
| `dependency_networks` | no | Existing dependency networks, for example a Redis cache network |
| `config_source_path` | yes | Controller-side configuration directory |
| `environment` | no | String key-value environment variables |
| `labels` | no | Container labels |
| `command` | no | Optional command override; omitted preserves the image default |
| `volumes` | no | Additional volume mappings |

The role uses the explicit frontend ingress and egress network names from
the instance. Dependency networks remain explicit and are neither created nor
inferred by the role.

## Network ownership

- The `docker_agentgateway` role owns and creates the internal frontend ingress
  network.
- This role verifies and attaches the MCP container to that network.
- This role creates and owns the derived dedicated egress network.
- Dependency networks must already exist and are only used by the container.
- This role does not create or remove frontend ingress or dependency networks.

The container uses its frontend ingress, egress, and dependency networks. MCP
servers do not publish host ports. The agentgateway is the frontend between
Traefik and the MCP server.

## Configuration and lifecycle

`config_source_path` points to a directory in the deployment project. Ansible
copies that directory to the managed host under the derived instance data
directory and mounts the resulting config directory read-only at `/app/config`.

The target config directory is managed by this role. Manual changes are not
persistent; update the source directory in the deployment project instead.
Stale target files are removed by default; set
`docker_s4u_mcp_server_config_cleanup: false` to disable that cleanup.

When an instance is removed from the list, its container, derived egress
network, and data directory are retained by default. Enable explicit lifecycle
cleanup to remove stale instances:

```yaml
docker_s4u_mcp_server_cleanup_stale_instances: true
```

Cleanup never removes frontend ingress or dependency networks, which are owned
by other roles.

The container runs with the shared `docker_uid:docker_gid` identity and uses a
read-only root filesystem, dropped capabilities, `no-new-privileges`, and a
hardened `/tmp` tmpfs.
