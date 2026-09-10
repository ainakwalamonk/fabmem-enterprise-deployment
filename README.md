# fabMem enterprise — self-hosted deployment

Code intelligence over your repositories, served to your developers' AI agents over MCP. Runs entirely inside
your infrastructure: your VM, your Postgres, your network. Source code is indexed in place and never leaves.

This repository contains only configuration. The application ships as a container image pulled from a private
registry with the credentials issued to you.

## What it does

1. Your CI uploads a built workspace to storage and tells fabMem where it is.
2. fabMem fetches it, runs compiler-grade SCIP indexing, and stores symbols + embeddings in Postgres.
3. Your developers' agents query it over MCP — `search_code`, `find_references`, `grep_code`, `read_file`.

## Requirements

- Linux host, x86-64, Docker with Compose v2
- 2 vCPU / 8 GB RAM minimum; disk sized for your source snapshots plus the index
- Nothing else. The container hosts its own PostgreSQL 17 — no database to provision
- Outbound HTTPS to your artifact storage (and to your SSO tenant, if you enable SSO)

## Install

```bash
docker login <registry>                      # credentials issued with your licence
cp .env.example .env                         # then edit — every required value is marked
docker compose pull
docker compose up -d
curl -s localhost:8080/health
```

Schema migrations apply automatically at boot. If the database is unreachable or a required variable is missing,
the container exits immediately rather than starting in a degraded state.

### Verifying the install

```bash
curl -s localhost:8080/ready | jq        # 200 when serving, 503 with the failing checks
docker compose exec app node enterprise/dist/doctor-cli.js   # same checks, human-readable, exits non-zero
```

`/ready` checks the database connection, both required extensions, the BM25 indexes, the schema, that `DATA_DIR`
is writable and persistent, the SSO configuration and its JWKS endpoint, and that the indexers and embedding
model are present in the image. Compose polls it, so a container that cannot serve reports **unhealthy** in
`docker compose ps` rather than looking fine.

`/health` is separate and deliberately shallow: it answers 200 whenever the process is running.

### The database

The container runs its own PostgreSQL 17 with `vector` (semantic search) and `pg_textsearch` (BM25 keyword
ranking) already provisioned. There is no database service to deploy and no password to manage.

Both extensions are required. `pg_textsearch` needs `shared_preload_libraries` and is not on the extension
allowlist of any major managed Postgres — checked September 2026 against Azure Flexible Server, RDS and Aurora.
Pointing `DATABASE_URL` at a managed instance still starts and still answers, but every keyword lane returns
nothing: the migration logs `pg_textsearch not available - BM25 indexes not created` and continues. Use your own
Postgres only where you control the server enough to install the module.

The container runs as uid 10001, not root, because Postgres refuses to `initdb` as root.

### Storage

One volume holds everything that must survive: the database, the extracted source snapshots, the indexer cache
and the embedding model. It survives container restarts, image upgrades and `docker compose down`; only
`down -v` destroys it.

```bash
FABMEM_DATA_PATH=/mnt/fabmem-data   # bind-mount your own disk; must be writable by uid 10001
```

Back it up as a unit. The source snapshots are rebuildable by re-sending builds, but the database is not.

## Sending builds

`POST /ingest/build?sphere=<project>` with the CI token. The body is a pointer, not an upload:

```bash
curl -X POST "https://fabmem.internal/ingest/build?sphere=default" \
  -H "Authorization: Bearer $FABMEM_INGEST_TOKEN" \
  -H "content-type: application/json" \
  -d '{"url":"https://…","repo":"acme/api","commit":"'"$GIT_SHA"'"}'
```

Returns `202 { buildId, status: "pending" }`. A worker then fetches the URL, extracts it, and indexes it.

**`url`** must be an HTTP(S) URL the container can reach, returning a tar archive (`.tar`, `.tar.gz`, `.tgz` —
compression is auto-detected). An **S3 presigned URL is the recommended form**: the container needs no AWS
credentials, no bucket policy and no IAM role, and the grant expires on its own.

```yaml
# GitHub Actions
- run: tar czf build.tgz .                      # the BUILT workspace (deps installed, artifacts present)
- run: aws s3 cp build.tgz s3://$BUCKET/$GITHUB_SHA.tgz
- run: |
    URL=$(aws s3 presign s3://$BUCKET/$GITHUB_SHA.tgz --expires-in 3600)
    curl -fsS -X POST "$FABMEM_URL/ingest/build?sphere=default" \
      -H "Authorization: Bearer ${{ secrets.FABMEM_INGEST_TOKEN }}" \
      -H 'content-type: application/json' \
      -d "$(jq -nc --arg u "$URL" --arg r "$GITHUB_REPOSITORY" --arg c "$GITHUB_SHA" \
             '{url:$u, repo:$r, commit:$c}')"
```

Any storage works — S3, GCS, Azure Blob, Artifactory, an internal HTTP file server — as long as the URL is
reachable from the container and the body is a tar archive.

Upload the **built** workspace: dependencies installed and build artifacts present. Indexing does not rebuild the
project or fetch dependencies; the toolchains in the image resolve symbols against what you send.

Re-posting the same repo replaces its snapshot. Send one build per repo per commit; `sphere` separates projects,
and each sphere is isolated at the database level.

## Connecting agents

Point an MCP client at `https://fabmem.internal/mcp` — one endpoint for the whole organisation. Every tool
takes a `sphere` argument (the id or name from the dashboard's Spheres page) naming the project to query; omit it
and the deployment's default sphere is used. Each call is authorized against the sphere it names.

**Token mode** (default) — pass `FABMEM_QUERY_TOKEN` as a bearer token. Fine for a small team or an evaluation.

**SSO mode** — set `WORKOS_ISSUER` in `.env`. `/mcp` then requires a WorkOS AuthKit token, developers sign in with
Google in their client, and there are no shared tokens to distribute or rotate. The server publishes OAuth
discovery at `/.well-known/oauth-protected-resource`; compatible clients find the login flow on their own.

```bash
claude mcp add --scope user --transport http fabmem https://fabmem.internal/mcp
```

Behind a load balancer, set `FABMEM_MCP_RESOURCE_URL` to the public HTTPS URL — discovery advertises this value to
clients, and deriving it from the `Host` header is wrong when a proxy rewrites it.

## Operations

**Upgrade** — change `FABMEM_IMAGE` to the new tag, then `docker compose pull && docker compose up -d`. Migrations
apply on boot. Read CHANGELOG.md first; tags are pinned deliberately and never `latest`.

**Backup** — the single data volume is the source of truth.

**Logs** — `docker compose logs -f app`. Structured JSON, one object per line.

**Health** — `GET /health` = process up. `GET /ready` = able to serve, with a per-check report; compose uses
`/ready`, so `docker compose ps` shows unhealthy while anything is misconfigured.

## Security

- Source code and indexes stay on your infrastructure. The container makes outbound connections only to the
  artifact URLs you send it and, if SSO is enabled, to your identity provider's public JWKS endpoint.
- No fabMem-operated telemetry, licence check or phone-home.
- Bootstrap credentials `FABMEM_INGEST_TOKEN` (CI, write) and `FABMEM_QUERY_TOKEN` (read) come from `.env`.
  Once running, issue per-caller API keys from the dashboard instead: revocable individually, last-used
  recorded, stored as salted scrypt hashes, and rotatable without restarting.
- With SSO enabled, `WORKOS_ALLOWED_DOMAINS` / `WORKOS_ALLOWED_ORG_ID` are what restrict who may sign in.
  Leaving both empty admits any authenticated user of that tenant.
- Terminate TLS in front of the container; it serves plain HTTP on `PORT`.
- Presigned upload URLs mean no long-lived cloud credentials are held by fabMem.

## Support

Include `docker compose logs app --tail=200`, your image tag, and the `buildId` if the problem involves ingestion.
