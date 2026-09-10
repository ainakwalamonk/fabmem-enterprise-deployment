# SSO — WorkOS AuthKit

Enabling SSO switches both `/mcp` and the dashboard from static tokens to WorkOS AuthKit. Developers sign in with
Google; there are no shared tokens to distribute or rotate.

## What to configure in WorkOS

WorkOS dashboard, in this order.

### 1. Authentication methods

**Authentication → Methods.** Enable the providers your developers use (Google OAuth for a Google Workspace org).
Test-mode tenants have Google enabled by default.

### 2. Redirect URI

**Redirects → Redirect URIs → Add URI:**

```
https://<your-fabmem-host>/auth/callback
```

Exact match, https, no trailing slash. This is the dashboard's sign-in callback. Without it, sign-in ends on
WorkOS's error page.

MCP clients (Claude, Cursor) do **not** use this URI. They register their own callback dynamically — nothing to
add by hand.

### 3. Dynamic client registration

**Applications → Configuration.** Dynamic Client Registration must be enabled for remote MCP clients to connect.
Verify against the tenant's own metadata:

```bash
curl -s https://<tenant>.authkit.app/.well-known/oauth-authorization-server | jq '{registration_endpoint, client_id_metadata_document_supported}'
```

Both fields must be present. `registration_endpoint` is DCR; `client_id_metadata_document_supported: true` is the
CIMD alternative some clients use.

### 4. Values to copy

| WorkOS dashboard location | `.env` variable | Format |
|---|---|---|
| Applications → your app → **Client ID** | `WORKOS_CLIENT_ID` | `client_01ABC…` |
| API Keys → **Secret key** | `WORKOS_API_KEY` | `sk_test_…` / `sk_live_…` |
| Authentication → AuthKit → **AuthKit domain** | `WORKOS_ISSUER` | `https://<tenant>.authkit.app` |
| Organizations → your org → **Organization ID** (optional) | `WORKOS_ALLOWED_ORG_ID` | `org_01ABC…` |

Plus, in `.env`:

```
FABMEM_MCP_RESOURCE_URL=https://<your-fabmem-host>
WORKOS_ALLOWED_DOMAINS=yourcompany.com
```

## Access control

`WORKOS_ALLOWED_DOMAINS` and `WORKOS_ALLOWED_ORG_ID` decide who may sign in. **Empty means any authenticated user
of that WorkOS tenant.** Set at least one before exposing the deployment.

`WORKOS_ALLOWED_DOMAINS` matches the email domain. AuthKit session tokens carry no `email` claim, so the server
resolves it from the WorkOS API — `WORKOS_API_KEY` must be set for the domain gate to work.

Beyond sign-in, each sphere is gated separately. A signed-in user may read a sphere when any of these holds:

- `users.is_org_admin` is true, or
- the sphere's `visibility` is `org`, or
- an explicit `sphere_members` row grants it.

New spheres default to `private`. The first user to sign in is provisioned with `is_org_admin = false` and no
memberships, so they can see the dashboard but no sphere data until granted.

## How each variable is used

| Variable | `/mcp` | Dashboard |
|---|---|---|
| `WORKOS_ISSUER` | required — issuer + JWKS signature | required |
| `WORKOS_CLIENT_ID` | not checked as an audience | required — first-party sign-in fails without it |
| `WORKOS_API_KEY` | resolves the email for the domain gate | required — server-side code exchange |
| `FABMEM_MCP_RESOURCE_URL` | advertised in OAuth discovery | — |

`/mcp` does not pin the token audience to `WORKOS_CLIENT_ID`: a remote MCP client registers its own client, so its
token's `aud` is that client's id, never this one. Identity is bound by the issuer and the JWKS signature.

`FABMEM_MCP_RESOURCE_URL` must be the public https URL when running behind a proxy. OAuth discovery advertises
this value; deriving it from the `Host` header is wrong when a proxy rewrites the header.

## Verifying

```bash
# 1. The server advertises its authorization server
curl -s https://<host>/.well-known/oauth-protected-resource

# 2. /mcp refuses anonymous access and points at discovery
curl -si -X POST https://<host>/mcp -H 'content-type: application/json' -d '{}' | grep -i www-authenticate

# 3. The dashboard reports a real client id (not a metadata-document URL)
curl -s https://<host>/api/auth/config
```

Expected: (1) lists your AuthKit domain, (2) `401` with
`www-authenticate: Bearer resource_metadata="https://<host>/.well-known/oauth-protected-resource"`,
(3) `"clientId":"client_01…"`.

Then open `https://<host>/` and sign in, and add `https://<host>/mcp` as a custom connector in your MCP client.

An MCP client that connects without prompting has a live AuthKit session in that browser. Sign out at
`https://<tenant>.authkit.app/sessions/logout` or use a private window to see the full sign-in.
