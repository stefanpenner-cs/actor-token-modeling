# actor-token-modeling

Modeling the identity and ACL system GitHub uses in Actions.

Each finding below was produced by a runnable GitHub Actions workflow in this repo. Re-run them via `workflow_dispatch` to reproduce.

---

## Table of Contents

- [1. Token Identity](#1-token-identity)
- [2. Permission Boundaries](#2-permission-boundaries)
- [3. Permission Escalation (write-all)](#3-permission-escalation-write-all)
- [4. Default Permissions (no permissions key)](#4-default-permissions-no-permissions-key)
- [5. Git Clone and Rate Limits](#5-git-clone-and-rate-limits)
- [6. Cross-Repo Access](#6-cross-repo-access)

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

## 5. Git Clone and Rate Limits

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

## 6. Cross-Repo Access

**`GITHUB_TOKEN` is scoped to a single repository. Cross-repo access behaves the same as unauthenticated.**

Workflow: [`probe-permission-boundary.yml`](.github/workflows/probe-permission-boundary.yml) — [Run #25325490388](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490388)

Tested `GET /repos/stefanpenner/ember-cli/contents` with and without the token — both returned HTTP 301 (repo redirect). The token provides no elevated access to other repositories. `/installation/repositories` confirmed the token only sees its own repo.

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
```

---

## Workflow Index

| Workflow | Purpose | Latest Run |
|---|---|---|
| [`probe-token-identity.yml`](.github/workflows/probe-token-identity.yml) | Who is the token? GraphQL viewer, headers, installation endpoint | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490393) |
| [`probe-permission-boundary.yml`](.github/workflows/probe-permission-boundary.yml) | Declared vs undeclared permissions, cross-repo access | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490388) |
| [`probe-permission-escalation.yml`](.github/workflows/probe-permission-escalation.yml) | What does `write-all` actually grant? | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490402) |
| [`probe-default-no-permissions.yml`](.github/workflows/probe-default-no-permissions.yml) | What happens with no `permissions:` key? | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25325490412) |
| [`test-clone-rate-limit.yml`](.github/workflows/test-clone-rate-limit.yml) | Does git clone consume API rate limits? | [Run](https://github.com/stefanpenner-cs/actor-token-modeling/actions/runs/25324960405) |
