# Studio Hub Site — New Session Brief
> Paste this entire file as the first message in the new Claude Sonnet 5.5 session.

---

## Goal

Build and deploy a professional product studio hub site at `https://debasisr66.github.io/studio/` — a GitHub Pages static site (no build step, no npm, no frameworks) that:

- Showcases all public GitHub repos owned by `debasisr66` as products
- Has sections: Products, Blog, Updates, Reading List, Books
- Auto-syncs product data (description, releases, stars, changelog) from each product repo via GitHub Actions
- Has a CMS admin layer (same pattern as existing personal site) for blog/reads/books — unlocked with a GitHub PAT stored in localStorage
- Has a distinct **product-studio brand look** (NOT matching the personal site at debasisr66.github.io/debasis-web)

**Known product repos to showcase (start with these):**
- `debasisr66/PennyExpenseManager` — personal finance / expense tracker app

More repos will be added via `content/products.json` (the registry file).

---

## Repository Setup

**New repo:** `debasisr66/studio` (already named by owner — create it on GitHub if not done)  
**Local path:** `/Users/debasisray/CoworkProjects/studio-site/`  
**GitHub Pages:** Settings → Pages → Source: `main` branch, root `/`

```bash
mkdir /Users/debasisray/CoworkProjects/studio-site
cd /Users/debasisray/CoworkProjects/studio-site
git init
git remote add origin https://github.com/debasisr66/studio.git
```

---

## File Structure

```
studio-site/
├── index.html                    ← main hub (landing + product grid + updates)
├── content/
│   ├── products.json             ← REGISTRY: owner-curated list (which repos, overrides)
│   ├── cache.json                ← AUTO-GENERATED: live data pulled from GitHub API
│   ├── blog.json                 ← CMS-editable blog posts
│   └── reads.json                ← CMS-editable reading list + books
└── .github/
    └── workflows/
        ├── sync.yml              ← nightly + on repository_dispatch: refreshes cache.json
        └── pages.yml             ← GitHub Pages deploy workflow (if needed)
```

---

## Safety Rules — CRITICAL (read before writing any code)

These rules protect the GitHub repos from corruption. Enforce them everywhere.

### A. GitHub API Write Safety

1. **Always GET SHA before PUT** — every write to a file via GitHub API must first fetch the file's current SHA. A PUT without the correct SHA is rejected by GitHub (409 Conflict). Never skip this step.

2. **Validate JSON before committing** — parse and re-serialize the JSON payload before base64-encoding it. If `JSON.parse()` throws, abort and log the error. Never commit malformed JSON.

3. **One file per commit** — never batch multiple file changes into one PUT. If one file fails, others remain clean.

4. **Idempotent writes** — compare new content to current content before committing. If unchanged, skip the PUT entirely. This prevents spurious commits and reduces API calls.

5. **No force-push, ever** — the sync workflow must never use `git push --force` or `--force-with-lease`. If a push fails due to divergence, log it and exit with an error. Let the next scheduled run retry.

6. **Read-only access to product repos** — the hub site and its workflows NEVER write to product repos. They only read via GitHub API. The cross-repo notification goes the other way: product repo → dispatches to hub repo.

### B. GitHub Actions Workflow Safety

7. **Minimal permissions** — sync.yml uses `contents: write` ONLY on the hub repo. All other permissions default to `read`. Never use `write-all`.

8. **Dry-run check** — sync.yml accepts an input `dry_run: true` (workflow_dispatch) that logs what would be committed without actually pushing.

9. **Failure isolation** — if fetching data for one product repo fails (API error, rate limit, 404), log the error and continue with the others. Never let one failure abort the entire sync.

10. **Rate limit guard** — check `X-RateLimit-Remaining` header after each API call. If remaining < 10, stop fetching and use whatever was already collected. Authenticated requests use the `GH_TOKEN` secret (5000/hour limit).

11. **Atomic cache update** — write all product data to a temp object in memory, validate the full JSON, then commit once. Never partially update cache.json.

### C. Cross-Repo Dispatch Safety

12. **Product repos are never modified by hub** — the `repository_dispatch` event flows ONE WAY: product repo → hub repo. The hub never writes back to product repos.

13. **Dispatch token is scoped** — the `HUB_DISPATCH_TOKEN` secret stored in product repos has ONLY `repo` scope on the hub repo, not the product repos themselves. Use a fine-grained PAT limited to `debasisr66/studio` with Actions: write permission only.

14. **Dispatch is fire-and-forget** — product repo workflows don't wait for the hub sync to succeed. They dispatch and exit. Hub sync failures don't propagate back.

### D. Recovery Mechanisms

15. **Git history IS the backup** — every cache.json commit is recoverable via `git revert` or `git checkout <sha> -- content/cache.json`. Never squash or force-push the sync commits.

16. **Manual sync trigger** — sync.yml has `workflow_dispatch:` with optional `dry_run` input, so the owner can trigger a fresh sync anytime from the GitHub Actions UI.

17. **Fallback rendering** — `index.html` always renders from `cache.json` if available, falls back to `products.json` registry data (name + tagline only) if cache is missing/corrupt, and shows a graceful empty state if both fail.

18. **Admin PAT is never committed** — the GitHub PAT for the CMS admin layer lives in `localStorage('studio-admin-pat')` only. It is never written to any file or logged.

---

## Content Files Spec

### `content/products.json` (owner-maintained registry)

```json
[
  {
    "repo": "debasisr66/PennyExpenseManager",
    "slug": "penny",
    "name": "Penny",
    "tagline": "Personal expense manager — simple, private, offline-first",
    "category": "Productivity",
    "featured": true,
    "hidden": false
  }
]
```

Fields: `repo` (required), `slug` (URL-safe id), `name`/`tagline` (override GitHub description), `category`, `featured` (shows in hero grid), `hidden` (exclude from public listing).

### `content/cache.json` (auto-generated by sync.yml)

```json
{
  "synced_at": "2026-10-13T03:00:00Z",
  "products": {
    "debasisr66/PennyExpenseManager": {
      "description": "...",
      "stars": 12,
      "language": "Swift",
      "topics": ["ios", "finance"],
      "pushed_at": "2026-10-10T14:23:00Z",
      "latest_release": {
        "tag": "v1.2.0",
        "name": "Release 1.2.0",
        "body": "## What's new\n- Feature A\n- Bug fix B",
        "published_at": "2026-10-10T14:00:00Z",
        "download_url": "https://github.com/debasisr66/PennyExpenseManager/releases/tag/v1.2.0",
        "assets": [
          { "name": "Penny.ipa", "download_url": "...", "size": 12400000 }
        ]
      }
    }
  }
}
```

### `content/blog.json`

```json
{
  "posts": [
    {
      "id": 1728820000000,
      "date": "2026-10-13",
      "title": "...",
      "body": "...",
      "link": ""
    }
  ]
}
```

### `content/reads.json`

```json
{
  "books": [
    { "title": "...", "author": "...", "year": 2024, "note": "..." }
  ],
  "articles": [
    { "title": "...", "url": "...", "note": "..." }
  ]
}
```

---

## GitHub Actions Workflows

### `sync.yml` — the core safety-critical workflow

```yaml
name: Sync product cache

on:
  schedule:
    - cron: '0 3 * * *'        # 3am UTC daily
  repository_dispatch:
    types: [product-updated]    # fired by product repos
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Dry run (log only, no commit)'
        type: boolean
        default: false

permissions:
  contents: write               # only what's needed

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Fetch and validate product data
        id: fetch
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DRY_RUN: ${{ inputs.dry_run || 'false' }}
        run: node scripts/sync-products.js

      - name: Commit cache if changed
        if: steps.fetch.outputs.changed == 'true' && inputs.dry_run != 'true'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add content/cache.json
          git diff --staged --quiet || git commit -m "sync: refresh product cache $(date -u +%Y-%m-%dT%H:%M:%SZ)"
          git push
```

The actual fetch logic lives in `scripts/sync-products.js` (Node.js, no npm deps — uses built-in `fetch`). It:
- Reads `content/products.json`
- Calls GitHub API for each repo (with rate-limit guard)
- Validates the full JSON object
- Compares to existing `content/cache.json`
- Sets `outputs.changed=true` only if different
- Exits 0 on partial failure (logs error, continues)

### Cross-repo notify workflow (goes in EACH product repo)

```yaml
# PennyExpenseManager/.github/workflows/notify-studio.yml
name: Notify studio hub

on:
  release:
    types: [published]
  push:
    branches: [main]
    paths:
      - 'CHANGELOG.md'
      - 'README.md'

jobs:
  dispatch:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger studio sync
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.STUDIO_DISPATCH_TOKEN }}   # fine-grained PAT: studio repo Actions write only
          repository: debasisr66/studio
          event-type: product-updated
          client-payload: '{"source_repo":"${{ github.repository }}"}'
```

**Secret to add in each product repo:** `STUDIO_DISPATCH_TOKEN` — a fine-grained PAT with:
- Repository access: `debasisr66/studio` only
- Permission: Actions → Read and write

---

## Design Direction

**Aesthetic:** Product studio — think Linear, Raycast, Expo. Dark-first, clean grid, mono accent.

- **Color:** Dark background `#0D0D0F`, accent `#6E56CF` (purple — distinct from personal site blue)
- **Type:** Inter (UI) + JetBrains Mono (code/version tags) from Google Fonts
- **Layout:** Full-width hero → product card grid (3-col desktop, 1-col mobile) → updates feed → reads
- **Product cards:** Show name, tagline, category tag, language dot, latest version, download button
- **Mobile:** Single column, bottom-sticky nav (Products / Blog / Reads)
- **Theme:** Dark default with light mode support; all colors as CSS tokens

---

## CMS Admin Layer (same pattern as personal site)

- Gear icon `⚙` fixed bottom-right, semi-transparent
- PAT modal → validate against `GET https://api.github.com/user`
- Edit mode: add/edit/delete blog posts, reading list entries
- Publish: GET sha → PUT `content/blog.json` or `content/reads.json` via GitHub API
- **Safety:** validate JSON before PUT; show error if PUT fails; never leave site in broken state
- PAT stored in `localStorage('studio-admin-pat')`, never committed

---

## Implementation Order

1. Create repo + enable GitHub Pages
2. Create `content/products.json` with Penny entry
3. Create `scripts/sync-products.js` (Node, no deps)
4. Create `.github/workflows/sync.yml`
5. Run sync manually to generate `content/cache.json`
6. Build `index.html` (renders from cache.json + products.json)
7. Add CMS layer for blog + reads
8. Commit + push everything
9. Verify GitHub Pages deploy
10. Add `notify-studio.yml` to PennyExpenseManager repo

**Do NOT start building until you have confirmed:**
- [ ] Repo `debasisr66/studio` exists on GitHub
- [ ] GitHub Pages is enabled on it
- [ ] Local directory `/Users/debasisray/CoworkProjects/studio-site/` created

---

## Owner Context

- **Debasis Ray** — Engineering Executive, Cybage Software
- Professional/CxO audience; minimalistic, high-quality output
- Existing personal site: `https://debasisr66.github.io/debasis-web/` — DO NOT touch this repo
- Local personal site path: `/Users/debasisray/CoworkProjects/claude-code/` — read-only reference only
