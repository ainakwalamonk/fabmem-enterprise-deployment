# Changelog

Image tags are pinned. Upgrade by changing `FABMEM_IMAGE` in `.env`, then
`docker compose pull && docker compose up -d`. Schema migrations apply on boot.

## Unreleased

- Initial self-hosted stack: app + optional bundled Postgres, env-driven configuration,
  build ingestion by pointer URL, MCP with static-token or WorkOS SSO access.
