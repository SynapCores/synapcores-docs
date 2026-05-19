# API reference

Every SynapCores AIDB binary ships an OpenAPI 3.1 specification of its REST API
and a browsable Swagger UI. Both are public endpoints — no authentication
required — and live on the same port as the rest of the gateway (default
`:8080`).

## Endpoints

| What | URL | Notes |
|---|---|---|
| **OpenAPI 3.1 schema (JSON)** | `GET /v1/openapi.json` | The machine-readable contract. Point codegen tools at this. |
| **Swagger UI (HTML)** | `GET /v1/api-docs` | Human-browsable; redirects to `/v1/api-docs/`. |

The spec is generated at compile time from `crates/aidb-gateway/src/openapi.rs`
annotations, so it always reflects the routes the binary actually serves —
there is no hand-maintained JSON file that can drift out of sync.

## Quick probe

If you've just installed and want to confirm the gateway is healthy and the
spec is reachable:

```bash
# Health check
curl -i http://127.0.0.1:8080/health

# OpenAPI spec — first ~200 chars to confirm it's a real spec
curl -s http://127.0.0.1:8080/v1/openapi.json | head -c 200

# Swagger UI in a browser
open http://127.0.0.1:8080/v1/api-docs
```

Adjust the host/port if you've changed `listen_addr` in your
`synapcores.toml`.

## Common entrypoints

The spec is the source of truth, but the routes most integrations need are:

| Operation | Route | Method |
|---|---|---|
| Execute SQL | `/v1/query/execute` | `POST` |
| Batch SQL | `/v1/query/execute/batch` | `POST` |
| List tables | `/v1/schema/tables` | `GET` |
| Vector search | `/v1/vectors/collections/{name}/search` | `POST` |
| AutoML predict | `/v1/automl/models/{id}/predict` | `POST` |
| MCP (JSON-RPC) | `/v1/mcp` | `POST` |
| MCP info | `/v1/mcp/info` | `GET` |

> **Two paths that 404 because they don't exist.** If you've seen these, you're
> looking for one of the routes above:
>
> - `POST /v1/query` — there is no bare query route; use `/v1/query/execute`.
> - `POST /api/v1/<anything>` — the API is mounted at `/v1/`, not `/api/v1/`.

## Generating a client

The published `@synapcores/sdk` (npm) and `synapcores` (PyPI) packages already
wrap the most common endpoints, but if you're integrating from a language we
don't ship for — PHP, Go, Java, Ruby, etc. — you can generate a typed client
directly from the spec.

=== "TypeScript / JavaScript"

    ```bash
    # openapi-typescript: lightweight types, no runtime
    npx openapi-typescript http://127.0.0.1:8080/v1/openapi.json \
      -o src/synapcores-types.ts

    # Or Orval: full client with React Query / Axios hooks
    npx orval --input http://127.0.0.1:8080/v1/openapi.json \
              --output src/synapcores-client.ts
    ```

=== "Python"

    ```bash
    # openapi-python-client: typed client with httpx
    pipx run openapi-python-client generate \
      --url http://127.0.0.1:8080/v1/openapi.json
    ```

=== "Go"

    ```bash
    # oapi-codegen
    go run github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen \
      --config oapi.yaml \
      http://127.0.0.1:8080/v1/openapi.json
    ```

=== "PHP / Java / Ruby / ..."

    ```bash
    # The full OpenAPI Generator covers 50+ targets
    npx @openapitools/openapi-generator-cli generate \
      -i http://127.0.0.1:8080/v1/openapi.json \
      -g php \
      -o ./synapcores-php-client
    ```

## Authentication

The spec advertises two authentication schemes on the protected routes:

- `Bearer` JWT — obtained via `POST /v1/auth/login` with email + password
- `X-API-Key` header — programmatic, created via `POST /v1/api-keys`

`/v1/openapi.json` and `/v1/api-docs` themselves are unauthenticated so codegen
tools and CI runners can pull the spec without juggling credentials. The
endpoints the spec *describes* enforce auth as usual.

## Versioning

`/v1/openapi.json` always reflects the binary you're running. To pin a client
to a specific release, snapshot the spec at deploy time:

```bash
curl -s http://127.0.0.1:8080/v1/openapi.json > synapcores-v1.6.5.4-ce.json
```

…and feed the snapshot to your codegen instead of the live URL.

## Reporting drift

If you call a route exactly as described in the spec and get a 404, that's a
bug — either the spec annotation is wrong or the route isn't wired up. Open
an issue at [github.com/mataluis2k/aidb/issues](https://github.com/mataluis2k/aidb/issues)
with the request + response and we'll fix it.
