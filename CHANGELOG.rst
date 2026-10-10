Changelog
=========

This project follows `Semantic Versioning <https://semver.org/>`_.

0.1.0
-----

- Accept null or empty Docker port bindings in internal-container integration
  tests while independently rejecting publish-all-ports configuration.
- Add opt-in Traefik REST, MCP and agent API middleware chains with fixed
  rate/concurrency budgets, streaming-safe MCP/agent profiles and API headers;
  preserve the legacy ``security-chain`` and support custom source templates.
- Apply merged Agentgateway labels to Docker containers.
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
- Add opt-in stale instance lifecycle cleanup for MCP server and agents runtime
  roles.
- Add the ``docker_maintenance`` role for explicit Docker resource pruning.
- Add the ``system_base`` role for reusable Ubuntu host preparation.
- Add the ``system_updates`` role for explicit package updates and optional
  reboot handling.
- Add Ubuntu host bootstrap with Python detection, locale, NTP, troubleshooting
  tools, and emergency disk reserve support.
