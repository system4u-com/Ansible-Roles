# docker_s4u_agents_runtime

Deploys one or more System4u agents runtime containers. Each container can
host multiple agents; the role manages one container per item in
`docker_s4u_agents_runtime_instances`.

The role is intended for runtime containers behind an agentgateway. It does
not add Traefik labels, connect runtime containers directly to Traefik, or
publish backend ports on the host.

## Usage

Define the instances in the deployment project and invoke the role once:

```yaml
# group_vars/grp_docker_hosts/agents_runtime.yml
docker_s4u_agents_runtime_instances:
  - name: support
    image: ghcr.io/system4u-com/s4u-agents-runtime:2026.10.0
    config_source_path: >-
      {{ playbook_dir }}/templates/s4u-agents-runtime/config
    dependency_networks: []
    environment:
      - key: TZ
        value: "{{ system_timezone }}"
```

```yaml
- name: Deploy agents runtimes
  hosts: grp_docker_hosts
  become: true
  roles:
    - role: system4u.infra.docker_s4u_agents_runtime
```

For multiple independent runtime containers, add more items to the same list.
Each item gets its own derived container, frontend ingress network, egress
network, data directory, and configuration directory.

Images are built and published by the deployment project's CI, for example to
GHCR. Use an immutable release tag or digest rather than `latest` in production.

## Derived resource names

The `name` field is a logical instance suffix, not the final Docker resource
name. With the default naming settings:

```yaml
name: support
```

produces:

```text
container:       s4u-agents-runtime-support
frontend ingress: s4u-agents-runtime-support-ingress-net
egress:           s4u-agents-runtime-support-egress-net
```

The prefix and suffixes can be changed through role defaults when a deployment
needs a different naming namespace.

## Instance fields

The public variable is a list. Each item supports the following fields:

| Field | Required | Description |
| --- | --- | --- |
| `name` | yes | Lowercase logical instance suffix |
| `image` | yes | Container image |
| `dependency_networks` | no | Existing dependency networks, for example a Redis cache network |
| `config_source_path` | yes | Controller-side runtime configuration directory |
| `environment` | no | String key-value environment variables |
| `labels` | no | Container labels |
| `command` | no | Optional command override; omitted preserves the image default |
| `volumes` | no | Additional volume mappings |

The role derives the frontend ingress and egress network names from `name`.
Dependency networks remain explicit and are neither created nor inferred by the
role.

## Network ownership

- The `docker_agentgateway` role owns and creates the internal frontend ingress
  network.
- This role verifies and attaches the runtime container to that network.
- This role creates and owns the derived dedicated egress network.
- Dependency networks must already exist and are only used by the container.
- This role does not create or remove frontend ingress or dependency networks.

The container uses its frontend ingress, egress, and dependency networks. It
does not publish host ports by default. The agentgateway is the frontend between
Traefik and the agents runtime.

## Configuration and lifecycle

`config_source_path` points to a directory in the deployment project. Ansible
copies that directory to the managed host under the derived instance data
directory and mounts the resulting config directory read-only at `/app/config`.

The target config directory is managed by this role. Manual changes are not
persistent; update the source directory in the deployment project instead.
Stale target files are removed by default; set
`docker_s4u_agents_runtime_config_cleanup: false` to disable that cleanup.

The container can host multiple agents and runs with the shared
`docker_uid:docker_gid` identity. It uses a read-only root filesystem, dropped
capabilities, `no-new-privileges`, and a hardened `/tmp` tmpfs.
