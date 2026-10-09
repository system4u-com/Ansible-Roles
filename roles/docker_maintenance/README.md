# docker_maintenance

Explicitly prunes unused Docker resources. All prune operations are disabled
by default because pruning is destructive and cannot be undone through Docker.

## Usage

Enable only the resource types that should be cleaned:

```yaml
- name: Run Docker maintenance
  hosts: grp_docker_hosts
  become: true
  roles:
    - role: system4u.infra.docker_maintenance
      vars:
        docker_maintenance_prune_containers: true
        docker_maintenance_prune_images: true
```

The role performs one Docker prune operation. It does not inspect or delete
resources owned by a particular Ansible role, and it does not remove running
containers.

## Defaults

```yaml
docker_maintenance_prune_containers: false
docker_maintenance_prune_images: false
docker_maintenance_prune_networks: false
docker_maintenance_prune_volumes: false
docker_maintenance_prune_builder_cache: false
```

By default, stopped containers older than seven days are eligible when
container pruning is enabled:

```yaml
docker_maintenance_containers_filters:
  until: 168h
```

By default, only dangling images are pruned:

```yaml
docker_maintenance_images_filters:
  dangling: true
```

To prune all unused images, including tagged images that are not used by a
container, explicitly set:

```yaml
docker_maintenance_images_filters:
  dangling: false
```

## Networks and volumes

Network pruning removes only unused Docker networks. A network with an active
container endpoint is not removed; Docker returns an error instead of silently
disconnecting the container.

Volume pruning can delete unused volumes and must be enabled deliberately.
Review the target host before enabling it.

## Recommended lifecycle

Run this role as a separately scheduled maintenance play, not automatically as
part of every application deployment:

```yaml
- name: Scheduled Docker maintenance
  hosts: grp_docker_hosts
  become: true
  roles:
    - role: system4u.infra.docker_maintenance
      vars:
        docker_maintenance_prune_containers: true
        docker_maintenance_prune_images: true
        docker_maintenance_prune_networks: true
```

Do not enable volume pruning without an explicit data-retention policy.
