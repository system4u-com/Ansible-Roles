Changelog
=========

This project follows `Semantic Versioning <https://semver.org/>`_.

0.1.0
-----

- Initial collection with the ``docker`` role.
- Add the ``helper_merge_kv`` helper role.
- Add the ``docker_traefik`` role with ACME, isolated networks, and dynamic
  configuration support.
- Add the ``docker_redis_cache`` role for ephemeral Redis caching.
- Add lint, sanity, build, and Redis integration CI jobs.
- Skip CI workflow runs for documentation-only changes to Markdown and RST
  files.
- Add the ``docker_s4u_mcp_server`` role with plural instances and derived
  container/network names.
- Add the ``docker_s4u_agents_runtime`` role with plural instances and derived
  container/network names.
- Add deployment-managed configuration copy and stale-file cleanup for the
  MCP and agents runtime roles.
- Add Docker runtime integration tests for the MCP server and agents runtime
  roles.
