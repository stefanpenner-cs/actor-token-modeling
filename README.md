# actor-token-modeling

Modeling the identity and ACL system GitHub uses in Actions.

Each finding below was produced by a runnable GitHub Actions workflow in this repo. Re-run them via `workflow_dispatch` to reproduce.

---

## Table of Contents

- [1. Token Identity](#1-token-identity)
- [2. Permission Boundaries](#2-permission-boundaries)
- [3. Permission Escalation (write-all)](#3-permission-escalation-write-all)
- [4. Default Permissions (no permissions key)](#4-default-permissions-no-permissions-key)
- [5. Actor Permission Ceiling](#5-actor-permission-ceiling)
- [6. Git Clone and Rate Limits](#6-git-clone-and-rate-limits)
- [7. Cross-Repo Access](#7-cross-repo-access)
- [8. OIDC Token Claims](#8-oidc-token-claims)
- [9. Architecture: OIDC Token Vending for Cross-Repo Access](#9-architecture-oidc-token-vending-for-cross-repo-access)

---

## 1. Token Identity

**`GITHUB_TOKEN` is a GitHub App installation access token (`ghs_` prefix). It does NOT authenticate as the triggering user.**

Workflow: [`probe-token-identity.yml`](.github/workflows/probe-token-identity.yml) — [Run #25325490393](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490393)

### Key findings

| Question | Answer |
|---|---|
| Who does `GET /user` say the token is? | **403 Forbidden** — this endpoint rejects installation tokens |
| Who does GraphQL `{ viewer { login } }` say? | `github-actions[bot]` (user ID `41898282`) |
| What does `github.actor` report? | The human who triggered the workflow (e.g. `stefanpenner`) |
| What does `github.triggering_actor` report? | Same as `github.actor` for direct pushes |
| Is `GET /installation/repositories` accessible? | **Yes** — confirms it's an app installation token, scoped to 1 repo |
| What does `GET /repos/:repo` `.permissions` show? | `{admin: false, maintain: false, pull: false, push: false, triage: false}` — always false for installation tokens |
| What does `/collaborators/:actor/permission` show? | Reports the **actor's** (human's) permission: `admin` |
| Are `X-OAuth-Scopes` headers present? | **No** — only returned for classic PATs (`ghp_`). Absence = installation token or fine-grained PAT |
| What is `X-Accepted-GitHub-Permissions`? | Returned on 403 responses, tells you what permissions were needed |

### Identity model

```
┌─────────────────────────────────────────────────────┐
│                  Workflow Run                        │
│                                                     │
│  github.actor = "stefanpenner"     (who triggered)  │
│  github.actor_id = 1377                             │
│                                                     │
│  GITHUB_TOKEN authenticates as:                     │
│    → github-actions[bot] (ID 41898282)              │
│    → Type: GitHub App installation token (ghs_)     │
│    → Scoped to: this repository only                │
│    → Permissions: set by workflow YAML + org defaults│
│                                                     │
│  The token is NOT the actor.                        │
│  API calls are attributed to the bot, not the user. │
└─────────────────────────────────────────────────────┘
```

### Gotcha: `.permissions` on repo endpoint is useless

`GET /repos/:repo` returns `.permissions` as all `false` for `GITHUB_TOKEN`. This field only reflects meaningful values for user tokens (PATs, OAuth). Don't use it to introspect installation token capabilities.

---

## 2. Permission Boundaries

**The `permissions:` key in the workflow YAML strictly controls what the token can do. Undeclared permissions are denied with HTTP 403.**

Workflow: [`probe-permission-boundary.yml`](.github/workflows/probe-permission-boundary.yml) — [Run #25325490388](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490388)

### Test: declared `contents: read`, `issues: write`, `pull-requests: read`

| Operation | Permission needed | Declared? | Result |
|---|---|---|---|
| List repo contents | `contents: read` | Yes | **ALLOWED** (HTTP 200) |
| Create an issue | `issues: write` | Yes | **ALLOWED** (HTTP 201) |
| List pull requests | `pull-requests: read` | Yes | **ALLOWED** (HTTP 200) |
| Create a file | `contents: write` | No (only `read`) | **DENIED** (HTTP 403) |
| Delete a workflow run | `actions: write` | No | **DENIED** (HTTP 403) |
| Update repo settings | `admin` | No | **DENIED** (HTTP 403) |
| List org packages | `packages: read` | No | **ALLOWED** (HTTP 200) |

### Surprise: packages:read was allowed without being declared

Listing organization packages succeeded even though `packages: read` was not in the `permissions:` block. This suggests either:
- The `packages: read` permission is implicitly granted (possibly tied to `metadata: read` which is always granted)
- Or the endpoint doesn't require specific token permissions for public package listing

---

## 3. Permission Escalation (write-all)

**`permissions: write-all` grants broad access but NOT admin. Security-related endpoints are also denied.**

Workflow: [`probe-permission-escalation.yml`](.github/workflows/probe-permission-escalation.yml) — [Run #25325490402](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490402)

### What `write-all` actually grants

| Operation | Result |
|---|---|
| `contents:read` (list files) | **GRANTED** |
| `issues:read` (list issues) | **GRANTED** |
| `pull-requests:read` (list PRs) | **GRANTED** |
| `actions:read` (list runs) | **GRANTED** |
| `packages:read` (list packages) | **GRANTED** |
| `metadata:read` (repo info) | **GRANTED** |
| `security:read` (vulnerability alerts) | **DENIED** (HTTP 403) |
| `issues:write` (create label) | **GRANTED** |
| `contents:write` (create branch) | **GRANTED** |
| `admin` (update repo description) | **DENIED** (HTTP 403) |

### Key takeaways

- **`write-all` ≠ admin.** The token cannot modify repository settings even with maximum permissions requested.
- **Security/vulnerability endpoints are denied** even with `write-all`.
- The `/installation/repositories` endpoint confirms the token can only see **1 repo** (the workflow's own repo).

---

## 4. Default Permissions (no permissions key)

**When no `permissions:` key is present, the org/repo default applies. For this repo (in the `stefanpenner-cs` org), the default is permissive (read/write).**

Workflow: [`probe-default-no-permissions.yml`](.github/workflows/probe-default-no-permissions.yml) — [Run #25325490412](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490412)

### Effective permissions with no `permissions:` key

The runner's "Set up job" step logged these token permissions:
```
Actions: write
ArtifactMetadata: write
Contents: write
Issues: write
Metadata: read
Packages: write
PullRequests: write
```

| Operation | Result |
|---|---|
| `contents:read` (list files) | **GRANTED** |
| `issues:read` (list issues) | **GRANTED** |
| `pull-requests:read` (list PRs) | **GRANTED** |
| `actions:read` (list runs) | **GRANTED** |
| `metadata:read` (repo info) | **GRANTED** |
| `contents:write` (create branch) | **GRANTED** |
| `issues:write` (create label) | **GRANTED** |
| `admin` (update repo) | **DENIED** |

### Implication

Without an explicit `permissions:` block, this org defaults to **read/write** on most scopes. This means any workflow in this org can create branches, issues, labels, etc. unless explicitly restricted. **Best practice: always declare `permissions:` to follow least-privilege.**

---

## 5. Actor Permission Ceiling

**The actor's repository access is the ceiling for `GITHUB_TOKEN` permissions. The `permissions:` key cannot escalate beyond what the triggering user can do.**

Workflow: [`probe-actor-ceiling.yml`](.github/workflows/probe-actor-ceiling.yml) — [Run #25326146968](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25326146968) (admin baseline)

### Baseline: admin actor with `write-all`

When triggered by `stefanpenner` (admin on this repo):

| Operation | Result |
|---|---|
| `contents:read` | **GRANTED** |
| `issues:read` | **GRANTED** |
| `actions:read` | **GRANTED** |
| `issues:write` (create label) | **GRANTED** |
| `contents:write` (create branch) | **GRANTED** |
| `admin` (update repo) | **DENIED** |

Even an admin actor with `write-all` cannot get admin API access through `GITHUB_TOKEN`. The token's permission model has a hard ceiling imposed by GitHub regardless of the actor's role.

### To fully test the actor ceiling

To prove that a read-only collaborator gets *fewer* permissions than an admin despite both declaring `write-all`, we need a second user with limited access to trigger this workflow. The workflow is designed for this comparison — add a collaborator with `read` permission and have them trigger it via `workflow_dispatch`.

**Expected result for read-only actor:** writes should be DENIED even though the YAML declares `write-all`, because `effective = min(YAML, defaults, actor_access)`.

---

## 6. Git Clone and Rate Limits

**Neither authenticated (`GITHUB_TOKEN`) nor unauthenticated git clone operations consume any REST API rate limit.**

Workflow: [`test-clone-rate-limit.yml`](.github/workflows/test-clone-rate-limit.yml) — [Run #25324960405](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25324960405)

### Evidence

A control API call between snapshots proved the rate limit counter is live:

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

`GET /rate_limit` itself is free — consistent with [GitHub docs](https://docs.github.com/en/rest/rate-limit/rate-limit).

### Caveats

- Git operations may have their own server-side compute quotas not exposed through `/rate_limit`.
- Tested with the automatic `GITHUB_TOKEN`, not a PAT or GitHub App installation token from a custom app.

---

## 7. Cross-Repo Access

**`GITHUB_TOKEN` is scoped to a single repository. Cross-repo access behaves the same as unauthenticated.**

Workflow: [`probe-permission-boundary.yml`](.github/workflows/probe-permission-boundary.yml) — [Run #25325490388](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490388)

Tested `GET /repos/stefanpenner/ember-cli/contents` with and without the token — both returned HTTP 301 (repo redirect). The token provides no elevated access to other repositories. `/installation/repositories` confirmed the token only sees its own repo.

---

## 8. OIDC Token Claims

**GitHub Actions can issue OIDC tokens (`id-token: write`) with rich claims about the workflow context. These are ideal for federating identity to external services.**

Workflow: [`probe-oidc-token.yml`](.github/workflows/probe-oidc-token.yml) — [Run #25326193808](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25326193808)

### JWT structure

**Header:**
```json
{"alg": "RS256", "kid": "38826b17-6a30-5f9b-b169-8beb8202f723", "typ": "JWT"}
```

**Full claims payload:**
```json
{
  "sub": "repo:stefanpenner-cs/actor-token-modeling:ref:refs/heads/main",
  "aud": "https://github.com/stefanpenner-cs",
  "iss": "https://token.actions.githubusercontent.com",
  "repository": "stefanpenner-cs/actor-token-modeling",
  "repository_owner": "stefanpenner-cs",
  "repository_owner_id": "109183179",
  "repository_id": "1228935499",
  "repository_visibility": "public",
  "actor": "stefanpenner",
  "actor_id": "1377",
  "workflow": "Probe: OIDC Token Claims",
  "workflow_ref": "stefanpenner-cs/actor-token-modeling/.github/workflows/probe-oidc-token.yml@refs/heads/main",
  "job_workflow_ref": "stefanpenner-cs/actor-token-modeling/.github/workflows/probe-oidc-token.yml@refs/heads/main",
  "event_name": "push",
  "ref": "refs/heads/main",
  "ref_type": "branch",
  "sha": "e317e2176511ce4621622b808b442d9dbafa7f86",
  "environment": null,
  "runner_environment": "github-hosted",
  "run_id": "25326193808",
  "run_number": "2",
  "run_attempt": "1"
}
```

### Claims available for policy decisions

| Claim | Example value | Policy use |
|---|---|---|
| `sub` | `repo:org/repo:ref:refs/heads/main` | Primary subject — match repo + ref |
| `repository` | `stefanpenner-cs/actor-token-modeling` | Which repo is requesting access |
| `repository_owner` | `stefanpenner-cs` | Org-level policy |
| `actor` | `stefanpenner` | Who triggered the workflow |
| `ref` | `refs/heads/main` | Only allow from specific branches |
| `workflow_ref` | `.../.github/workflows/probe-oidc-token.yml@refs/heads/main` | Pin to specific workflow file + ref |
| `job_workflow_ref` | Same as workflow_ref (differs for reusable workflows) | Pin to calling workflow |
| `event_name` | `push` | Only allow specific triggers |
| `environment` | `null` (or `"production"`) | Require GitHub Environment protection |
| `repository_visibility` | `public` | Restrict to private repos only |
| `runner_environment` | `github-hosted` | Reject self-hosted runners |

### Key behaviors

| Test | Result |
|---|---|
| Custom `audience` per request | **Works** — `aud` set to whatever you pass (e.g. `https://token-vending.example.com`) |
| Same audience, multiple requests | **Different tokens each time** (different `jti`, potentially different `exp`) |
| Token lifetime | **~5 minutes** (`exp - iat = 300s`) |
| OIDC discovery endpoint | `https://token.actions.githubusercontent.com/.well-known/openid-configuration` |
| Signing keys (JWKS) | 4 RSA256 keys available at `.../.well-known/jwks` |

---

## 9. Architecture: OIDC Token Vending for Cross-Repo Access

The `GITHUB_TOKEN` is scoped to a single repo and you can't get cross-repo access from it. Storing PATs or GitHub App private keys as secrets works but puts long-lived credentials in the repo.

**The alternative: use the OIDC token to authenticate to an external token-vending service that mints short-lived, policy-scoped credentials.**

### How it works

```
┌──────────────────────────────────────────────────────────────────┐
│                    GitHub Actions Workflow                        │
│                                                                  │
│  1. Request OIDC token (id-token: write)                        │
│     → JWT signed by GitHub, contains repo/actor/ref/workflow     │
│                                                                  │
│  2. Send OIDC token to Token Vending Service                    │
│     POST https://tokens.example.com/github/token                │
│     Authorization: Bearer <oidc-jwt>                            │
│     Body: { "repos": ["org/other-repo"], "permissions": ... }   │
│                                                                  │
│  3. Receive short-lived credential                               │
│     → GitHub App installation token scoped to requested repos   │
│     → Or fine-grained PAT, or cloud provider credential         │
│                                                                  │
│  4. Use credential for cross-repo operations                     │
│     git clone, API calls, etc.                                   │
└──────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Token Vending Service                          │
│                                                                  │
│  Validates:                                                      │
│    ✓ JWT signature via GitHub JWKS                               │
│    ✓ iss = https://token.actions.githubusercontent.com           │
│    ✓ aud = https://tokens.example.com (prevents token reuse)    │
│    ✓ exp/nbf (token not expired)                                │
│                                                                  │
│  Policy engine evaluates claims:                                 │
│    ✓ repository ∈ allowed_repos                                  │
│    ✓ ref matches allowed branch pattern                          │
│    ✓ actor ∈ allowed_actors (optional)                           │
│    ✓ workflow_ref matches pinned workflow (prevents tampering)   │
│    ✓ environment = "production" (optional, for deploy keys)     │
│    ✓ runner_environment = "github-hosted" (reject self-hosted)  │
│    ✓ requested repos ⊆ policy-allowed repos for this source     │
│                                                                  │
│  Mints:                                                          │
│    → GitHub App installation token (1hr, scoped to target repos)│
│    → With minimum necessary permissions                          │
│    → Logged for audit trail                                      │
└──────────────────────────────────────────────────────────────────┘
```

### Why this is better than secrets

| | Secrets (PAT/App key) | OIDC + Token Vending |
|---|---|---|
| Credentials in repo | Yes — stored as GitHub Secrets | **None** — OIDC token is ephemeral |
| Credential lifetime | Long-lived (until rotation) | **~5 min OIDC + ~1hr minted token** |
| Scope control | At secret creation time | **Per-request, policy-evaluated** |
| Rotation | Manual or scheduled | **Automatic — nothing to rotate** |
| Audit | Who used the secret? Hard to tell | **Full claim-based audit trail** |
| Blast radius of compromise | All repos the PAT can access | **Only what policy allows for that specific workflow + ref** |
| Branch restrictions | None — any branch can use the secret | **Policy can require `ref = refs/heads/main`** |
| Workflow pinning | None — any workflow can use the secret | **Policy can pin to specific `workflow_ref`** |

### Example policy

```yaml
# Token vending policy: what can each repo request?
policies:
  - name: "actor-token-modeling can clone shared-libs"
    match:
      repository: "stefanpenner-cs/actor-token-modeling"
      ref: "refs/heads/main"
      workflow_ref: "stefanpenner-cs/actor-token-modeling/.github/workflows/build.yml@refs/heads/main"
    grant:
      repos: ["stefanpenner-cs/shared-libs"]
      permissions:
        contents: read

  - name: "deploy workflows can write to infrastructure"
    match:
      repository: "stefanpenner-cs/*"
      environment: "production"
      runner_environment: "github-hosted"
    grant:
      repos: ["stefanpenner-cs/infrastructure"]
      permissions:
        contents: write
        pull-requests: write
```

### Implementation options

The token-vending service itself can be:

1. **A GitHub App you host** — receives OIDC JWT, validates it, uses its own private key to mint installation tokens for target repos. Most flexible.
2. **A lightweight serverless function** (Lambda, Cloud Run, etc.) — same pattern, minimal infrastructure.
3. **HashiCorp Vault** — has built-in [JWT/OIDC auth method](https://developer.hashicorp.com/vault/docs/auth/jwt) that can validate GitHub OIDC tokens and issue dynamic secrets.
4. **Cloud provider native** — AWS, GCP, and Azure already accept GitHub OIDC tokens for cloud credentials (no custom service needed for cloud access).

### Workflow usage

```yaml
permissions:
  id-token: write   # needed to request OIDC token
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Get cross-repo token
        id: token
        run: |
          OIDC=$(curl -s \
            -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
            "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=https://tokens.example.com" \
            | jq -r '.value')

          RESULT=$(curl -s -X POST \
            -H "Authorization: Bearer $OIDC" \
            -H "Content-Type: application/json" \
            -d '{"repos":["stefanpenner-cs/shared-libs"],"permissions":{"contents":"read"}}' \
            https://tokens.example.com/github/token)

          echo "::add-mask::$(echo $RESULT | jq -r '.token')"
          echo "token=$(echo $RESULT | jq -r '.token')" >> "$GITHUB_OUTPUT"

      - name: Clone private repo
        run: |
          git clone https://x-access-token:${{ steps.token.outputs.token }}@github.com/stefanpenner-cs/shared-libs.git
```

---

## Summary Model

```
GITHUB_TOKEN effective permissions = min(
  workflow YAML `permissions:` declaration,
  org/repo default token permissions,
  actor's own repository access
)

Where:
  - Identity:    github-actions[bot] (not the actor)
  - Token type:  GitHub App installation token (ghs_ prefix)
  - Scope:       single repository only
  - Rate limit:  5000 req/hr core (separate from git operations)
  - Admin:       never granted, even with write-all
  - Security:    vulnerability alerts denied even with write-all

For cross-repo access without secrets:
  OIDC token (id-token: write) → Token Vending Service → scoped credential
  - OIDC token lifetime: ~5 minutes
  - Claims include: repo, actor, ref, workflow_ref, environment, visibility
  - Audience is configurable per-request
  - Signed RS256, verifiable via GitHub's public JWKS
```

---

## Workflow Index

| Workflow | Purpose | Latest Run |
|---|---|---|
| [`probe-token-identity.yml`](.github/workflows/probe-token-identity.yml) | Who is the token? GraphQL viewer, headers, installation endpoint | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490393) |
| [`probe-permission-boundary.yml`](.github/workflows/probe-permission-boundary.yml) | Declared vs undeclared permissions, cross-repo access | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490388) |
| [`probe-permission-escalation.yml`](.github/workflows/probe-permission-escalation.yml) | What does `write-all` actually grant? | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490402) |
| [`probe-default-no-permissions.yml`](.github/workflows/probe-default-no-permissions.yml) | What happens with no `permissions:` key? | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490412) |
| [`probe-actor-ceiling.yml`](.github/workflows/probe-actor-ceiling.yml) | Can `permissions:` exceed actor's access? (dispatch-only) | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25326146968) |
| [`probe-oidc-token.yml`](.github/workflows/probe-oidc-token.yml) | Decode OIDC JWT claims, test audiences, discovery doc | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25326193808) |
| [`test-clone-rate-limit.yml`](.github/workflows/test-clone-rate-limit.yml) | Does git clone consume API rate limits? | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25324960405) |
