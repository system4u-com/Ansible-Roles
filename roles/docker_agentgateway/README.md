# docker_agentgateway

Deploys an agentgateway instance as an HTTP backend behind Traefik.
Traefik terminates TLS and handles Let's Encrypt certificates; this role does
not configure TLS or publish ports to the host by default.

## Network model

The Traefik role owns and creates the internal ingress network:

```yaml
docker_agentgateway_ingress_network: agentgateway-a-ingress-net
```

When the Traefik frontend is enabled, the ingress network must already
exist and be declared in `docker_traefik_backend_ingress_networks`. This role
does not create or remove it.

The Traefik frontend can be disabled for direct published-port operation:

```yaml
docker_agentgateway_enable_traefik_frontend: false
docker_agentgateway_container_published_ports:
  - "127.0.0.1:4000:4000"
```

In that mode the ingress network is not required and no Traefik labels are
added. The agentgateway still uses its own egress and downstream networks.

This role creates its own egress network and optional downstream/MCP networks:

```yaml
docker_agentgateway_egress_network: agentgateway-a-egress-net
docker_agentgateway_downstream_networks:
  - agentgateway-a-mcp-files-net
  - agentgateway-a-mcp-github-net
```

The container is attached to its ingress, egress, and downstream networks.
Traefik is attached only to the ingress network. MCP service roles should
attach only to their own downstream network.

## Example topology

```text
agentgateway-a-ingress-net  internal: true
├── traefik
└── agentgateway-a

agentgateway-a-egress-net
└── agentgateway-a

agentgateway-a-mcp-files-net  internal: true
├── agentgateway-a
└── mcp-files
```

## Traefik labels

Routing is disabled by default. Enable it explicitly:

```yaml
docker_agentgateway_container_labels_override:
  - key: "traefik.enable"
    value: "true"
  - key: "traefik.http.routers.agentgateway-a.rule"
    value: "Host(`agentgateway-a.example.com`)"
  - key: "traefik.http.routers.agentgateway-a.entrypoints"
    value: "websecure"
  - key: "traefik.http.routers.agentgateway-a.tls"
    value: "true"
  - key: "traefik.http.routers.agentgateway-a.tls.certresolver"
    value: "le"
  - key: "traefik.http.services.agentgateway-a.loadbalancer.server.port"
    value: "4000"
```

When the Traefik frontend is enabled, the role supplies the explicit
`traefik.docker.network` label using the configured ingress network. This does
not enable routing by itself; set `traefik.enable` and the router/service
labels explicitly through `docker_agentgateway_container_labels_override`.

## Configuration

The default configuration is stored in the role template and provides:

- an HTTP gateway listener on port `4000`;
- no database;
- no UI;
- no local configuration storage.

The gateway is intentionally a stateless HTTP proxy. The default configuration
only creates the gateway listener; it does not configure MCP servers, agents,
routes, or backends. Provide an instance-specific configuration through
`docker_agentgateway_config_source_path`. TLS termination and Let's Encrypt
certificates are handled by Traefik.

This first version does not configure the agentgateway UI or database. Add
those features only as a deliberate extension when their authentication and
storage requirements are defined. Such an extension would require separate
writable data/database storage instead of making the whole `/config` mount
writable.

Override the source file for a host or instance:

```yaml
docker_agentgateway_config_source_path: >-
  {{ playbook_dir }}/templates/{{ inventory_hostname }}/agentgateway-a/config.yaml.j2
```

Source paths are controller-local and checked without privilege escalation.
Destination files are created on the managed host under its normal become policy.
The role renders the source into the read-only `/config` mount. The current
stateless configuration does not enable UI or database features and therefore
does not need container-side writes.

## Additional mounts

`docker_agentgateway_container_extra_volumes` is an empty list by default.
It appends Docker volume bindings without replacing the read-only `/config`
mount. For example, a deployment-managed secret directory can be mounted with:

```yaml
docker_agentgateway_container_extra_volumes:
  - /opt/docker4u/gateway-secrets:/etc/agentgateway/secrets:ro
```

The deployment must create source paths before running the role and owns their
content, permissions, rotation and removal. For secrets, use a private directory
owned by root with mode 0750 and files with mode 0640, readable by the container's
numeric group but not writable by its non-root user. Use `:ro` explicitly; do
not mount over `/config` or expose another instance's secrets. This role neither
fetches credentials nor changes ownership of additional mount sources.

## Multiple instances

Include the role once per instance with distinct names, networks, data paths,
and labels. Each instance must have its own ingress and egress networks:

```yaml
- name: Deploy Agentgateway A
  ansible.builtin.include_role:
    name: system4u.infra.docker_agentgateway
  vars:
    docker_agentgateway_container_name: agentgateway-a
    docker_agentgateway_ingress_network: agentgateway-a-ingress-net
    docker_agentgateway_egress_network: agentgateway-a-egress-net
```
