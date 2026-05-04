# actor-token-modeling

Modeling the identity and ACL system GitHub uses in Actions.

## Experiments

### `test-clone-rate-limit`

Tests whether cloning a repo with `GITHUB_TOKEN` consumes REST API rate limits.

GitHub's git protocol operations (clone/fetch/push) go through a separate transport layer
than the REST/GraphQL API. This workflow measures the core API rate limit before and after
clone operations to determine if they share quota.

The workflow compares:
- Clone using `GITHUB_TOKEN` (authenticated git-over-https)
- Clone without credentials (unauthenticated git-over-https)

Run manually via `workflow_dispatch` or automatically on push to `main`.
