# Publishing the OpenSpec extension

This document describes the two distribution channels:

1. **Internal releases via GitHub Actions** (primary) — `.vsix` artifacts published to GitHub Releases on `MotusLabs/openspec-ext`, installable with "Install from VSIX…".
2. **Public marketplaces** (optional) — VS Code Marketplace and Open VSX, published manually.

---

## 1. Internal release via GitHub Actions (primary)

CI lives in `.github/workflows/`:

- `ci.yml` — runs on every PR to `main`: tests, lint (`eslint src/`), and a packaging smoke test; the produced `.vsix` is uploaded as a workflow artifact for review.
- `release.yml` — runs on every push to `main` and on `v*` tags: tests, packages, and publishes a GitHub Release with the `.vsix` attached. No secrets are required (it uses the built-in `GITHUB_TOKEN`).

### 1.1 Rolling releases (every PR merge)

Each merge to `main` via a PR automatically publishes a **prerelease** named after the PR:

- Version: `<package.json version>-<PR number>` (e.g. `0.2.2-123`); the `.vsix` carries this version.
- Git tag / release: `v0.2.2-123`, marked as a pre-release so stable tags keep the "latest" pointer.
- Release notes come from the PR title, body, and link.

Direct pushes to `main` without an associated PR run the tests but do not create a release.

### 1.2 Stable releases (tag push)

1. Bump `version` in `package.json`.
2. Move the `## [Unreleased]` items in `CHANGELOG.md` under a new `## [X.Y.Z] - YYYY-MM-DD` heading (Keep a Changelog format).
3. Commit, then tag and push:

   ```bash
   git tag vX.Y.Z
   git push origin main vX.Y.Z
   ```

CI verifies the tag matches `package.json` (the run fails with instructions otherwise), extracts the changelog section as release notes, and creates the GitHub Release with `openspec-workflow-X.Y.Z.vsix` attached. Tags like `vX.Y.Z-rc.1` are published as prereleases.

### 1.3 Installing internally

The repo is private, so release assets require GitHub authentication (browser download while signed in, or `gh`):

```bash
gh auth login                      # once
gh release download vX.Y.Z -R MotusLabs/openspec-ext -p '*.vsix'
code --install-extension openspec-workflow-X.Y.Z.vsix
```

Or in the UI: Extensions view → `…` → **Install from VSIX…**.

> **Extension ID note:** the internal build is published as `motuslabs.openspec-workflow` (publisher `motuslabs`). It is a distinct extension ID from the older public `randysss.openspec-workflow`; uninstall the old build to avoid duplicates. Settings keys (`openspec.*`) are unchanged.

---

## 2. Public marketplace publishing (optional)

Ensure `package.json` version is bumped and changes are committed before publishing. The `publisher` field (`motuslabs`) must exist as a namespace on each marketplace before the first publish.

### 2.1 VS Code Marketplace (marketplace.visualstudio.com)

#### Create a publisher (first time only)

VS Code uses Azure DevOps for the Marketplace. You need a **publisher** and a **Personal Access Token (PAT)**.

1. **Personal Access Token**
   - Go to [Azure DevOps](https://go.microsoft.com/fwlink/?LinkId=307137). Create an [organization](https://learn.microsoft.com/azure/devops/organizations/accounts/create-organization) if needed.
   - User settings (profile) → **Personal access tokens** → **New Token**.
   - Under **Scopes**, choose **Custom defined** → **Marketplace** → **Manage** (and show all scopes if needed). Create the token and copy it to a safe place.

2. **Create publisher**
   - Go to [Marketplace publisher management](https://marketplace.visualstudio.com/manage).
   - Sign in with the same Microsoft account used for the PAT.
   - Click **Create publisher**.
   - Set **ID** to `motuslabs` (must match `publisher` in `package.json`; cannot be changed later). Click **Create**.

3. **Log in with vsce**
   - Run: `pnpm exec vsce login motuslabs`
   - When prompted, paste your Personal Access Token. You only need to do this once per machine.

#### Package and publish

```bash
# Build and create .vsix
pnpm run package

# Publish to VS Code Marketplace (uses vsce login)
pnpm run publish:marketplace
```

For CI or non-interactive use, you can use the `VSCE_PAT` environment variable instead of `vsce login`. Do not commit the PAT.

### 2.2 Open VSX (open-vsx.org)

#### Create namespace and token (first time only)

1. **Account and namespace**
   - Go to [open-vsx.org](https://open-vsx.org) and sign in (e.g. with GitHub).
   - Create a **namespace** named `motuslabs` that matches the `publisher` in `package.json`.

2. **Personal Access Token**
   - Open [Open VSX user settings → Tokens](https://open-vsx.org/user-settings/tokens).
   - Create a new token. Copy it to a safe place.

3. **Token must not be committed**
   - Use the token only via environment variable (e.g. `OVSX_TOKEN`). Do not put it in the repo or in committed `.env` files. CI should inject it as a secret.

#### Publish to Open VSX

Set your token and run the publish script. You can use **make** (reads `OVSX_TOKEN` from `.env` if present) or run the steps manually:

**Option A: make (recommended if you use .env)**

```bash
# Put OVSX_TOKEN=your-token in project root .env (do not commit .env)
make publish-ovsx
```

`make publish-ovsx` loads `.env` if it exists, then runs `pnpm run package` and `pnpm run publish:openvsx`. No need to export `OVSX_TOKEN` in the shell.

**Option B: pnpm (set token in environment)**

```bash
# Build and create .vsix (if not already done)
pnpm run package

# Publish to Open VSX (requires OVSX_TOKEN)
OVSX_TOKEN=your-token-here pnpm run publish:openvsx
```

If `OVSX_TOKEN` is not set, the script exits with a clear error and does not publish.

On Windows (PowerShell):

```powershell
$env:OVSX_TOKEN = "your-token-here"
pnpm run publish:openvsx
```

---

## 3. Local packaging (any channel)

```bash
pnpm install                 # uses the committed pnpm-lock.yaml
pnpm run package             # produces openspec-workflow-<version>.vsix in the repo root
```

---

## 4. Optional: .env.example

If you use a local `.env` for tokens, add `.env` to `.gitignore` and optionally provide:

```bash
# .env.example (no real values – do not commit tokens)
# OVSX_TOKEN=    # Open VSX PAT from https://open-vsx.org/user-settings/tokens
# VSCE_PAT=      # Azure DevOps PAT with Marketplace scope (for vsce publish)
```

Load `.env` before running publish (e.g. with `dotenv` or your shell). Never commit `.env` or real tokens.
