# actor-token-modeling

Modeling the identity and ACL system GitHub uses in Actions.

## Findings

### Git clone does not consume REST API rate limits

**Neither authenticated (`GITHUB_TOKEN`) nor unauthenticated git clone operations consume any REST API core rate limit.**

Git-over-HTTPS operates on a completely separate rate-limiting plane from the REST/GraphQL API. The `GITHUB_TOKEN`'s 5,000 req/hr core quota is unaffected by clone/fetch/push operations.

#### Evidence

The workflow inserts a control API call (`GET /repos/...`) between rate limit snapshots to prove the counter is live and responsive, then measures the impact of clone operations:

```
Core rate limit remaining at each snapshot:
  T0 (baseline):                5000
  T1 (after control API call):  4999    ← counter moved, proving it's live
  T2 (after token clone):       4999    ← clone didn't touch it
  T3 (after unauth clone):      4999    ← nor did unauth clone

Delta T0→T1 (control API call): 1      ← CONTROL OK
Delta T1→T2 (token clone):      0
Delta T2→T3 (unauth clone):     0
```

Additionally, `GET /rate_limit` itself is free — it does not decrement the counter, consistent with [GitHub's documentation](https://docs.github.com/en/rest/rate-limit/rate-limit).

#### Workflow runs

| Run | Description | Result |
|-----|-------------|--------|
| [#25324960405](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25324960405) | With control API call proving counter is live | Confirmed: clone consumes 0 rate limit |
| [#25324783767](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25324783767) | Initial run (no control call) | Showed 0 consumed, but lacked proof counter was live |

#### Caveats

- This only measures the **REST API core rate limit**. Git operations may have their own server-side compute quotas that are not exposed through the `/rate_limit` endpoint.
- Tested with the automatic `GITHUB_TOKEN` provided to GitHub Actions, not a PAT or GitHub App installation token. Other token types may behave differently.

## Workflow

- [`test-clone-rate-limit.yml`](.github/workflows/test-clone-rate-limit.yml) — measures rate limit before/after clone operations with a control API call to validate the counter is live.
