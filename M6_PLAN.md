# M6 Plan: Cut over `adrianhall.github.io` to a redirect

> **Purpose of this document.** This is a self-contained handoff for a
> **separate session in the `adrianhall/adrianhall.github.io` repository**.
> That session will not have access to this conversation, the
> `adrianhall/blog` repo, or `MIGRATION_PLAN.md` — everything needed to
> execute M6 safely is captured here. This is Milestone 6 of a larger
> Jekyll→Astro/Cloudflare migration; M1–M5 are complete and are summarized
> below only for context. **Do not re-open or second-guess M1–M5 decisions**
> — this plan is scoped to this repo only.

---

## Context (read this first)

The blog previously served from GitHub Pages at `adrianhall.github.io`
(Jekyll, no custom domain) has been fully rebuilt on Astro and is now live,
verified, and CI-deployed at **`https://blog.adrianhall.uk`** (Cloudflare
Workers Static Assets, in the separate `adrianhall/blog` repo). All content
was preserved path-for-path during that migration.

**This repo's (`adrianhall.github.io`) job now is to become a thin redirect
layer**, sending every request for its old content to the equivalent path
on `blog.adrianhall.uk`. Nothing else about this repo's purpose changes: it
keeps its name, stays a public GitHub repo, stays on GitHub Pages (default
`*.github.io` hosting — **no custom domain is being added here**, and there
never was a `CNAME` file in this repo to begin with).

**Why not a real HTTP 301?** GitHub Pages on a bare `*.github.io` domain has
no server-side configuration layer — there is no way to issue a redirect
status code for arbitrary paths. The only options are a client-side
(JS/meta-refresh) redirect or nothing. Since every old URL in this repo has
an **identical path** on the new site (that was a hard requirement of the
whole migration), a single generic, path-preserving client-side redirect
covers the entire site with no per-URL mapping table — except for **one**
known exception, detailed below.

---

## Critical pre-flight gate — DO THIS FIRST, before touching anything

**This repo may still be an active publishing target.** At the time this
plan was written, `_posts/` in this repo had posts dated all the way up to
*today*, and the new site (`blog.adrianhall.uk`) already had every one of
those posts too — the two were in sync. But time may have passed between
writing this plan and executing it. **Do not proceed with the cutover if
this repo has posts that don't yet exist on the new site** — that would
delete/redirect-away content that hasn't been migrated yet.

Run this check and read its output before doing anything else:

```bash
# From the root of this repo (adrianhall.github.io):
find _posts -name "*.md" | sort | tail -5
git log -1 --format="%ci %s"
```

Then compare against the new site's most recent post — either by asking
the repo owner directly, or (if you have `curl` and the new site is
reachable) by checking:

```bash
curl -s https://blog.adrianhall.uk/feed.xml | grep -o "<title>[^<]*</title>" | head -3
```

**If the dates/titles don't line up with this repo's latest `_posts` entry,
STOP and tell the owner** — new posts need to land in `adrianhall/blog`
(the Astro repo) before this repo's content becomes redundant. Proceeding
with a stale sync would silently drop unpublished-on-the-new-site content
behind a redirect.

Also worth 10 seconds: confirm this repo still has no `CNAME` file
(`ls CNAME` should fail) and no custom domain configured
(`gh api repos/adrianhall/adrianhall.github.io/pages --jq .cname` should be
`null`). If either exists, **stop and ask the owner** — that would mean
something about the hosting setup changed since this plan was written.

---

## The one known URL exception

Every old URL maps to the **identical path** on `blog.adrianhall.uk`,
**except**:

| Old (this repo, currently live) | New (`blog.adrianhall.uk`) |
|---|---|
| `/posts/2025/2025-08-01-oss-ai-editors.html` | `/posts/2025/2025-08-03-oss-ai-editors.html` |

This is not staleness or a mistake to fix — it's a genuine, permanent
mismatch: the post's front matter has `date: 2025-08-01` but its filename
says `2025-08-03`. Jekyll's permalink pattern
(`/posts/:year/:year-:month-:day-:title:output_ext`) resolves `:year`/
`:month`/`:day` from the **front-matter date**, so this repo serves it at
the `08-01` path. The Astro conversion (in the M1/M2 milestones of the
sibling migration) resolved the same post's URL from the **filename**
instead, so the new site serves it at the `08-03` path. Both are internally
consistent with their own site's rules; they just disagree with each other
for this one post. **Do not "fix" either site to match the other** — that's
out of scope for this milestone and would re-open closed migration
decisions. Just make sure the redirect script accounts for it (see
implementation below).

**Re-verify this list is still complete before implementing**, in case new
posts were added since this plan was written (it's a cheap, mechanical
check — do it):

```bash
# From this repo, after confirming the pre-flight gate above is clear:
for f in $(find _posts -name "*.md"); do
  fname_date=$(basename "$f" | grep -oE '^[0-9]{4}-[0-9]{2}-[0-9]{2}')
  fm_date=$(grep -m1 -E '^date:' "$f" | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}')
  if [ -n "$fm_date" ] && [ "$fname_date" != "$fm_date" ]; then
    echo "MISMATCH: $f  filename=$fname_date  frontmatter=$fm_date"
  fi
done
```

If this prints anything other than the one `oss-ai-editors` line already
known, add the corresponding entry to the `PATH_OVERRIDES` map in the
redirect script below before deploying.

---

## Locked decisions

| Topic | Decision |
|---|---|
| Redirect target | `https://blog.adrianhall.uk` (**not** bare `adrianhall.uk` — that's the zone, but the site itself lives on the `blog` subdomain; confirmed live in M5) |
| Redirect mechanism | Client-side: `location.replace()` + `<meta http-equiv="refresh">` fallback + `<link rel="canonical">`, since a real 301 isn't possible here |
| Path handling | Fully generic (`location.pathname` preserved as-is) **except** the one override above |
| Hosting | Stays on GitHub Pages, `*.github.io`, no custom domain, no CNAME — unchanged |
| Jekyll toolchain | Removed entirely (Gemfile, `_layouts/`, `_includes/`, `_sass/`, `_data/`, `_pages/`, `_posts/`, `.jekyll-cache/`) — there is nothing left to build |
| `assets/` (37MB of images) | Removed from the live tree — every image these posts reference was already copied into `adrianhall/blog`'s `public/assets/images/**` at the identical path, so once a post's own URL redirects to the new domain, images resolve from there. Nothing on the new domain depends on this repo serving assets. |
| History/rollback | Nothing is force-deleted. Tag the pre-cutover state before changing anything (see below) so a full rollback is one `git checkout`/revert away. |
| GitHub Pages Actions build | Replaced with a much smaller workflow that just publishes two static files — no Ruby/Jekyll step needed anymore |

---

## Implementation steps

### 1. Safety backup (before changing anything)

```bash
git tag archive/jekyll-final
git push origin archive/jekyll-final
```

This preserves the entire pre-cutover repo (Jekyll source, `_posts`,
`assets/`, everything) at a named, permanent, pushed ref. If anything about
this cutover needs to be undone, `git checkout archive/jekyll-final` gets
the old site's full source back.

### 2. Create the redirect site

Create a new `redirect/` directory (keeps the redirect payload separate
from repo-root files like `README.md`, `LICENSE`, `MIGRATION_PLAN.md` that
should stay) with **two files with identical content**:

- `redirect/index.html` — serves `/` (and would serve any other path that
  happens to exist as a real file, but there won't be any others)
- `redirect/404.html` — GitHub Pages serves this (with an HTTP `404` status,
  but with this file's *content*) for every path that isn't a real file —
  which, once the old Jekyll output is gone, is **every old post, tag,
  category, and feed URL**. This is intentional and is what makes the
  generic redirect work without a giant per-URL routing table.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>This blog has moved · Because Developers are Awesome</title>
<meta name="robots" content="noindex">
<link rel="canonical" id="canonical-link" href="https://blog.adrianhall.uk/">
<meta http-equiv="refresh" content="0; url=https://blog.adrianhall.uk/" id="meta-refresh">
<style>
  body { font: 16px/1.5 -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
         max-width: 40rem; margin: 4rem auto; padding: 0 1.5rem; color: #1a1a1a; }
  a { color: #2563eb; }
</style>
</head>
<body>
  <p>
    This blog has moved to
    <a id="fallback-link" href="https://blog.adrianhall.uk/">blog.adrianhall.uk</a>.
    You should be redirected automatically — if not, follow the link above.
  </p>
  <script>
  (function () {
    var TARGET_ORIGIN = 'https://blog.adrianhall.uk';

    // Known one-off path remaps: URLs whose shape differs between this
    // repo and the new site (front-matter-date vs. filename-date
    // mismatches, etc.). See M6_PLAN.md ("The one known URL exception")
    // for why this exists and how it was verified. This is a safety net
    // for KNOWN exceptions only -- it is not expected to grow, but if a
    // future audit finds another mismatch, add it here.
    var PATH_OVERRIDES = {
      '/posts/2025/2025-08-01-oss-ai-editors.html': '/posts/2025/2025-08-03-oss-ai-editors.html'
    };

    var path = location.pathname;
    var mapped = PATH_OVERRIDES[path] || path;
    var target = TARGET_ORIGIN + mapped + location.search + location.hash;

    var canonical = document.getElementById('canonical-link');
    if (canonical) canonical.href = target;
    var refresh = document.getElementById('meta-refresh');
    if (refresh) refresh.setAttribute('content', '0; url=' + target);
    var fallback = document.getElementById('fallback-link');
    if (fallback) fallback.href = target;

    location.replace(target);
  })();
  </script>
</body>
</html>
```

Copy this exact file to both `redirect/index.html` and `redirect/404.html`
— they must be byte-identical (or at least script-identical); there's no
reason for them to differ.

**Known, accepted limitation**: `/feed.xml` and `/feed.json` will also hit
`404.html` and get this HTML redirect page instead of a valid feed. RSS/JSON
feed readers generally won't execute the JS redirect or follow a
meta-refresh. There's no static-hosting way to serve a "this feed has
moved" response with the correct `content-type` here. This is accepted as
an unavoidable limitation of a static, JS-based redirect — subscribers will
need to resubscribe at `https://blog.adrianhall.uk/feed.xml` (which is
itself linked from every page of the new site via
`<link rel="alternate">`). Do not attempt to "fix" this with more
complexity (e.g. a build step that generates a redirect feed) — it's out of
scope and not worth the added maintenance surface for a retired site.

### 3. Replace the GitHub Actions workflow

Replace `.github/workflows/deploy-blog.yml` with a much smaller workflow —
no Ruby, no Jekyll, no build step, just publish the two static files:

```yaml
name: Deploy redirect site to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  deploy:
    name: deploy
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Pages
        uses: actions/configure-pages@v6

      - name: Upload redirect site
        uses: actions/upload-pages-artifact@v5
        with:
          path: ./redirect

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

(Keeps the same `actions/checkout@v6` / `configure-pages@v6` /
`upload-pages-artifact@v5` / `deploy-pages@v5` versions already in use in
this repo's current workflow — no reason to change those.)

### 4. Remove the retired Jekyll source

Once the backup tag from step 1 is pushed, remove from the working tree
(these are all safely recoverable from `archive/jekyll-final` if ever
needed):

```bash
git rm -r Gemfile Gemfile.lock _config.yml _data _includes _layouts _pages _posts _sass assets .jekyll-cache index.html
```

Leave alone: `README.md`, `LICENSE`, `.editorconfig`, `.gitignore`,
`.devcontainer/`, `.vscode/`, `MIGRATION_PLAN.md` (a copy of the sibling
repo's planning doc — harmless as historical record; optionally add a
one-line note to `README.md` pointing at the new repo/site, but this is
optional polish, not required for the cutover).

Update `.gitignore` if it references now-deleted Jekyll paths (e.g.
`_site/`, `.jekyll-cache/`) — harmless to leave them, but fine to trim if
you're already in there.

### 5. Commit and push

```bash
git add -A
git commit -m "M6: cut over to a redirect-only site

Content has moved to https://blog.adrianhall.uk (see the sibling
adrianhall/blog repo). This repo now serves a generic, path-preserving
client-side redirect for every old URL instead of rebuilding the Jekyll
site. A true HTTP 301 isn't possible on a bare *.github.io host, so
this is index.html + 404.html with a location.replace() + meta-refresh
fallback. One path override is baked in for a known URL-shape mismatch
(see M6_PLAN.md). Pre-cutover source is preserved at the
archive/jekyll-final tag."
git push origin main
```

Pushing triggers the new Actions workflow automatically.

---

## Verification (after the Actions run completes)

### Automated spot-check

Run this against a representative sample — every URL below was confirmed
live on the old site via its own `sitemap.xml` at the time this plan was
written (re-fetch `https://adrianhall.github.io/sitemap.xml` yourself if
you want a fresher list before relying on this one):

```bash
#!/usr/bin/env bash
set -euo pipefail

urls=(
  "/"
  "/posts/"
  "/posts/page/2/"
  "/posts/2017/2017-05-08-building-a-service-in-the-cloud.html"
  "/posts/2018/2018-06-01-how-developers-can-auth-with-aws-appsync.html"
  "/posts/2019/2019-02-12-documenting-opensource-projects.html"
  "/posts/2020/2020-05-03-swiftui-masks.html"
  "/posts/2021/2021-11-12-zumo-irepository.html"
  "/posts/2022/2022-11-28-bicep-type-checking.html"
  "/posts/2023/2023-07-12-cleanup-azure-resources.html"
  "/posts/2024/2024-09-13-aspnet-identity-part2.html"
  "/posts/2025/2025-08-01-ai-editors.html"
  "/posts/2025/2025-08-01-oss-ai-editors.html"   # <-- the known override; must land on 2025-08-03
  "/posts/2026/2026-07-01-cf-caching.html"
  "/privacy.html"
  "/tags/"
  "/tags/cloudflare/"
  "/tags/page/2/"
  "/categories/cloud-development/"
  "/feed.xml"
  "/feed.json"
)

for path in "${urls[@]}"; do
  final=$(curl -s -o /dev/null -w "%{url_effective}" -L "https://adrianhall.github.io${path}")
  code=$(curl -s -o /dev/null -w "%{http_code}" -L "https://adrianhall.github.io${path}")
  echo "$code  $path  ->  $final"
done
```

Every row should show `200` and a `final` URL under `https://blog.adrianhall.uk/`
— **except** the `oss-ai-editors` row, which should land on
`.../2025-08-03-oss-ai-editors.html` (not `08-01`) if the override is
working.

### Manual/browser checks

- Open `https://adrianhall.github.io/` in a real browser — confirm it lands
  on the new home page (not a blank/broken page, not stuck on the redirect
  notice).
- Open a couple of the post URLs above directly — confirm you land on the
  actual matching post (not just *a* page on the new site).
- Check the browser's dev tools console on a couple of pages for JS errors
  from the redirect script itself (there shouldn't be any — it's a handful
  of lines with no dependencies).
- Confirm search engines/crawlers aren't going to index the redirect page
  itself as real content: view-source on `https://adrianhall.github.io/`
  and confirm `<meta name="robots" content="noindex">` is present.

### Full-list check (optional, more thorough)

If you have access to clone `adrianhall/blog` too, its
`scripts/legacy-urls.json` is the authoritative frozen list of every
migrated URL (305 entries, generated in M5). You can adapt the loop above
to iterate that whole list instead of the smaller sample here for full
coverage. Not required for sign-off, but worth doing if you want maximum
confidence before calling this done.

---

## What this milestone deliberately does NOT do

- **No DNS changes.** `adrianhall.github.io` stays exactly as it is
  (default GitHub Pages hosting, no custom domain, no CNAME record). Only
  the *content* served changes.
- **No changes to `blog.adrianhall.uk` or the `adrianhall/blog` repo.**
  This milestone is scoped entirely to this repo.
- **No per-post redirect mapping**, beyond the single documented exception.
  If you find yourself wanting to add a second, third, fourth override,
  stop and re-run the mismatch-detection script from the pre-flight section
  — something is probably wrong with the assumption that paths are
  identical, and that's worth understanding rather than papering over with
  more overrides.
- **No attempt to preserve feed URLs' content-type.** Documented above as
  an accepted limitation.
- **No repo rename, transfer, or deletion.** It keeps existing exactly
  where it is, under the same name, just serving different content.

---

## Milestone checklist

- [ ] Pre-flight: confirmed this repo's latest post matches the new site's
  latest post (no un-migrated content)
- [ ] Pre-flight: confirmed no `CNAME` / custom domain exists on this repo
- [ ] Pre-flight: re-ran the front-matter/filename date mismatch scan;
  confirmed the override list above is still complete
- [ ] Safety tag `archive/jekyll-final` created and pushed
- [ ] `redirect/index.html` + `redirect/404.html` created (identical
  content, override map included)
- [ ] `.github/workflows/deploy-blog.yml` replaced with the static-only
  workflow
- [ ] Old Jekyll source removed from the working tree
- [ ] Committed, pushed, Actions run completed successfully
- [ ] Automated spot-check script run — all rows `200`, correct final URLs,
  override row lands on `08-03`
- [ ] Manual browser check on home page + a couple of posts — no console
  errors, `noindex` present
- [ ] (Optional) full-list check against `adrianhall/blog`'s
  `scripts/legacy-urls.json`

## M6 implementation notes

*(Fill in during/after execution — what actually happened, any surprises,
anything this plan got wrong or missed. Match the style of the sibling
`MIGRATION_PLAN.md`'s M1–M5 implementation-notes sections if you want
consistency across both repos' histories.)*
