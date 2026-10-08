# Repository instructions

## Project

This repository contains the `system4u.infra` Ansible collection. Roles are
intended to be reusable building blocks for Docker-based infrastructure.

## Collection identity

- Namespace: `system4u`
- Collection name: `infra`
- Fully qualified role names use `system4u.infra.<role>`.
- Collection metadata lives in `galaxy.yml`.
- Collection dependencies are declared in both `galaxy.yml` and
  `requirements.yml` when applicable.
- Update `CHANGELOG.rst` for every user-visible role, variable, behavior,
  dependency, CI, or documentation change. Keep entries grouped by release
  and use reStructuredText syntax.

## Role structure

Use the standard role layout:

```text
roles/<role_name>/
├── defaults/main.yml
├── meta/main.yml
├── meta/argument_specs.yml
├── tasks/main.yml
├── templates/
├── files/
└── README.md
```

Split task files by responsibility. Keep `tasks/main.yml` as a readable
orchestration layer and use descriptive action-oriented names such as
`configure_files.yml`, `create_networks.yml`, `create_<resource>.yml`, and
`remove_<resource>.yml`.

## Naming and variables

- Role names use lowercase `snake_case`; Docker roles use the `docker_` prefix.
- Every role variable must use the role prefix, for example
  `docker_traefik_` or `docker_redis_cache_`.
- Do not introduce unprefixed role variables. `ansible-lint` enforces this.
- Shared infrastructure values may come from inventory/group variables. Current
  conventions include:
  - `group_vars/all/system.yml`: `system_timezone`
  - `group_vars/all/ssl.yml`: `ssl_acme_email`
  - `group_vars/grp_docker_hosts.yml`: Docker user, group, UID/GID and
    `docker_data_basepath`
- Do not duplicate shared inventory values in every role. Derive role-local
  values from the shared variables when appropriate and document the contract.
- Use numeric UID/GID values when they represent container file ownership.

## Docker conventions

- Use `community.docker` modules with fully qualified collection names.
- Do not publish service ports to the host unless explicitly required. Defaults
  should use an empty `published_ports` list for internal services.
- Create only the Docker networks a role owns or explicitly declares.
- Network names use the `-net` suffix, for example `redis-cache-net` and
  `traefik-egress-net`.
- Use `internal: true` for networks that should not provide external
  connectivity, such as Redis cache or Docker socket proxy networks.
- Keep backend services on separate networks when network isolation is
  required. A shared reverse-proxy network must not be introduced merely for
  convenience.
- Traefik-specific network conventions:
  - `traefik-docker-socket-proxy-net` is an `internal: true` control-plane
    network shared only by Traefik and the Docker socket proxy.
  - `traefik-egress-net` is Traefik's non-internal outbound network. Backend
    containers must not be attached to it by default.
  - Each routed backend should use a dedicated ingress network and, when
    outbound connectivity is required, a dedicated egress network. For example:
    `agentgateway-a-ingress-net` and `agentgateway-a-egress-net`.
  - The Traefik role owns and creates declared backend ingress networks as
    `internal: true`. Backend service roles must not create or remove their
    ingress network; they only attach their container to the declared network.
  - A backend ingress network is shared only by Traefik and that backend. The
    backend service role owns its dedicated egress network, which must not be
    shared with other backends.
  - Traefik may join each backend ingress network, but must not join backend
    egress networks. Backend containers must not join another backend's
    networks.
  - Containers attached to multiple networks must set an explicit
    `traefik.docker.network` label so Traefik selects the intended ingress
    network.
  - Do not introduce a shared `traefik-ingress-net` merely for routing when
    backend-to-backend isolation is required.
  - The optional Whoami debug container uses its own isolated network and must
    remain disabled by default.
- Containers should use conservative defaults where compatible: read-only root
  filesystem, `no-new-privileges`, dropped capabilities, and writable mounts
  only where required.
- Use `no_log: true` for container tasks that receive secrets in environment
  variables, labels, commands, or module arguments.
- Secrets belong in Ansible Vault, not in role defaults or plain-text examples.

## Configuration and defaults

- Keep `defaults/main.yml` focused on actual defaults and short comments.
- Put long examples and usage guidance in the role README, not in defaults.
- Make destructive operations opt-in and explicit. Disabling a feature should
  not delete existing containers, networks, or data unless a separate cleanup
  variable is enabled.
- For generated directories, clearly document ownership and cleanup behavior.
- If a role replaces generated configuration, keep the target directory
  managed by the role and provide a warning README when useful.
- Prefer explicit, pinned image tags for stateful or production services.
  Stateless debug images may use `latest` only when documented and disabled by
  default.

## Helper role

Use `helper_merge_kv` for string key-value dictionaries such as container
labels and environment variables. It supports defaults, overrides, and removal
sentinels. Keep sensitive merge tasks protected with `no_log`.

## Documentation and tests

Every reusable role should have:

- `meta/main.yml` with supported platforms and minimum Ansible version,
- `meta/argument_specs.yml` for public variables,
- a role README with requirements, defaults, examples, security notes, and
  lifecycle/cleanup behavior,
- lint-clean tasks,
- integration coverage for important configuration branches.

When adding a feature, update defaults, argument specs, README, and tests
consistently.

## Validation commands

Before submitting changes, run:

```bash
ansible-lint .
git diff --check
ansible-galaxy collection build --force
```

Remove the generated collection archive after validation if it is not being
published. CI also runs `ansible-test sanity --docker` on the collection.

## Scope and changes

- Read existing files before editing them.
- Prefer small, focused changes over unrelated refactors.
- Preserve existing public variable names unless a deliberate breaking change
  is documented.
- Do not add application-specific resources to a generic base role. Put
  service-specific networks, containers, and configuration in the corresponding
  service role.
- Keep the working tree free of generated archives and temporary files.
