# Docker Compose Audit — Setup Guide

Automatically audits Docker Compose files on push, files Gitea issues for hardening violations, and tags stacks in Komodo with `audit:pass` or `audit:fail`.

---

## How it works

1. A push to `main` triggers the workflow
2. Only the changed compose file(s) are audited
3. If issues are found, a Gitea issue is opened (or updated) and labelled with the service name
4. The corresponding Komodo stack is tagged `audit:pass` or `audit:fail`
5. If a previously failing file is fixed, the issue is automatically closed

---

## Prerequisites

- Gitea instance with Actions enabled
- A Gitea Actions runner attached to the repository
- Komodo instance (optional — tagging is skipped if credentials are not provided)

---

## Step 1 — Enable Gitea Actions

In your Gitea instance, ensure Actions are enabled. In `app.ini`:

```ini
[actions]
ENABLED = true
```

Restart Gitea if you changed this.

---

## Step 2 — Create a Gitea API token

Go to **Gitea → Settings → Applications → Manage Access Tokens** and generate a new token with these scopes:

| Scope | Level |
|---|---|
| issue | Read and Write |
| repository | Read |

All other scopes can remain at No Access.

Save the token value — you will not see it again.

---

## Step 3 — Add Gitea repository secret

Go to your repository → **Settings → Secrets** and add:

| Name | Value |
|---|---|
| `REPO_TOKEN` | The token generated in Step 2 |

---

## Step 4 — Add Komodo secrets (optional)

If you want Komodo stack tagging, go to your repository → **Settings → Secrets** and add:

| Name | Value |
|---|---|
| `KOMODO_URL` | Your Komodo URL e.g. `https://komodo.homenook.xyz` |
| `KOMODO_API_KEY` | API key from Komodo user settings |
| `KOMODO_API_SECRET` | API secret from Komodo user settings |

To generate a Komodo API key, go to **Komodo → User Settings → API Keys → Create**.

If these secrets are not present the workflow will still run — Komodo tagging is simply skipped.

---

## Step 5 — Add the workflow file

Create the file `.gitea/workflows/compose-audit.yaml` in your repository with the workflow contents.

Commit and push to `main` to activate it.

---

## Step 6 — Verify runner availability

Go to your repository → **Settings → Actions → Runners** and confirm a runner is listed and online. The workflow uses `runs-on: ubuntu-latest` — ensure your runner satisfies this label.

---

## Repository structure

The workflow expects compose files to live in per-service subdirectories:

```
docker_compose/
├── uptime-kuma/
│   └── compose.yaml
├── authentik/
│   └── compose.yaml
├── backrest/
│   └── compose.yaml
└── .gitea/
    └── workflows/
        └── compose-audit.yaml
```

The folder name becomes the service name used in Gitea issue titles, labels, and Komodo tags.

---

## Checks performed

| Check | Description |
|---|---|
| `no-new-privileges:true` | Must be set in `security_opt` |
| `cap_drop: ALL` | All capabilities must be dropped |
| `read_only: true` | Root filesystem should be read-only |
| No `privileged: true` | Privileged mode should not be used |
| No inline `environment:` | Use `env_file: .env` instead |
| `restart:` policy | A restart policy must be defined |
| No `network_mode: host` | Host networking should not be used |
| Internal service isolation | Databases/caches must not be on the ingress network or expose host ports |
| Network isolation | Services spanning multiple networks should have at least one marked `internal: true` |
| Internal network naming | Networks named `db`, `backend`, `cache`, etc. should have `internal: true` |

The ingress network name is hardcoded as `nginx` in the workflow — update `INGRESS_NETWORK` at the top of the script if yours differs.

---

## Issue lifecycle

- **New findings** → issue is created with the service label
- **Repeat push with same findings** → existing issue body is updated, no duplicate created
- **Push that fixes all findings** → open issue is automatically closed with a note

---

## Komodo tag lifecycle

- **Findings exist** → stack is tagged `audit:fail`, `audit:pass` is removed
- **All checks pass** → stack is tagged `audit:pass`, `audit:fail` is removed
- **Stack name not found in Komodo** → tagging is skipped with a warning, audit continues

The tags `audit:pass` and `audit:fail` are created automatically in Komodo if they do not already exist.
