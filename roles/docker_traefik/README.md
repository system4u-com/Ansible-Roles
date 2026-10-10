# docker_traefik

Deploys Traefik with a Docker socket proxy and isolated backend networks.

## Requirements

The role is intended for hosts in the `grp_docker_hosts` inventory group.
The base Docker role must run before this role so that the shared container
user and group exist.

Common variables are maintained in the following inventory variable files:

```text
group_vars/all/system.yml              # system_timezone
group_vars/all/ssl.yml                 # ssl_acme_email
group_vars/grp_docker_hosts.yml       # docker_user, docker_group,
                                      # docker_uid, docker_gid,
                                      # docker_data_basepath
```

For example:

```yaml
# group_vars/all/system.yml
system_timezone: Europe/Prague

# group_vars/all/ssl.yml
ssl_acme_email: admin@example.com

# group_vars/grp_docker_hosts.yml
docker_user: docker4u
docker_group: docker4u
docker_uid: 9000
docker_gid: 9000
docker_data_basepath: /opt/docker4u
```

The role derives its local values from these shared variables:

```yaml
docker_traefik_acme_email: "{{ ssl_acme_email }}"
docker_traefik_container_data_basepath: "{{ docker_data_basepath }}/traefik"
```

## Dynamic configuration

Traefik also watches a read-only mounted directory for dynamic configuration.
The role ships with default templates in `templates/dynamic/`:

```yaml
docker_traefik_dynamic_config_source_path: ""
docker_traefik_dynamic_config_basepath: /opt/docker4u/traefik/dynamic
docker_traefik_dynamic_config_container_path: /etc/traefik/dynamic
docker_traefik_dynamic_config_watch: true
docker_traefik_dynamic_config_cleanup: true
```

An empty source path uses the role defaults. To replace them for one host, set
the path in that host's variables:

```yaml
# host_vars/edge-01.yml
docker_traefik_dynamic_config_source_path: >-
  {{ playbook_dir }}/templates/edge-01/traefik
```

The replacement directory should contain the complete desired configuration:

```text
templates/
└── edge-01/
    └── traefik/
        ├── middlewares.yml.j2
        └── tls.yml.j2
```

The role deploys all top-level `.yml`, `.yaml`, `.yml.j2`, and `.yaml.j2`
files from the selected source directory. The `.j2` suffix is removed at the
destination. With cleanup enabled, stale generated files are removed from the
target directory. The target directory contains a generated `README.md` warning
that it is managed by the role; do not make manual changes there. The Docker socket proxy network (`traefik-docker-socket-proxy-net`) is an
internal control-plane network. Only Traefik and the socket proxy should use
it; backend containers should remain on their own backend networks.

Traefik also uses a dedicated non-internal egress network
(`traefik-egress-net`) for Let's Encrypt, DNS provider APIs, and other outbound
traffic. Backend containers are not attached to this network by the role.

The role provides reusable default files for:

- `middlewares.yml.j2`: security headers, compression, and a reusable
  `security-chain`.
- `tls.yml.j2`: a `modern-tls` TLS option with TLS 1.2 as the minimum.
- `middlewares-strict.yml.j2`: an optional fixed `strict-chain` with body,
  request-rate and concurrent-request limits.

These definitions are not applied automatically to every router. Attach them
explicitly to a router through Docker labels:

```yaml
labels:
  traefik.http.routers.app.middlewares: security-chain@file
  traefik.http.routers.app.tls.options: modern-tls@file
```

This avoids changing backend behavior implicitly. If a host-specific source
path is configured, it replaces these defaults, so copy and adapt the files
that should remain enabled.

The directory is mounted read-only into the Traefik container.

The default `security-chain` enables HSTS preload. Use it only after confirming
that the domain and all subdomains are permanently served over HTTPS.

### Optional strict profile

The existing `security-chain@file` provides headers and compression. Select
`strict-chain@file` instead to add a 1 MiB request body limit (128 KiB in memory),
120 requests per minute per client remote IP (burst 30), and 20 concurrent HTTP
requests per hostname, followed by the existing security headers. Long-lived
streaming requests count toward concurrency; these are not TCP connection limits.
The HSTS domain requirement above applies to both chains.

```yaml
docker_agentgateway_container_labels_override:
  - key: traefik.enable
    value: "true"
  - key: traefik.http.routers.gateway.middlewares
    value: strict-chain@file
  # Define hostname, entrypoint, TLS and backend service labels separately.
```

The profile is optional and its fixed thresholds are a starting point, not
universal best practice. No router receives it automatically. For different
limits, use `docker_traefik_dynamic_config_source_path` with your own templates;
the custom directory replaces all shipped files, so include required headers
and TLS definitions too. No additional role parameters are needed.

## Security defaults

The role applies conservative container defaults:

- Traefik runs with a read-only root filesystem.
- Traefik drops all capabilities and adds only `NET_BIND_SERVICE`.
- Traefik and the Docker socket proxy use `no-new-privileges`.
- The Docker socket proxy uses a read-only Docker socket mount.
- The Traefik dashboard API is disabled by default.

Enable the dashboard only together with an authenticated, explicitly configured
router:

```yaml
docker_traefik_enable_dashboard: true
```

TLS options, security headers, and other HTTP middleware are intentionally not
part of these container defaults. Configure them through Traefik dynamic
configuration and middleware.

## ACME CA server and staging

The default empty value uses the configured ACME provider default. During
initial deployment or DNS validation troubleshooting, use the Let's Encrypt
staging directory to avoid production rate limits:

```yaml
docker_traefik_acme_ca_server: >-
  https://acme-staging-v02.api.letsencrypt.org/directory
```

Staging certificates are not trusted by normal clients. Switch back to
production only after validation succeeds:

```yaml
docker_traefik_acme_ca_server: >-
  https://acme-v02.api.letsencrypt.org/directory
```

The ACME storage file is persistent. When switching between staging and
production, use separate storage files or remove the staging `acme.json` after
stopping Traefik; otherwise Traefik may continue using certificates issued by
the previous CA.

## ACME DNS challenge

HTTP challenge is used by default:

```yaml
docker_traefik_acme_challenge_type: http
```

For DNS validation, configure the challenge type and provider:

```yaml
docker_traefik_acme_challenge_type: dns
docker_traefik_acme_dns_provider: cloudflare
docker_traefik_container_environment_override:
  - key: CF_DNS_API_TOKEN
    value: "{{ vault_cloudflare_dns_api_token }}"
```

The environment variable names depend on the selected Traefik DNS provider.
Store credentials in Ansible Vault. Do not put API tokens in role defaults or
plain-text inventory. Optional DNS propagation settings are available:

```yaml
docker_traefik_acme_dns_resolvers:
  - "1.1.1.1:53"
  - "8.8.8.8:53"
docker_traefik_acme_dns_delay_before_check: 10
```

The role passes `TZ` automatically through
`docker_traefik_container_environment_default`. Additional environment values
or provider credentials should be supplied through
`docker_traefik_container_environment_override`.

## Log level

The default log level is `INFO`. Override it for troubleshooting when needed:

```yaml
docker_traefik_log_level: DEBUG
```

## Backend network model

The role creates the networks listed in
`docker_traefik_backend_ingress_networks` as `internal: true` before creating
the Traefik container. Each backend should use its own ingress network.
Traefik is attached to all declared backend ingress networks, while each
backend container should be attached only to its own ingress and dedicated
egress network. Backend service roles must not create or remove their ingress
network; the Traefik role owns it.

```yaml
docker_traefik_backend_ingress_networks:
  - agentgateway-a-ingress-net
  - agentgateway-b-ingress-net
```

A backend container connected to more than one network must explicitly select
its Traefik network:

```yaml
labels:
  traefik.enable: "true"
  traefik.docker.network: agentgateway-a-ingress-net
```

The backend role owns and creates its dedicated egress network, but it only
attaches to the ingress network declared by the Traefik role. This prevents
Traefik from selecting the wrong network when a container has multiple network
attachments.

## Dashboard labels

The dashboard is disabled by default. Enable it by overriding the default
labels. The role does not provide a default dashboard password. Store the
password hash in Ansible Vault rather than using a plaintext password in
inventory:

```yaml
docker_traefik_container_labels_override:
  - key: "traefik.enable"
    value: "true"
  - key: "traefik.http.routers.dashboard.entrypoints"
    value: "websecure8443"
  - key: "traefik.http.routers.dashboard.rule"
    value: "Host(`traefik.example.com`)"
  - key: "traefik.http.routers.dashboard.tls"
    value: "true"
  - key: "traefik.http.routers.dashboard.tls.certresolver"
    value: "le"
  - key: "traefik.http.routers.dashboard.service"
    value: "api@internal"
  - key: "traefik.http.routers.dashboard.middlewares"
    value: "auth"
  - key: "traefik.http.middlewares.auth.basicauth.users"
    value: "admin:{{ vault_traefik_dashboard_password_hash }}"
```

## Whoami debug container

The stateless `traefik/whoami:latest` container is disabled by default:

```yaml
docker_traefik_enable_whoami: false
```

To enable it:

```yaml
docker_traefik_enable_whoami: true
docker_traefik_whoami_container_labels_override:
  - key: "traefik.enable"
    value: "true"
  - key: "traefik.http.routers.whoami.rule"
    value: "Host(`whoami.example.com`)"
```

It uses its own `traefik-whoami-net` network. The role adds the
`traefik.docker.network` label automatically. Setting
`docker_traefik_enable_whoami: false` does not remove an existing container or
network. For explicit cleanup, set:

```yaml
docker_traefik_remove_whoami_when_disabled: true
```

The container is removed before its network.
