# docker_redis_cache

Deploys Redis as an isolated, ephemeral Docker cache.

## Network access

The role creates only the networks listed in
`docker_redis_cache_networks`. Every network created by this role is always an
internal Docker network. Redis is not published on host ports by default:

```yaml
docker_redis_cache_networks:
  - redis-cache-net
docker_redis_cache_container_published_ports: []
```

Only application containers explicitly attached to one of these networks can
reach Redis. The role does not attach Redis to Traefik or any egress network.

## Cache defaults

This role intentionally does not configure Redis persistence. Redis data is
lost when the container is removed or recreated. That is appropriate for a
cache; applications must be able to repopulate the data from their source.

```yaml
docker_redis_cache_maxmemory: 512mb
docker_redis_cache_maxmemory_policy: allkeys-lru
docker_redis_cache_maxmemory_samples: 10
```

Applications should set TTLs on their own keys. `maxmemory-policy` controls
which keys Redis evicts when the configured memory limit is reached; it does
not set a TTL.

If a persistent Redis datastore is needed, use a separate role rather than
extending this cache role with persistence behavior.

## Authentication

Authentication is optional and disabled by default. The role enables a
container healthcheck by default; when authentication is enabled, the
healthcheck uses the configured `REDISCLI_AUTH` environment variable. Use
Ansible Vault for the password:

```yaml
docker_redis_cache_requirepass: "{{ vault_redis_cache_password }}"
```

## Shared container identity

The Redis process runs under the shared `docker_uid` and `docker_gid` values.
The generated configuration is readable by that identity and is not stored in
persistent application data.
