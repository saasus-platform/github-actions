# saasus-platform/github-actions

Composite actions shared by the SDK repositories' contract tests.

| Action | Purpose |
|---|---|
| `resolve-latest-stable-tag` | Resolve the calling repository's latest stable tag (`vX.Y.Z`) |
| `fetch-openapi-spec` | Fetch the OpenAPI specs of the requested modules from the api repository (`Anti-Pattern-Inc/saasus-api`) |
| `setup-prism` | Start one Prism mock per module (sequential ports from `4010`) |

See each `.github/actions/*/action.yml` for the inputs and outputs.

## Usage

```yaml
- uses: saasus-platform/github-actions/.github/actions/fetch-openapi-spec@main
  id: spec
  with:
    token: ${{ steps.app-token.outputs.token }}  # contents:read on api
- uses: saasus-platform/github-actions/.github/actions/setup-prism@main
  with:
    spec-dir: ${{ steps.spec.outputs.spec-dir }}
    modules: ${{ steps.spec.outputs.modules }}
```

`setup-prism` exports each mock's URL to the job environment as
`PRISM_URL_<MODULE>` (e.g. `PRISM_URL_AUTH=http://127.0.0.1:4010`). Prism serves
`paths:` at the root, so set the SDK's baseURL to that URL as is, without the
spec's `servers` path (e.g. `/v1/auth`).

## Constraints

- The specs are private: never log the token, a spec body or a response body.
- Keep fetched specs inside the workspace; never publish them as an artifact.
- Removing an input makes the callers' `with:` silently ignored and fall back to
  the default. Grep every caller before removing one.
