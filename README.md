<p align="center">
  <img src="branding.png" alt="Takumi Guard — a panda security guard scanning PyPI packages" width="300" />
</p>

<h1 align="center">Takumi Guard for PyPI</h1>

<p align="center">
  <strong>Stop malicious PyPI packages before they reach your CI.</strong><br />
  A GitHub Action that routes installs through a security proxy — no secrets, no config files, two lines of YAML.
</p>

<p align="center">
  <a href="https://github.com/flatt-security/setup-takumi-guard-pypi/actions/workflows/test.yml"><img src="https://github.com/flatt-security/setup-takumi-guard-pypi/actions/workflows/test.yml/badge.svg" alt="CI" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/flatt-security/setup-takumi-guard-pypi" alt="License" /></a>
</p>

---

> **Not using CI?** For local setup on your laptop, see the [email registration & token management appendix](#appendix-email-registration--token-management) below.

## Contents

- [What is Takumi Guard?](#what-is-takumi-guard)
- [Quickstart (3 steps)](#quickstart)
- [Setup modes](#setup-modes)
- [Migrating existing projects](#migrating-existing-projects)
- [Inputs](#inputs)
- [Outputs](#outputs)
- [Troubleshooting](#troubleshooting)
- [Security](#security)
- [Appendix: Email registration & token management](#appendix-email-registration--token-management)

---

## What is Takumi Guard?

Every `pip install` in your CI is a trust decision. Takumi Guard sits between your workflow and PyPI, **blocking known-malicious packages before they execute**.

- **How it works** -- Routes installs through a security proxy (`pypi.flatt.tech`) that checks packages against a threat database in real time.
- **What you change** -- One step in your workflow YAML. No config files, no secrets to manage.
- **What it supports** -- **pip**, **uv**, and **poetry**.

---

## Quickstart

**Goal:** Add Takumi Guard to any GitHub Actions workflow. No account required.

**Step 1.** Add the action to your workflow file (e.g. `.github/workflows/ci.yml`):

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: flatt-security/setup-takumi-guard-pypi@v1   # <-- add this line

  - run: pip install -r requirements.txt
  - run: pytest
```

**Step 2.** Push the change. Every `pip install` in this job now runs through the Takumi Guard proxy. Malicious packages are blocked automatically.

**Step 3.** *(Optional)* **Want audit logging and a dashboard?** Add a Bot ID for full visibility into package activity:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # Required for authentication
      contents: read
    steps:
      - uses: actions/checkout@v4

      - uses: flatt-security/setup-takumi-guard-pypi@v1
        with:
          bot-id: "YOUR_BOT_ID"

      - run: pip install -r requirements.txt
```

> **Where do I get a Bot ID?** Create one at [Shisho Cloud byGMO](https://cloud.shisho.dev) -- or skip this entirely. Blocking works without it. The Bot ID is a public reference key, not a secret.

---

## Setup modes

| Mode | Blocks malware | Audit logging | Account needed | Best for |
|---|:---:|:---:|:---:|---|
| **[Blocking only](#blocking-only)** | Yes | No | No | OSS projects, quick evaluation |
| **[Full protection](#full-protection)** | Yes | Yes | Yes | Production workloads |
| **[Auth-only](#auth-only-advanced)** | You manage | Yes | Yes | Custom pip.conf setups |

---

### Blocking only

> **No account needed.** Add one line and you are protected.

Blocks known-malicious packages. No signup, no authentication.

```yaml
- uses: flatt-security/setup-takumi-guard-pypi@v1
```

Good for open-source projects or quick evaluation.

---

### Full protection

> **Recommended for production.** Blocks threats _and_ logs all package activity to your dashboard.

```yaml
permissions:
  id-token: write

steps:
  - uses: flatt-security/setup-takumi-guard-pypi@v1
    with:
      bot-id: "YOUR_BOT_ID"
```

**Key details:**
- Auth is handled via **GitHub's built-in OIDC** -- no PATs or secrets to rotate.
- If authentication fails, **blocking remains active** but logging is degraded. The build continues with a warning.
- Get a Bot ID from [Shisho Cloud byGMO](https://cloud.shisho.dev).

---

### Auth-only (advanced)

> **For custom setups.** You manage `PIP_INDEX_URL` / `UV_INDEX_URL` yourself. The action only handles authentication.

```yaml
- uses: flatt-security/setup-takumi-guard-pypi@v1
  with:
    bot-id: "YOUR_BOT_ID"
    set-index-url: false
```

**Key details:**
- Useful for projects that need full control over pip/uv configuration.
- Requires `PIP_INDEX_URL=https://pypi.flatt.tech/simple/` (and `UV_INDEX_URL` for uv) set in your environment or config files.
- If authentication fails, **the action exits with an error** -- there is no fallback.

---

## Migrating existing projects

Unlike npm, pip does not use lockfiles that reference a specific registry URL by default. Most projects can adopt Takumi Guard without any lockfile changes.

**For pip / uv:** No migration needed. The action sets `PIP_INDEX_URL` (for pip) and `UV_INDEX_URL` (for uv) so all installs automatically route through the proxy.

**For poetry** (with `poetry.lock`):

```bash
# Add the Takumi Guard source
poetry source add --priority=primary takumi-guard https://pypi.flatt.tech/simple/

# Regenerate lockfile
poetry lock --no-update

# Commit
git add pyproject.toml poetry.lock
git commit -m "Route installs through Takumi Guard"
```

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `bot-id` | No | -- | Bot ID from Shisho Cloud byGMO. Omit for blocking-only mode. |
| `set-index-url` | No | `true` | Set `PIP_INDEX_URL` and `UV_INDEX_URL` environment variables. Set to `false` if you manage them yourself. |
| `registry-url` | No | `https://pypi.flatt.tech` | Registry endpoint. |
| `sts-url` | No | `https://sts.cloud.shisho.dev` | STS endpoint for token exchange. |
| `expires-in` | No | `1800` | Token lifetime in seconds (max 86400). |

---

## Outputs

| Output | Description |
|---|---|
| `token-expires-at` | ISO 8601 timestamp of token expiration. Only set when authenticated. |

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `OIDC not available` | Missing permission on the job | Add `permissions: { id-token: write }` to your job |
| `invalid ID token` | Trust condition mismatch | Check the bot's trust settings in Shisho Cloud byGMO |
| `invalid request` | Malformed bot-id | Double-check the bot-id value from your console |
| `Authentication failed ... Falling back` | STS token exchange failed | Verify bot-id and trust settings. Blocking is still active. |

> **Still stuck?** Open an issue on this repository with your error output and workflow file (redact any IDs).

---

## Security

- **Short-lived tokens** -- 30 minutes by default, 24 hours max.
- **Auto-masked** -- Tokens and authenticated URLs are automatically masked in workflow logs.
- **Environment-scoped** -- The action sets `PIP_INDEX_URL` and `UV_INDEX_URL` for the current job only. Your global pip/uv config is untouched.
- **Basic auth in URL** -- pip uses `https://token:ACCESS_TOKEN@host/simple/` format. The full URL is masked in logs.

---

## Appendix: Email registration & token management

> **Optional.** Register your email to receive breach notifications if a package you installed is later flagged as malicious. This works for local development -- CI workflows should use [Full protection](#full-protection) instead.

### Register

```bash
curl -X POST https://pypi.flatt.tech/api/v1/tokens \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'
```

Check your inbox and click the verification link. You will receive a token like `tg_anon_xxx...`.

**Language preference:** Add `"language": "ja"` to receive emails in Japanese. Defaults to English (`"en"`) if omitted.

```bash
curl -X POST https://pypi.flatt.tech/api/v1/tokens \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "language": "ja"}'
```

### Configure pip / uv

Set the index URL with your token:

```bash
# For pip (and uv pip commands)
export PIP_INDEX_URL=https://token:tg_anon_xxx...@pypi.flatt.tech/simple/
# For uv project commands (add/sync/lock)
export UV_INDEX_URL=https://token:tg_anon_xxx...@pypi.flatt.tech/simple/
```

To make this permanent, add the lines to your shell profile or configure pip in `~/.config/pip/pip.conf` (Linux/macOS) or `%APPDATA%\pip\pip.ini` (Windows):

```ini
[global]
index-url = https://token:tg_anon_xxx...@pypi.flatt.tech/simple/
```

After this, `pip install` and `uv` commands route through Takumi Guard with your identity attached. If a package you downloaded is later found to be malicious, you will receive a breach notification email.

### Check token status

```bash
curl -H "Authorization: Bearer tg_anon_xxx..." \
  https://pypi.flatt.tech/api/v1/tokens/status
```

### Rotate your key

```bash
curl -X POST -H "Authorization: Bearer tg_anon_xxx..." \
  https://pypi.flatt.tech/api/v1/tokens/regenerate
```

Returns a new API key. The old one is invalidated immediately. Update your pip configuration with the new key.

### Revoke a token

```bash
curl -X DELETE -H "Authorization: Bearer tg_anon_xxx..." \
  https://pypi.flatt.tech/api/v1/tokens
```

---

<p align="center">
  Built by <a href="https://flatt.tech">GMO Flatt Security Inc.</a><br />
  <a href="LICENSE">MIT License</a>
</p>
